Below is a step-by-step guide (with sample code) on how you can split out the _chart_-related logic and HTML from the _table_-related logic and HTML to help your page load faster. The key ideas are:

1. **Create a new view** (e.g. `get_status_chart`) dedicated to **only** fetching the data needed for plotting the chart, and rendering a partial template (or returning JSON if that suits you better).
    
2. **Remove** the chart canvas and JS for the chart from the existing `status_table.html` and place them in a new partial template (e.g. `status_chart.html`).
    
3. **Modify** your main template (`annotation_status.html`) so that:
    
    - It has **two placeholders** (e.g., `<div>` elements) into which you can load:
        1. The table snippet (via `get_status_table`).
        2. The chart snippet (via `get_status_chart`).
    - Each is fetched via AJAX calls (or you can do `include` tags, but AJAX is often best for perceived faster loading).

Below is one possible restructuring. Adjust as needed for your project’s naming conventions or style.

---

## 1. Create a **new** view for chart data: `get_status_chart`

We will extract the high-level progress (cumulative annotation progress) logic from your `get_status_table` and put it into a new function: `get_status_chart`. This function can render an HTML snippet with the canvas and pass down the necessary variables for plotting (e.g., `hl_progress`).

```python
# annotation_status_view.py

import pandas as pd
import json
from django.http.response import JsonResponse
from django.shortcuts import render
from django.views.generic import TemplateView
from django.contrib.auth.mixins import LoginRequiredMixin
from django.views.decorators.csrf import csrf_exempt
import logging
import logging.config
from datetime import datetime

logging.config.fileConfig("logging.ini")
logger = logging.getLogger("csdLogger")

from ..models import (
    NMTModel,
    AnnotationFormField,
    AnnotationTrack,
    AnnotationTrackSample,
)
from dashboard.utils.cte_item_ops import get_cte_catalog_data, post_cte_catalog_data

class AnnotationStatusView(LoginRequiredMixin, TemplateView):
    template_name = "annotation_status.html"

    def get(self, request, model_id):
        nmt_model = NMTModel.objects.get(pk=model_id)
        return render(
            request,
            self.template_name,
            {
                "nmt_model": nmt_model,
            },
        )

    @csrf_exempt
    def publish_now(request, model_id):
        body = json.loads(request.body)
        logger.info(f"request body {body}")
        tokens= body["payload"].split(";")
        track_id = tokens[-1]
        try:
            handle_publish_now(track_id)
            return JsonResponse({"status": 200, "message": "Correction Published Successfully"})
        except Exception as error:
            return JsonResponse({"status": 400, "message": str(error)})


    def get_status_table(request, model_id):
        """
        Returns the HTML snippet for the status table only.
        """
        nmt_model = NMTModel.objects.get(pk=model_id)
        tracks = AnnotationTrack.objects.filter(model=nmt_model)
        col_names = []
        rows = []

        if not tracks.exists():
            return render(
                request,
                "includes/status_table.html",
                {
                    "col_names": col_names,
                    "rows": rows,
                    "nmt_model": nmt_model,
                    # We won't handle hl_progress here,
                    # because the chart is rendered separately now.
                    "hl_progress": {},
                },
            )

        # ... the same logic as before to produce df2show, except
        # the part that deals with df2plot & hl_progress is removed
        # or left out, since we'll handle that in get_status_chart.

        l2_fields = list(
            AnnotationFormField.objects.filter(
                form=tracks.first().form, is_high_level_key=True
            ).values_list("name", flat=True)
        )
        samples_qs = AnnotationTrackSample.objects.select_related(
            "track", "annotated_by"
        ).filter(track__model=nmt_model)

        sample_data = []
        for a in samples_qs:
            data_row = dict(a.data)
            data_row.update(
                {
                    "track_id": a.track.id,
                    "track": a.track.name,
                    "created_at": a.created_at,
                    "annotated_at": a.annotated_at if a.annotated_by else None,
                    "status": a.status,
                }
            )
            sample_data.append(data_row)

        if not sample_data:
            return render(
                request,
                "includes/status_table.html",
                {
                    "col_names": col_names,
                    "rows": rows,
                    "nmt_model": nmt_model,
                    "hl_progress": {},
                    "message": "No data to show",
                },
            )

        all_df = pd.DataFrame(sample_data)
        all_df["word_count"] = all_df["text to translate"].str.split().str.len()

        groupby_cols = l2_fields + ["track", "track_id"]

        agg_all = (
            all_df.groupby(groupby_cols)
            .agg(
                total_words=("word_count", "sum"),
                total_texts=("text to translate", "count"),
                last_insertion=("created_at", "max"),
                last_annotation=("annotated_at", "max"),
            )
            .reset_index()
        )

        status_agg = (
            all_df.groupby(groupby_cols + ["status"])["word_count"]
            .sum()
            .unstack(fill_value=0)
            .reset_index()
        )

        df2show = pd.merge(agg_all, status_agg, on=groupby_cols, how="left")

        status_mapping = {
            "PENDING": "pending",
            "ANNOTATED": "annotated",
            "VALIDATED": "validated",
            "PUBLISHED": "published",
            "PUBLISH_FAIL": "publish_failed",
        }
        for status_key, col_name in status_mapping.items():
            if status_key not in df2show.columns:
                df2show[col_name] = 0.0
            else:
                df2show[col_name] = df2show.apply(
                    lambda row: round(
                        (
                            (row[status_key] / row["total_words"] * 100)
                            if row["total_words"]
                            else 0
                        ),
                        2,
                    ),
                    axis=1,
                )
            if status_key in df2show.columns:
                df2show.drop(columns=[status_key], inplace=True)

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

        def show_publish_button(row):
            if row["status"] in ["PUBLISH FAILED", "VALIDATED"]:
                publish_data = (
                    "publishaction;"
                    + ";".join(str(row.get(f, "")) for f in l2_fields)
                    + ";"
                    + str(row["track_id"])
                )
                return publish_data
            return ""

        df2show["publish action"] = df2show.apply(show_publish_button, axis=1)
        df2show["last_insertion"] = df2show["last_insertion"].apply(
            lambda ts: ts.strftime("%Y-%m-%d %H:%M:%S") if pd.notnull(ts) else ""
        )
        df2show["last_annotation"] = df2show["last_annotation"].apply(
            lambda ts: ts.strftime("%Y-%m-%d %H:%M:%S") if pd.notnull(ts) else ""
        )

        display_cols = groupby_cols + [
            "last_insertion",
            "last_annotation",
            "status",
            "publish action",
        ]
        df2show = df2show[display_cols]

        # We'll no longer generate df2plot or hl_progress here.

        col_names = ["Index"] + list(df2show.columns)
        if "track_id" in col_names:
            col_names.remove("track_id")
        if "track_id" in df2show.columns:
            df2show = df2show.drop(columns=["track_id"])
        rows = df2show.to_records(index=True).tolist()

        return render(
            request,
            "includes/status_table.html",
            {
                "col_names": col_names,
                "rows": rows,
                "nmt_model": nmt_model,
                # Provide empty or no hl_progress since chart is separate
                "hl_progress": {},
            },
        )


    def get_status_chart(request, model_id):
        """
        Returns the HTML snippet (partial) for the chart only.
        """
        nmt_model = NMTModel.objects.get(pk=model_id)
        tracks = AnnotationTrack.objects.filter(model=nmt_model)

        # We only care about the portion that produces "hl_progress".
        hl_progress = {}

        if not tracks.exists():
            # If no tracks, no chart data
            return render(
                request,
                "includes/status_chart.html",
                {
                    "nmt_model": nmt_model,
                    "hl_progress": hl_progress,
                }
            )

        # Build the data needed for the chart. Similar to the old logic:
        samples_qs = AnnotationTrackSample.objects.select_related(
            "track", "annotated_by"
        ).filter(track__model=nmt_model)

        sample_data = []
        for a in samples_qs:
            data_row = dict(a.data)
            data_row.update(
                {
                    "track_id": a.track.id,
                    "track": a.track.name,
                    "created_at": a.created_at,
                    "annotated_at": a.annotated_at if a.annotated_by else None,
                    "status": a.status,
                }
            )
            sample_data.append(data_row)

        if not sample_data:
            return render(
                request,
                "includes/status_chart.html",
                {
                    "nmt_model": nmt_model,
                    "hl_progress": hl_progress,
                }
            )

        df = pd.DataFrame(sample_data)
        df["last_annotation"] = pd.to_datetime(df["annotated_at"], errors="coerce")

        # filter relevant statuses for the chart
        statuses_for_chart = ["ANNOTATED", "VALIDATED", "PUBLISHED", "PUBLISH_FAIL"]
        df = df[df["status"].isin(statuses_for_chart)].copy()
        df = df.dropna(subset=["last_annotation"])

        if not df.empty:
            df_grouped = (
                df.groupby(pd.Grouper(key="last_annotation", freq="ME"))
                .size()
                .reset_index(name="count")
            )
            df_grouped["last_annotation"] = df_grouped["last_annotation"].dt.strftime("%Y %b")
            df_grouped["cumulative_sum"] = df_grouped["count"].cumsum()

            # Convert to dict for usage in chart
            hl_progress = df_grouped.set_index("last_annotation").to_dict(orient="index")

        return render(
            request,
            "includes/status_chart.html",
            {
                "nmt_model": nmt_model,
                "hl_progress": hl_progress,
            },
        )


def handle_publish_now(track_id):
    # (Unmodified from your example)
    ...
```

---

## 2. Create a **new** partial template for the chart: `status_chart.html`

Move the chart’s canvas markup and script from `includes/status_table.html` into something like `includes/status_chart.html`. For example:

```html
{# includes/status_chart.html #}
{% if hl_progress|length > 0 %}
  <div class="chart">
    <canvas id="chart-line-hl-progress" class="chart-canvas" height="300"></canvas>
  </div>
{% else %}
  <p>There is no chart data to display.</p>
{% endif %}

<script>
  // This variable is used in your chart JS
  var hl_progress = {{ hl_progress|safe }};
</script>
```

> **Note:** If you have more JavaScript for the chart initialization, you can either:
> 
> - Put it directly in this partial.
> - Put it in a dedicated `.js` file and simply ensure you have the `hl_progress` object available.

---

## 3. Update your **existing** table partial `status_table.html` to remove the chart markup

Remove the `<canvas>` and chart-related scripts from `status_table.html`, since you’ll now be using `status_chart.html`. For example:

```html
{# includes/status_table.html #}
<div class="table-responsive p-3">
    <table id="data-table-{{nmt_model.id}}" class="table table-striped table-bordered" cellspacing="0" width="100%">
        <thead>
            <tr>
                {% for col_name in col_names %}
                <th class="text-uppercase text-secondary text-xxs font-weight-bolder opacity-7">
                    {{col_name}}
                </th>
                {% endfor %}
            </tr>
        </thead>
        <tbody>
            {% for row in rows %}
            <tr>
                {% for cel in row %}
                <td>
                    {% if "publishaction" in cel %}
                    <button
                        data-url="{% url 'dashboard:annotations_status_publish_now' model_id=nmt_model.id %}"
                        data="{{cel}}"
                        onclick="handlePublishAction(event)"
                        class="btn btn-primary btn-sm {{forloop.counter0}}"
                    >Publish Now</button>
                    {% else %}
                    <span class="text-secondary text-xs font-weight-bold">{{cel}}</span>
                    {% endif %}
                </td>
                {% endfor %}
            </tr>
            {% endfor %}
        </tbody>
    </table>
</div>

<div class="toast-holder position-fixed end-0 bottom-0 p-3"></div>
```

Notice we removed:

- The `<canvas>` element.
- The `<script>` that references `hl_progress`.

---

## 4. Asynchronously load the two partials in `annotation_status.html`

Your main template, `annotation_status.html`, can render almost immediately. Then use JavaScript (or fetch calls) to load each snippet from their respective URLs. This results in faster perceived load times for the user:

```html
{# annotation_status.html #}
{% extends 'layouts/base.html' %}
{% load static %}
{% load i18n %}

{% block content %}
<div class="container-fluid py-4">
  <div class="row">
    <div class="col-lg-12">
      <div class="card">

        <div class="card-header pb-0">
          <h5>Annotation Progress (at high level)</h5>
        </div>

        <div class="card-body p-3 pb-0">
          <!-- A spinner or placeholder while we fetch chart data -->
          <div id="chart-container" class="d-flex justify-content-center">
            <div class="spinner-grow text-secondary m-5" role="status">
              <span class="visually-hidden">Loading...</span>
            </div>
          </div>

          <!-- Another spinner or placeholder while we fetch table data -->
          <div id="table-container" class="d-flex justify-content-center">
            <div class="spinner-grow text-secondary m-5" role="status">
              <span class="visually-hidden">Loading...</span>
            </div>
          </div>
        </div>

      </div>
    </div>
  </div>
</div>

<script src="{% static 'js/annotation_status.js' %}" type="text/javascript"></script>

<script>
  // On page load, fetch the partial HTML for both the chart and the table
  document.addEventListener("DOMContentLoaded", function() {
    let chartUrl = "{% url 'dashboard:annotations_status_chart' model_id=nmt_model.id %}";
    let tableUrl = "{% url 'dashboard:annotations_status_table' model_id=nmt_model.id %}";

    // Load the chart partial
    fetch(chartUrl)
      .then(response => response.text())
      .then(html => {
        document.getElementById('chart-container').innerHTML = html;
        // If you have extra chart initialization, do it here
        // e.g., new Chart(...) using var hl_progress
      })
      .catch(err => {
        console.error('Error fetching chart partial:', err);
        document.getElementById('chart-container').innerHTML = '<p>Failed to load chart</p>';
      });

    // Load the table partial
    fetch(tableUrl)
      .then(response => response.text())
      .then(html => {
        document.getElementById('table-container').innerHTML = html;
      })
      .catch(err => {
        console.error('Error fetching table partial:', err);
        document.getElementById('table-container').innerHTML = '<p>Failed to load table</p>';
      });
  });
</script>

{% endblock content %}
```

Now you have:

- **`get_status_chart`**: returning `status_chart.html`, which only includes chart markup & chart-related data (`hl_progress`).
- **`get_status_table`**: returning `status_table.html`, which only includes the table markup & table rows/data.
- **`annotation_status.html`**: an almost empty shell that just has two `div`s (`chart-container` & `table-container`) and some spinners. After the page loads, it makes **two asynchronous requests** to fill those containers.

---

## 5. Update `urls.py`

Make sure your Django `urls.py` (wherever you define these routes) has separate paths for `get_status_table` and `get_status_chart`. For example:

```python
# urls.py

from django.urls import path
from .views.annotation_status_view import (
    AnnotationStatusView,
)

urlpatterns = [
    # ...
    path(
        'annotation_status/<int:model_id>/table/',
        AnnotationStatusView.get_status_table,
        name='annotations_status_table'
    ),
    path(
        'annotation_status/<int:model_id>/chart/',
        AnnotationStatusView.get_status_chart,
        name='annotations_status_chart'
    ),
    path(
        'annotation_status/<int:model_id>/publish/',
        AnnotationStatusView.publish_now,
        name='annotations_status_publish_now'
    ),
    # ...
]
```

Make sure the URL names match what you used in the `<script>` in `annotation_status.html`.

---

### That’s it!

With these changes:

- The main page (`annotation_status.html`) loads quickly because it has minimal server-side work done before sending HTML.
- The heavier aggregation logic is now **split** into two endpoints (`get_status_table` and `get_status_chart`).
- Each partial is fetched via AJAX, so the user sees the page’s skeleton (plus spinners) almost immediately, and each section populates independently. This can feel faster to the end user, especially if the chart data is large or if the table is large.

You can, of course, tweak the approach: you could return JSON for the chart data only and build the chart fully on the client side. But the above approach—**separate partial templates** loaded asynchronously—is a straightforward way to split the logic and lighten the initial page load.