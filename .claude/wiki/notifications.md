# Project Wiki: Notifications — Inventory Management
# triggers: notification, alert, event, publish, notify, trigger, emit

## Mechanism
This demo has no Kafka or message broker. Notifications are simulated as a structured
event log written to `server/data/alert_events.json`.

## Required Event Fields
```json
{
  "event_id":         "uuid-v4",
  "event_type":       "inventory.low_stock_detected",
  "sku":              "PCB-001",
  "warehouse":        "San Francisco",
  "quantity_on_hand": 150,
  "reorder_point":    200,
  "triggered_at":     "2025-09-30T10:30:00Z",
  "severity":         "warning"
}
```

Rules:
- `event_id` is always UUID v4 — `str(uuid.uuid4())`
- `triggered_at` is always UTC ISO 8601
- `severity` must be one of: `"critical"` (quantity_on_hand == 0), `"warning"` (quantity_on_hand < reorder_point), `"info"` (quantity_on_hand == reorder_point)
- Never omit any field
- Items where `reorder_point` is null are excluded from alerts and returned in a `warnings` collection instead

See `database.md` for the atomic read→append→write pattern.
