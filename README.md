# Org Designer

![Tests](https://img.shields.io/badge/tests-72%20passing-brightgreen)
![Coverage](https://img.shields.io/badge/coverage-lib%20%7C%20store-blue)
![License](https://img.shields.io/badge/license-MIT-lightgrey)

> An interactive org chart tool for exploring and redesigning team structure — without touching the source of truth.

Org Designer loads a snapshot of your organization from LDAP and lets you experiment with structure through drag-and-drop, filtering, and metrics. Nothing is ever written back.

---

## Quickstart

**Prerequisites:** Node 20+, Python 3.10+, VPN access to Red Hat LDAP (for data refresh only)

```bash
make deps          # install Node dependencies
make fetch-users   # pull org data from LDAP (VPN required)
make dev           # start Vite (port 5173) + API server (port 3001)
```

Open http://localhost:5173.

> **Already have a baseline?** Drop a colleague's `data/baseline.json` into `data/` and run `make dev` — no LDAP access needed.

---

## Features

- **Interactive org chart** — expand/collapse managers, pan, zoom, multi-select (Cmd/Ctrl+click)
- **Drag-and-drop reorg** — drag a card onto another to reparent; hold Shift to move the individual only (reports stay with old manager)
- **Undo / Redo** — all changes are overlay actions on top of an immutable baseline; nothing is written back to LDAP or org.git
- **Delete with confirmation** — Delete key or context menu; supports multi-select delete
- **Search** — scored full-text search across name, title, geo, and country (⌘K / `/`)
- **Filters** — narrow by geo, country, job role, or team; highlight or hide non-matching nodes
- **Metrics** — headcount, IC:manager ratio, span of control, breakdowns by geo / role / country / team
- **Team integration** — org.git YAML teams shown as scope nodes and color-coded on person cards
- **Export** — PNG, SVG, PDF, or JSON (JSON can be re-imported as a scenario)
- **Configurable cards** — toggle title, location, city, hire date, tenure, report counts, team name; compact/default/comfortable density; vertical or horizontal layout

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
│   ├── store/index.ts       # Zustand store — baseline, overlay, undo/redo, filters, config
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
│   │   └── role-colors.ts     # Job role → color mapping
│   │
│   └── components/
│       ├── chart/           # OrgChart · PersonNode · ScopeNode · OrgChartEdge
│       ├── layout/          # Toolbar · Sidebar · SearchBar
│       ├── panels/          # MetricsDashboard · FilterPanel · ConfigPanel
│       └── dialogs/         # AddPersonDialog · DeleteConfirmDialog
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

---

## Commands

```bash
make deps            # install Node dependencies
make fetch-users     # pull org data from LDAP → data/all_users.json (VPN required)
make import          # rebuild data/baseline.json from all_users.json + org YAML
make dev             # start dev server (Vite + API)
make build           # production bundle → dist/
make test            # run unit + integration tests
make ci              # typecheck + lint + format check + tests
make clean           # remove dist/, data/, node_modules/
make clean-baseline  # remove baseline.json only (keep all_users.json)
```
