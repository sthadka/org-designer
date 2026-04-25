# Org Designer

![Tests](https://img.shields.io/badge/tests-72%20passing-brightgreen)
![Coverage](https://img.shields.io/badge/coverage-lib%20%7C%20store-blue)
![License](https://img.shields.io/badge/license-MIT-lightgrey)

> An interactive org chart tool for exploring and redesigning team structure — without touching the source of truth.

Org Designer loads a snapshot of your organization from LDAP and lets you experiment with structure through drag-and-drop, filtering, and metrics — saving named scenarios you can share or revisit. Nothing is ever written back.

---

## Quickstart

**Prerequisites:** Node 20+, Python 3.10+, VPN access to Red Hat LDAP (for data refresh only)

```bash
# 1. Install dependencies
make deps

# 2. Fetch org data from LDAP (requires VPN + ldap-utils)
make fetch-users

# 3. Start the dev server
make dev              # Vite (port 5173) + API server (port 3001)
```

Open http://localhost:5173.

> **Already have a baseline?** If a colleague shared `data/baseline.json`, drop it in `data/` and run `make dev` — steps 2 and 3 are skipped automatically.

---

## User Guide

### Navigating the Chart

| Action                              | How                                       |
| ----------------------------------- | ----------------------------------------- |
| Expand/collapse a manager's reports | Click the chevron button on the card edge |
| Expand all / Collapse all           | Toolbar ⤢ / ⤡ buttons                     |
| Pan                                 | Click and drag the canvas                 |
| Zoom                                | Scroll wheel                              |
| Select a node                       | Click it                                  |
| Multi-select                        | Cmd/Ctrl+click additional nodes           |

### Reparenting (Drag to Reorg)

Drag a card onto another to make it the new manager. A highlight indicates the drop target.

- **Default:** moves the person _and their entire subtree_
- **Hold Shift:** moves only the individual — reports are re-parented to the old manager

All changes are overlay actions on top of the immutable baseline. Use **Undo / Redo** (↺ / ↻) to step through changes.

### Filters

The **Filters** tab narrows the chart by geo, country, job role, team, or name/title search.

- **Highlight mode:** non-matching cards dim to 25% opacity
- **Hide mode:** non-matching cards are removed from the layout

### Configure

| Setting          | Description                                                      |
| ---------------- | ---------------------------------------------------------------- |
| Job Title        | Show/hide the full job title line                                |
| Location         | Show/hide Geo · Country                                          |
| City             | Show/hide city/locality                                          |
| Hire Date        | Show/hide hire date                                              |
| Tenure           | Show/hide years of tenure                                        |
| Report Counts    | Show/hide direct/total report count on manager cards             |
| Team Name        | Show/hide the org.git team assignment                            |
| Card Density     | Compact / Default / Comfortable — controls spacing between cards |
| Layout Direction | Vertical (top-to-bottom) or Horizontal (left-to-right)           |
| Snap to Grid     | Snap cards back to layout position when drag doesn't reparent    |

### Export

Use the **Export** dropdown in the toolbar to save the chart as PNG, SVG, PDF, or JSON. JSON export saves the current scenario overlay and can be re-imported later.

---

## Architecture

```
org-designer/
├── server/
│   └── index.ts             # Express API server (port 3001)
│       ├── GET  /api/baseline
│       └── CRUD /api/scenarios/:name
│
├── src/
│   ├── App.tsx              # Root layout: Toolbar · Sidebar · OrgChart
│   │
│   ├── store/index.ts       # Single Zustand store — baseline, overlay,
│   │                        #   undo/redo, selection, filters, config
│   │
│   ├── types/
│   │   ├── person.ts        # PersonRecord · TeamRecord · BaselineData
│   │   ├── overlay.ts       # OverlayAction union (move, scope_*, add/edit/delete)
│   │   └── org.ts           # OrgScopeNode · EffectiveState
│   │
│   ├── lib/                 # Pure business logic (fully unit-tested)
│   │   ├── overlay-engine.ts  # applyOverlay(): baseline + actions → EffectiveState
│   │   ├── layout-engine.ts   # computeLayout(): dagre → ReactFlow nodes/edges
│   │   ├── filter-utils.ts    # matchesFilter() · computeFilteredIds()
│   │   ├── hierarchy-utils.ts # buildChildrenMap() · getSubtreeIds()
│   │   ├── search-utils.ts    # searchPeople() with scored ranking
│   │   ├── team-colors.ts     # Deterministic team ID → color mapping
│   │   ├── role-colors.ts     # Job role → color mapping
│   │   └── role-abbreviation.ts
│   │
│   └── components/
│       ├── chart/
│       │   ├── OrgChart.tsx     # ReactFlow canvas, drag-and-drop, filter overlay
│       │   ├── PersonNode.tsx   # Person card (all field toggles, direction-aware buttons)
│       │   ├── ScopeNode.tsx    # Team/scope label node
│       │   ├── OrgChartEdge.tsx # Custom org-chart style edges (shared trunk pattern)
│       │   └── BreadcrumbBar.tsx
│       ├── layout/
│       │   ├── Toolbar.tsx      # Undo/redo · expand/collapse · import/export
│       │   ├── Sidebar.tsx      # Tab switcher (Metrics · Filters · Configure)
│       │   └── SearchBar.tsx
│       ├── panels/
│       │   ├── MetricsDashboard.tsx  # Totals · spans · breakdown by geo/role/country/team
│       │   ├── FilterPanel.tsx       # Chip filters + manager dropdown
│       │   └── ConfigPanel.tsx       # Card field toggles · density · direction
│       └── dialogs/
│           ├── AddPersonDialog.tsx
│           └── DeleteConfirmDialog.tsx
│
├── scripts/
│   ├── import.ts            # all_users.json + org YAML → data/baseline.json
│   ├── ldif_to_json.py      # ldapsearch LDIF → JSON
│   └── enrich_users.py      # Adds geocoding + report counts
│
└── data/                    # gitignored — contains PII
    ├── all_users.json        # Raw LDAP dump (make fetch-users)
    └── baseline.json         # Processed snapshot (make import)
```

### Data Flow

```
LDAP  →  ldif_to_json.py  →  enrich_users.py  →  all_users.json
                                                         │
org.git YAML ────────────────────────────────────────────┤
                                                         ▼
                                                   import.ts
                                                         │
                                                         ▼
                                                baseline.json  ←── Express API
                                                                        │
                                                                        ▼
                                                            React app (Vite)
                                                                        │
                                                    + overlay actions (in memory)
                                                                        │
                                                                        ▼
                                                        EffectiveState (computed)
                                                                        │
                                                                        ▼
                                                        dagre layout → ReactFlow
```

### Key Design Decisions

**Baseline + overlay pattern:** All reorg changes are an ordered list of `OverlayAction[]` applied on top of an immutable baseline. Nothing is written back to LDAP or org.git. Undo/redo is trivial — replay fewer actions.

**Controlled ReactFlow:** `nodes` live in local `useState`, seeded from dagre. `applyNodeChanges` handles drag position updates without discarding layout. On state changes (reparent, expand), dagre re-seeds the positions with CSS transition animations.

**No PII in git:** `data/` is gitignored. Baseline is regenerated locally from LDAP.

**org.git integration:** Teams from the org.git YAML hierarchy are converted into scope nodes and used for team-colored cards, "By Team" metrics, and team chip filters. The underlying manager hierarchy comes entirely from LDAP.

---

## Build & Test

```bash
make dev              # Start dev server (requires data/baseline.json)
make fetch-users      # Re-fetch LDAP data (VPN required)
make import           # Rebuild baseline from all_users.json + org YAML
make build            # Production bundle → dist/
make check            # TypeScript type-check (no emit)
make test             # Run unit + integration tests
make test-coverage    # Run tests with coverage report
make clean            # Remove dist/, data/, .vite/, node_modules/
make clean-baseline   # Remove baseline.json only
```

### Testing

Tests are written with [Vitest](https://vitest.dev/) and cover all pure business logic in `src/lib/` and the Zustand store in `src/store/`.

| Category                             | Files                                   | Tests |
| ------------------------------------ | --------------------------------------- | ----- |
| Overlay engine (all 8 action types)  | `lib/__tests__/overlay-engine.test.ts`  | 12    |
| Filter logic + ancestor preservation | `lib/__tests__/filter-utils.test.ts`    | 12    |
| Hierarchy tree traversal             | `lib/__tests__/hierarchy-utils.test.ts` | 8     |
| Search scoring and ranking           | `lib/__tests__/search-utils.test.ts`    | 9     |
| Team color determinism               | `lib/__tests__/team-colors.test.ts`     | 5     |
| Layout height arithmetic             | `lib/__tests__/layout-engine.test.ts`   | 7     |
| Store: undo/redo, selection, delete  | `store/__tests__/store.test.ts`         | 15    |

Run `make test-coverage` to generate an HTML coverage report in `coverage/`.

---

## Makefile Reference

| Target                | Description                                                   |
| --------------------- | ------------------------------------------------------------- |
| `make deps`           | Install Node dependencies (`npm install`)                     |
| `make dev`            | Start dev server (Vite + API) — requires `data/baseline.json` |
| `make fetch-users`    | Re-fetch LDAP data → `data/all_users.json` (VPN required)     |
| `make import`         | Rebuild `data/baseline.json` from `all_users.json` + org YAML |
| `make build`          | Production bundle → `dist/`                                   |
| `make check`          | TypeScript type-check without building                        |
| `make test`           | Run unit + integration tests                                  |
| `make test-watch`     | Run tests in watch mode                                       |
| `make test-coverage`  | Run tests with coverage report                                |
| `make clean`          | Remove `dist/`, `data/`, `.vite/`, `node_modules/`            |
| `make clean-baseline` | Remove `data/baseline.json` only (keep `all_users.json`)      |
