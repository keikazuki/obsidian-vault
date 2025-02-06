Below is one example of how you might “refactor” the view so that it only hits the database once for the samples and then uses vectorized Pandas operations (and a pivot‐table) rather than repeating almost the same code five times. (Be sure to test thoroughly on your data before deploying.) In our version we:

1. Use `select_related` to prefetch each sample’s related track and annotator.
2. Build one list of dictionaries (adding in the JSONField data plus extra fields including the sample’s status) rather than doing separate queries.
3. Create one DataFrame from all samples and compute the word counts using the vectorized `.str.split().str.len()`.
4. Group once for the “all” aggregation and then use a pivot table to “spread” each status’ word counts.
5. Loop over the status names to compute percentage completions and then determine an overall status.

The overall functionality (grouping by the high‐level keys, computing the completion percentages, setting the “publish action” and building a plot summary) remains the same.

Below is the complete optimized code:

---

```python
def get_status_table(request, model_id):
    # Get the NMT model and its annotation tracks
    nmt_model = NMTModel.objects.get(pk=model_id)
    tracks = AnnotationTrack.objects.filter(model=nmt_model)
    col_names = []
    rows = []
    
    # If there are no tracks, return an empty table
    if not tracks.exists():
        return render(
            request,
            "includes/status_table.html",
            {
                "col_names": col_names,
                "rows": rows,
                "hl_progress": {},
                "hl_progress_title": "",
                "nmt_model": nmt_model,
            },
        )

    # Get the high-level key field names from the form attached to the first track.
    l2_fields = list(
        AnnotationFormField.objects.filter(
            form=tracks.first().form, is_high_level_key=True
        ).values_list("name", flat=True)
    )

    # Fetch all samples for the model in one query. Use select_related to reduce extra queries.
    samples_qs = AnnotationTrackSample.objects.select_related("track", "annotated_by").filter(
        track__model=nmt_model
    )

    # Build a list of dictionaries. Note that we include the sample’s status.
    sample_data = []
    for a in samples_qs:
        # Start with the JSON data, then add extra fields.
        data_row = dict(a.data)
        data_row.update({
            "track_id": a.track.id,
            "track": a.track.name,
            "created_at": a.created_at,
            "annotated_at": a.annotated_at if a.annotated_by else None,
            "status": a.status,
        })
        sample_data.append(data_row)

    # If there are no sample records, return an empty table.
    if not sample_data:
        return render(
            request,
            "includes/status_table.html",
            {
                "col_names": col_names,
                "rows": rows,
                "hl_progress": {},
                "hl_progress_title": "",
                "nmt_model": nmt_model,
            },
        )

    # Create a DataFrame from the list of sample dictionaries.
    all_df = pd.DataFrame(sample_data)

    # Compute word counts using the vectorized string methods.
    # (Assumes every row has a "text to translate" field.)
    all_df["word_count"] = all_df["text to translate"].str.split().str.len()

    # Define the grouping columns – these are the high-level fields plus the track info.
    groupby_cols = l2_fields + ["track", "track_id"]

    # Aggregate overall data per group (total words, total texts, and the most recent datetimes)
    agg_all = all_df.groupby(groupby_cols).agg(
        total_words=("word_count", "sum"),
        total_texts=("text to translate", "count"),
        last_insertion=("created_at", "max"),
        last_annotation=("annotated_at", "max"),
    ).reset_index()

    # Get the total word count per group per status via a pivot table.
    # (This replaces the repeated DataFrame creations and groupbys.)
    status_agg = (
        all_df.groupby(groupby_cols + ["status"])["word_count"]
        .sum()
        .unstack(fill_value=0)
        .reset_index()
    )

    # Merge the overall aggregation with the status-specific word counts.
    df2show = pd.merge(agg_all, status_agg, on=groupby_cols, how="left")

    # Define a mapping from the model’s status names to the names used in the UI.
    status_mapping = {
        "PENDING": "pending",
        "ANNOTATED": "annotated",
        "VALIDATED": "validated",
        "PUBLISHED": "published",
        "PUBLISH_FAIL": "publish_failed",
    }

    # For each status, calculate the completion percentage (if total_words is nonzero).
    for status_key, col_name in status_mapping.items():
        if status_key not in df2show.columns:
            # If there were no samples with this status, set completion to 0.
            df2show[col_name] = 0.0
        else:
            df2show[col_name] = df2show.apply(
                lambda row: round((row[status_key] / row["total_words"] * 100)
                                  if row["total_words"] else 0, 2),
                axis=1,
            )
        # Optionally, drop the raw status column if it exists.
        if status_key in df2show.columns:
            df2show.drop(columns=[status_key], inplace=True)

    # Determine an overall status for each group.
    def get_status(row):
        if row["publish_failed"] > 0:
            return "PUBLISH FAILED"
        if row["pending"] == 100:
            return "PENDING"
        if row["annotated"] == 100:
            return "ANNOTATED"
        if row["validated"] == 100:
            return "VALIDATED"
        if row["published"] == 100:
            return "PUBLISHED"
        return "WIP"

    df2show["status"] = df2show.apply(get_status, axis=1)

    # Determine if the publish button should be shown.
    def show_publish_button(row):
        if row["status"] in ["PUBLISH FAILED", "VALIDATED"]:
            # Build a semicolon-delimited string using the high-level field values and track_id.
            publish_data = "publishaction;" + ";".join(
                str(row.get(f, "")) for f in l2_fields
            ) + ";" + str(row["track_id"])
            return publish_data
        return ""

    df2show["publish action"] = df2show.apply(show_publish_button, axis=1)

    # Format the datetime columns for display.
    df2show["last_insertion"] = df2show["last_insertion"].apply(
        lambda ts: ts.strftime("%Y-%m-%d %H:%M:%S") if pd.notnull(ts) else ""
    )
    df2show["last_annotation"] = df2show["last_annotation"].apply(
        lambda ts: ts.strftime("%Y-%m-%d %H:%M:%S") if pd.notnull(ts) else ""
    )

    # Select only the columns we need to show in the table.
    display_cols = groupby_cols + ["last_insertion", "last_annotation", "status", "publish action"]
    df2show = df2show[display_cols]

    # Prepare the data for plotting.
    # We use only rows with one of the following statuses.
    df2plot = df2show[df2show["status"].isin(
        ["ANNOTATED", "VALIDATED", "PUBLISHED", "PUBLISH FAILED"]
    )].copy()
    # Convert the last_annotation column back to datetime for grouping.
    df2plot["last_annotation"] = pd.to_datetime(df2plot["last_annotation"], errors="coerce")
    df2plot = df2plot.dropna(subset=["last_annotation"])
    if not df2plot.empty:
        # Group by month-end frequency.
        df2plot_grouped = (
            df2plot.groupby(pd.Grouper(key="last_annotation", freq="ME"))
            .size()
            .reset_index(name="count")
        )
        df2plot_grouped["last_annotation"] = df2plot_grouped["last_annotation"].dt.strftime("%Y %b")
        df2plot_grouped["cumulative_sum"] = df2plot_grouped["count"].cumsum()
        hl_progress = df2plot_grouped.set_index("last_annotation").to_dict(orient="index")
    else:
        hl_progress = {}

    # Prepare column names and row data for the HTML table.
    col_names = ["Index"] + list(df2show.columns)
    if "track_id" in col_names:
        col_names.remove("track_id")
    # Also drop track_id from the actual data.
    if "track_id" in df2show.columns:
        df2show = df2show.drop(columns=["track_id"])
    rows = df2show.to_records(index=False).tolist()

    return render(
        request,
        "includes/status_table.html",
        {
            "col_names": col_names,
            "rows": rows,
            "hl_progress": hl_progress,
            "hl_progress_title": "",
            "nmt_model": nmt_model,
        },
    )
```

---

### A Few Notes on This Version

- **Database hits minimized:** We fetch all samples in one query (using `select_related`) and then split the “status” counts in memory.
- **Vectorized string operations:** Instead of calling a Python lambda on every row (as in your original multiple `.apply(len)` calls), we use the faster `str.split().str.len()`.
- **Pivot table instead of repeated merges:** The status‐by‐group aggregation is done in one pass using a pivot table.
- **Functionality preserved:** The grouping by your “high level” fields, the computation of completion percentages, the “get_status” logic, and the final render context remain the same.

Test this optimized version on your staging data to be sure it works as expected.