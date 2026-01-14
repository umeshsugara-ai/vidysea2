
**Logic 4: AI Query Builder Examples (Multi-Collection Schema):**


**Objective:** The AI acts as a "Schema Mapper". It groups filters by collection so the developer knows exactly how to construct Joins/Lookups. It also expands acronyms ("MBA" -> "Master of Business Administration") to match the standard DB data.


**Example 1: Simple Intent (Cross-Collection)**
*   **User:** "MBA in USA"
*   **AI JSON Plan:**
    ```json
    {
      "intent": "program_search",
      "primary_collection": "programmes",
      "join_required": true,
      "pipeline_steps": {
        "universities": {
          "basic_info.location.country": "USA"
        },
        "programmes": {
          "name": { "$regex": "Master of Business Administration", "$options": "i" }, // AI expands acronym
          "program_type": "Masters"
        }
      },
      "sort": null
    }
    ```
*   **Developer Logic:** 
    1. Query `universities` for "USA" -> Get List of `_id`.
    2. Query `programmes` where `university_id` IN `[ids]` AND `name` matches "Master...".


**Example 2: Complex Constraint (Multi-Field)**
*   **User:** "Cheap CS masters in Germany under 15k with no application fee"
*   **AI JSON Plan:**
    ```json
    {
      "intent": "program_search",
      "primary_collection": "programmes",
      "join_required": true,
      "pipeline_steps": {
        "universities": {
          "basic_info.location.country": "Germany",
          "admissions.application_fee_usd": 0 // "No application fee" mapped to University Admission logic
        },
        "programmes": {
          "name": { "$regex": "Computer Science", "$options": "i" }, // "CS" -> "Computer Science"
          "program_fees_usd_approx": { "$lt": 15000 }
        }
      },
      "sort": { "collection": "programmes", "field": "program_fees_usd_approx", "order": 1 }
    }
    ```


**Example 3: Edge Case (Tests & Time-Sensitive)**
*   **User:** "Universities accepting TOEFL with spring intake"
*   **AI JSON Plan:**
    ```json
    {
      "intent": "university_search", // User asked for "Universities"
      "primary_collection": "universities",
      "join_required": true,
      "pipeline_steps": {
        "universities": {
          "admissions.application_opening_date": { "$regex": "Spring|January", "$options": "i" }
        },
        "programmes": {
          "language_test_requirements": "TOEFL iBT" // Filter programs that accept this test
        }
      },
      "sort": null
    }
    ```


**Example 4: Specific Test Search**
*   **User:** "IELTS test centers and fees"
*   **AI JSON Plan:**
    ```json
    {
      "intent": "test_search",
      "primary_collection": "tests",
      "join_required": false,
      "pipeline_steps": {
        "tests": {
          "testname": { "$regex": "IELTS", "$options": "i" }
        }
      },
      "projection": ["test_format_mode", "registration_fee_currency", "registration_fee"]
    }
    ```


**Example 5: Direct University Lookup (No Join)**
*   **User:** "Stanford University address"
*   **AI JSON Plan:**
    ```json
    {
      "intent": "university_search",
      "primary_collection": "universities",
      "join_required": false,
      "pipeline_steps": {
        "universities": {
          "basic_info.name": { "$regex": "Stanford University", "$options": "i" }
        }
      },
      "projection": ["basic_info.location", "basic_info.name"]
    }
    ```


---