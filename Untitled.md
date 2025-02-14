Below are a few suggestions for further optimizations beyond fixing the N+1 query in **get_status_table**:

1. **Use `select_related` in `handle_publish_now`:**  
    In the loop over samples, you access `s.track.form.name` for each sample. This can trigger extra queries per sample. You can optimize by fetching the related track and form in the initial query. For example:
    
    ```python
    samples_qs = AnnotationTrackSample.objects.select_related("track", "track__form").filter(
        track_id=track_id,
        status__in=[
            AnnotationTrackSample.Status.VALIDATED,
            AnnotationTrackSample.Status.PUBLISH_FAIL,
        ],
    )
    for s in samples_qs:
        if s.track.form.name == "Catalog Correction-based Form":
            # process as before
    ```
    
2. **Bulk Update Sample Statuses:**  
    After calling the external API, the code loops over the samples to update their status and calls `save()` on each instance. This issues one query per sample. If you don’t need per-object signals or post-save hooks, you can use a bulk update to update all matching samples at once:
    
    ```python
    from django.utils import timezone
    
    now = timezone.now()
    updated = AnnotationTrackSample.objects.filter(pk__in=sample_ids).update(
        status=AnnotationTrackSample.Status.PUBLISHED, updated_at=now
    )
    ```
    
    Similarly, in the failure branch, update all samples in one go.
    
3. **Refactor Python Loops in Pandas:**  
    In **get_status_table**, you build the `sample_data` list by iterating over `samples_qs` in a loop. You can make this a list comprehension for a small readability (and slight performance) boost:
    
    ```python
    sample_data = [
        {
            **a.data,
            "track_id": a.track.id,
            "track": a.track.name,
            "created_at": a.created_at,
            "annotated_at": a.annotated_at if a.annotated_by else None,
            "status": a.status,
        }
        for a in samples_qs
    ]
    ```
    
4. **Vectorize Percentage Calculations in Pandas:**  
    The code uses `.apply()` with a lambda to compute the percentage completion for each status. If your DataFrame is large, this row-wise operation can be slower than a vectorized operation. For example, instead of:
    
    ```python
    df2show[col_name] = df2show.apply(
        lambda row: round((row[status_key] / row["total_words"] * 100) if row["total_words"] else 0, 2),
        axis=1,
    )
    ```
    
    You might do something like:
    
    ```python
    if status_key in df2show.columns:
        df2show[col_name] = (
            (df2show[status_key] / df2show["total_words"] * 100)
            .fillna(0)
            .round(2)
        )
        df2show.drop(columns=[status_key], inplace=True)
    else:
        df2show[col_name] = 0.0
    ```
    
5. **Datetime Formatting:**  
    When formatting datetime columns, if the columns are already datetime types, you can use vectorized methods. For instance, instead of applying a lambda on each row:
    
    ```python
    df2show["last_insertion"] = df2show["last_insertion"].dt.strftime("%Y-%m-%d %H:%M:%S")
    df2show["last_annotation"] = df2show["last_annotation"].dt.strftime("%Y-%m-%d %H:%M:%S")
    ```
    
    This is both more concise and efficient.
    
6. **Optional – Use `get_object_or_404`:**  
    In your view’s `get()` and possibly in `publish_now`, you’re using `NMTModel.objects.get(pk=model_id)`. For better error handling (and potentially a slight performance benefit by reducing try/except boilerplate), consider using Django’s `get_object_or_404`.
    

Overall, these changes should help reduce the number of database queries and make your DataFrame operations more efficient, especially when dealing with large datasets.