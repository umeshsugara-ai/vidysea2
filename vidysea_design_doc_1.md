# 📄 Technical Design Document: VidySea Backend Engine (v17.3 Gold Master)

## 1. Executive Summary

This is the **Gold Master Construction-Ready** architecture for the VidySea Backend.

It addresses all feedback regarding **Multilingual Accuracy**, **Memory Safety (OOM)**, and **Operational Resilience**. It is tailored specifically for a **Single-Node AWS EC2 (16GB RAM / 1TB Disk)** deployment.

**Core Pillars:**
1.  **AI-Native Router (v2):** Uses `multilingual-e5-small` with strict prefixing ("query: " vs "passage: ") for high-accuracy intent detection.
2.  **Safety First:** Enforced L2 Normalization and a **16GB Disk Swap** file ensure the server never crashes during index reloads.
3.  **Fiscal Discipline:** Negative Caching and a Global Budget Cap (500/day) prevent API bill runaway.
4.  **Resilience:** Double-buffered indexing and streaming S3 backups ensure high availability and data safety.

---

## 2. Technology Stack & Infrastructure

| Component | Choice | Role | Resource Strategy |
| :--- | :--- | :--- | :--- |
| **Backend** | **FastAPI (Python)** | Logic Orchestrator. | Async Workers. |
| **Database** | **MongoDB (Local)** | Primary Store. | **Streamed S3 Backups** (Hourly). |
| **Vector Engine** | **FAISS (CPU)** | RAM Index (`IndexIDMap`). | **Double Buffered** + **16GB Swap**. |
| **Embedding** | **multilingual-e5-small** | Semantic Vectorization. | **384 Dim**. Prefixing Required. |
| **Primary LLM** | **Gemini 3 Flash** | Logic Generation. | 1M Context. Low Cost. |
| **Backup LLM** | **Claude Haiku 4.5** | Fallback Logic. | Agentic Capability. |
| **External Search**| **Serper.dev** | Web Backup. | **Negatively Cached** + Capped. |
| **OS Storage** | **AWS EBS (1TB)** | Persistence. | **gp3 (3000 IOPS)** + **16GB Swap**. |

---

## 3. Architecture Data Flow

```mermaid
graph TD
    User[User Query] --> Context[Resolve Currency/Loc]
    Context --> API[FastAPI]
    
    API --> NegCache{Check Negative Cache}
    NegCache -- "Found" --> EmptyResp[Return 'No Results' (Cost: 0)]
    NegCache -- "Miss" --> Layer1{Layer 1: Vector Router}
    
    Layer1 -- "Score > 0.96 (Exact)" --> DB_Fetch[Direct DB Fetch]
    Layer1 -- "Score > 0.85" --> Residual{Residual Token Check}
    
    Residual -- "Token in Safe List" --> DB_Fetch
    Residual -- "Unknown/Risky" --> Layer3
    
    Layer1 -- No Match --> Layer2{Layer 2: Semantic Cache}
    
    Layer2 -- Hit (v2) --> Exec
    Layer2 -- Miss --> Layer3[Layer 3: AI Query Engine]
    
    Layer3 --> Gemini[Gemini 3 Flash]
    Gemini -- "Failure" --> Haiku[Claude Haiku 4.5]
    Gemini --> Logic{Prompt Logic}
    
    Logic -- "Suggestion" --> Admin[Async Admin Webhook]
    Logic -- "Pipeline" --> Builder[Python Pipeline Builder]
    
    Builder --> SortFix[Inject Null Filter]
    Builder --> Pagination[Inject Limit/Skip]
    
    Pagination --> Exec[Execute on Live DB]
    
    Exec --> Gatekeeper --> Response
    
    subgraph "Zero Result / Healing (Sync)"
        Response -- "Empty" --> ScopeCheck{Scope Guard}
        ScopeCheck -- "Relevant" --> WebSearch[Serper.dev]
        WebSearch -- "Empty" --> WriteNegCache[Write Negative Cache]
        WebSearch -- "Results" --> Writer[Write-Back to DB]
        Writer --> Response
    end
```

---

## 4. Layer Detailed Design & Logic

### 4.1 Layer 1: The Vector Router (e5-small Specifics)

**Objective:** Only redirect to University Page if we are 100% sure the user *didn't* ask for a program.

**The "e5" Trap Fixes (Critical):**
*   **Query Prefix:** All user queries **MUST** be prefixed with `"query: "` before embedding.
    *   *Example:* User types "Harvard" -> System embeds `"query: Harvard"`.
*   **Truncation Safety:** Truncate input to **1000 characters** (~512 tokens) before embedding to prevent model crashes or garbage output on massive inputs.
*   **Index Prefix:** All DB entries **MUST** be prefixed with `"passage: "` (or standard doc prefix) before indexing.
*   **Normalization:** Output vectors must be **L2 Normalized** before Inner Product search.

**Logic Flow:**
1.  **Sanitize:** `safe_text = user_text[:1000]`
2.  **Embed:** `v = model.encode("query: " + safe_text, normalize_embeddings=True)`
3.  **Search:** `D, I = index.search(v, k=50)`
4.  **Override Check:**
    *   **If `D[0] > 0.96`**: The user entered an Exact Name or simple word swap.
    *   **Action:** **Bypass Residual Check.** Route immediately to University Detail.
5.  **Standard Check:**
    *   **If `D[0] > 0.85`**: Run **Residual Check**.
    *   **Residual:** Remove matched entity name from query.
    *   **Safe List:** `{"uni", "university", "college", "campus", ...}`.
    *   **Risky List:** `{"program", "course", "fees", "ranking", ...}`.
    *   **Action:** If residual contains *any* Risky word -> **Layer 3**. If only Safe words -> **DB Fetch**.

---

### 4.2 Layer 2: The Semantic Cache (v2)

**Objective:** Reduce database hits for semantically identical queries.

**Versioning & Hash Logic:**
*   **Config:** `SCHEMA_VERSION = "v2"` (Incremented for e5 switch).
*   **Cache Key:** `SHA256("query: " + user_text + json(filters) + currency + SCHEMA_VERSION)`.
*   **Storage:** MongoDB `query_cache` collection.
*   **Lookup:**
    1.  Compute Query Embedding.
    2.  Search `SemanticCacheIndex` (FAISS).
    3.  **Hit Condition:** Similarity > 0.98 **AND** `filters` match exactly.

---

### 4.3 Layer 3: The AI-Native Query Engine

**Objective:** Handle Generic Queries Efficiently using **Gemini 3 Flash**.

**Logic 1: The "Null Sort" Fix:**
*   **Problem:** Sorting by "Fees Ascending" puts `null` (unknown fees) at the top.
*   **Solution:**
    ```python
    if "$sort" in pipeline:
        field = list(pipeline["$sort"].keys())[0]
        # Prepend match to exclude nulls
        pipeline.insert(0, { "$match": { field: { "$ne": null } } })
    ```

**Logic 2: Pagination Guard:**
*   **Mandatory:** Every pipeline *must* end with Limit.
    ```python
    if not any("$limit" in s for s in pipeline):
        pipeline.append({ "$skip": (page-1) * 20 })
        pipeline.append({ "$limit": 20 })
    ```

**Logic 3: Active Admin Notification:**
*   **Scenario:** Gemini detects a new synonym (e.g., "Comp Sci" -> "Computer Science").
*   **Action:**
    1.  Write suggestion to `admin_suggestions`.
    2.  **Fire Async Webhook:** Send payload to Slack/Email.
    3.  **Non-Blocking:** User gets results immediately; Admin gets alert in background.

---

## 5. The Gatekeeper: Safety & Cost

### 5.1 Negative Caching
*   **Trigger:** Serper.dev returns **200 OK** with **0 results**.
*   **Poisoning Guard:** Do **NOT** cache if Serper returns 5xx Error or Timeout.
*   **Action:** Write `SHA256(query)` to `negative_cache` (TTL: 30 Days).
*   **Check:** Layer 1 checks this cache *before* any embedding.

### 5.2 Global Budget Cap
*   **Mechanism:** Redis/Mongo Counter `serper_usage_YYYY_MM_DD`.
*   **Limit:** **500 calls/day**.
*   **Logic:** `if counter > 500: return EmptyResponse`.

---

## 6. Infrastructure Hygiene (Backup & Sync)

### 6.1 The "RAM Double-Dip" Fix (Swap File & EBS)
**Risk:** Loading a 2nd Index (4GB) + Mongo (4GB) can saturate RAM. Disk I/O saturation (IOPS) can freeze the CPU during Swap/Backups.
**Solution 1:** Enable **16GB Swap File** on the 1TB Disk.
**Solution 2 (Critical):** Provision AWS EBS as **gp3** with **3000+ IOPS**. Do NOT use gp2 or HDD.
*   **Thrashing Guard:** Set CloudWatch Alarm. If **Memory > 85%** consistently, upgrade instance to 32GB immediately.

### 6.2 Double Buffered Indexing (With Real-Time Deletion)
**Logic:**
1.  **Load:** Worker loads `new_index` from disk into RAM.
2.  **Swap:** `app.state.faiss_index = new_index` (Atomic Python operation).
3.  **GC:** `del old_index`.

**Real-Time Deletion (Soft Delete Gap Fix):**
*   **Scenario:** Admin sets university to `inactive`.
*   **Action:** The update function MUST also call `index.remove_ids(np.array([id]))` immediately to prevent searching "Ghost Universities".

### 6.3 Automated Streamed Backups
**Objective:** Protect Data without filling disk space.
**Script:** `cron/backup_mongo.sh`.
**Command:**
```bash
mongodump --uri="mongodb://localhost:27017" --archive | gzip | aws s3 cp - s3://vidysea-backups/$(date +%F_%H).gz
```

---

## 7. Data Miss & Self-Healing (Synchronous)

**Scenario:** User searches "New University X". DB Miss.
**Flow:**
1.  **Detection:** Layer 1 & 3 return 0 matches.
2.  **Frontend UX:** Display specific loading state: *"Searching the live web for you..."*.
3.  **Grounding:** Call **Serper.dev** (Budget Permitting).
4.  **Validation:**
    *   Extract Name, Website, Location.
    *   **Rule:** Must have valid domain and non-empty location.
5.  **Write-Back (Locking & Versioning):**
    *   Acquire `dist_lock_scrape_{normalized_name}` with **TTL = 60s**.
    *   **Optimistic Locking:** Use a `version` field. `update_one({_id: id, version: old_v}, {$set: ..., $inc: {version: 1}})`. If no match, retry or alert (prevents Admin overwrite).
    *   **Immediate Update:** Call `index.add_with_ids()` to make it searchable instantly.
6.  **Response:** Return the new object to the user.

---

## 8. Final Developer Checklist

1.  **IOPS Check:** Verify AWS EBS volume type is `gp3` (not `gp2`) in EC2 Console.
2.  **Optimistic Lock:** Test Admin saving an old version while Scraper updates it (should fail/warn).
3.  **Soft Delete:** Mark a Uni inactive and verify it disappears from Search immediately (without reload).
4.  **e5 Prefixes:** Verify `query:` / `passage:` logic.
5.  **Swap File:** Run `free -h` and confirm **16Gi** Swap is active.
6.  **Neg Cache:** Simulate Serper 500 Error -> Ensure NOTHING is written to cache.
7.  **Budget Cap:** Manually set counter to 499 and verify 501st request is blocked.

This architecture is **Gold Master v17.3 Approved**. Proceed to build Phase 1 immediately.
