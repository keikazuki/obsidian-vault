Below is a concise yet detailed description you can put into your GitHub PR to explain the reasoning and benefits behind these changes:

---

## Summary

This PR refactors the `get_annotation_data` function to improve performance and efficiency when loading annotation data. The primary objectives are to reduce the number of database queries and to leverage pandas’ vectorized operations more effectively.

## Key Changes

1. **Reduced Database Queries**
    
    - Utilized `select_related` for `annotated_by` and `validated_by` to avoid the “N+1” query problem.
    - Switched to using `.values(...)` and `pd.DataFrame.from_records(...)` to fetch the necessary fields in a single query and construct the DataFrame in one pass.
2. **Vectorized Data Handling in Pandas**
    
    - Expanded JSON data with `pd.Series` instead of iterating in Python.
    - Computed word counts, pending statuses, and aggregated statistics (e.g., per-annotator summaries) using pandas’ built-in vectorized methods (`.str.split().str.len()`, `.groupby().agg()`, etc.).
3. **Clean Column Management**
    
    - Dynamically added missing columns with default values and ensured consistent column ordering.
    - Reliably converted date/time columns to string format only once via `pd.to_datetime(...).dt.strftime(...)`.
4. **Improved Code Readability**
    
    - Removed nested loops and manual expansions in favor of direct DataFrame operations.
    - Centralized logic for pending text calculations and aggregator functions.

## Performance Benefits

- **Fewer DB hits**: `select_related` dramatically cuts down the number of queries needed for related `annotated_by` and `validated_by` fields.
- **Faster DataFrame Construction**: Moving from a list comprehension that calls object attributes repeatedly to `.values(...)` → `from_records(...)` eliminates Python overhead.
- **Leveraging Pandas Vectorization**: Aggregations (word counts, group stats) now happen at the C level, which is faster than Python loops.

## Potential Considerations

- **Large JSON Fields**: If the `data` field is extremely large or deeply nested, expanding it with `pd.Series` might still be expensive in memory and runtime.
- **Dataset Size**: Very large tables may require additional optimizations or a chunked approach to avoid exceeding memory limits.
- **Data Irregularities**: By default, the code converts invalid/missing data to empty dictionaries and coerces date/time fields, which is generally robust but can mask input anomalies.

Overall, these changes should lead to a noticeable reduction in page load times and resource usage when rendering large annotation datasets.

---

Feel free to adjust or reword any parts to match the style of your project’s documentation or team guidelines.