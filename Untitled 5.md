Below is one way to improve the performance of the `get_annotation_data` function without changing its external behavior. The main ideas are to:

1. **Reduce Database Hits:**  
    Instead of iterating over model instances and then accessing related fields (which can trigger additional queries), use Django’s `select_related` and `values()` to fetch all needed data in one query.
    
2. **Leverage Vectorized Operations in Pandas:**  
    Replace per-row Python functions (like the use of `.apply(len)` and lambda functions for datetime formatting) with vectorized operations such as Pandas’ `.str.split().str.len()` and `.dt.strftime()`.
    

Below is the refactored code for `get_annotation_data`:

---

```python
from datetime import datetime
import html
import pandas as pd
from django.shortcuts import render
from django.contrib.auth.mixins import LoginRequiredMixin
from ..models import (
    AnnotationFormField,
    AnnotationTrack,
    AnnotationTrackSample,
)

class DataView(LoginRequiredMixin, TemplateView):
    template_name = "data.html"

    def get_annotation_data(request, track_id):
        # Fetch the track and its associated fields
        track = AnnotationTrack.objects.get(pk=track_id)
        fields_qs = AnnotationFormField.objects.filter(form=track.form).order_by("order_position")
        field_names = list(fields_qs.values_list("name", flat=True))
        l2_fields_qs = fields_qs.filter(is_high_level_key=True)
        l2_fields_names = list(l2_fields_qs.values_list("name", flat=True))

        # Use select_related and values() to get all needed data in one go
        samples_qs = AnnotationTrackSample.objects.filter(track=track) \
            .select_related("annotated_by", "validated_by") \
            .values(
                "data",
                "annotated_at",
                "validated_at",
                "status",
                "id",
                "annotated_by__username",
                "validated_by__username",
            )

        # Build the list of dictionaries by merging the JSON field with extra fields
        data_list = []
        for item in samples_qs:
            # Ensure that we work with a copy of the dictionary from the JSONField
            row = item["data"].copy() if isinstance(item["data"], dict) else {}
            row["annotator"] = item.get("annotated_by__username")
            row["annotated_at"] = item.get("annotated_at")
            row["validator"] = item.get("validated_by__username")
            row["validated_at"] = item.get("validated_at")
            row["status"] = item.get("status")
            row["sample_id"] = item.get("id")
            data_list.append(row)

        if data_list:
            df = pd.DataFrame(data_list)

            # Reorder columns: first the form field names, then the extra columns.
            required_cols = field_names + [
                "validation",
                "annotator",
                "annotated_at",
                "validator",
                "validated_at",
                "status",
                "sample_id",
            ]
            df = df[required_cols].reset_index(drop=True).fillna("")

            # Use a vectorized string method for word count calculation
            if "text to translate" in df.columns:
                df["word_count"] = df["text to translate"].str.split().str.len()
            else:
                df["word_count"] = 0

            # Format datetime columns using vectorized operations
            if "annotated_at" in df.columns:
                df["annotated_at"] = pd.to_datetime(df["annotated_at"], errors="coerce") \
                    .dt.strftime("%Y-%m-%d %H:%M:%S").fillna("")
            if "validated_at" in df.columns:
                df["validated_at"] = pd.to_datetime(df["validated_at"], errors="coerce") \
                    .dt.strftime("%Y-%m-%d %H:%M:%S").fillna("")

            # Group by annotator to calculate summary statistics
            annotated_data_per_annotator = (
                df.groupby("annotator")
                  .agg(
                      total_words=("word_count", "sum"),
                      total_texts=("text to translate", "count"),
                      last_annotation=("annotated_at", "max"),
                  )
                  .to_dict(orient="index")
            )

            context = {
                "track_id": html.escape(str(track.id)),
                "track_next_deadline": track.next_deadline,
                "col_names": list(df.columns),
                "rows": df.to_records(index=False).tolist(),
                "total_texts": len(df),
                "total_words": int(df["word_count"].sum()),
                "pending_texts": int(
                    df[df["status"].isin([
                        AnnotationTrackSample.Status.PENDING,
                        AnnotationTrackSample.Status.LOADED,
                    ])].shape[0]
                ),
                "pending_words": int(
                    df[df["status"].isin([
                        AnnotationTrackSample.Status.PENDING,
                        AnnotationTrackSample.Status.LOADED,
                    ])]["word_count"].sum()
                ),
                "total_hl_keys": int(df.drop_duplicates(subset=l2_fields_names).shape[0]),
                "hl_keys": ", ".join(l2_fields_names),
                "data_per_annotator": annotated_data_per_annotator,
            }
            return render(request, "includes/data_table.html", context)
        else:
            # No data: return an empty table context
            context = {
                "total_texts": 0,
                "total_words": 0,
                "pending_texts": 0,
                "pending_words": 0,
                "total_hl_keys": 0,
                "hl_keys": ", ".join(l2_fields_names),
            }
            return render(request, "includes/data_table.html", context)
```

---

### Explanation of the Changes

1. **Queryset Optimization:**
    
    - We now use:
        
        ```python
        samples_qs = AnnotationTrackSample.objects.filter(track=track) \
            .select_related("annotated_by", "validated_by") \
            .values("data", "annotated_at", "validated_at", "status", "id", "annotated_by__username", "validated_by__username")
        ```
        
        This retrieves all the fields (including related user fields) in one query. Using `select_related` avoids additional queries when accessing `annotated_by` and `validated_by`.
2. **Flattening the Data Efficiently:**
    
    - Instead of iterating over model instances and merging dictionaries (which can be slower), we build a list of dictionaries from the queryset’s values. This list is then passed to Pandas to create a DataFrame.
3. **Vectorized Pandas Operations:**
    
    - The word count is now computed with:
        
        ```python
        df["word_count"] = df["text to translate"].str.split().str.len()
        ```
        
        which is faster than applying a Python function row‑by‑row.
    - Similarly, instead of using `apply` with a lambda for date formatting, we convert the columns with:
        
        ```python
        df["annotated_at"] = pd.to_datetime(df["annotated_at"], errors="coerce").dt.strftime("%Y-%m-%d %H:%M:%S").fillna("")
        ```
        
        This leverages Pandas’ datetime capabilities for faster processing.
4. **Preserving the Existing Functionality:**
    
    - The final context passed to the template remains unchanged. All summary statistics (total words, pending texts/words, etc.) are calculated in the same way as before.

By applying these changes, the function should process the annotation data more efficiently, particularly when the number of samples is large.

Feel free to adjust further (for example, by adding pagination if the dataset grows very large) while keeping the overall functionality intact.

Below is an updated version of the `get_annotation_data` function where the summary statistics are computed more efficiently. In this version, we pre‑compute a boolean mask for pending statuses (instead of filtering the DataFrame repeatedly) and use that mask to compute both the pending texts count and the pending words sum. We also isolate the columns needed for counting unique high‑level keys so that we don’t carry the entire DataFrame unnecessarily when calculating that value.

Below is the optimized code snippet:

---

```python
from datetime import datetime
import html
import pandas as pd
from django.shortcuts import render
from django.contrib.auth.mixins import LoginRequiredMixin
from ..models import (
    AnnotationFormField,
    AnnotationTrack,
    AnnotationTrackSample,
)

class DataView(LoginRequiredMixin, TemplateView):
    template_name = "data.html"

    def get_annotation_data(request, track_id):
        # Retrieve the track and its related form fields
        track = AnnotationTrack.objects.get(pk=track_id)
        fields_qs = AnnotationFormField.objects.filter(form=track.form).order_by("order_position")
        field_names = list(fields_qs.values_list("name", flat=True))
        l2_fields_qs = fields_qs.filter(is_high_level_key=True)
        l2_fields_names = list(l2_fields_qs.values_list("name", flat=True))

        # Use select_related and values() to pull all needed fields in a single query
        samples_qs = AnnotationTrackSample.objects.filter(track=track) \
            .select_related("annotated_by", "validated_by") \
            .values(
                "data",
                "annotated_at",
                "validated_at",
                "status",
                "id",
                "annotated_by__username",
                "validated_by__username",
            )

        # Build a list of dictionaries combining the JSONField data with additional fields.
        data_list = []
        for item in samples_qs:
            row = item["data"].copy() if isinstance(item["data"], dict) else {}
            row["annotator"] = item.get("annotated_by__username")
            row["annotated_at"] = item.get("annotated_at")
            row["validator"] = item.get("validated_by__username")
            row["validated_at"] = item.get("validated_at")
            row["status"] = item.get("status")
            row["sample_id"] = item.get("id")
            data_list.append(row)

        if data_list:
            df = pd.DataFrame(data_list)

            # Reorder and fill missing values
            required_cols = field_names + [
                "validation",
                "annotator",
                "annotated_at",
                "validator",
                "validated_at",
                "status",
                "sample_id",
            ]
            df = df[required_cols].reset_index(drop=True).fillna("")

            # Compute word counts using vectorized operations
            if "text to translate" in df.columns:
                df["word_count"] = df["text to translate"].str.split().str.len().fillna(0)
            else:
                df["word_count"] = 0

            # Vectorized datetime formatting for annotated_at and validated_at
            if "annotated_at" in df.columns:
                df["annotated_at"] = pd.to_datetime(df["annotated_at"], errors="coerce") \
                    .dt.strftime("%Y-%m-%d %H:%M:%S").fillna("")
            if "validated_at" in df.columns:
                df["validated_at"] = pd.to_datetime(df["validated_at"], errors="coerce") \
                    .dt.strftime("%Y-%m-%d %H:%M:%S").fillna("")

            # --- Optimized Summary Statistics Calculations ---

            # Pre-calculate pending statuses using a boolean mask (only one DataFrame filtering)
            pending_mask = df["status"].isin([
                AnnotationTrackSample.Status.PENDING,
                AnnotationTrackSample.Status.LOADED,
            ])
            pending_texts = int(pending_mask.sum())
            pending_words = int(df.loc[pending_mask, "word_count"].sum())

            # Total texts and words
            total_texts = len(df)
            total_words = int(df["word_count"].sum())

            # Calculate the number of unique high-level keys using only the relevant columns
            total_hl_keys = int(df[l2_fields_names].drop_duplicates().shape[0]) if l2_fields_names else 0

            # Group by annotator for per-annotator summary statistics in one operation
            annotated_data_per_annotator = (
                df.groupby("annotator")
                  .agg(
                      total_words=("word_count", "sum"),
                      total_texts=("text to translate", "count"),
                      last_annotation=("annotated_at", "max"),
                  )
                  .to_dict(orient="index")
            )

            context = {
                "track_id": html.escape(str(track.id)),
                "track_next_deadline": track.next_deadline,
                "col_names": list(df.columns),
                "rows": df.to_records(index=False).tolist(),
                "total_texts": total_texts,
                "total_words": total_words,
                "pending_texts": pending_texts,
                "pending_words": pending_words,
                "total_hl_keys": total_hl_keys,
                "hl_keys": ", ".join(l2_fields_names),
                "data_per_annotator": annotated_data_per_annotator,
            }
            return render(request, "includes/data_table.html", context)
        else:
            context = {
                "total_texts": 0,
                "total_words": 0,
                "pending_texts": 0,
                "pending_words": 0,
                "total_hl_keys": 0,
                "hl_keys": ", ".join(l2_fields_names),
            }
            return render(request, "includes/data_table.html", context)
```

---

### Explanation of the Optimizations

1. **Pre-Computed Pending Mask:**  
    Instead of filtering the DataFrame twice for pending texts and pending words, we create a single boolean mask:
    
    ```python
    pending_mask = df["status"].isin([AnnotationTrackSample.Status.PENDING, AnnotationTrackSample.Status.LOADED])
    ```
    
    Then we use this mask for both:
    
    - `pending_texts = int(pending_mask.sum())`
    - `pending_words = int(df.loc[pending_mask, "word_count"].sum())`
2. **Selective Column Duplicate Removal:**  
    When calculating the number of unique high‑level keys, only the columns in `l2_fields_names` are passed to the `drop_duplicates` method. This avoids unnecessary processing on columns that are not relevant:
    
    ```python
    total_hl_keys = int(df[l2_fields_names].drop_duplicates().shape[0]) if l2_fields_names else 0
    ```
    
3. **Single Groupby Operation:**  
    The per‑annotator summary is calculated using a single `groupby` call with multiple aggregations, which is both vectorized and efficient.
    

These changes help reduce the number of passes over the data and avoid redundant filtering, which should lead to a noticeable performance improvement when processing large datasets.

