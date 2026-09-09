# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

Factory Inventory Management System Demo with GitHub integration - Full-stack application with Vue 3 frontend, Python FastAPI backend, and in-memory mock data (no database).

> ⚠️ **This repository and any fork you create are PUBLIC.** Do not commit credentials, internal hostnames, or private registry URLs. `client/.npmrc` pins the public npm registry and `client/package-lock.json` is gitignored to prevent locally-configured registries from leaking into commits — leave both in place.

## Critical Tool Usage Rules

### Subagents
Use the Task tool with these specialized subagents for appropriate tasks:

- **vue-expert**: Use for Vue 3 frontend features, UI components, styling, and client-side functionality
  - Examples: Creating components, fixing reactivity issues, performance optimization, complex state management
  - **MANDATORY RULE: ANY time you need to create or significantly modify a .vue file, you MUST delegate to vue-expert**
- **code-reviewer**: Use after writing significant code to review quality and best practices
- **security-auditor**: Use for security review of changes, especially around auth, filtering/query params, and CORS
- **Explore**: Use for understanding codebase structure, searching for patterns, or answering questions about how components work
- **general-purpose**: Use for complex multi-step tasks or when other agents don't fit

### Skills
- **backend-api-test** skill: Use when writing or modifying tests in `tests/backend` directory with pytest and FastAPI TestClient

### MCP Tools
- **ALWAYS use GitHub MCP tools** (`mcp__github__*`) for ALL GitHub operations
  - Exception: Local branches only - use `git checkout -b` instead of `mcp__github__create_branch`
- **ALWAYS use Playwright MCP tools** (`mcp__playwright__*`) for browser testing
  - Test against: `http://localhost:3000` (frontend), `http://localhost:8001` (API)

### Nested CLAUDE.md files
- `server/CLAUDE.md` has detailed FastAPI/Pydantic conventions and patterns (RESTful design, error handling, filter implementation examples)
- `client/CLAUDE.md` has detailed Vue 3 Composition API conventions and patterns (composables, reactivity, component structure examples)
- Read the relevant nested file before making non-trivial changes in that directory — this root file only covers what's shared across the stack.

## Stack
- **Frontend**: Vue 3 + Composition API + Vite (port 3000)
- **Backend**: Python FastAPI (port 8001)
- **Data**: JSON files in `server/data/` loaded via `server/mock_data.py`

## Commands

```bash
# One-command startup (macOS/Linux)
./scripts/start.sh   # installs deps if missing, starts both servers
./scripts/stop.sh

# Backend (manual)
cd server
uv venv && uv sync
uv run python main.py         # http://localhost:8001, docs at /docs

# Frontend (manual)
cd client
npm install
npm run dev                   # http://localhost:3000
npm run build                 # production build -> client/dist/

# Backend tests (51 tests, run from tests/ not server/)
cd tests
uv run pytest -v
uv run pytest backend/test_inventory.py -v                                   # single file
uv run pytest backend/test_inventory.py::TestInventoryEndpoints -v           # single class
uv run pytest backend/test_inventory.py::TestInventoryEndpoints::test_get_all_inventory -v  # single test
uv run pytest --cov=../server --cov-report=html                              # coverage
```

There is no lint/format tooling configured in this repo (no ESLint/Prettier config, no Python linter).

## Architecture

**Filter system**: 4 filters (Time Period, Warehouse, Category, Order Status) apply across the app via query params. Flow: Vue filter state (`client/src/composables/useFilters.js`) → `client/src/api.js` → FastAPI query params → `apply_filters()`/`filter_by_month()` in `server/main.py` → in-memory list filtering → Pydantic `response_model` validation → Vue computed properties derive display data from raw refs.

- Inventory has no time dimension — inventory filters don't accept `month`.
- Month filtering also accepts quarters (`Q1-2025`..`Q4-2025`), mapped in `QUARTER_MAP` in `server/main.py`.
- Filters use the sentinel value `'all'` (not empty string/null) to mean "no filter" on both ends of the stack.

**Backend data flow**: `server/mock_data.py` loads all JSON files from `server/data/` into module-level lists/dicts at import time (`inventory_items`, `orders`, `demand_forecasts`, `backlog_items`, `spending_summary`, `monthly_spending`, `category_spending`, `recent_transactions`, `purchase_orders`). `server/main.py` filters these in memory per-request — nothing is persisted, so edits during a running session are lost on restart, and changes to the JSON files require a server restart to take effect.

**Frontend structure**:
- Views (`client/src/views/*.vue`) are routed page-level components registered in `client/src/main.js`; note `Backlog.vue` exists but is not currently wired into the router (only linked from within `Dashboard.vue`).
- Reusable modals/widgets live in `client/src/components/*.vue`.
- Cross-cutting state lives in composables: `useFilters.js` (global filter state), `useAuth.js` (mock auth — always authenticated, no real login/tokens), `useI18n.js` (en/ja translations, locale persisted to `localStorage`, currency auto-derived from locale: `en`→USD, `ja`→JPY via `client/src/utils/currency.js`).
- Raw API data is held in refs (`allOrders`, `inventoryItems`, etc.); all derived/filtered/aggregated values are computed properties — don't recompute derived data manually in methods.

## API Endpoints
- `GET /api/inventory` - Filters: warehouse, category
- `GET /api/orders` - Filters: warehouse, category, status, month
- `GET /api/dashboard/summary` - All filters
- `GET /api/demand`, `/api/backlog` - No filters
- `GET /api/spending/*` - Summary, monthly, categories, transactions

## Code Style
- Always document non-obvious logic changes with comments.

## Common Issues
1. Use unique keys in v-for (not `index`) - use `sku`, `month`, etc.
2. Validate dates before `.getMonth()` calls
3. Update Pydantic models when changing JSON data structure
4. Inventory filters don't support month (no time dimension)
5. Revenue goals: $800K/month single, $9.6M YTD all months

## File Locations
- Views: `client/src/views/*.vue`
- API Client: `client/src/api.js`
- Backend: `server/main.py`, `server/mock_data.py`
- Data: `server/data/*.json`
- Styles: `client/src/App.vue`

## Design System
- Colors: Slate/gray (#0f172a, #64748b, #e2e8f0)
- Status: green/blue/yellow/red
- Charts: Custom SVG, CSS Grid for layouts
- No emojis in UI
