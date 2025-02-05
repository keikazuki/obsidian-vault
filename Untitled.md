Below is one example of how you can rework your view so that it minimizes the number of database queries and “vectorizes” as many operations as possible. The idea is to:

1. **Make a single query** that returns all the fields you need (including the JSON data, status, related track info, and dates).
2. **Build one DataFrame** from that query (instead of separate DataFrames for each status).
3. **Pivot or group the data in pandas** so you can compute sums and percentages in a vectorized (column‐wise) way instead of repeatedly merging and using per‑row apply calls.
4. **Use vectorized functions (or np.select)** to compute your “status” and “publish action” columns.

Below is one refactored version of your function. (Make sure you test it on your data because small details—such as the JSON field structure and the names of your columns—might require some adjustments.)

---

```python
import numpy as np
import pandas as pd
from django.shortcuts import render

def get_status_table(request, model_id):
    # Get the model and its annotation tracks
    nmt_model = NMTModel.objects.get(pk=model_id)
    tracks = AnnotationTrack.objects.filter(model=nmt_model)
    if not tracks.exists():
        # Render an empty table if there are no tracks
        return render(request, "includes/status_table.html", {
            "col_names": [],
            "rows": [],
            "hl_progress": {},
            "hl_progress_title": "",
            "nmt_model": nmt_model,
        })

    # Get the high-level fields (l2_fields)
    l2_fields = list(
        AnnotationFormField.objects.filter(
            form=tracks.first().form, 
            is_high_level_key=True
        ).values_list("name", flat=True)
    )

    # ─── STEP 1: ONE QUERY FOR ALL SAMPLES ──────────────────────────────
    # Instead of running a separate query for each status,
    # fetch all samples along with the fields you need.
    qs = AnnotationTrackSample.objects.filter(track__model=nmt_model).select_related("track").values(
        "data", 
        "track_id", 
        "track__name", 
        "created_at", 
        "annotated_at", 
        "annotated_by", 
        "status"
    )
    
    # Build one list of dictionaries. Note that each sample’s JSON data
    # (stored in the "data" field) is merged with additional fields.
    data_rows = []
    for sample in qs:
        # If your JSONField is already a dict, you can merge it directly.
        # (Otherwise, you may need to json.loads(sample["data"]) first.)
        row = sample["data"].copy()  # make sure to copy so you don't modify the original
        row.update({
            "track_id": sample["track_id"],
            "track": sample["track__name"],
            "created_at": sample["created_at"],
            # Only include annotated_at if there is an annotator
            "annotated_at": sample["annotated_at"] if sample["annotated_by"] else None,
            "status": sample["status"],
        })
        data_rows.append(row)

    # Create one DataFrame with all samples
    all_df = pd.DataFrame(data=data_rows)
    
    # ─── STEP 2: COMPUTE BASIC COLUMNS ────────────────────────────────────
    # Ensure the column "text to translate" exists.
    if "text to translate" not in all_df.columns:
        all_df["text to translate"] = ""
    
    # Compute word count in a vectorized way (count non‐whitespace groups)
    all_df["word_count"] = all_df["text to translate"].str.count(r"\S+")
    
    # Make sure date columns are datetime objects
    all_df["created_at"] = pd.to_datetime(all_df["created_at"], errors="coerce")
    all_df["annotated_at"] = pd.to_datetime(all_df["annotated_at"], errors="coerce")
    
    # Define the columns on which you want to group your data
    groupby_on = l2_fields + ["track", "track_id"]

    # ─── STEP 3: AGGREGATE ALL DATA AT ONCE ─────────────────────────────
    # Compute the overall aggregates for each group
    agg_main = all_df.groupby(groupby_on).agg(
        total_words=("word_count", "sum"),
        total_texts=("text to translate", "count"),
        last_insertion=("created_at", "max"),
        last_annotation=("annotated_at", "max")
    ).reset_index()
    
    # Pivot the DataFrame so that for each status we have the sum of word_count.
    # (This replaces your separate pending_df, annotated_df, etc.)
    pivot = all_df.pivot_table(
        index=groupby_on,
        columns="status",
        values="word_count",
        aggfunc="sum",
        fill_value=0
    ).reset_index()
    
    # Merge the overall aggregates with the pivot table data.
    df2show = pd.merge(agg_main, pivot, on=groupby_on, how="left")
    
    # ─── STEP 4: COMPUTE PERCENTAGES FOR EACH STATUS ─────────────────────
    # Define a vectorized helper to compute percentage (avoid divide-by-zero)
    def calc_percentage(col_name):
        # If the status column isn’t present in the merged frame, return 0.
        if col_name not in df2show.columns:
            return 0
        # Compute percentage: (status_words / total_words * 100)
        return np.where(
            df2show["total_words"] > 0,
            np.round(100 * df2show[col_name] / df2show["total_words"], 2),
            0
        )
    
    # Compute the percentages for the statuses you care about.
    # Note that in your original code the status names were:
    # PENDING, ANNOTATED, VALIDATED, PUBLISHED, and PUBLISH_FAIL.
    df2show["pending"] = calc_percentage("PENDING")
    df2show["annotated"] = calc_percentage("ANNOTATED")
    df2show["validated"] = calc_percentage("VALIDATED")
    df2show["published"] = calc_percentage("PUBLISHED")
    df2show["publish_failed"] = calc_percentage("PUBLISH_FAIL")
    
    # ─── STEP 5: DETERMINE THE GROUP STATUS ──────────────────────────────
    # Use np.select for a vectorized assignment of the final status
    conditions = [
        df2show["publish_failed"] > 0,
        df2show["pending"] == 100,
        df2show["annotated"] == 100,
        df2show["validated"] == 100,
        df2show["published"] == 100,
    ]
    choices = ["PUBLISH FAILED", "PENDING", "ANNOTATED", "VALIDATED", "PUBLISHED"]
    df2show["status"] = np.select(conditions, choices, default="WIP")
    
    # ─── STEP 6: CREATE THE "PUBLISH ACTION" COLUMN ───────────────────────
    # For rows with status "PUBLISH FAILED" or "VALIDATED", create the publish action string.
    df2show["publish action"] = ""
    mask = df2show["status"].isin(["PUBLISH FAILED", "VALIDATED"])
    if mask.any():
        # For these rows, join the values of all high-level key columns (l2_fields)
        df2show.loc[mask, "publish action"] = (
            "publishaction;" +
            df2show.loc[mask, l2_fields].astype(str).agg(";".join, axis=1) +
            ";" +
            df2show.loc[mask, "track_id"].astype(str)
        )
    
    # ─── STEP 7: FINAL FORMATTING FOR DISPLAY ──────────────────────────────
    # Format the date columns as strings
    df2show["last_insertion"] = pd.to_datetime(df2show["last_insertion"]).dt.strftime("%Y-%m-%d %H:%M:%S")
    df2show["last_annotation"] = pd.to_datetime(df2show["last_annotation"]).dt.strftime("%Y-%m-%d %H:%M:%S")
    
    # Choose the final columns (here we keep the groupby fields and the calculated ones)
    final_cols = groupby_on + ["last_insertion", "last_annotation", "status", "publish action"]
    df2show_final = df2show[final_cols].copy()

    # Prepare data for the progress plot.
    # Filter rows whose status is in the set used for plotting.
    df2plot = df2show_final[df2show_final["status"].isin(["ANNOTATED", "VALIDATED", "PUBLISHED", "PUBLISH FAILED"])].copy()
    # Convert last_annotation back to a datetime so we can group by month
    df2plot["last_annotation_dt"] = pd.to_datetime(df2plot["last_annotation"], errors="coerce")
    # Group by month (using a monthly frequency) and count the rows per month.
    df2plot_grouped = df2plot.groupby(pd.Grouper(key="last_annotation_dt", freq="M")).size().reset_index(name="count")
    df2plot_grouped["last_annotation"] = df2plot_grouped["last_annotation_dt"].dt.strftime("%Y %b")
    df2plot_grouped["cumulative_sum"] = df2plot_grouped["count"].cumsum()

    # Prepare column names and rows for the template.
    # (If you want to hide the track_id column on the UI, remove it from the list.)
    col_names = ["Index"] + list(df2show_final.columns)
    if "track_id" in col_names:
        col_names.remove("track_id")
    # Convert the final DataFrame to a list of dictionaries.
    rows = df2show_final.drop(columns=["track_id"], errors="ignore").to_dict(orient="records")
    
    # Render the template with the calculated data.
    return render(
        request,
        "includes/status_table.html",
        {
            "col_names": col_names,
            "rows": rows,
            "hl_progress": df2plot_grouped.set_index("last_annotation").to_dict(orient="index"),
            "hl_progress_title": "",
            "nmt_model": nmt_model,
        }
    )
```

---

### Explanation

1. **Single Query & DataFrame Construction:**  
    The code uses one query (`qs`) with a `select_related` and `values` call so that all samples (regardless of status) are retrieved in one hit. In the loop, each sample’s JSON data is merged with extra fields (track info, dates, status). This avoids the six separate queries you had before.
    
2. **Vectorized Operations:**  
    Word counts are computed with a regular expression (which avoids a Python‑level split and apply) and grouping is done using pandas’ built‑in groupby and pivot functions.
    
3. **Aggregation & Pivot:**  
    Instead of separately grouping by status (e.g. pending_df, annotated_df, etc.) and then merging each one, we pivot the DataFrame so that each status becomes a column (with the sum of word counts). Then a simple (vectorized) division computes the percentage for each status.
    
4. **Determining “Status” and “Publish Action”:**  
    The final status of each group is computed with `np.select` (which is much faster than applying a custom function row‑by‑row). The “publish action” column is created only for the relevant rows, again in a vectorized way.
    
5. **Plot Data:**  
    Finally, the code groups by month (using a monthly frequency on the last_annotation date) to create the cumulative plot data.
    

This refactoring should greatly reduce the per‑row Python loops and repeated merges, thereby reducing the overall processing time. You may still need to tweak details (for example, ensuring that the JSON field is correctly handled) to match your exact schema, but this overall structure avoids many common performance pitfalls.

Feel free to ask further questions or clarifications!