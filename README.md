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
| **Database** | **MongoDB (Local)** | Primary Store + Text Search. | **Streamed S3 Backups** (Hourly). |
| **Vector Engine** | **FAISS (Disk-Mapped)** | Vector Search (`IO_FLAG_MMAP`). | **Zero RAM Overhead** + **Hybrid Search**. |
| **Embedding** | **multilingual-e5-small** | Semantic Vectorization. | **384 Dim**. Prefixing Required. |
| **Primary LLM** | **Gemini 3 Flash** | Logic Generation. | 1M Context. Low Cost. |
| **Backup LLM** | **Claude Haiku 4.5** | Fallback Logic. | Agentic Capability. |
| **External Search**| **Serper.dev** | Web Backup. | **Negatively Cached** + Capped. |
| **OS Storage** | **AWS EBS (1TB)** | Persistence. | **gp3 (3000 IOPS)** + **16GB Swap**. |


---


## 3. Architecture Data Flow

### ASCII Flowchart (For Non-Mermaid Viewers)

```
┌─────────────────────────────────────────────────────────────────────────────────────────────┐
│                                    VIDYSEA QUERY FLOW                                       │
└─────────────────────────────────────────────────────────────────────────────────────────────┘

                                    ┌──────────────┐
                                    │  User Query  │
                                    └──────┬───────┘
                                           │
                                           ▼
                              ┌────────────────────────┐
                              │ Resolve Currency/Loc   │
                              └───────────┬────────────┘
                                          │
                                          ▼
                                   ┌──────────────┐
                                   │   FastAPI    │
                                   └──────┬───────┘
                                          │
                                          ▼
                          ┌───────────────────────────────┐
                          │    Check Negative Cache       │
                          └───────────────┬───────────────┘
                                          │
                     ┌────────────────────┴────────────────────┐
                     │ Found                                   │ Miss
                     ▼                                         ▼
          ┌──────────────────┐               ┌─────────────────────────────────┐
          │ Return 'No       │               │   LAYER 1: Vector Router        │
          │ Results'         │               └────────────────┬────────────────┘
          └──────────────────┘                                │
                                    ┌─────────────────────────┼─────────────────────────┐
                                    │                         │                         │
                            Score > 0.96              Score > 0.85                 No Match
                              (Exact)                         │                         │
                                    │                         ▼                         │
                                    │        ┌────────────────────────────┐             │
                                    │        │  Residual Token Check      │             │
                                    │        └───────────┬────────────────┘             │
                                    │           ┌────────┴────────┐                     │
                                    │           │                 │                     │
                                    │    Token in           Unknown/Risky               │
                                    │    Safe List                │                     │
                                    │           │                 │                     │
                                    ▼           ▼                 │                     ▼
                          ┌──────────────────────┐                │     ┌───────────────────────────┐
                          │   Direct DB Fetch    │                │     │  LAYER 2: Plan Cache      │
                          └──────────────────────┘                │     └─────────────┬─────────────┘
                                    │                             │            ┌──────┴──────┐
                                    │                             │           Hit           Miss
                                    │                             │            │             │
                                    │                             │            │             ▼
                                    │                             │            │  ┌─────────────────────────┐
                                    │                             └────────────┼──► LAYER 3: AI Query      │
                                    │                                          │  │ Engine                  │
                                    │                                          │  └───────────┬─────────────┘
                                    │                                          │              │
                                    │                                          │              ▼
                                    │                                          │    ┌──────────────────┐
                                    │                                          │    │ Gemini 3 Flash   │──Failure──┐
                                    │                                          │    └────────┬─────────┘           │
                                    │                                          │             │                     ▼
                                    │                                          │             │         ┌───────────────────┐
                                    │                                          │             │         │ Claude Haiku 4.5  │
                                    │                                          │             │         └─────────┬─────────┘
                                    │                                          │             ▼                   │
                                    │                                          │    ┌──────────────────┐         │
                                    │                                          │    │  Prompt Logic    │◄────────┘
                                    │                                          │    └────────┬─────────┘
                                    │                                          │    ┌────────┴────────┐
                                    │                                          │ Suggestion        Pipeline
                                    │                                          │    │                 │
                                    │                                          │    ▼                 ▼
                                    │                                          │  ┌────────────┐  ┌─────────────────────┐
                                    │                                          │  │ Async      │  │ Python Pipeline     │
                                    │                                          │  │ Admin      │  │ Builder             │
                                    │                                          │  │ Webhook    │  └──────────┬──────────┘
                                    │                                          │  └────────────┘             │
                                    │                                          │                             ▼
                                    │                                          │              ┌──────────────────────────┐
                                    │                                          │              │  Inject Null Filter      │
                                    │                                          │              └────────────┬─────────────┘
                                    │                                          │                           │
                                    │                                          │                           ▼
                                    │                                          │              ┌──────────────────────────┐
                                    │                                          │              │  Inject Limit/Skip       │
                                    │                                          │              └────────────┬─────────────┘
                                    │                                          │                           │
                                    ▼                                          ▼                           ▼
                          ┌────────────────────────────────────────────────────────────────────────────────────┐
                          │                         Execute on Live DB                                         │
                          └────────────────────────────────────────┬───────────────────────────────────────────┘
                                                                   │
                                                                   ▼
                                                        ┌──────────────────┐
                                                        │   Gatekeeper     │
                                                        └────────┬─────────┘
                                                                 │
                                                                 ▼
                                                        ┌──────────────────┐
                                                        │    Response      │
                                                        └────────┬─────────┘
                                                                 │
┌────────────────────────────────────────────────────────────────┼────────────────────────────────────────────┐
│  ZERO RESULT / HEALING (Sync)                            Empty │                                            │
│                                                                ▼                                            │
│                                                     ┌──────────────────┐                                    │
│                                                     │   Scope Guard    │                                    │
│                                                     └────────┬─────────┘                                    │
│                                                              │ Relevant                                     │
│                                                              ▼                                              │
│                                                     ┌──────────────────┐                                    │
│                                                     │   Serper.dev     │                                    │
│                                                     │   (Web Search)   │                                    │
│                                                     └────────┬─────────┘                                    │
│                                              ┌───────────────┴───────────────┐                              │
│                                           Empty                           Results                           │
│                                              │                               │                              │
│                                              ▼                               ▼                              │
│                                   ┌─────────────────────┐        ┌─────────────────────┐                    │
│                                   │ Write Negative      │        │ Write-Back to DB    │                    │
│                                   │ Cache               │        └──────────┬──────────┘                    │
│                                   └─────────────────────┘                   │                               │
│                                                                             ▼                               │
│                                                                   ┌──────────────────┐                      │
│                                                                   │    Response      │                      │
│                                                                   └──────────────────┘                      │
└─────────────────────────────────────────────────────────────────────────────────────────────────────────────┘
```

**3-Layer Architecture Summary:**

| Layer | Name | Purpose |
|-------|------|---------|
| **Layer 1** | Vector Router | Semantic similarity matching (Score thresholds: >0.96 = Exact, >0.85 = Check residual tokens) |
| **Layer 2** | Plan Cache | Cached execution plans for previously seen query patterns |
| **Layer 3** | AI Query Engine | Gemini 3 Flash (primary) → Claude Haiku 4.5 (fallback) for pipeline generation |

**Key Decision Points:**
1. **Negative Cache** - Quickly reject known "no result" queries
2. **Vector Router Score** - Determines if query is exact match or needs AI processing
3. **Residual Token Check** - Safety validation for partially matched queries
4. **Zero Result Healing** - Auto-recovery via web search when DB returns empty

---


## 4. Layer Detailed Design & Logic


### 4.1 Layer 1: The Vector Router (Hybrid Disk-Mapped Architecture)


**Objective:** Eliminate RAM bottlenecks and support hybrid (Keyword + Vector) search.


**The "Sidecar" Index Strategy (Write-Safety):**
*   **Problem:** Writing to a Memory-Mapped file (MMAP) while reading causes Segfaults.
*   **Solution:**
    1.  **Main Index (Read-Only):** The massive 10GB index on Disk (MMAP).
    2.  **Sidecar Index (Read-Write):** A small In-Memory `IndexFlatIP` for items added *today*.
*   **Search Flow:** `results = merge(main_index.search(q), sidecar_index.search(q))`
*   **Merge Policy:** Nightly cron job merges Sidecar into Main and resets Sidecar.
*   **Hybrid Search (The "Magic Glue"):**
    *   **Vector Branch:** Returns "Meaning" match (e.g., "Rice University" -> 0.92 similarity).
    *   **Keyword Branch:** Returns "Exact Text" match (e.g., "Rice Research Institute" -> 15.5 score).
    *   **The Fusion (RRF):** We combine these incomparable scores using **Reciprocal Rank Fusion**.
    
    ### RRF (Reciprocal Rank Fusion) Explained
    
    **The Problem:**
    You have two search engines running in parallel:
    1.  **Vector Search (Semantic):** Returns results with a score of 0.0 to 1.0 (e.g., "Rice University" = 0.92).
    2.  **Keyword Search (Mongo Text):** Returns results with a score of 0 to Infinity (e.g., "Rice University" = 15.5 because the word "Rice" appears 3 times).
    
    **The Challenge:**
    How do you combine them?
    *   If you just add them (`0.92 + 15.5`), the Keyword Score (15.5) totally dominates. The Vector score becomes irrelevant.
    *   You cannot normalize them easily because you don't know the maximum Keyword score (it could be 5, or 500).
    
    **The Solution:**
    RRF ignores the *raw score* and looks at the **Rank** (1st, 2nd, 3rd place).
    
    **The Formula:**
    `Score = 1 / (k + Rank)`
    
    **Example:**
    *   **Vector Search Results:**
        1.  University A (Rank 1) -> Score = `1 / (60 + 1)` = 0.0163
        2.  University B (Rank 2) -> Score = `1 / (60 + 2)` = 0.0161
    *   **Keyword Search Results:**
        1.  University B (Rank 1) -> Score = `1 / (60 + 1)` = 0.0163
        2.  University C (Rank 2) -> Score = `1 / (60 + 2)` = 0.0161
    
    **Final Combined Score:**
    *   **University B:** 0.0161 (Vector) + 0.0163 (Keyword) = **0.0324 (Winner!)**
    *   **University A:** 0.0163 (Vector) + 0 (Not found in Keyword) = 0.0163
    
    **Why k = 60?**
    The constant `k` controls how much you trust the "Top 1" result versus the "Top 10".
    *   `k = 1`: The #1 result gets a HUGE score. If Vector is wrong about #1, the whole result list sucks.
    *   `k = 60`: The curve is flatter. Being #1 is good, but being #5 in *both* lists is better. This is the industry standard (used by Elasticsearch/Solr) for robust search.
    
    **Conclusion:**
    RRF is the "Magic Glue" that lets you combine Vector (Meaning) and Text (Keywords) fairly, without one dominating the other.


**The "ETL Mandate" (Data Hygiene):**
*   **Rule:** **NEVER** store numeric data as Strings.
*   **Why:** String sorting is alphabetical ("10%" < "2%"). This breaks range queries (`$lt`, `$gt`).
*   **Action:** Ingestion scripts MUST convert:
    *   `"5.5%"` -> `0.055` (Number)
    *   `"15 Lakhs"` -> `1500000` (Number)
    *   `"$20,000"` -> `20000` (Number)


**The "e5" Trap Fixes (Critical):**
*   **Query Prefix:** All user queries **MUST** be prefixed with `"query: "` before embedding.
    *   *Example:* User types "Harvard" -> System embeds `"query: Harvard"`.
*   **Truncation Safety:** Truncate input to **1000 characters** (~512 tokens) before embedding to prevent model crashes or garbage output on massive inputs.
*   **Index Prefix:** All DB entries **MUST** be prefixed with `"passage: "` (or standard doc prefix) before indexing.
*   **Normalization:** Output vectors must be **L2 Normalized** before Inner Product search.


**Logic Flow:**
1.  **Sanitize:** `safe_text = user_text[:1000]`
2.  **Embed:** `v = model.encode("query: " + safe_text, normalize_embeddings=True)`
3.  **Search:**
    *   **Standard:** `k=50`.
    *   **Dynamic Oversampling (Filter Logic):** If query contains Location/Category keywords (e.g., "Germany", "Master"), boost `k` to **500**. This prevents the "Filtered Vector Blindspot" where relevant items are cut off before filtering.
4.  **Override Check (The "Bypass" Guard):**
    *   **If `D[0] > 0.96`**: The user entered an Exact Name.
    *   **Residual Check (Safety):** Remove the matched entity name from the query.
        *   If `residual_tokens` contain *any* meaningful filter words ("Canada", "Online", "Cheap") -> **ABORT BYPASS**. Route to Layer 3.
        *   Only Bypass if residual is empty or stop-words.
5.  **Standard Check:**
    *   **If `D[0] > 0.85`**: Run **Residual Check**.
    *   **Residual:** Remove matched entity name from query.
    *   **Safe List:** `{"uni", "university", "college", "campus", ...}`.
    *   **Risky List:** `{"program", "course", "fees", "ranking", ...}`.
    *   **Action:** If residual contains *any* Risky word -> **Layer 3**. If only Safe words -> **DB Fetch**.


---


### 4.2 Layer 2: The Plan Cache (Exact Match)


**Objective:** Eliminate AI latency for repeated exact queries (Pagination Optimized).


**Why Hashing? (Developer Note):**
*   **Speed:** Searching for a 64-char SHA256 string is O(1) and extremely fast in MongoDB B-Tree indexes.
*   **Normalization:** It handles whitespace/case standardization once, creating a unique "fingerprint" for the query plan.
*   **Storage:** Storing the "Plan" (JSON) is lighter than storing "Results".


**Versioning & Hash Logic:**
*   **Config:** `SCHEMA_VERSION = "v3"`
*   **Cache Key Strategy:**
    *   **Goal:** User asking for "Page 2" should hit the *same* cache as "Page 1" for the AI Logic.
    *   **The Key:** `SHA256("query:{safe_text}|filt:{filters}|curr:{currency}|date:{YYYY-MM-DD}")`
    *   **Excluded:** `page`, `limit` (These are applied *after* cache retrieval).
*   **Storage:** MongoDB `query_cache` collection.
*   **Content:** We store the **AI Query Helper JSON** (The "Plan"), NOT the results.
    *   *Hit Flow:* Fetch Plan -> Update `$skip/$limit` in Python -> Run Query.


---


### 4.3 Layer 3: The AI-Native Query Engine


**Objective:** Handle Generic Queries Efficiently using **Gemini 3 Flash** with Strict Schema Validation.


**Logic 1: Strict AI "Query Helper" JSON Schema:**
*   **Goal:** Prevent hallucinated fields. The AI does *not* write raw Mongo code. It outputs a strictly typed JSON plan.
*   **Spelling Correction:** AI must normalize typos (e.g., "Hahvard" -> "Harvard") in the `semantic_rewrite` field *before* pipeline generation.
*   **Strict Schema:**
    ```json
    {
      "intent": "program_search", // Enum: [university_search, program_search, test_search]
      "filters": {
        "program_fees_domestic_usd": { "$lt": 20000 },
        "country": "USA"
      },
      "sort": { "field": "qs_world", "order": -1 }, // Mapped to actual DB fields
      "currency_convert": true,
      "semantic_rewrite": "computer science" // Normalized term for $text search
    }
    ```
*   **Validator:** A Python layer validates keys against `VALID_DB_FIELDS` set. If invalid key -> Log Warning & Drop Filter.


**Logic 2: The "Null Sort" Fix:**
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


**Logic 4: AI Pipeline Generator (The Airport Security Model):**


**System Constraints (The "Truth Source"):**
1.  **Not a Chatbot:** We are a Database Search Engine, not a general knowledge bot. If the data isn't in the DB, we return "No Results" (or trigger Web Search), we do NOT hallucinate answers.
2.  **English-Only DB:** All data is stored in English. Multi-lingual queries MUST be translated to English terms by the AI before pipeline generation.
3.  **USD Base Currency:** All fees are queried in USD (using `fees_usd_approx`). The frontend handles display conversion.
4.  **Standard Names Only:** DB stores "Master of Business Administration", not "MBA". AI MUST expand all user acronyms to standard forms.
5.  **Concept Mapping (Seasons):** AI Prompt MUST map abstract timeframes to Regex (Default: Northern Hemisphere):
    *   "Fall" -> `August|September|October`
    *   "Spring" -> `January|February`
    *   "Summer" -> `May|June|July`
    *   *Note:* If User Location = "Australia/NZ", AI MUST invert seasons.


**The "Variety Pack" Examples (Covering Simple, Multilingual, Complex, and Edge Cases):**


### A. Simple & Direct (1-3)


**1. "MBA in USA"**
*   **JSON Pipeline:**
    ```json
    [
      { "$match": { "program_type": "Masters", "name": { "$regex": "Master of Business Administration", "$options": "i" } } },
      { "$lookup": { "from": "universities", "localField": "university_id", "foreignField": "_id", "as": "uni" } },
      { "$unwind": "$uni" },
      { "$match": { "uni.basic_info.location.country": "USA" } }
    ]
    ```


**2. "Stanford University address"**
*   **JSON Pipeline:**
    ```json
    [
      { "$match": { "basic_info.name": { "$regex": "Stanford University", "$options": "i" } } },
      { "$project": { "name": "$basic_info.name", "address": "$basic_info.location.address" } }
    ]
    ```


**3. "IELTS test fees"**
*   **JSON Pipeline:**
    ```json
    [
      { "$match": { "testname": { "$regex": "IELTS", "$options": "i" } } },
      { "$project": { "fee": "$registration_fee", "currency": "$registration_fee_currency" } }
    ]
    ```


### B. Medium Complexity (English) (4-6)


**4. "Universities with < 20k fees in Canada"**
*   **JSON Pipeline:**
    ```json
    [
      { "$match": { "program_fees_usd_approx": { "$lt": 20000 } } },
      { "$lookup": { "from": "universities", "localField": "university_id", "foreignField": "_id", "as": "uni" } },
      { "$unwind": "$uni" },
      { "$match": { "uni.basic_info.location.country": "Canada" } }
    ]
    ```


**5. "Spring 2026 Masters in Data Science"**
*   **AI Insight:** "Spring 2026" -> Regex on `start_date` for Jan/Feb/March 2026.
*   **JSON Pipeline:**
    ```json
    [
      { "$match": {
          "program_type": "Masters",
          "name": { "$regex": "Data Science", "$options": "i" },
          "start_date": { "$regex": "2026-0(1|2|3)|Jan.*2026|Feb.*2026", "$options": "i" }
      }}
    ]
    ```


**6. "Colleges in UK accepting IELTS 6.5"**
*   **JSON Pipeline:**
    ```json
    [
      { "$match": { "test_scores": { "$elemMatch": { "testname": "IELTS", "cutoff_range": { "$lte": "6.5" } } } } },
      { "$lookup": { "from": "universities", "localField": "university_id", "foreignField": "_id", "as": "uni" } },
      { "$unwind": "$uni" },
      { "$match": { "uni.basic_info.location.country": "United Kingdom" } } // "UK" -> "United Kingdom"
    ]
    ```


### C. Multilingual Examples (7-10)


**7. "Wirtschaftsinformatik in Deutschland" (German)**
*   **Translation:** "Business Informatics" in "Germany".
*   **JSON Pipeline:**
    ```json
    [
      { "$match": { "name": { "$regex": "Business Informatics|Wirtschaftsinformatik", "$options": "i" } } },
      { "$lookup": { "from": "universities", "localField": "university_id", "foreignField": "_id", "as": "uni" } },
      { "$unwind": "$uni" },
      { "$match": { "uni.basic_info.location.country": "Germany" } }
    ]
    ```


**8. "Universidades baratas en Canadá" (Spanish)**
*   **Translation:** "Cheap Universities" -> Sort Fee Ascending.
*   **JSON Pipeline:**
    ```json
    [
      { "$lookup": { "from": "programmes", "localField": "_id", "foreignField": "university_id", "as": "progs" } },
      { "$match": { "basic_info.location.country": "Canada" } },
      { "$unwind": "$progs" },
      { "$sort": { "progs.program_fees_usd_approx": 1 } },
      { "$limit": 20 }
    ]
    ```


**9. "Saste colleges USA me Computer Science ke liye" (Hindi)**
*   **Translation:** "Cheap colleges in USA for Computer Science".
*   **JSON Pipeline:**
    ```json
    [
      { "$match": { 
          "name": { "$regex": "Computer Science", "$options": "i" },
          "program_fees_usd_approx": { "$lt": 15000 } // AI infers "Saste" (Cheap) = < $15k threshold
      }},
      { "$lookup": { "from": "universities", "localField": "university_id", "foreignField": "_id", "as": "uni" } },
      { "$unwind": "$uni" },
      { "$match": { "uni.basic_info.location.country": "USA" } }
    ]
    ```


**10. "美国计算机硕士学费" (Chinese)**
*   **Translation:** "USA Computer Masters Tuition Fee".
*   **JSON Pipeline:**
    ```json
    [
      { "$match": { 
          "program_type": "Masters",
          "name": { "$regex": "Computer", "$options": "i" }
      }},
      { "$lookup": { "from": "universities", "localField": "university_id", "foreignField": "_id", "as": "uni" } },
      { "$unwind": "$uni" },
      { "$match": { "uni.basic_info.location.country": "USA" } },
      { "$project": { "name": 1, "program_fees_usd_approx": 1 } }
    ]
    ```


### D. Complex & Edge Cases (11-13)


**11. "Cheap CS masters in Germany under 15k with no application fee"**
*   **JSON Pipeline:**
    ```json
    [
      { "$match": {
          "name": { "$regex": "Computer Science", "$options": "i" },
          "program_type": "Masters",
          "program_fees_usd_approx": { "$lt": 15000 }
      }},
      { "$lookup": { "from": "universities", "localField": "university_id", "foreignField": "_id", "as": "uni" } },
      { "$unwind": "$uni" },
      { "$match": { 
          "uni.basic_info.location.country": "Germany",
          "uni.admissions.application_fee_usd": 0 
      }}
    ]
    ```


**12. "Top 50 Universities accepting TOEFL > 90"**
*   **JSON Pipeline:**
    ```json
    [
      { "$match": { "additional_fields.rankings.qs_world": { "$lte": 50 } } },
      { "$lookup": { "from": "programmes", "localField": "_id", "foreignField": "university_id", "as": "progs" } },
      { "$unwind": "$progs" },
      { "$match": { 
          "progs.test_scores": { 
              "$elemMatch": { "testname": "TOEFL", "cutoff_range": { "$gte": "90" } } 
          } 
      }}
    ]
    ```


**13. "Programs starting in Jan 2026 with internship"**
*   **JSON Pipeline:**
    ```json
    [
      { "$match": { "start_date": { "$regex": "2026-01|Jan.*2026", "$options": "i" } } },
      { "$lookup": { "from": "universities", "localField": "university_id", "foreignField": "_id", "as": "uni" } },
      { "$unwind": "$uni" },
      { "$match": { "uni.additional_fields.internship": { "$regex": "Yes|Available", "$options": "i" } } }
    ]
    ```


### E. Negative Logic (14)


**14. "Universities NOT in USA"**
*   **AI Insight:** "NOT" -> `$ne` (Not Equal) operator.
*   **JSON Pipeline:**
    ```json
    [
      { "$match": { "basic_info.location.country": { "$ne": "USA" } } }
    ]
    ```


### F. The "Failure Zone" (Where this System Breaks)


**15. Numeric Comparison on String Fields**
*   **Query:** "Acceptance rate > 50%" (Stored as "55%")
*   **Fail:** `$gt: "50%"` does alphabetical compare ("9%" > "10%").
*   **Fix:** Schema MUST store numbers as `Number`. AI cannot fix bad schema data types reliably.


**16. Abstract Concepts**
*   **Query:** "Universities with good vibes"
*   **Fail:** No field for "vibes".
*   **Result:** AI returns empty filter or hallucinates `student_life.vibe`.
*   **Fix:** Vector Search (Layer 1) handles this. Layer 3 (SQL/Mongo) fails.


**17. "Best ROI" (Return on Investment)**
*   **Query:** "Best ROI for MBA"
*   **Fail:** Requires Math: `(Median Salary - Fees) / Fees`.
*   **Result:** AI struggles to write complex `$project` math in Aggregation without hallucinations.
*   **Fix:** Pre-compute `roi_score` in ETL pipeline.


---


## 5. Data Lifecycle & Scheduling (Tiered Updates)


**Objective:** Ensure data freshness without "Update All" storms.


### 5.1 The Scheduler Strategy (Cron + Async)


| Frequency | Target Data | Time (IST) | Strategy |
| :--- | :--- | :--- | :--- |
| **Daily** | **Currency Rates** | 02:00 AM | Fetch API -> Update `rates_collection` (Single Doc). |
| **Monthly** | **Fees/Deadlines** | 1st, 03:00 AM | Select `high_volatility` fields. Priority Crawl queue. |
| **Yearly** | **Static Info** | Jan 15 | Select `static` fields. Background verification queue. |


### 5.2 Currency Handling (The "Dual-Storage" Model)


**The Strategy:** Maintain "Historical Truth" (Native Currency) while serving "User Convenience" (USD).


1.  **Storage Schema (The "Truth" + "Cache"):**
    *   `fees_native`: Number (e.g., 10000).
    *   `currency_native`: String (e.g., "EUR").
    *   `program_fees_usd_approx`: Number (Indexed, e.g., 11000). *This is the field used for User Filters.*
2.  **The "Weekly Float" Script (Cron):**
    *   **Schedule:** Every Sunday at 03:00 AM.
    *   **Action:** Recalculate `program_fees_usd_approx` for ALL records using the *current* exchange rate.
    *   **Benefit:** If Euro crashes, the "Under $10k" filter updates automatically without re-scraping.
3.  **Filtering (User Experience):**
    *   User sees and filters by USD (or their local currency converted to USD).
    *   Backend query: `db.programs.find({ program_fees_usd_approx: { $lt: limit_usd } })`.
4.  **Display:**
    *   Frontend receives both Native and USD values.
    *   Can show: "€10,000 (~$10,800)". This builds trust.


---



## 6. The Gatekeeper: Safety & Cost


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


### 6.3 Automated Streamed Backups (and Restore)
**Objective:** Protect Data without filling disk space.
**Backup Script:** `cron/backup_mongo.sh`
```bash
mongodump --uri="mongodb://localhost:27017" --archive | gzip | aws s3 cp - s3://vidysea-backups/$(date +%F_%H).gz
```
**Restore Procedure (DR Drill):**
```bash
aws s3 cp s3://vidysea-backups/LATEST.gz - | gunzip | mongorestore --archive
```


### 6.4 User Rate Limiting (Security)
*   **Risk:** Botnet bypassing cache to exhaust AI budget.
*   **Mechanism:** `SlowAPI` (Token Bucket) on Nginx/FastAPI.
*   **Limit:** **60 req/min** per IP.
*   **Response:** `429 Too Many Requests`.


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


## 9. Known Edge Cases & Mitigations (The "Developer FAQ")


This section addresses critical risks identified during architectural review. Developers MUST implement these mitigations.


### 9.1 Operational Edge Cases (Infrastructure)


**1. Disk Thrashing (IOPS Spike):**
*   **The Risk:** 50 concurrent users (Random Reads) + Background Backup (Sequential Writes) = IOPS Saturation (>3000). System freezes.
*   **The Fix:**
    *   **Backups:** MUST run with `ionice -c 3` (Idle Priority) to yield disk access to user traffic.
    *   **Monitoring:** CloudWatch Alarm on `DiskReadOps`. If > 2500 sustained, upgrade to `gp3` (6000 IOPS).
    *   **Mongo:** Set `{ allowDiskUse: true }` in Aggregations to offload RAM sorting to disk.


**2. Cold Start Latency (The MMAP "Page Fault" Problem):**
*   **The Intern Explanation:** When we use `IO_FLAG_MMAP`, we don't load the 10GB index into RAM at startup. We just tell the Operating System "The file is here".
*   **The Physics:** When the first user searches, the OS realizes "Oh, I need that specific byte from the disk". It pauses the CPU (Page Fault), reads the slow disk, loads it into RAM, and then continues. This takes 100ms-500ms per query.
*   **The Fix: The Warm-up Script.**
    *   **Action:** Immediately after server boot, run a script that fires 50 random vector searches.
    *   **Why?** This forces the OS to "touch" the disk sectors and load them into the Page Cache (RAM). Real users then hit the RAM, which is fast.
    *   **Warm-up Query List:**
        ```python
        WARMUP_QUERIES = [
            "Computer Science", "MBA", "Data Science", "Psychology", "Engineering",
            "Business", "Medicine", "Law", "Art", "Design",
            "History", "Biology", "Chemistry", "Physics", "Math",
            "Economics", "Finance", "Accounting", "Marketing", "Management",
            "Nursing", "Education", "Teaching", "Philosophy", "Sociology",
            "Political Science", "Geography", "Geology", "Astronomy", "Botany",
            "Zoology", "Ecology", "Environmental Science", "Anthropology", "Archaeology",
            "Linguistics", "Literature", "Music", "Theatre", "Dance",
            "Film", "Photography", "Journalism", "Communication", "Media",
            "Sports", "Health", "Fitness", "Nutrition", "Yoga"
        ]
        ```


**3. Rate Limit & Race Conditions:**
*   **The Risk:** Currency API fails / Multiple users triggering same scrape.
*   **The Fix:**
    *   **Currency:** If API fails, use "Last Good Known Rate" (cache for 48h). Log Critical Alert.
    *   **Scrape:** Use **Pending Scrape Cache** (Redis/KeyDB). If `scrape_lock_{query}` exists, wait/poll.


### 9.2 Search Logic Edge Cases (The "Gotchas")


**4. The "Common Word" Trap (Why "University" Kills the CPU):**
*   **The Scenario:** User searches "University of Science".
*   **The Math:**
    *   "University" appears in 98,000 documents.
    *   "of" appears in 99,900 documents.
    *   "Science" appears in 5,000 documents.
*   **The Crash:** MongoDB Text Search tries to fetch ALL IDs for all 3 words and find the intersection. It has to scan ~200,000 index entries. This spikes CPU to 100% and blocks other queries.
*   **The Fix: Aggressive Stop Word List.**
    *   **Rule:** Before sending to MongoDB `$text`, remove these words: `["of", "the", "in", "for", "and", "a", "an", "at", "to", "university", "college", "institute", "school"]`.
    *   **Result:** The query becomes just "Science". Mongo scans only 5,000 entries. Fast.


**5. RRF Tuning ("Apples vs Oranges"):**
*   **The Risk:** MongoDB Text Scores are unbounded (e.g., 50.0), while Vector Scores are 0-1. Fear that Text Score will dominate.
*   **The Explanation:** RRF uses **RANK**, not **SCORE**. Even if Text Score is 5,000,000, RRF only cares that it is Rank #1.
*   **The Formula:** `Score = 1 / (60 + Rank)`. This naturally normalizes both engines to a 0.0-0.01 range, making them comparable. `k=60` ensures the curve is flat enough that one engine doesn't dominate.


**6. "Description Noise" (Oxford Style / Acronym Collision):**
*   **The Risk:** Generic college mentions "Oxford-style" or "MIT professor". Keyword search ranks it high.
*   **The Fix:** **Aggressive Field Weighting** in MongoDB Index:
    *   `name`, `abbreviation`: Weight **50** (Critical Boost)
    *   `description`: Weight **1** (Low Boost)
    *   This ensures "MIT" in the Name dominates "MIT" in the Description.
    *   **Multiple Matches:** If "MIT" matches multiple valid universities (e.g., Massachusetts vs. Manipal), **Show All**. The system relies on Pagination/List View to let the user distinguish; it does not force a single "Winner".
    and if there are more than 1 matching result for MIT , we can show them all to user as a list. we have a facilty of pagination also. it not like we just have to show the exact 1 response right. 


**7. "Translated Token" Mismatch:**
*   **The Risk:** User searches "Informatik" (German). DB has "Computer Science" (English). Searching "Informatik" in Text Index returns 0 results.
*   **The Fix:**
    *   **Vector Branch:** Handles this natively (Multilingual Model).
    *   **Keyword Branch:** AI MUST translate "Informatik" -> "Computer Science".
    *   **Rule:** Keyword Search **ONLY** searches for the English Translated Term.
    *   **Why?** Since the Database is strictly English-Only, searching for "Informatik" via MongoDB Text Search will result in **0 matches**. It adds noise/latency for no gain. We rely on the Vector Engine to bridge the language gap, and the Keyword Search to find the exact *English* text match.


### 9.3 Data & AI Edge Cases


**8. Numeric Comparison on String Fields (The Sort Breaker):**
*   **The Risk:** "Acceptance rate > 20%". If stored as String "10%", it sorts *after* "2%". Filter breaks.
*   **The Fix:** **ETL Mandate.** All numeric data (Fees, Rates, Scores) MUST be stored as `Number` type (e.g., `0.10`, `20000`). Strings are banned for these fields.


**9. Abstract Concepts:**
*   **The Risk:** "Universities with good vibes".
*   **The Fix:** Layer 3 (Mongo) returns empty. Layer 1 (Vector) handles the "Vibe" match.


**10. "Best ROI" (Complex Math):**
*   **The Risk:** AI struggles to write `(Salary - Fees)/Fees` in Aggregation without hallucinations.
*   **The Fix:** Pre-compute `roi_score` in ETL pipeline.


**11. Currency Unit Ambiguity:**
*   **The Risk:** User types "Fees under 50k".
*   **The Problem:** Is it USD or INR? If user is in India, 50k = $600. If assumed USD, result is wrong.
*   **The Fix:** **Context Injection.** AI Prompt must include `User_Location`.
    *   Rule: If symbol missing, infer from Location (India -> INR, EU -> EUR).
    *   AI output must convert to USD for the DB Query (`50k INR` -> `$600`).


**12. Boolean Exclusion Logic:**
*   **The Risk:** "Canada except Ontario".
*   **The Fix:** Python Pipeline Validator MUST explicitly support and allow `$ne` (Not Equal) and `$nin` (Not In) operators.


**13. Debuggability of RRF:**
*   **The Risk:** "Why is this University #1?" (Vector vs Text mix is opaque).
*   **The Fix:** API must support `?debug=true` flag.
*   **Response:** Return `{ "vector_rank": 5, "text_rank": 1, "final_rrf_score": 0.03 }` in the JSON metadata.


**14. Synchronous Healing Latency (The "Bounce" Risk):**
*   **The Risk:** Scraping takes 5 seconds. User closes tab.
*   **The Fix:** **UI Loading State.** The Frontend MUST show specific status: *"Searching live sources..."*. We stick to Sync for v1 simplicity, but if bounce rate > 30%, move to WebSocket (v2).


**15. Schema Mismatch (Critical Blocker):**
*   **The Conflict:** TDD says "Store Numbers", but Schema has "String".
*   **The Fix: Transformation Table (ETL):**
    *   `acceptance_rate` (String "10%") -> `acceptance_rate_val` (Number 0.10)
    *   `student_faculty_ratio` (String "15:1") -> `student_faculty_ratio_val` (Number 15)
    *   **Rule:** Logic/Sorting uses `_val` fields. Display uses String fields.


**16. Currency Cache Stale:**
*   **The Risk:** Cache holds "Under $10k" logic. Currency crashes. Logic is stale for 7 days.
*   **The Fix:** Include `{date: YYYY-MM-DD}` in the Cache Key. This forces daily cache invalidation for all currency-related queries.


**17. Multilingual Stop Words:**
*   **The Risk:** Spanish user types "Universidad de Madrid". "de" is common.
*   **The Fix:** Expanded Stop Word List: `["de", "la", "en", "und", "der", "le", "les", "des"]` + English list.


**18. "Number-ish" String Disaster (ETL Hygiene):**
*   **Scenario:** Scraper sees "20k" or "10,000".
*   **The Risk:** Failed conversion -> `null` or crash.
*   **The Fix:** **Dirty Data Log.** If `parse_int()` fails, Log the ID and **SKIP** the record. Do NOT pollute the DB with strings in Number fields.


**19. "Stuck Lock" (Server Crash):**
*   **Scenario:** Server dies while scraping "Harvard". `scrape_lock` remains forever.
*   **The Fix:** **Hard TTL.** All Redis/KeyDB locks MUST have an expiry (e.g., `expire=300` for 5 mins). No infinite locks allowed.


**20. "Near Me" (Geolocation) - v2 Scope:**
*   **Scenario:** "Colleges near me".
*   **The Constraint:** We rely on Text/Vector. We do not have Lat/Long data in v1.
*   **The Fix:** AI must reply: *"Please specify a city or country. Geolocation is coming in v2."*


**21. Stale Negative Cache:**
*   **Scenario:** Admin adds "New University". User searches it. Cache says "No Result" (valid for 30 days).
*   **The Fix:** **Invalidation Hook.** `add_university()` function MUST run `delete_many({ type: "negative_cache" })`. Flushing the whole negative cache is safer/cheaper than finding specific keys.


**22. Thundering Herd (Cold Start):**
*   **Scenario:** Server restarts. 50 warm-up queries run. 100 real users hit at same second.
*   **The Fix:** **Queue Buffer.** If `warmup_complete == False`, API returns "503 Service Unavailable (Starting Up)" or queues requests for 10s. Do NOT let users hit disk while cache is cold.


**23. Pagination Drift (Infinite Scroll Duplicates):**
*   **Scenario:** User sees Page 1. Admin deletes Record #5. Record #21 moves to Page 1. User loads Page 2 (skipping Record #21).
*   **The Constraint:** Cursor pagination is too complex for v1.
*   **The Fix:** **Frontend Deduplication.** The Frontend MUST deduplicate results by `_id`. Do NOT rely on Backend consistency for live data.


**24. Date Format Hell (ETL Gatekeeper):**
*   **Scenario:** Data comes as "Fall '25", "01/09/2025".
*   **The Risk:** Regex filters fail.
*   **The Fix:** **ISO 8601 Mandate.** ETL MUST normalize ALL dates to `YYYY-MM-DD` before insert. Unparseable dates = `null`. Do NOT store raw date strings.


**25. Healing Timeout (Network Lag):**
*   **Scenario:** Serper takes 8s. Browser/FastAPI times out at 5s.
*   **The Fix:**
    *   **Backend:** Set FastAPI timeout to **30s** for search endpoints.
    *   **Frontend:** Set Client timeout to **30s**. Show "Still searching..." message after 5s.


**26. The "No Results" Loop (Spelling Trap):**
*   **Scenario:** User searches "Hahvard". System finds nothing -> Caches "No Result".
*   **The Fix:** **Vector First.** The Vector Search (Layer 1) matches "Hahvard" to "Harvard" (Score > 0.88). We find the result *before* triggering the Negative Cache.
*   **AI Backup:** Layer 3 AI explicitly fixes spelling in `semantic_rewrite` field.


**27. Broad Query Horizon (Developer Note: Why not "Threshold"?):**
*   **The Question:** "Why `k=50`? Why not `score > 0.7`?"
*   **The Physics:** Vector Indices (HNSW/IVF) are built for **Nearest Neighbor** (finding the Top K closest points). They are *not* built for Range Scans ("Find all points in this radius").
*   **The Cost:** A Threshold query (`> 0.7`) might force a scan of the entire index (O(N)), killing performance. A Top-K query is O(log N).
*   **The Fix:** **Dynamic K.** If Text Search returns > 100 hits, we auto-increase FAISS `k` to **200** to widen the net safely without scanning the whole disk.

**28. FAISS Update Ghosting:**
*   **Scenario:** Update Description. `index.add(id, new_vector)`.
*   **The Risk:** FAISS appends. Now ID has 2 vectors (Old + New). Search might return the Old one.
*   **The Fix:** **Atomic Replacement.** The update function MUST be: `index.remove_ids([id])` -> `index.add_with_ids([id], [vector])`. Never skip the remove step.

**29. Cultural Semantic Trap (AI Prompting):**
*   **Scenario:** "Public School" means "Private" in UK, "Government" in India.
*   **The Fix:** **AI System Prompt.** We do not hardcode rules.
    *   *Instruction:* "Analyze the User's Location (e.g., UK) and Query (e.g., 'Public School'). Apply filters that match the *local cultural meaning* of the terms."
    *   *Result:* AI adds `{"institution_type": "Private"}` for UK users automatically.

**30. The "False Friend" Trap (Language-Gated RRF):**
*   **Scenario:** German user types "Gift" (Poison). English DB has "Gifted Students".
*   **The Risk:** Keyword Search finds "Gifted" (High Score). Vector Search finds "Toxicology". RRF might rank "Gifted" #1.
*   **The Fix:** **Language Gate.**
    *   Detect User Language.
    *   If `Language != English`: **DISABLE Keyword Search Branch**. Rely 100% on Multilingual Vector Search.
    *   *Why not simple weighting?* Keyword scores are unbounded. A massive keyword match can often override even a weighted vector score. Disabling is safer for preventing dangerous mistranslations.

**31. The "Synonym Loop" (Search Expansion):**
*   **Scenario:** AI rewrites "Law" -> "Legal Studies". DB record has "School of Law".
*   **The Risk:** Record matches "Law" but not "Legal". Search returns 0.
*   **The Fix:** **OR Logic.**
    *   Keyword Search Query: `("Law" OR "Legal Studies")`.
    *   Never *replace* the user's term; always *expand* it.

**32. Sorting vs. RRF (The "Cheapest" Problem):**
*   **Scenario:** User sorts by "Price Low".
*   **The Risk:** RRF ranks by *Relevance*. The "Cheapest" college might be irrelevant (Rank #500) and not returned.
*   **The Fix:** **Sort Bypass.**
    *   If `sort` parameter is present (Price, Rank, Date) -> **Bypass Layer 1 (Vector/RRF)**.
    *   Route strictly to Layer 3 (Mongo) to execute the sort on the *entire* dataset.
    *   *Note:* Frontend sorting is insufficient because it only sorts the *page* of results it receives. The Backend must sort the whole DB to find the true cheapest items.

**33. The "Freshness Lie" (Source Dating):**
*   **Scenario:** Scraper finds 2019 blog post. `last_updated` = Today. User thinks fees are current.
*   **The Risk:** Misleading the user about price accuracy.
*   **The Fix:** **Source Date Extraction.**
    *   **AI Instruction:** "Extract source date. If missing, look for intake years (e.g., '2020 intake')."
    *   **Flag:** If `source_date < (Today - 2 Years)`, set `data_warning: "potentially_outdated"`.
    *   **UI:** Show "⚠️ Data from 2019" badge.


---


## 10. Final Developer Checklist


1.  **IOPS Check:** Verify AWS EBS volume type is `gp3` (not `gp2`) in EC2 Console.
2.  **Optimistic Lock:** Test Admin saving an old version while Scraper updates it (should fail/warn).
3.  **Soft Delete:** Mark a Uni inactive and verify it disappears from Search immediately (without reload).
4.  **e5 Prefixes:** Verify `query:` / `passage:` logic.
5.  **Swap File:** Run `free -h` and confirm **16Gi** Swap is active.
6.  **Neg Cache:** Simulate Serper 500 Error -> Ensure NOTHING is written to cache.
7.  **Budget Cap:** Manually set counter to 499 and verify 501st request is blocked.
8.  **Lock TTL:** Kill server during scrape -> Verify lock vanishes after 5 mins.
9.  **Dirty Data:** Feed `"fees": "20k"` to ETL -> Verify it logs to `error.log` and does NOT insert.
10. **Cache Invalidation:** Add a new University -> Verify Negative Cache is flushed.
11. **Cold Start:** Hit API during reboot -> Verify it waits/503s until Warm-up finishes.
12. **Frontend Dedupe:** Simulate DB delete between pages -> Verify UI doesn't crash/show duplicates.
13. **Date Logic:** Feed "Fall 2025" to ETL -> Verify DB stores `2025-09-01`.
14. **Timeout:** Add `sleep(10)` to Serper mock -> Verify Frontend waits and doesn't timeout.


---


## 11. Testing Strategy (QA & CI/CD)


**Objective:** Prevent regression in a complex multi-layer system.


### 11.1 Unit Tests (PyTest)
*   **Schema Validator:** Feed invalid JSON (e.g., "filters": {"unknown_field": 1}) -> Assert it drops the field.
*   **ETL Transformation:** Feed "10,000" -> Assert `10000` (Number). Feed "Free" -> Assert `0`.
*   **RRF Math:** Mock Vector Score 0.9, Text Score 50. Assert RRF Score calculation is correct.


### 11.2 Integration Tests
*   **Full Search Flow:**
    1.  Inject "Test Uni" into DB.
    2.  Wait for Index Refresh.
    3.  Search "Test Uni".
    4.  Assert Result in Top 3.
*   **Cache Hit:**
    1.  Search "MBA in USA".
    2.  Search "MBA in USA" again.
    3.  Assert Response Time < 50ms (Cache Hit).


### 11.3 Load Testing (Locust)
*   **Scenario:** 50 Concurrent Users.
*   **Expectation:**
    *   p95 Latency < 800ms.
    *   Error Rate < 1%.
    *   Swap Usage < 20%.


well here are some more points that will be added 

* in the db apart from searching only on the names, for which we also have vectors stored in the database for each record, we can also do token searching for each one of the program/test/university like description and all 

* in caching we will store query vector and also the response from the ai that will be in json

* ai  will not give the exact query. it will give something that can help developer to make the query in the backend, one solution might be json, or any other possible way


now rate this design document out of 100 and give pros and cons and highlight hidden edge cases which we did not addresed yet. 
Note:  we  already had addressed some of the raised edge cases so  please not raise the same. find the hidden edge cases, not which are already addressed
