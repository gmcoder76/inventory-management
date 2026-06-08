# Project Wiki: Approved Libraries — Inventory Management
# triggers: import, install, dependency, library, package, pip, require, module

## Backend — Approved
- `fastapi` — web framework (installed)
- `pydantic` — data validation (installed)
- `uvicorn` — ASGI server (installed)
- Python standard library: `uuid`, `json`, `os`, `datetime`, `hashlib`, `hmac`

## Backend — DO NOT ADD
- `sqlalchemy` — no database in this system
- `celery` — no task queue needed
- `confluent-kafka` / `aiokafka` — no Kafka in this demo
- `aiohttp` / `httpx` — use FastAPI built-in async if needed

## Frontend — Approved
- `Vue 3` with Composition API (installed)
- `Vite` — build tool (installed)

## Existing Utilities — Always Use
| Utility | Location | Purpose |
|---|---|---|
| `apply_filters()` | `server/main.py` | Filter by warehouse/category |
| `filter_by_month()` | `server/main.py` | Filter by month/quarter |
| `load_json_file()` | `server/mock_data.py` | Load data from server/data/ |
