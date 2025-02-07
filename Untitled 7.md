To improve the performance of the `get_annotation_data` function while maintaining correct aggregations and implementing pagination with caching, follow these steps:

### 1. Optimize Summary Statistics with Django ORM
Replace Pandas-based aggregations with optimized database queries to calculate totals without loading all data into memory.

```python
from django.core.paginator import Paginator
from django.db.models import Count, Q, F
from django.db.models.functions import Cast, Coalesce
from django.contrib.postgres.fields.jsonb import KeyTextTransform

def get_annotation_data(request, track_id):
    track = AnnotationTrack.objects.get(pk=track_id)
    page_number = request.GET.get('page', 1)
    page_size = 25  # Adjust based on needs

    # Get base queryset
    samples_qs = AnnotationTrackSample.objects.filter(track=track)
    
    # Calculate statistics using optimized ORM queries
    total_texts = samples_qs.count()
    
    pending_status = [AnnotationTrackSample.Status.PENDING, AnnotationTrackSample.Status.LOADED]
    pending_texts = samples_qs.filter(status__in=pending_status).count()

    # Word count calculation (using database functions)
    word_count_expr = Coalesce(
        Cast(
            KeyTextTransform('text_to_translate', 'data'), 
            output_field=IntegerField()
        ), 
        0
    )
    total_words = samples_qs.aggregate(
        total=Sum(word_count_expr)
    )['total'] or 0
    
    pending_words = samples_qs.filter(status__in=pending_status).aggregate(
        total=Sum(word_count_expr)
    )['total'] or 0

    # High-level keys calculation
    l2_fields = AnnotationFormField.objects.filter(
        form=track.form, 
        is_high_level_key=True
    ).values_list('name', flat=True)
    
    if l2_fields:
        total_hl_keys = samples_qs.values(*[f'data__{field}' for field in l2_fields]).distinct().count()
    else:
        total_hl_keys = 0

    # Pagination
    paginator = Paginator(samples_qs.select_related('annotated_by', 'validated_by'), page_size)
    page = paginator.get_page(page_number)

    # Build data list for current page
    data_list = []
    for sample in page:
        row = sample.data.copy()
        row.update({
            'annotator': sample.annotated_by.username if sample.annotated_by else '',
            'annotated_at': sample.annotated_at.strftime("%Y-%m-%d %H:%M:%S") if sample.annotated_at else '',
            'validator': sample.validated_by.username if sample.validated_by else '',
            'validated_at': sample.validated_at.strftime("%Y-%m-%d %H:%M:%S") if sample.validated_at else '',
            'status': sample.status,
            'sample_id': sample.id,
            'word_count': len(sample.data.get('text_to_translate', '').split())
        })
        data_list.append(row)

    context = {
        'rows': data_list,
        'total_texts': total_texts,
        'total_words': total_words,
        'pending_texts': pending_texts,
        'pending_words': pending_words,
        'total_hl_keys': total_hl_keys,
        'current_page': page.number,
        'total_pages': paginator.num_pages,
        # Include other necessary context
    }
    
    return render(request, "includes/data_table.html", context)
```

### 2. Implement Server-Side Pagination in JavaScript
Modify the JavaScript to handle pagination requests and cache pages:

```javascript
// data.js
const PAGE_SIZE = 25;
let currentPage = 1;
let currentTrackUrl = '';
const dataCache = new Map();

function loadTable(track, page=1, useCache=true) {
    const cacheKey = `${track.datasetTableUrl}-${page}`;
    
    if (useCache && dataCache.has(cacheKey)) {
        renderTable(dataCache.get(cacheKey));
        return;
    }

    showSpinner();
    
    fetch(`${track.datasetTableUrl}?page=${page}&page_size=${PAGE_SIZE}`)
        .then(response => response.text())
        .then(html => {
            dataCache.set(cacheKey, html);
            renderTable(html);
            currentPage = page;
            currentTrackUrl = track.datasetTableUrl;
        });
}

function renderTable(html) {
    const container = document.querySelector('.data-table-container');
    container.innerHTML = html;
    initializeDataTable();
    
    // Update pagination controls
    updatePaginationControls(container);
}

function updatePaginationControls(container) {
    const paginationDiv = document.createElement('div');
    const pageInfo = JSON.parse(container.querySelector('#page-info').dataset.info);
    
    paginationDiv.innerHTML = `
        <button ${pageInfo.current_page === 1 ? 'disabled' : ''} 
                onclick="loadPage(${pageInfo.current_page - 1})">Previous</button>
        <span>Page ${pageInfo.current_page} of ${pageInfo.total_pages}</span>
        <button ${pageInfo.current_page === pageInfo.total_pages ? 'disabled' : ''} 
                onclick="loadPage(${pageInfo.current_page + 1})">Next</button>
    `;
    
    container.appendChild(paginationDiv);
}

function loadPage(newPage) {
    if (newPage === currentPage) return;
    loadTable(currentTrackUrl, newPage);
}

// Initialize with first page
document.addEventListener('DOMContentLoaded', () => {
    const initialTrack = document.querySelector('[data-table-url]');
    loadTable(initialTrack);
});
```

### 3. Update Template to Include Pagination Information
Modify the template to include pagination metadata:

```html
<!-- includes/data_table.html -->
<div id="page-info" data-info='{"current_page": {{ current_page }}, "total_pages": {{ total_pages }}}'></div>

<table>
    <!-- Table headers -->
    <tbody>
        {% for row in rows %}
            <tr>
                <!-- Table cells -->
            </tr>
        {% endfor %}
    </tbody>
</table>
```

### 4. Cache Invalidation on Data Updates
When new data is added, clear the cache for affected tracks:

```javascript
// In add-data success handler
function handleAddDataSuccess() {
    // Clear cache for this track
    const trackId = /* get current track ID */;
    Array.from(dataCache.keys()).forEach(key => {
        if (key.startsWith(trackId)) dataCache.delete(key);
    });
    loadTable(currentTrackUrl, 1, false);
}
```

### Key Improvements:
1. **Efficient Aggregations**: Statistics are calculated using database optimizations instead of in-memory Pandas operations.
2. **Server-Side Pagination**: Only the current page's data is processed and transferred.
3. **Client-Side Caching**: Previously visited pages are stored to avoid redundant requests.
4. **Maintained Functionality**: All original functionality (validation, data display) remains intact.
5. **Scalability**: Handles large datasets more efficiently by reducing memory usage and database load.

This approach significantly reduces initial load time by only processing the first page of data while maintaining accurate aggregations across the entire dataset. Subsequent pages load quickly from cache when available, and statistics remain accurate through optimized database queries.