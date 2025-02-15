
in annotation_status_view.py
-. **Use `select_related` in `handle_publish_now`:**  
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
    
in data_view.py
- Below are several suggestion qs and minor code tweaks that could further optimize the remaining functions in **data_view.py**:

---

### 1. Optimize `get_add_data_form`

- **Use `select_related` for the Track’s Form:**  
    When fetching the track, you later access `track.form.name`. You can reduce a join by doing:
    
    ```python
    track = AnnotationTrack.objects.select_related("form").get(pk=track_id)
    ```
    
- **Minimize Repeated Conversions:**  
    You’re converting `track.id` to a string (and escaping it) multiple times. Save it in a variable for clarity and reuse:
    
    ```python
    track_id_str = html.escape(str(track.id))
    ```
    
    Then use `track_id_str` for both the prefix and context.
    

---

### 2. Optimize the `get` Method

- **Use `get_object_or_404`:**  
    Instead of `NMTModel.objects.get(pk=model_id)`, you might consider using Django’s `get_object_or_404` for more robust error handling.
    
- **Preload Related Data on Tracks:**  
    If the template uses track details like the associated form or model, you can reduce additional queries by selecting related fields:
    
    ```python
    nmt_model = get_object_or_404(NMTModel, pk=model_id)
    tracks = (
        AnnotationTrack.objects.filter(model=nmt_model, form__form_type=form_type)
        .select_related("form", "model")
        .order_by("priority")
    )
    ```
    

---

### 3. Optimize `add_data`

- **Fetch the Track with Related Data:**  
    Since you access both `track.form` and `track.model` multiple times, fetch them upfront:
    
    ```python
    track = AnnotationTrack.objects.select_related("form", "model").get(pk=track_id)
    ```
    
- **Cache Repeated Query Results for Form Fields:**  
    In conditions where you check for fields like `"product id"`, `"gtin"`, or `"fields to correct"`, avoid multiple filtering calls. For example:
    
    ```python
    product_field = l2_fields.filter(name="product id").first()
    gtin_field = l2_fields.filter(name="gtin").first()
    fields_to_correct_field = l2_fields.filter(name="fields to correct").first()
    
    if product_field and gtin_field and fields_to_correct_field:
        wpids = form.cleaned_data[f"field_{product_field.id}"].splitlines()
        gtins = form.cleaned_data[f"field_{gtin_field.id}"].splitlines()
        fields_to_correct = form.cleaned_data[f"field_{fields_to_correct_field.id}"].splitlines()
        incoming_df = prepare_incoming_df(wpids, gtins, fields_to_correct)
    ```
    
    Similarly, for the `"seed queries"` check, retrieve the field once instead of calling `l2_fields.filter(name='seed queries').first()` repeatedly.
    
- **Simplify Existing Samples Query:**  
    Instead of filtering by `track__model=track.model, track__name=track.name`, you can filter directly by the track:
    
    ```python
    existing_samples = AnnotationTrackSample.objects.filter(track=track).values("data", "id")
    existing_df = pd.DataFrame(
        [
            {**sample["data"], "sample_id": sample["id"]}
            for sample in existing_samples
        ]
    ).drop_duplicates(primary_fields)
    ```
    
    Using `.values(...)` avoids instantiating full model objects if you only need the data.
    
- **Consider Refactoring Long Conditional Logic:**  
    The multiple `elif` branches (for different track types) can be refactored into a dispatch table or helper functions. This isn’t a performance change per se but will improve maintainability.
    

---

### 4. Optimize `validate_sample`

- **Check User Permission Efficiently:**  
    Instead of loading all annotators to check membership, use a query to see if the current user is in the set:
    
    ```python
    if sample.track.annotators.filter(pk=request.user.pk).exists() or request.user.is_superuser:
    ```
    
    This avoids building a list of all annotators, which can be especially useful if the list grows large.
    
- **Use Timezone-Aware Datetimes:**  
    If your project uses time zones (and it’s generally a good practice), use Django’s timezone utilities:
    
    ```python
    from django.utils import timezone
    # ...
    sample.validated_at = timezone.now()
    ```
    

---

### Summary

While the heavy-lifting (like fixing the N+1 problem in `get_annotation_data`) is already done, these improvements can help:

- Reduce repeated queries by using `select_related` and caching query results.
- Simplify and vectorize DataFrame operations where possible.
- Use more efficient permission checks and standard Django utilities (like `get_object_or_404` and `timezone.now()`).

Implementing these suggestions should help reduce overhead and improve both the performance and maintainability of your views.

Below is a detailed explanation of potential optimizations in your **view.py** code—both in the two specific functions you mentioned and some broader suggestions for faster page loading.

---

## 1. Optimizing Specific Views

### A. **Index View**

#### **Observations & Potential Tweaks:**

1. **Timezone Consistency:**
    
    - **Current:** You use `datetime.now()` and then convert to an aware datetime with `make_aware()`.
    - **Optimization:** Use Django’s `timezone.now()` directly. This ensures your datetimes are already aware and aligns with Django’s timezone settings.
        
        ```python
        from django.utils import timezone
        now = timezone.now()
        ```
        
2. **Avoid Redundant COUNT Queries:**
    
    - You calculate `len(annotated_data)`, `len(last_annotated_data)`, etc. Each call to `len(queryset)` may trigger a separate COUNT query.
    - **Optimization:** Store counts in variables so that you don’t run the same query multiple times.
        
        ```python
        total_count = annotated_data.count()
        last_count = last_annotated_data.count()
        prev_count = prev_annotated_data.count()
        ```
        
    - Then compute percentages and deltas using these numbers.
3. **Simplify the Percentage Calculation:**
    
    - The current inline expression for `"last_annotation_percent"` is a bit hard to read. Once you have your counts, you can compute it in a clearer way:
        
        ```python
        if total_count:
            last_annotation_percent = int(last_count / total_count * 100)
        else:
            last_annotation_percent = 100 if last_count else 0
        ```
        

---

### B. **get_core_data View**

#### **Observations & Potential Tweaks:**

4. **Avoiding N+1 Queries When Fetching Related Data:**
    
    - In your list comprehension for each model, you do:
        
        ```python
        "annotation_forms": [
            {"id": f.id, "name": f.name}
            for f in m.annotationtrack_set.all()
        ]
        ```
        
    - **Optimization:** Use `prefetch_related` on your initial queryset so that Django loads the related `AnnotationTrack` objects in a single additional query instead of one query per model.
        
        ```python
        models = NMTModel.objects.prefetch_related("annotationtrack_set").all()
        ```
        
    - This change minimizes database hits and improves performance when iterating over each model’s annotation forms.
5. **Check for Metrics Query:**
    
    - Currently, you assign:
        
        ```python
        metrics = NMTModel.objects.all()
        ```
        
    - If your intent is to return “metrics” different from models, double-check whether you’re querying the correct model. If the same model is used for both, consider renaming or combining the two lists to reduce confusion.

---

### C. **get_users_performance View**

#### **Observations & Potential Tweaks:**

6. **Use Efficient Count/Existence Checks:**
    
    - Instead of calling `len(samples)` (which triggers a COUNT query) repeatedly or iterating over filtered querysets in loops, use:
        - `.count()` when you need the count.
        - `.exists()` for a boolean check.
    - For example, in the loop over tracks:
        
        ```python
        pending_count = track_samples.filter(status=AnnotationTrackSample.Status.PENDING).count()
        loaded_count = track_samples.filter(annotated_by=u, status=AnnotationTrackSample.Status.LOADED).count()
        if pending_count and loaded_count:
            current_tracks_names.append(track.name)
        ```
        
    - This can be more efficient than repeatedly converting querysets to lists or using `len()` on them.
7. **Minimize Repeated Queries per User and Track:**
    
    - You iterate over each user and then for each user you query `AnnotationTrack.objects.filter(annotators=u)`.
    - **Optimization:** Consider prefetching the related tracks or even annotating counts at the database level. For example, if you need to know the number of pending or loaded samples per track, you might use Django’s aggregation to do this in one query per user rather than one query per track.
    - Alternatively, if the data set isn’t huge, you could cache results of subqueries within the loop.
8. **Raw SQL for Word Count:**
    
    - You use a RawSQL expression to calculate the word count in the `"text to translate"` field. This can be computationally expensive if run on a large number of rows.
    - **Optimization Considerations:**
        - If the word count is used frequently, consider storing or caching the value in the database (for example, by adding a computed column or updating it via a signal).
        - Otherwise, ensure that your database can handle the calculation and that you’re not processing more rows than necessary.
9. **Sorting the Results:**
    
    - At the end, you sort the result dictionary based on the daily average. This is done in Python.
    - **Optimization:** If the number of users is small (as in your hard-coded list), this is acceptable. For larger sets, consider sorting at the database level using annotated fields.

---

## 2. Additional General Optimizations for Faster Page Loading

Beyond view-level query optimizations, consider these broader strategies:

10. **Caching:**
    
    - **Per-View or Template Fragment Caching:**  
        Use Django’s caching framework to cache expensive query results or rendered HTML fragments that change infrequently.
    - **Low-Level Cache API:**  
        Cache data that doesn’t change often (like a list of models with pre-fetched related objects).
11. **Database Indexing:**
    
    - Ensure that columns used frequently in filters (such as `status`, `annotated_at`, and foreign keys like `annotated_by`) are indexed. Django automatically indexes foreign keys, but you may need additional indexes for other fields.
12. **Reduce Redundant Calculations:**
    
    - When you calculate aggregate values (such as counts or averages), store them in local variables to avoid recalculating them multiple times.
13. **Prefetch/Select Related:**
    
    - Always consider using `select_related` and `prefetch_related` to fetch related data in fewer queries. This is especially useful in list comprehensions or loops that access related models.
14. **Asynchronous Tasks:**
    
    - For heavy operations (e.g., complex aggregations over large data sets or operations involving external API calls), consider offloading to asynchronous tasks using Celery or Django Q. This keeps the response time for web requests short.
15. **Pagination and Lazy Loading:**
    
    - If any view returns a large dataset to be rendered in a table or list, use pagination. This reduces the amount of data processed and sent to the client, speeding up the page load.
16. **Static File Optimization:**
    
    - Ensure your static assets (CSS, JavaScript, images) are optimized, minified, and served via a CDN if possible. Although this isn’t in your Python code, it has a significant impact on overall page load performance.

---

## Summary

- **Index View:**  
    Use `timezone.now()` and cache counts to avoid multiple COUNT queries.
    
- **get_core_data:**  
    Prefetch related objects (using `prefetch_related("annotationtrack_set")`) to avoid N+1 queries when iterating over related annotation forms.
    
- **get_users_performance:**  
    Replace repeated use of `len(queryset)` with `.count()` or `.exists()`, and consider aggregating counts at the database level to avoid nested loops issuing many queries.
    
- **General Optimizations:**  
    Use caching, proper indexing, and asynchronous processing when applicable. Also, consider pagination and front-end optimizations for a smoother user experience.
    

Implementing these adjustments should reduce database load, lower response times, and make your views more efficient overall.

Below is an **example** of how your `view.py` **could look** after incorporating the optimizations we discussed. This example attempts to keep the same functionality while demonstrating:

1. Using Django’s `timezone.now()` instead of `datetime.now()` + `make_aware()`.
2. Storing `.count()` results in local variables instead of using `len(queryset)`.
3. Using `prefetch_related` in `get_core_data` to avoid N+1 queries.
4. Replacing repeated calls to `len()` on a queryset with `.count()` or `.exists()`.
5. Slightly cleaner calculations for percentages and sets.

Feel free to adjust any variable names or additional logic to match your application’s exact needs and style.

---

```python
from datetime import timedelta

from django.shortcuts import render, redirect
from django.contrib.auth.views import LoginView
from admin_soft.forms import LoginForm
from django.http.response import JsonResponse
from django.contrib.auth.decorators import login_required
from django.views.decorators.csrf import csrf_exempt
from django.utils import timezone
from django.contrib.auth.models import User
from django.db.models import Count, Avg, Max, Min, Sum
from django.db.models.functions import TruncDay
from django.db.models.expressions import RawSQL
from django.db.models import IntegerField

from ..models import NMTModel, AnnotationTrackSample, AnnotationTrack


def index(request):
    # If user in wg_users group, redirect
    if request.user.groups.filter(name="wg_users").exists():
        return redirect("dashboard:white_glove_form")

    # Otherwise load annotated data
    annotated_data = AnnotationTrackSample.objects.filter(
        status__in=[
            AnnotationTrackSample.Status.ANNOTATED,
            AnnotationTrackSample.Status.VALIDATED,
            AnnotationTrackSample.Status.PUBLISHED,
        ]
    )

    # Use Django's timezone-aware now
    now = timezone.now()

    # Compute start/end of last week
    start_of_last_week = now - timedelta(days=now.weekday() + 7)
    end_of_last_week = start_of_last_week + timedelta(days=6)

    # Compute start/end of previous week
    start_of_prev_week = now - timedelta(days=now.weekday() + 14)
    end_of_prev_week = start_of_prev_week + timedelta(days=6)

    # Filter the relevant annotated_data
    last_annotated_data = annotated_data.filter(
        annotated_at__range=(start_of_last_week, end_of_last_week)
    )
    prev_annotated_data = annotated_data.filter(
        annotated_at__range=(start_of_prev_week, end_of_prev_week)
    )

    # Convert to set of user IDs (or user objects); no need to materialize entire rows
    last_active_users = set(last_annotated_data.values_list("annotated_by", flat=True))
    prev_active_users = set(prev_annotated_data.values_list("annotated_by", flat=True))

    # Store counts to avoid multiple queries
    total_count = annotated_data.count()
    last_count = last_annotated_data.count()

    if total_count:
        last_annotation_percent = int(last_count / total_count * 100)
    else:
        # If total_count is 0, fallback logic
        last_annotation_percent = 100 if last_count else 0

    return render(
        request,
        "index.html",
        {
            "segment": "index",
            "annotatated_data": total_count,
            "last_annotation_percent": last_annotation_percent,
            "active_users": last_active_users,
            # Simple difference in active users between two weeks
            "active_users_delta": len(last_active_users) - len(prev_active_users),
        },
    )


def profile(request):
    if request.user.groups.filter(name="wg_users").exists():
        return redirect("dashboard:white_glove_form")
    else:
        return render(request, "includes/profile.html", {"segment": "profile"})


@login_required
def get_current_user(request):
    user = request.user
    return JsonResponse({"username": user.username, "email": user.email})


# Authentication
class UserLoginView(LoginView):
    template_name = "accounts/login.html"
    form_class = LoginForm

    def get_success_url(self):
        # Change the redirect URL here
        return "/profile/"

    def get_context_data(self, **kwargs):
        context = super().get_context_data(**kwargs)
        # Retrieve the flag from the query parameters
        show_register_opt = (
            self.request.GET.get("show_register_opt", "false").lower() == "true"
        )
        context["show_register_opt"] = show_register_opt
        return context


@csrf_exempt
def get_core_data(request):
    # Use prefetch_related to avoid N+1 queries on annotationtrack_set
    models = NMTModel.objects.prefetch_related("annotationtrack_set").all()

    # Double-check if 'metrics' really should be the same queryset as 'models'
    metrics = NMTModel.objects.all()

    return JsonResponse(
        {
            "models": [
                {
                    "id": m.id,
                    "name": m.name,
                    "src_locale": m.src_locale,
                    "trg_locale": m.trg_locale,
                    "annotation_forms": [
                        {"id": track.id, "name": track.name}
                        for track in m.annotationtrack_set.all()
                    ],
                }
                for m in models
            ],
            # If metrics are truly also NMTModel, keep as is;
            # otherwise, change this to query the correct model/table.
            "metrics": [{"id": s.id, "name": s.name} for s in metrics],
        }
    )


@csrf_exempt
def get_users_performance(request):
    # Hardcoded user list
    user_list = [
        "DianaLiceaga",
        "sparvaiz",
        "JulioMadrid",
        "nicole.aguilera0@walmart.com",
        "EldaBenet",
        "TaniaAckerman",
    ]

    users = User.objects.filter(username__in=user_list)

    result = {}
    for u in users:
        # Samples annotated by this user
        samples_qs = AnnotationTrackSample.objects.filter(annotated_by=u)

        # Instead of len(samples_qs), use .count()
        total_samples = samples_qs.count()
        if total_samples > 0:
            current_tracks_names = []
            # Tracks that this user is assigned to
            user_tracks = AnnotationTrack.objects.filter(annotators=u)

            # For each track, check if user still has pending samples
            for track in user_tracks:
                track_samples = AnnotationTrackSample.objects.filter(track=track)
                # Use count(), not len()
                pending_count = track_samples.filter(
                    status=AnnotationTrackSample.Status.PENDING
                ).count()
                loaded_count = track_samples.filter(
                    annotated_by=u, status=AnnotationTrackSample.Status.LOADED
                ).count()

                if pending_count and loaded_count:
                    current_tracks_names.append(track.name)

            # Annotate each sample with a raw-sql-based word count
            samples_qs = samples_qs.annotate(
                text2translate_word_count=RawSQL(
                    """
                    LENGTH(Cast(data->>'text to translate' as TEXT))
                    - LENGTH(REPLACE(Cast(data->>'text to translate' as TEXT), ' ', ''))
                    + 1
                    """,
                    [],
                    output_field=IntegerField(),
                )
            )

            # Group by day of annotation
            row_each_day = (
                samples_qs.annotate(day=TruncDay("annotated_at"))
                .values("day")
                .annotate(text_count=Count("id"))
                .annotate(word_count=Sum("text2translate_word_count"))
            )

            # Aggregate day-level stats
            performance = row_each_day.aggregate(
                samples_avg=Avg("text_count"),
                samples_max=Max("text_count"),
                samples_min=Min("text_count"),
                words_avg=Avg("word_count"),
                words_max=Max("word_count"),
                words_min=Min("word_count"),
            )

            result[u.username] = {
                "all": total_samples,
                "daily_work": performance,
                "current_tracks": current_tracks_names,
            }

    # Sort the result dictionary by 'samples_avg' descending
    # If 'samples_avg' is None for some user, handle gracefully with a fallback
    def sort_key(item):
        username = item[0]
        data = item[1]
        daily_work = data["daily_work"]
        # If daily_work['samples_avg'] is None, default to 0
        return daily_work["samples_avg"] or 0

    sorted_result = dict(
        sorted(result.items(), key=sort_key, reverse=True)
    )

    return JsonResponse(sorted_result)
```

### Notable Changes

6. **Replaced `datetime.now()`** with `timezone.now()` in `index` view.
7. **Used `.count()`** to get the size of a queryset instead of `len(queryset)`.
8. **Stored counts** in variables so we only query once instead of multiple times.
9. **In `get_core_data`**, used `prefetch_related("annotationtrack_set")` to avoid N+1 queries.
10. **In `get_users_performance`**:
    - Changed `len(samples_qs)` to `samples_qs.count()`.
    - Used `.count()` in track checks (`pending_count` and `loaded_count`).
    - Added a simple `sort_key` function to handle cases where `samples_avg` might be `None`.
11. **Kept** the general structure and logic intact so it should work the same way as before.

---

This code example should serve as a good starting point if you want to check that everything is **“done correctly”** according to the previous suggestions. Of course, you can further **adjust, refactor, or clean up** based on your project’s specific requirements and coding style.