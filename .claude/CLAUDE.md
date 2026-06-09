# inventory-management — Project Context

> Read this before building anything. It is ~100 tokens — far cheaper than reading main.py.

## What this is
FastAPI backend (port 8001) + Vue 3 frontend (port 3000) for factory inventory management.
Single-file backend: `server/main.py`. In-memory data from `server/data/*.json`.

## The 3 rules you must follow
1. Filter with `apply_filters(items, warehouse?, category?, status?)` — never inline
2. Paginate with `paginate(data, page, page_size)` → returns `(items, total)` — never unbounded
3. Update `openapi.yaml` BEFORE changing any endpoint

## Key helpers (server/main.py)
- `apply_filters()` line 35
- `filter_by_month()` line 19
- `paginate()` line 55

## Before building any feature
Read `.claude/wiki/service-card.md` — it has the full model list, all endpoints,
data shapes, and the code map of every function and class.

## Run tests
```
cd server && uv run pytest tests/ -v
```
