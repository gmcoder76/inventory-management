# Project Wiki: Testing — Inventory Management
# triggers: test, pytest, coverage, assert, mock, fixture, testclient

## Framework
pytest + FastAPI TestClient

```python
from fastapi.testclient import TestClient
import sys, os
sys.path.insert(0, os.path.join(os.path.dirname(__file__), "../../server"))
from main import app

client = TestClient(app)
```

## 8 Required Test Cases — Low Stock Alert Endpoint
The coverage-grader checks for ALL of these. Missing any will fail the pre-review.

1. `test_low_stock_returns_items_below_reorder_point` — happy path
2. `test_low_stock_empty_when_all_stock_adequate` — 200 with empty list, not 404
3. `test_low_stock_boundary_equal_to_reorder_point` — quantity==reorder_point → NOT returned
4. `test_low_stock_null_reorder_point_goes_to_warnings` — null safety
5. `test_low_stock_filters_by_warehouse` — filter param applied correctly
6. `test_low_stock_pagination_envelope_present` — {items, page, page_size, total} present
7. `test_low_stock_event_written_to_alert_events` — side effect verified
8. `test_low_stock_event_shape_complete` — all required event fields present

## Assertion Style
Assert on logic and shape, not specific SKU names or counts:

```python
# CORRECT
assert response.status_code == 200
data = response.json()
assert all(item["quantity_on_hand"] < item["reorder_point"] for item in data["items"])

# WRONG — brittle
assert data["items"][0]["sku"] == "PCB-001"
```
