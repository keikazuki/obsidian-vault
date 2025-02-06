### Pull Request Notes for Code Review

#### **Summary of Changes**

This PR refactors the `get_status_table` function to improve performance, readability, and maintainability. The previous implementation had multiple inefficiencies, redundant DataFrame operations, and unnecessary loops. The new implementation addresses these issues while maintaining the same functionality.

---

### **Key Improvements**

#### **1. Performance Optimization**

- **Reduced Database Queries**:
    
    - Used `.select_related("track", "annotated_by")` in the query to minimize the number of queries executed for related models.
    - Fetched all `AnnotationTrackSample` records in one query instead of filtering multiple times.
- **Replaced Redundant DataFrame Creations**:
    
    - Previously, separate DataFrames were created for each sample status (`pending_df`, `annotated_df`, etc.), causing unnecessary computations.
    - Now, a **single DataFrame (`all_df`)** is used, and a pivot table (`status_agg`) is leveraged to compute status-specific aggregations in one step.
- **Optimized Word Count Calculation**:
    
    - Used vectorized `str.split().str.len()` instead of applying a function row by row.

---

#### **2. Code Readability and Maintainability**

- **Refactored Data Processing Logic**:
    
    - Moved repeated logic into a single loop instead of handling each status separately.
    - Introduced `status_mapping` for a cleaner approach to mapping statuses.
- **Improved Data Merging Approach**:
    
    - Used **merge()** operations efficiently to join datasets instead of performing multiple filter and groupby operations.
- **Better Function Naming**:
    
    - Used more descriptive function names like `show_publish_button()` instead of `showPublishButton()`.
- **Error Handling and Edge Cases**:
    
    - Ensured the function correctly handles cases when there are **no tracks** or **no samples**.
    - Avoids errors due to missing columns in DataFrame operations.

---

#### **3. Formatting and UI Consistency**

- **Datetime Formatting**:
    
    - Used `strftime()` to properly format dates for display.
- **Column Selections**:
    
    - Ensured only necessary columns are passed to the UI by dropping `track_id` where appropriate.
- **Cumulative Sum Calculation for Progress Chart**:
    
    - Handles cases where there are no annotations to avoid processing empty DataFrames.

---

### **Performance Gains**

- **Fewer Queries**: Reduced database queries significantly by fetching all necessary data at once.
- **Faster Execution**: Optimized DataFrame operations to minimize computational overhead.
- **Reduced Memory Usage**: No longer creating unnecessary intermediate DataFrames.

---

### **Impact of These Changes**

- The function is now **faster, cleaner, and more efficient**.
- Maintains the same functionality while significantly reducing complexity.
- Ensures **scalability** as the dataset grows.

---

### **Next Steps**

- Additional testing with large datasets to confirm speed improvements.
- Consider caching results for frequently accessed data to further enhance performance.

Let me know if you have any questions or suggestions! 🚀

### **Causes of Slow Performance in the Original Code & Reasons for Refactoring**

The original implementation had several inefficiencies that contributed to slow performance. Below is a breakdown of the key issues and why they were addressed in the refactored version.

---

## **Causes of Slow Performance**

### **1. Too Many Database Queries (N+1 Query Problem)**

- **Issue:** The original code made multiple separate database queries:
    - One query to get `NMTModel`
    - One query to fetch `AnnotationTrack`
    - One query to get `AnnotationFormField`
    - Multiple queries (one per status) to fetch `AnnotationTrackSample` records.
- **Impact:** This caused unnecessary repeated database hits, increasing execution time.
- **Fix:** Used **a single query** with `.select_related("track", "annotated_by")` to fetch all necessary data in one go, eliminating redundant queries.

---

### **2. Inefficient DataFrame Creation**

- **Issue:** The code created **separate DataFrames** for each annotation status (e.g., `pending_df`, `annotated_df`, `validated_df`, etc.).
- **Impact:** This was redundant and significantly increased memory usage and computation time.
- **Fix:**
    - Used **a single DataFrame (`all_df`)** to hold all samples.
    - Used **a pivot table (`status_agg`)** to aggregate word counts per status in one step instead of multiple groupby calls.

---

### **3. Redundant Row-Wise Operations in Pandas**

- **Issue:** The code used `.apply(lambda x: …)` on individual rows for:
    - Word count calculations.
    - Date formatting.
    - Status completion calculations.
- **Impact:** `.apply()` on DataFrame rows is slow and should be avoided in favor of vectorized operations.
- **Fix:**
    - Used vectorized Pandas operations such as `.str.split().str.len()` for word count.
    - Used `.agg()` instead of applying functions row-wise.

---

### **4. Inefficient Data Merging**

- **Issue:** The code used `pd.merge()` **multiple times** on `df2show`, merging different status-specific DataFrames one by one.
- **Impact:** Each merge operation adds overhead, especially on large datasets.
- **Fix:**
    - Used **a single pivot table** (`status_agg`) to aggregate per-status word counts and then merged **only once**.

---

### **5. Unnecessary Data Processing on Empty DataFrames**

- **Issue:** The code attempted to compute DataFrame operations (e.g., grouping, aggregation) even when no data was present.
- **Impact:** This led to unnecessary computations and potential errors.
- **Fix:**
    - Introduced early exits: If there are **no tracks** or **no samples**, return immediately with empty data.

---

### **6. Inefficient Status Determination**

- **Issue:** The `get_status()` function used multiple **if conditions** to determine status.
- **Impact:** It was not optimized for vectorized execution.
- **Fix:** Applied Pandas-based column operations instead of iterating over rows.

---

### **7. Inefficient Handling of UI Formatting**

- **Issue:** Date formatting used `.apply()` row by row, which is slow.
- **Impact:** Increased processing time unnecessarily.
- **Fix:** Used **vectorized** `.strftime()` on the entire column.

---

## **Summary of Fixes & Performance Gains**

|Issue|Fix|Performance Impact|
|---|---|---|
|Too many DB queries|Used `.select_related()` to fetch data in **one** query|🚀 Reduced DB hits significantly|
|Repetitive DataFrame creation|Used **one DataFrame** instead of multiple|🚀 Reduced memory usage & processing time|
|`.apply()` used for word count|Used **vectorized** `.str.split().str.len()`|🚀 Faster execution|
|Multiple `merge()` operations|Used **pivot table** to consolidate data|🚀 Eliminated redundant merging|
|Unnecessary computations on empty data|Added **early exit** conditions|🚀 Avoids unnecessary processing|
|Row-wise operations for status|Applied **column-based** logic|🚀 Better performance on large datasets|
|Inefficient date formatting|Used **vectorized** `.strftime()`|🚀 Improved UI responsiveness|

---

### **Why These Changes Were Necessary**

- **Performance Gain**: Faster execution, especially on large datasets.
- **Memory Efficiency**: Avoids creating unnecessary DataFrames.
- **Scalability**: Handles increasing data volume efficiently.
- **Maintainability**: Cleaner and easier-to-read code.

These changes significantly improve the speed and efficiency of the function while maintaining the same output.

Let me know if you’d like me to add anything else! 🚀