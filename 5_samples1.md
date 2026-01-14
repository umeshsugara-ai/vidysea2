**Logic 4: AI Query Builder Examples (Strict JSON):**


**Example 1: Simple Intent**
*   **User:** "MBA in USA"
*   **AI JSON Plan:**
    ```json
    {
      "intent": "program_search",
      "filters": {
        "program_type": "Masters",
        "name": { "$regex": "MBA", "$options": "i" },
        "country": "USA"
      }
    }
    ```
*   **Result:** Exact match on fields. Fast execution.


**Example 2: Complex Constraint**
*   **User:** "Cheap CS masters in Germany under 15k with no application fee"
*   **AI JSON Plan:**
    ```json
    {
      "intent": "program_search",
      "filters": {
        "program_name": { "$regex": "Computer Science", "$options": "i" }, // Semantic rewrite
        "country": "Germany",
        "program_fees_usd_approx": { "$lt": 15000 },
        "application_fee": 0
      },
      "sort": { "field": "program_fees_usd_approx", "order": 1 } // Infer "Cheap" = Sort Asc
    }
    ```


**Example 3: Edge Case (Time-Sensitive)**
*   **User:** "Universities with spring intake"
*   **AI JSON Plan:**
    ```json
    {
      "intent": "university_search",
      "filters": {
        "admissions.application_opening_date": { "$regex": "Spring|January", "$options": "i" }
      }
    }
    ```


---