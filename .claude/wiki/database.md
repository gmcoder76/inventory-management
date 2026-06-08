# Project Wiki: Data Model — Inventory Management
# triggers: inventory, sku, reorder, stock, quantity, warehouse, alert, item, category, data, field

## Architecture
In-memory mock system. Data lives in JSON files loaded at startup. NO database, NO ORM.

Data files: `server/data/`
Loading:    `server/mock_data.py` via `load_json_file()`

```python
from mock_data import inventory_items  # CORRECT
json.load(open("server/data/inventory.json"))  # WRONG — bypass the data layer
```

## Inventory Item Schema
```json
{
  "id": "1",
  "sku": "PCB-001",
  "name": "Single Layer PCB Assembly",
  "category": "Circuit Boards",
  "warehouse": "San Francisco",
  "quantity_on_hand": 450,
  "reorder_point": 200,
  "unit_cost": 24.99,
  "location": "Warehouse A-12",
  "last_updated": "2025-09-30T10:30:00"
}
```

## Critical Field Names
- `quantity_on_hand` — current stock level (int, nullable)
- `reorder_point`    — threshold below which stock is low (int, nullable)
- `sku`             — unique identifier (string, format: XXX-NNN)
- `warehouse`       — location name (e.g. "San Francisco", "New York")
- `category`        — product category (e.g. "Circuit Boards", "Sensors")

## Low Stock Logic
An item is LOW STOCK when: `quantity_on_hand < reorder_point`

BOUNDARY: `quantity_on_hand == reorder_point` → NOT low stock (not below threshold).
Always test the boundary case explicitly.

## Null Safety (MANDATORY)
Both `reorder_point` and `quantity_on_hand` CAN be null in legacy records.

```python
# CORRECT
if item.get("reorder_point") is None or item.get("quantity_on_hand") is None:
    warnings.append({"sku": item["sku"], "reason": "missing_threshold"})
    continue
if item["quantity_on_hand"] < item["reorder_point"]:
    alerts.append(item)

# WRONG — crashes on null
if item["quantity_on_hand"] < item["reorder_point"]:
```

## Alert Event Writing
Write to `server/data/alert_events.json` atomically (read → append → write):

```python
import json, os, uuid
from datetime import datetime, timezone

def write_alert_event(alerts: list, data_dir: str):
    path = os.path.join(data_dir, "alert_events.json")
    existing = []
    if os.path.exists(path):
        with open(path) as f:
            existing = json.load(f)
    for alert in alerts:
        existing.append({
            "event_id":        str(uuid.uuid4()),
            "event_type":      "inventory.low_stock_detected",
            "sku":             alert["sku"],
            "warehouse":       alert["warehouse"],
            "quantity_on_hand": alert["quantity_on_hand"],
            "reorder_point":   alert["reorder_point"],
            "triggered_at":    datetime.now(timezone.utc).isoformat()
        })
    with open(path, "w") as f:
        json.dump(existing, f, indent=2)
```

Write events AFTER the endpoint logic succeeds. If the event write fails, log and continue — do NOT fail the API response.
