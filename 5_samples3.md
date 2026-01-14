
**Logic 4: AI Pipeline Generator (The Airport Security Model):**


**Objective:** The AI generates a **Validated Aggregation Pipeline (JSON List)**. It acts as the "Passenger" packing the bag. The Developer acts as the "X-Ray Machine" (Validator) scanning for dangerous operators (`$out`, `$merge`, `$function`) before execution. The AI is fed a minified schema via a config file (`config/db_schemas.json`).


**The "Variety Pack" Examples (Covering Simple, Multilingual, Complex, and Edge Cases):**


### A. Simple & Direct (1-3)


**1. "MBA in USA"**
*   **JSON Pipeline:**
    ```json
    [
      { "$match": { "program_type": "Masters", "name": { "$regex": "MBA", "$options": "i" } } },
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


### B. Multilingual & Semantic (4-6)


**4. "Wirtschaftsinformatik in Deutschland" (Business Informatics in Germany)**
*   **AI Insight:** Detects German. Translates intent to English Schema fields but keeps value matching logic relevant.
*   **JSON Pipeline:**
    ```json
    [
      { "$match": { "name": { "$regex": "Business Informatics|Wirtschaftsinformatik", "$options": "i" } } },
      { "$lookup": { "from": "universities", "localField": "university_id", "foreignField": "_id", "as": "uni" } },
      { "$unwind": "$uni" },
      { "$match": { "uni.basic_info.location.country": "Germany" } }
    ]
    ```


**5. "Universidades baratas en Canadá" (Cheap Universities in Canada)**
*   **AI Insight:** "Baratas" -> Sort by Fees Ascending.
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


**6. "Computer Science ke liye scholarships" (Hinglish)**
*   **JSON Pipeline:**
    ```json
    [
      { "$match": { "name": { "$regex": "Computer Science", "$options": "i" } } },
      { "$lookup": { "from": "universities", "localField": "university_id", "foreignField": "_id", "as": "uni" } },
      { "$unwind": "$uni" },
      { "$match": { "uni.additional_fields.scholarship_details": { "$exists": true, "$ne": "NA" } } }
    ]
    ```


### C. Complex & Constraints (7-10)


**7. "Cheap CS masters in Germany under 15k with no application fee"**
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


**8. "Top 50 Universities accepting TOEFL > 90"**
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


**9. "Programs starting in Jan 2026 with internship"**
*   **JSON Pipeline:**
    ```json
    [
      { "$match": { "start_date": { "$regex": "2026-01|Jan.*2026", "$options": "i" } } },
      { "$lookup": { "from": "universities", "localField": "university_id", "foreignField": "_id", "as": "uni" } },
      { "$unwind": "$uni" },
      { "$match": { "uni.additional_fields.internship": { "$regex": "Yes|Available", "$options": "i" } } }
    ]
    ```


**10. "Universities with < 10% acceptance rate and > 1000 international students"**
*   **JSON Pipeline:**
    ```json
    [
      { "$match": { 
          "admissions.acceptance_rate": { "$regex": "^[0-9]%", "$options": "i" } // AI must regex string range logic
      }},
      // Note: This query is RISKY. "10%" stored as String makes numeric compare hard.
      // AI might fallback to Regex or try $toInt. This is a potential FAIL CASE.
    ]
    ```


### D. The "Failure Zone" (Where this System Breaks)


**11. Numeric Comparison on String Fields**
*   **Query:** "Acceptance rate > 50%" (Stored as "55%")
*   **Fail:** `$gt: "50%"` does alphabetical compare ("9%" > "10%").
*   **Fix:** Schema MUST store numbers as `Number`. AI cannot fix bad schema data types reliably.


**12. Abstract Concepts**
*   **Query:** "Universities with good vibes"
*   **Fail:** No field for "vibes".
*   **Result:** AI returns empty filter or hallucinates `student_life.vibe`.
*   **Fix:** Vector Search (Layer 1) handles this. Layer 3 (SQL/Mongo) fails.


**13. "Best ROI" (Return on Investment)**
*   **Query:** "Best ROI for MBA"
*   **Fail:** Requires Math: `(Median Salary - Fees) / Fees`.
*   **Result:** AI struggles to write complex `$project` math in Aggregation without hallucinations.
*   **Fix:** Pre-compute `roi_score` in the ETL pipeline.


---
