Here's a concise yet informative template you can use in your PR description, highlighting the improvements and reasoning behind the changes:

---

### Summary

This PR refactors and improves the `get_annotation_data` function in the `DataView` class. The goal is to enhance performance, data handling robustness, and overall code readability. Below is a comparison of the key changes before and after the fix.

---

### What Changed

1. **Database Query Optimization**
    
    - Used `select_related("form")` and `select_related("annotated_by", "validated_by")` to reduce the number of queries.
    - Employed `values()` to retrieve only necessary fields.
    - This helps minimize the overhead of multiple queries and speeds up data retrieval.
2. **Data Expansion and Handling**
    
    - Replaced direct JSON expansion with a more robust approach using `pd.DataFrame.from_records(samples_qs)` followed by `pd.Series` expansion.
    - Ensured potential non-dict entries in the JSONField are handled gracefully.
3. **Column Management and Consistency**
    
    - Renamed columns (`id` to `sample_id`, etc.) to maintain consistency throughout the dataset.
    - Ensured all required columns exist, even if they are empty, so downstream operations remain stable.
4. **Date and Time Formatting**
    
    - Used vectorized datetime parsing and formatting to improve performance and keep the code clean.
    - Handled invalid or missing date strings consistently.
5. **Word Count Calculation**
    
    - Used a vectorized approach (`df["text to translate"].str.split().str.len()`) for word counts, which is more efficient than iterative methods.
6. **Aggregations & Group-By Statistics**
    
    - Per-annotator metrics are computed via a single `.groupby()` call with aggregated fields (`total_words`, `total_texts`, `last_annotation`).
    - Reduced multiple passes over the dataset and improved clarity.
7. **Pending Text and Word Computation**
    
    - Used a vectorized mask (`pending_mask`) for statuses to compute pending data, making it more concise and scalable.
8. **High-Level Key Counting**
    
    - Calculated unique combinations of high-level key fields using `drop_duplicates()`, clarifying the logic for counting unique categories.

---

### Why These Changes Are Needed

- **Performance**: Reducing the number of queries and using vectorized Pandas operations significantly speeds up the process, especially when handling large datasets.
- **Readability & Maintainability**: The new code is more structured, makes logical steps clear, and is easier for future contributors to extend or debug.
- **Robustness**: Additional checks for non-dictionary data types in the JSON field prevent edge-case errors.

---

### Testing & Validation

1. **Local Testing**
    
    - Verified the dataset renders correctly on the UI for both existing and newly created AnnotationTracks.
    - Checked that all statistics (word counts, text counts, pending data) match the expected values from the old code.
2. **Performance Benchmarks**
    
    - Compared response times for large track datasets before and after the fix. Observed a noticeable decrease in load and render time.
3. **Edge Cases**
    
    - Tested with empty data (no AnnotationTrackSamples) to confirm the table gracefully shows zeros without errors.
    - Ensured malformed or unexpected `data` fields do not break the DataFrame expansion.

---

### How to Review

- Focus on the transition to `select_related` and `values()`, ensuring no fields needed downstream are missing.
- Confirm that the expanded DataFrame columns line up with the original columns.
- Verify consistency with naming conventions (`sample_id` vs. `id`, etc.).
- Check that the template's context retains the same functionality and covers all original output fields.

---

### Next Steps

- Merge and deploy to a staging environment for further testing with real data.
- Monitor logs for any unexpected exceptions.
- Prepare any necessary documentation updates if the data model or output format for downstream processes has changed.

---

**Thank you** for taking the time to review this PR! Please let me know if there are any suggestions or areas we should revisit.


Here is an example PR body section you could use that specifically highlights the **problems** you discovered and **the improvements** you implemented:

---

### Problems Identified & Improvements Made

1. **Inefficient Data Queries**  
    **Problem**: The original code did not use `select_related` or `values()` effectively. This caused multiple database hits and unnecessary data retrieval.  
    **Improvement**: Added `select_related("form")` and `select_related("annotated_by", "validated_by")` along with `values()` to reduce the number of queries and improve performance.
    
2. **Handling Complex JSON Fields**  
    **Problem**: The code assumed the JSON `data` field would always be a valid dict. Non-dict entries could lead to errors or unexpected behavior.  
    **Improvement**: Implemented a check to ensure `data` is a dictionary before expanding and fallback to an empty dict otherwise. This makes the code more robust.
    
3. **Missing or Inconsistent Columns**  
    **Problem**: Some columns (e.g., `validation`, `sample_id`) could be absent in certain edge cases, or naming varied across stages, causing potential issues for downstream processes.  
    **Improvement**: Ensured all required columns exist (even if empty) and standardized column names (`id` → `sample_id`, etc.) for consistency.
    
4. **Non-Vectorized Word Count Calculation**  
    **Problem**: Word counting relied on iterative `.apply(len)`, which can be slower for large data sets.  
    **Improvement**: Switched to a vectorized approach using `df["text to translate"].str.split().str.len()`, improving performance significantly.
    
5. **Date & Time Formatting**  
    **Problem**: The code handled date/time conversion in a somewhat manual manner, leading to repetitive logic and potential for null issues.  
    **Improvement**: Used `pd.to_datetime(...).dt.strftime(...)` in a vectorized manner, simplifying the code and improving reliability.
    
6. **Complex Group-By Aggregations**  
    **Problem**: The aggregator logic was more complex and less clear, making maintenance difficult.  
    **Improvement**: Simplified to a single `.groupby("annotator")` with clear `agg` definitions (`total_words`, `total_texts`, `last_annotation`), improving readability and performance.
    
7. **Pending Status Computation**  
    **Problem**: The original code repeatedly filtered the DataFrame to find pending samples.  
    **Improvement**: Created a single boolean mask (`pending_mask`) to calculate pending texts and words in one pass, reducing redundancy.
    
8. **High-Level Key Counting**  
    **Problem**: Counting unique high-level keys was done through multiple steps that could be streamlined.  
    **Improvement**: Used `drop_duplicates()` on the subset of columns for high-level keys, making the logic more direct and scalable.
    

---

By addressing these issues, the new implementation is more **efficient**, **readable**, and **maintainable**. It not only speeds up data retrieval but also ensures that edge cases (such as missing or invalid JSON data) are handled gracefully. Let me know if you have any questions or further suggestions!