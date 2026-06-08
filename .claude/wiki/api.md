# Project Wiki: API Patterns — Inventory Management
# triggers: endpoint, route, api, get, post, filter, paginate, response

## Filter Pattern (this codebase)
The `apply_filters()` utility in `server/main.py` handles warehouse and category filtering.
ALWAYS use it. Never reimplement filtering inline.

```python
# CORRECT
from main import apply_filters
filtered = apply_filters(inventory_items, warehouse=warehouse, category=category)

# WRONG — reimplementing what already exists
filtered = [i for i in inventory_items if i.get("warehouse") == warehouse]
```

Supported filters on inventory endpoints:
- `warehouse`: optional string, default=None (returns all)
- `category`: optional string, default=None (returns all)

Note: Inventory endpoints do NOT support month/date filtering — stock levels have no time dimension.

## Pagination
Use the `paginate()` helper if it exists, or implement the standard envelope:
```json
{ "items": [...], "page": 1, "page_size": 20, "total": 47 }
```
Never return a raw array.

## API Prefix
All endpoints use the `/api/` prefix (e.g. `/api/inventory`, `/api/alerts/low-stock`).
This codebase does not yet use versioned paths — document that production would use `/v2/`.

## Pydantic Response Models
Response models live in `server/main.py`.
Naming convention: `{Entity}Response`, `{Entity}ListResponse`.
Always match field types to the JSON data files in `server/data/`.
