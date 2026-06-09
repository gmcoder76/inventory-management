# SERVICE CARD: inventory-management

> Auto-updated by post-commit hook. Do not edit the CODE MAP section manually.
> Last updated: 2026-06-09

---

## What This Service Does

Factory inventory management system for tracking stock across warehouses.
Monitors inventory levels, orders, demand forecasts, and alerts when items fall
below reorder thresholds.

**Stack:** Python 3.12 / FastAPI / Pydantic v2 / Uvicorn
**Frontend:** Vue 3 + Vite (separate process, port 3000)
**Backend port:** 8001
**Data:** In-memory — loaded from `server/data/*.json` at startup. No database.

---

## Key Files

| File | Purpose |
|------|---------|
| `server/main.py` | All routes + business logic (single file) |
| `server/mock_data.py` | Data loading — imports all JSON into module-level variables |
| `server/data/inventory.json` | 32 inventory items across SF / London / Tokyo |
| `server/data/orders.json` | Order history |
| `server/data/alert_events.json` | Appended by low-stock endpoint on every call |
| `server/data/demand_forecasts.json` | Demand forecast data |
| `openapi.yaml` | API contract — update this BEFORE changing endpoints |

---

## Pydantic Models (server/main.py)

| Model | Fields |
|-------|--------|
| `InventoryItem` | id, sku, name, category, warehouse, quantity_on_hand, reorder_point, unit_cost, location, last_updated |
| `InventoryItemUpdate` | name, category, warehouse, quantity_on_hand?, reorder_point?, unit_cost, location, last_updated |
| `Order` | id, order_number, customer, items, status, order_date, expected_delivery, total_value, actual_delivery?, warehouse?, category? |
| `DemandForecast` | id, item_sku, item_name, current_demand, forecasted_demand, trend, period |
| `BacklogItem` | id, order_id, item_sku, item_name, quantity_needed, quantity_available, days_delayed, priority, has_purchase_order? |
| `PurchaseOrder` | id, backlog_item_id, supplier_name, quantity, unit_cost, expected_delivery_date, status, created_date, notes? |
| `RestockingOrder` | id, order_number, items, total_cost, budget, status, order_date, expected_delivery |
| `LowStockResponse` | items, page, page_size, total |

---

## Helper Functions (server/main.py)

| Function | Signature | Purpose |
|----------|-----------|---------|
| `apply_filters` | `(items, warehouse?, category?, status?)` | Filter any list — ALWAYS use this, never inline filter |
| `filter_by_month` | `(items, month?)` | Filter by YYYY-MM or Q1-2025 quarter string |
| `paginate` | `(data, page, page_size)` | Returns `(page_items, total)` — ALWAYS use for list endpoints |
| `_write_low_stock_events` | `(alerts)` | Appends events to alert_events.json — internal use only |

**Rule:** Never write inline list comprehension filters. Always call `apply_filters()`.
**Rule:** Never return an unbounded list. Always call `paginate()`.

---

## API Endpoints

| Method | Path | Response Model | Notes |
|--------|------|---------------|-------|
| GET | `/api/inventory` | `List[InventoryItem]` | Filters: warehouse, category |
| GET | `/api/inventory/{item_id}` | `InventoryItem` | 404 if not found |
| GET | `/api/orders` | `List[Order]` | Filters: warehouse, category, status, month |
| GET | `/api/orders/{order_id}` | `Order` | 404 if not found |
| GET | `/api/demand` | `List[DemandForecast]` | No filters |
| GET | `/api/backlog` | `List[BacklogItem]` | Adds has_purchase_order flag |
| GET | `/api/dashboard/summary` | dict | Filters: warehouse, category, status, month |
| GET | `/api/spending/summary` | dict | - |
| GET | `/api/spending/monthly` | dict | - |
| GET | `/api/spending/categories` | dict | - |
| GET | `/api/spending/transactions` | dict | - |
| GET | `/api/reports/quarterly` | list | Computed from orders |
| GET | `/api/reports/monthly-trends` | list | Computed from orders |
| GET | `/api/restocking-orders` | `List[RestockingOrder]` | In-memory only |
| POST | `/api/restocking-orders` | `RestockingOrder` | Uses CATEGORY_LEAD_TIMES |
| GET | `/api/purchase-orders` | list | - |
| POST | `/api/purchase-orders` | dict | - |
| GET | `/api/alerts/low-stock` | `LowStockResponse` | Filters: warehouse, category, page, page_size. Writes alert_events.json |

---

## Data Shape: alert_events.json

Each event appended by `/api/alerts/low-stock`:
```json
{
  "event_id": "uuid",
  "event_type": "inventory.low_stock_detected",
  "sku": "TMP-201",
  "warehouse": "London",
  "quantity_on_hand": 125,
  "reorder_point": 150,
  "triggered_at": "2026-06-09T08:00:00+00:00",
  "severity": "warning"
}
```
`severity` = `"critical"` when `quantity_on_hand == 0`, else `"warning"`.

---

## Warehouses and Categories

**Warehouses:** San Francisco, London, Tokyo
**Categories:** Circuit Boards, Sensors, Actuators, Controllers, Power Supplies

**Low-stock items (as of last data update):**
- London: PCB-003, TMP-201, HMD-202, STP-303 (critical — qty 0), PSU-505
- Tokyo: SRV-301, SRV-302, STP-304, GYR-207 (critical — qty 0), PSU-508 (critical)
- San Francisco: all items above reorder point

---

## Patterns: What to Do / What Not to Do

| Situation | Correct Pattern | Wrong Pattern |
|-----------|----------------|---------------|
| Add a new list endpoint | Use `apply_filters()` + `paginate()` | Inline comprehensions + unbounded return |
| Add a new field to a response | Update Pydantic model AND openapi.yaml | Update only the code |
| Write an event | Append to `server/data/alert_events.json` via `_write_low_stock_events()` | Direct file write with different shape |
| Load data for a new endpoint | Add to `mock_data.py` and import at top of `main.py` | `json.load(open(...))` inside endpoint |
| Add a dependency | Add to `pyproject.toml` + comment why | Add silently or use a banned library |

---

## Banned Libraries

`sqlalchemy`, `celery`, `django`, `flask`, `requests` (use `httpx` instead)

---

## CODE MAP
<!-- AUTO-UPDATED by .git/hooks/post-commit — do not edit below this line -->
<!-- FORMAT: function_or_class | file | line | added_in_commit -->

filter_by_month | server/main.py | 19 | initial
apply_filters | server/main.py | 35 | initial
InventoryItem | server/main.py | 61 | initial
Order | server/main.py | 73 | initial
DemandForecast | server/main.py | 86 | initial
BacklogItem | server/main.py | 95 | initial
PurchaseOrder | server/main.py | 106 | initial
CreatePurchaseOrderRequest | server/main.py | 117 | initial
RestockingOrderItem | server/main.py | 125 | initial
RestockingOrder | server/main.py | 132 | initial
CreateRestockingOrderRequest | server/main.py | 142 | initial
root | server/main.py | 160 | initial
get_inventory | server/main.py | 164 | initial
get_inventory_item | server/main.py | 172 | initial
get_orders | server/main.py | 180 | initial
get_order | server/main.py | 192 | initial
get_demand_forecasts | server/main.py | 200 | initial
get_backlog | server/main.py | 205 | initial
get_dashboard_summary | server/main.py | 218 | initial
get_spending_summary | server/main.py | 246 | initial
get_monthly_spending | server/main.py | 251 | initial
get_category_spending | server/main.py | 256 | initial
get_recent_transactions | server/main.py | 261 | initial
get_quarterly_reports | server/main.py | 266 | initial
get_monthly_trends | server/main.py | 312 | initial
get_restocking_orders | server/main.py | 343 | initial
create_restocking_order | server/main.py | 348 | initial
<!-- END CODE MAP -->
