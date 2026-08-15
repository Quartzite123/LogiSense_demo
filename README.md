# LogiSense: Logistics Intelligence Platform

> A production **FastAPI + React** logistics analytics dashboard for parcel
> distribution companies, built for a real logistics aggregator (a Delhivery and
> Blue Dart channel partner). It ships with an **AI Insights engine** (statistical
> pattern detectors narrated by Groq / Llama 3.3 70B) and a grounded AI chat.
>
> This README is the single source of truth for the whole project: architecture,
> business logic, data pipeline, API surface, the insights engine, sessions and
> deployment, and the frontend. An engineer reading it straight through has the
> complete picture without needing any prior conversation.

---

## Live Demo

- **Dashboard:** https://logi-sense-demo.vercel.app/
- **Login:** demo@logisense.app / demo1234
- **API docs:** https://logisense-demo.onrender.com
- **Isolation:** each visitor gets their own session database. Uploads and
  reference-data edits affect only your session, never anyone else's.
- **Demo upload file:** `tools/demo_upload.xlsx` (upload it to see the
  What-Changed digest react)

---

## Table of Contents

1. [What This Project Does](#1-what-this-project-does)
2. [Tech Stack](#2-tech-stack)
3. [Project Structure](#3-project-structure)
4. [Core Business Logic (LOCKED)](#4-core-business-logic-locked)
5. [The Data Pipeline (`app/`)](#5-the-data-pipeline-app)
6. [The Database](#6-the-database)
7. [Backend API (`backend/`)](#7-backend-api-backend)
8. [The AI Insights Engine](#8-the-ai-insights-engine)
9. [The AI Assistant](#9-the-ai-assistant)
10. [Sessions & Deployment](#10-sessions--deployment)
11. [Frontend (`frontend/`)](#11-frontend-frontend)
12. [Design System](#12-design-system)
13. [Performance Optimizations](#13-performance-optimizations)
14. [Running the Project](#14-running-the-project)
15. [Testing](#15-testing)
16. [Known Quirks & Gotchas](#16-known-quirks--gotchas)
17. [Pending / Future Work](#17-pending--future-work)
18. [Migration History](#18-migration-history)
19. [Glossary](#19-glossary)

---

## 1. What This Project Does

A logistics company uploads its raw **Delhivery export file** (a 41-column `.xlsx`).
LogiSense then:

1. **Ingests** the file: parses, cleans, and deduplicates shipment records.
2. **Enriches** each shipment: resolves origin and destination zones, ODA status, expected TAT.
3. **Classifies** delivery performance: Early, On Time, or Late per order.
4. **Visualizes** everything: KPI cards, charts, and drill-down tables across seven sections.
5. **Explains the data** with an **AI Insights engine**: eight statistical detectors find
   real patterns (client churn, volume decline, structural ODA lateness, and more), a single
   Groq call narrates them in plain English, and a What-Changed digest compares this upload
   to the last one.
6. **Answers questions** with an AI chat grounded in the current data, with topic guardrails.

Every visitor to the public demo works in an **isolated session database**, so one person's
upload or edit never changes what anyone else sees.

The platform is used to monitor delivery-performance (E+OT) compliance, catch at-risk orders
before they breach, and evaluate per-client performance.

### Original design constraints (the founder's on-prem product)

These constraints describe the **founder's intended production build** (a local, offline
desktop tool). The public demo above is a separate hosted showcase on Vercel and Render, so
it deliberately relaxes the no-cloud and offline constraints while keeping the same codebase.

| # | Constraint | Why |
|---|---|---|
| 1 | **No cloud dependency** in the founder's build (nothing on AWS/GCP/Azure) | Data is operational and commercially sensitive |
| 2 | **No internet needed during operation** (the AI features are the sole optional exception) | Must work with unreliable connectivity |
| 3 | **Files uploaded manually** through the UI (no email fetch, no scheduled sync) | Simpler ops; founders export weekly |
| 4 | **Minimal install friction**: the hosted demo is zero-install (a web app); an on-prem build would package the same stack | Founders are non-technical |
| 5 | **Runs comfortably on 8 GB RAM Windows laptops** | Founder hardware varies |
| 6 | **Reference-data edits must NOT rewrite history**: matrix and ODA edits affect only future uploads; past shipments keep their stored values | Audit trust |

---

## 2. Tech Stack

| Layer | Technology | Notes |
|---|---|---|
| Data pipeline | Python 3.x (pandas, openpyxl) | Core pipeline, battle-tested with 28 unit tests |
| Database | SQLite (WAL mode) | One file per session; the shared template is `demo/demo.db` |
| Backend API | FastAPI + uvicorn | Async, auto-docs at `/docs` |
| Frontend | React 18 + Vite | SPA, lazy-loaded routes |
| Styling | Tailwind CSS + custom tokens | No component libraries |
| Charts | Recharts | Donut, composed line, grouped and stacked bars |
| Data fetching | TanStack React Query | 5-minute stale time, cache invalidation on upload |
| AI Insights | 8 SQL detectors + one Groq narration | Cached per upload in `insight_cache`, zero live calls on page load |
| AI chat | Groq API (Llama 3.3 70B) | Grounded in cached insights, topic guardrails, offline fallback |
| Animation | `lottie-react` | Upload-processing animation |
| Sessions | Cookie-scoped SQLite DB per visitor | `logi_session` cookie, DB under `/tmp/sessions` |
| Deployment | Vercel (frontend) + Render (backend) | Render uses a persistent disk; instant startup from `demo/demo.db` |
| Tests | pytest (28 tests) | Cover the `app/` pipeline only |

---

## 3. Project Structure

Verified against `git ls-files`. `app/pipeline/` and `app/store/` are the LOCKED
original pipeline; the FastAPI backend imports from them and never modifies them.

```
LogiSense/
├── app/                          # LOCKED: original data pipeline (do not modify)
│   ├── pipeline/
│   │   ├── ingest.py             # Upload processing: replace-then-insert, dedup, enrich
│   │   ├── dedup.py              # Keeps one winner snapshot per LRN
│   │   ├── sla.py                # compute_row(): TAT + Early/OnTime/Late classification
│   │   ├── zones.py              # Destination pincode to zone lookup
│   │   ├── oda.py                # lookup_oda(): YES/NO/UNKNOWN per pincode
│   │   └── origin_lookup.py      # 3-tier origin city to zone chain
│   ├── store/
│   │   ├── db.py                 # SQLite connection factory, WAL + perf PRAGMAs, session
│   │   │                         #   DB path (ContextVar), SCHEMA_SQL, indexes, init_db()
│   │   ├── seed.py               # First-run seeding: matrix, pincode master, STATE_ZONE
│   │   ├── schema.py             # Column definitions and snake_case mapping helpers
│   │   └── queries.py            # Original query functions (load_latest, trends, and more)
│   ├── reference/
│   │   ├── pincode_master.xlsx   # ~21,849 Indian pincodes with ODA flags (public data)
│   │   └── matrix.csv            # 5x5 zone TAT matrix
│   └── data/
│       └── origin_city_master.csv  # Indian cities with state and zone
│
├── backend/                      # FastAPI application
│   ├── main.py                   # App entry: SessionMiddleware, CORS, routers, lifespan
│   │                             #   (init_db, demo.db copy, seed, session cleanup),
│   │                             #   /api/health (session-exempt), static serving, load_dotenv
│   ├── schemas.py                # Pydantic response models + COLUMN_DISPLAY_NAMES
│   ├── session.py                # Per-visitor session DBs (get_or_create_session, cleanup)
│   ├── transit_risk.py           # Shared risk classifier (single source of truth)
│   ├── .env                      # GROQ_API_KEY (gitignored, never commit)
│   ├── insights/
│   │   ├── detectors.py          # 8 SQL pattern detectors + run_all_detectors
│   │   ├── groq_narrator.py      # Single Groq call + deterministic offline fallback
│   │   └── snapshot.py           # Snapshots, insight_cache read/write, generate_and_cache
│   └── routers/
│       ├── upload.py             # POST /api/upload + GET /api/export, regenerates insights
│       ├── landing.py            # /api/landing/kpis /donut /trend
│       ├── tat.py                # /api/tat/orders /summary /oda-chart
│       ├── transit.py            # /api/transit/orders /summary
│       ├── aggregate.py          # /api/aggregate/companies /monthly
│       ├── aggregate_transit.py  # /api/aggregate-transit/companies /company-detail
│       ├── customize.py          # /api/customize/orders (filtered)
│       ├── exports.py            # /api/export/{tat,transit,aggregate,...} to xlsx
│       ├── edit.py               # matrix + pincode read/edit/reset/upload endpoints
│       ├── insights.py           # /api/insights/digest /patterns /root-cause
│       └── assistant.py          # /api/assistant/chat + /suggestions (Groq + guardrails)
│
├── frontend/                     # React SPA
│   ├── index.html                # Entry, favicon
│   ├── vite.config.js            # /api to 127.0.0.1:8000 proxy (IPv4, not localhost)
│   ├── tailwind.config.js
│   ├── .env.development          # VITE_API_URL empty (dev proxy handles /api)
│   ├── .env.production           # VITE_API_URL = https://logisense-1dvc.onrender.com
│   └── src/
│       ├── main.jsx              # QueryClient (staleTime 5m, no window-focus refetch)
│       ├── App.jsx               # Routes, ProtectedShell layout, lazy Insights, Suspense
│       ├── index.css             # Global CSS, mobile media query, login shake keyframes
│       ├── components/icons.jsx  # Shared inline-SVG icon set (no icon library, no emoji)
│       ├── lib/
│       │   ├── api.js            # apiUrl, fetchJSON, sendJSON, download (credentials-aware)
│       │   └── useIsMobile.js    # Shared matchMedia(768px) hook
│       ├── context/ui.jsx        # Global upload-modal + toast state
│       ├── styles/tokens.js      # Design tokens (colors, radii, spacing, fonts)
│       ├── assets/airplane.json  # Lottie animation shown while an upload processes
│       ├── components/
│       │   ├── Sidebar.jsx       # Desktop nav (8 items) + Sign out; hidden on mobile
│       │   ├── MobileNav.jsx     # Mobile bottom bar + slide-up drawer + Sign out
│       │   ├── PageHeader.jsx    # Title + subtitle + Upload button
│       │   ├── KPICard.jsx       # Label/value/subtext/progress-bar/isDateCard
│       │   ├── DataTable.jsx     # Zebra, sticky sortable header, toolbar, expand
│       │   ├── StatusPill.jsx    # Colored pills for every status type
│       │   ├── ColumnPicker.jsx  # Show/hide columns + Sort By/Direction
│       │   ├── UploadModal.jsx   # Drag-drop modal + Lottie processing + success state
│       │   ├── Toast.jsx         # Top-right success/error toasts
│       │   ├── Skeleton.jsx      # Shimmer loading placeholder
│       │   ├── EmptyState.jsx    # Zero-data state with upload CTA
│       │   ├── filters/          # FilterPanel, FilterSelect, SegmentedToggle
│       │   ├── charts/           # chartTheme, Donut, TrendChart, GroupedBar, StackedBar, ChartPair
│       │   └── insights/
│       │       ├── DigestCard.jsx    # What-Changed 5-bullet card with up/down/neutral markers
│       │       ├── PatternCard.jsx   # Severity-coded pattern card + root-cause expander
│       │       ├── RootCausePanel.jsx# Inline "why is this happening" panel
│       │       └── ChatPanel.jsx     # Chat (inline on desktop, floating overlay on mobile)
│       └── pages/
│           ├── Landing.jsx        # 12 KPI cards + donut + trend + month table
│           ├── TAT.jsx            # Chips + ODA chart + column picker + table
│           ├── Transit.jsx        # Risk chips + donut + ChartPair + risk table
│           ├── Aggregate.jsx      # Company table + perf bar + monthly stacked
│           ├── AggregateTransit.jsx # Company dropdown + risk summary + overdue
│           ├── Customize.jsx      # Filters + toggle + result count + export
│           ├── Insights.jsx       # Digest + pattern grid + chat (replaces the old Assistant page)
│           ├── Login.jsx          # Demo login gate
│           └── Edit.jsx           # Region matrix + pincode master tabs (editable)
│
├── demo/
│   └── demo.db                   # Pre-built SQLite (4,017 shipments + cached insights),
│                                 #   the template each session and a fresh server copies
├── tools/
│   ├── generate_sample_data.py   # Synthetic 8-file demo generator (real ODA pincodes)
│   ├── generate_demo_upload.py   # 120-row demo upload file generator
│   ├── demo_upload.xlsx          # The visitor upload file
│   └── sample_data/              # 8 generated demo files (Jul 2025 to Feb 2026)
│
├── vercel.json                   # Vercel SPA config (build + rewrite to index.html)
├── render.yaml                   # Render service + persistent disk config
├── tests/                        # 28 pytest tests (app/ pipeline coverage)
├── logisense.db                  # Local SQLite database (gitignored)
└── requirements.txt              # Python deps
```

**Golden rule:** everything in `app/pipeline/` and `app/store/` is LOCKED. The FastAPI
backend imports from it and never modifies it. All 28 tests target this layer.

---

## 4. Core Business Logic (LOCKED)

These rules are the heart of the product. They were refined against real production data and
must never change without sign-off.

### 4.1 TAT (Turn-Around Time)

```
actual_tat = (Delivered Date minus Manifest Date) in calendar days
```

- **Start = Manifest Date** (not Pickup Date). Delhivery takes custody at manifest.
  Manifest is always at or before pickup; about 49% of orders manifest one day before pickup.
- Date-only subtraction: time components are stripped to avoid fractional days.
- Calendar days, not business days.
- The DB column is still named `pickup_date` for the pickup value; only the display label
  changed to "Manifest Date" in tables where manifest drives the calculation.

### 4.2 Zone Matrix (5x5, days)

|            | West | South | North | East | NE |
|------------|------|-------|-------|------|----|
| **West**   | 4    | 6     | 6     | 7    | 10 |
| **South**  | 6    | 4     | 6     | 7    | 10 |
| **North**  | 6    | 6     | 4     | 7    | 8  |
| **East**   | 7    | 7     | 7     | 4    | 6  |
| **NE**     | 10   | 10    | 8     | 6    | 4  |

- Row = origin zone, column = destination zone, value = expected TAT days.
- Stored in `sla_matrix_live`, seeded from `app/reference/matrix.csv`.
- Editable via the Edit section (edits affect FUTURE uploads only; past shipments keep their
  stored `_expected_tat_days`).

### 4.3 ODA (Out of Delivery Area)

- If the destination pincode is ODA: `expected_tat += 1`.
- Source values are normalized at pincode import: `"ODA"` becomes `YES`, `"Normal Service"`
  becomes `NO` (plus tolerant aliases: yes/y/1 become YES, no/n/0 become NO).
- `lookup_oda()` returns `YES`, `NO`, or `UNKNOWN` (UNKNOWN when the pincode is not in the master).
- **UNKNOWN is treated as NO** for Expected TAT: no +1 penalty is applied when the pincode
  can't be found in the master.
- **ODA is data-dependent.** If a dataset's destinations are all metro pincodes, ODA counts
  are legitimately 0. That is correct behavior, not a bug.

### 4.4 Delivery-performance classification (displayed as "E+OT" in the UI)

```
actual_tat <  expected_tat  → Early
actual_tat == expected_tat  → On Time
actual_tat >  expected_tat  → Late

E+OT % = (Early + On Time) / (Early + On Time + Late) × 100
```

- The old "SLA" label is renamed to **E+OT** in ALL UI labels (founders found "SLA" confusing).
- Internal names (`_sla_status`, function names, DB columns) are unchanged.
- The per-order column "SLA Status" displays as "**Delivery Status**".

### 4.5 State to Zone mapping (critical corrections)

| State | Zone | Why this matters |
|---|---|---|
| Chhattisgarh | **East** (not West) | Routes via the Kolkata/Raipur East hub |
| Sikkim | **East** (not NE) | Serviced via the Siliguri corridor |
| Odisha/Orissa | East | Both spellings handled |
| Daman & Diu / Daman and Diu | West | Alias handling |
| Pondicherry / Puducherry | South | Alias handling |
| Jammu & Kashmir / Jammu and Kashmir | North | Alias handling |

### 4.6 Upload behavior: ALWAYS REPLACE

- Every upload **deletes all existing data** before inserting
  (`DELETE FROM shipments_latest` and `shipments_raw`).
- DELETE runs **once per upload session** (before the file loop), so multi-file batches merge
  together and then jointly replace the old data.
- Rationale: founders upload complete Delhivery exports; accumulating periods would produce
  misleading mixed charts.
- After upload: `recompute_all_sla()` runs, the frontend invalidates all React Query caches,
  and (on the sessioned demo) new insights are regenerated for that session.

### 4.7 Transit risk classification

```
days_in_transit = (today minus manifest_date).days   [fallback: pickup_date]
days_remaining  = expected_tat_days minus days_in_transit

days_remaining < 0            → "At Risk (Xd overdue)"
days_remaining == 0           → "Due Today"
days_remaining > 0            → "" (on track, blank)
expected_tat_days IS NULL     → "Pending"
current_status = 'RTO'        → "RTO" (own bucket in /transit endpoints)
```

- The shared classifier lives in `backend/transit_risk.py`, the single source of truth used
  by Transit, Aggregate Transit, and exports.
- **Exception:** `/api/aggregate-transit/company-detail` uses the original 4-bucket date-based
  classification (At Risk/Due Today/On Track/Pending) where RTO orders are bucketed by date.
- The Transit endpoints also apply a 60-day recency window relative to the newest manifest date
  in the data (`manifest_date >= DATE(MAX(manifest_date), '-60 days')`), so a static demo
  dataset still shows in-flight orders.

### 4.8 The `client` vs `order_id` quirk (IMPORTANT)

In real Delhivery exports for aggregators:
- `client` = the Delhivery account name (constant per aggregator account), useless for grouping.
- `order_id` = the actual **end-client company name** (STELLARTECH SYSTEMS, MERIDIAN
  ELECTRICALS, and so on).

**Every backend company filter and grouping uses `order_id`, aliased as `company` in
responses.** `client` appears only as a display column.

---

## 5. The Data Pipeline (`app/`)

### 5.1 Ingest flow (`ingest.py`)

```
Upload .xlsx → validate REQUIRED_COLUMNS {LRN, Current Status, Pickup Date, Remarks}
→ parse 41 columns → normalize → dedup (one winner per LRN, see 5.2)
→ per-row enrichment via compute_row():
    origin city → origin zone      (origin_lookup.py, 3-tier chain)
    dest pincode → dest zone       (zones.py, pincode master then state fallback)
    dest pincode → ODA YES/NO      (oda.py, UNKNOWN = no penalty)
    matrix[origin][dest] + ODA adj → _expected_tat_days
    delivered minus manifest       → _actual_tat_days
    classification                 → _sla_status
→ INSERT into shipments_latest (winners) + shipments_raw (all rows, with a
  uuid batch_id + uploaded_at stamped on every row)
```

- A file missing any REQUIRED_COLUMN is rejected with an error.
- Derived columns are computed OUTSIDE the write transaction (nested cursors deadlock SQLite
  under WAL).
- Enriched columns are prefixed with `_`: `_oda`, `_origin_zone`, `_destination_zone`,
  `_expected_tat_days`, `_actual_tat_days`, `_tat_variance_days`, `_sla_status`.

### The 41-column Delhivery export schema

These are the exact column names as they appear in the raw Delhivery `.xlsx` export, stored
snake_cased in the database. Every pipeline function references them.

| # | Column name (raw) | DB column (snake_case) | Notes |
|---|---|---|---|
| 1 | LRN | lrn | Primary key. Unique shipment ID. |
| 2 | Order id | order_id | **Real company name** for aggregators (not `client`, see 4.8) |
| 3 | No of boxes | no_of_boxes | Integer |
| 4 | Client | client | Delhivery account name, constant per aggregator account, useless for grouping |
| 5 | Manifest Date | manifest_date | TAT clock start. Always at or before Pickup Date. |
| 6 | Pickup Date | pickup_date | Physical pickup from origin |
| 7 | Expected Date | expected_date | Delhivery's own promise date |
| 8 | Invoice Number | invoice_number | Client invoice ref |
| 9 | Consignee name | consignee_name | Recipient name + phone |
| 10 | Origin City | origin_city | Almost always "Aurangabad" for this deployment |
| 11 | Destination City | destination_city | |
| 12 | Client Location/warehouse | client_location_warehouse | |
| 13 | Pick up Address | pick_up_address | Often null |
| 14 | Pin code | pin_code | Destination pincode, drives zone + ODA lookup |
| 15 | Dispatch Count | dispatch_count | Number of dispatch events |
| 16 | First dispatch date | first_dispatch_date | |
| 17 | Last dispatch date | last_dispatch_date | |
| 18 | Last Scan Location | last_scan_location | Hub/city name |
| 19 | Last Scan Date | last_scan_date | Used as a timestamp tie-break in dedup |
| 20 | Current Status | current_status | Manifested / Dispatched / In Transit / Pending / Delivered / RTO |
| 21 | Status Type | status_type | Delivered / Undelivered / Returned |
| 22 | Remarks | remarks | Free-text status detail, a dedup tie-break source |
| 23 | Promise Date | promise_date | Same as Expected Date in most rows |
| 24 | Delivered Date | delivered_date | TAT clock end. Null if not delivered. |
| 25 | Payment Type | payment_type | Pre-paid / COD |
| 26 | Master Waybill | master_waybill | Delhivery waybill number |
| 27 | Additional Remarks | additional_remarks | POD audit notes |
| 28 | Return Promise Date | return_promise_date | RTO rows only |
| 29 | Transaction Type | transaction_type | Usually null |
| 30 | Transaction Mode | transaction_mode | Usually null |
| 31 | First Pending Date | first_pending_date | |
| 32 | Package Amount | package_amount | Declared value |
| 33 | Weight | weight | kg |
| 34 | First attempt date | first_attempt_date | |
| 35 | Last Attempt date | last_attempt_date | |
| 36 | Attempt Count | attempt_count | Delivery attempts |
| 37 | First Return Date | first_return_date | |
| 38 | Invoice Zone | invoice_zone | Delhivery billing zone (B/D1/D2/E) |
| 39 | RVP/ Forward identifier | rvp_forward_identifier | "Forward Shipment" or "Return" |
| 40 | PUR ID | pur_id | Delhivery internal ID |
| 41 | State | state | Destination state |

**Derived columns** (computed at ingest, stored alongside source columns):

| Column | Computed from | Values |
|---|---|---|
| `_origin_zone` | origin_city via the origin_lookup chain | West / South / North / East / NE / null |
| `_destination_zone` | pin_code via pincode master then state fallback | West / South / North / East / NE / null |
| `_oda` | pin_code via pincode_master_live | YES / NO / UNKNOWN |
| `_expected_tat_days` | matrix[origin][dest] + ODA adj | Integer days / null |
| `_actual_tat_days` | delivered_date minus manifest_date (date only) | Integer days / null |
| `_tat_variance_days` | actual minus expected | Signed integer / null |
| `_sla_status` | sign of variance | Early / On Time / Late / null |

### 5.2 Deduplication engine (`dedup.py`)

A Delhivery export can contain multiple snapshots of the same LRN (the shipment at different
lifecycle stages). Dedup picks ONE winner per LRN using a **tie-break ladder**, evaluated in
order:

```
1. Status rank      lifecycle position wins:
                    Manifested(1) < Dispatched(2) < In Transit(3)
                    < Pending(4) < Delivered(5) = RTO(5)
2. Remarks rank     regex keywords on the Remarks column break status ties
                    (for example "Out for Delivery" beats "Reached Hub")
3. Operational time newest of: Last Scan Date > Delivered Date > Pickup Date
4. Batch order      the later upload batch wins as the final tie-break
```

**Regression blocking:** a terminal status can never be downgraded. If the DB has LRN 123 =
Delivered and a new file contains LRN 123 = In Transit, the incoming row is skipped
(`skipped_regressions`). `pick_winner()` and `merge_into_latest()` are pure functions,
testable without a DB.

### 5.3 Architectural invariant: derived columns are STORED

`_expected_tat_days`, `_sla_status`, and the rest are computed **once at ingest** and stored
in the row, never recomputed at read time. Consequence: editing the matrix or pincode master
affects **only future uploads**; historical shipments keep the values they were classified with
(constraint #6, audit trust). The one deliberate exception: the first-ever pincode-master load
triggers `recompute_all_sla()` because rows ingested before the master existed have NULL
zone/ODA data that can now be resolved.

### 5.4 Origin lookup chain (`origin_lookup.py`)

1. `origin_recents` SQLite table (fast path, auto-populated).
2. `origin_city_master.csv` exact match (case-insensitive).
3. `origin_city_master.csv` fuzzy match (difflib, cutoff 0.80).
4. Unknown returns `None`, flagged for a warning.

Successful lookups (steps 2 and 3) upsert into `origin_recents` for future speed.

### 5.5 Seeding (`seed.py`)

On first launch, `seed_all_if_empty()`:
- Seeds the 5x5 matrix from `matrix.csv` into `sla_matrix_live`.
- Seeds the state-to-zone fallback map from `seed.py::STATE_ZONE` into `state_zone_fallback`.
- Seeds ~21,849 pincodes from `pincode_master.xlsx` into `pincode_master_live` (with ODA
  normalization) and triggers `recompute_all_sla()` if newly seeded.

On the deployed demo, seeding is short-circuited: the server copies the pre-built
`demo/demo.db` (which already contains the reference data, 4,017 shipments, and cached
insights) instead of running the full seed, so cold starts finish in well under a second.

---

## 6. The Database

Per-session SQLite (WAL mode). On the demo, each visitor's DB is a copy of `demo/demo.db`
under `/tmp/sessions/<uuid>.db`; locally, the file is `logisense.db` (gitignored).

### Tables

| Table | Purpose |
|---|---|
| `shipments_latest` | One row per LRN, the deduplicated current truth. All dashboards read this. |
| `shipments_raw` | Every uploaded row (pre-dedup) with `batch_id` + `uploaded_at`. Cleared on every upload along with `shipments_latest` (replace semantics, 4.6), so it archives the current upload session only. |
| `pincode_master_live` | ~21,849 pincodes: pincode, city, state, zone, oda (YES/NO). Editable. |
| `sla_matrix_live` | The editable 5x5 TAT matrix. |
| `state_zone_fallback` | State-to-zone map, used when a destination pincode is not in the master. |
| `origin_recents` | Origin-city cache: city_name PK, state, zone, last_seen, seen_count. |
| `uploads` | Upload history (batch_id, filename, row counts). |
| `upload_snapshots` | One lightweight metrics snapshot per upload (totals, E+OT, dates), for the What-Changed digest. |
| `snapshot_companies` | Per-company metrics for each snapshot. |
| `insight_cache` | Narrated insights (digest, patterns, root_causes) as JSON, keyed by snapshot_id. |

### Key `shipments_latest` columns

Source columns (from the Delhivery export, snake_cased): `lrn`, `order_id`, `no_of_boxes`,
`client`, `manifest_date`, `pickup_date`, `expected_date`, `invoice_number`, `consignee_name`,
`origin_city`, `destination_city`, `pin_code`, `current_status`, `status_type`, `remarks`,
`promise_date`, `delivered_date`, `payment_type`, `master_waybill`, `weight`, `package_amount`,
`state`, and the rest (41 total).

Derived columns: `_oda`, `_origin_zone`, `_destination_zone`, `_expected_tat_days`,
`_actual_tat_days`, `_tat_variance_days`, `_sla_status`.

### PRAGMAs and indexes (performance layer)

Every connection is opened by `get_conn()` in `app/store/db.py` with:
```sql
PRAGMA journal_mode=WAL;       -- concurrent read/write
PRAGMA synchronous=NORMAL;     -- faster than FULL, still safe under WAL
PRAGMA cache_size=-64000;      -- 64MB page cache
PRAGMA temp_store=MEMORY;
PRAGMA busy_timeout=5000;      -- wait instead of "database is locked"
PRAGMA foreign_keys=ON;
```

Indexes are defined in `SCHEMA_SQL` (`app/store/db.py`) and created by `init_db()`:
`idx_latest_status`, `idx_latest_pickup`, `idx_latest_company`, `idx_latest_sla_status`,
`idx_raw_lrn`, `idx_raw_batch`, `idx_pincode_city`, `idx_snap_companies`.

---

## 7. Backend API (`backend/`)

Base URL (dev): `http://127.0.0.1:8000`, interactive docs at `/docs`. All data endpoints run
inside the session middleware, so they read and write the caller's own session DB.

### Endpoint reference

| Method | Path | Returns |
|---|---|---|
| GET / HEAD | `/api/health` | `{status, service}`. Static, session-exempt, no DB. For uptime monitors. |
| POST | `/api/upload` | Multipart xlsx upload, returns `{success, rows_inserted}`. Clears all data first, then regenerates insights. |
| GET | `/api/landing/kpis` | Totals, delivered, in_transit, pending, rto, early, on_time, late, eot_count, eot_percent, oda_count, non_oda_count, date_min/max, cod_count. |
| GET | `/api/landing/donut` | `{labels, values, colors}`: Early / On Time / Late / Not Yet Delivered. |
| GET | `/api/landing/trend` | Per-month `{month, total_orders, early, on_time, late}`. |
| GET | `/api/tat/orders` | Delivered orders with all `_` derived columns. |
| GET | `/api/tat/summary` | total_delivered, early/on_time/late, eot%, avg TATs. |
| GET | `/api/tat/oda-chart` | `{oda?, non_oda?}`, an empty group is OMITTED (phantom-bar fix). |
| GET | `/api/transit/orders` | Non-delivered orders + days_in_transit, days_remaining, risk_status. |
| GET | `/api/transit/summary` | total_in_flight, at_risk, due_today, on_track, rto_count, pending_count. |
| GET | `/api/aggregate/companies` | Per-company (by `order_id`): totals, statuses, eot_percent, avg TAT. |
| GET | `/api/aggregate/monthly?company=X` | `[{month, early, on_time, late, not_delivered}]` for the stacked chart. |
| GET | `/api/aggregate-transit/companies` | Per-company in-flight counts, sorted at_risk desc. |
| GET | `/api/aggregate-transit/company-detail?company=X` | `{company, risk_summary, days_overdue_breakdown, orders}`. |
| GET | `/api/customize/orders` | Filtered rows (company, status, sla_status, oda, date_from, date_to, zone). |
| GET | `/api/export/{tat\|transit\|aggregate\|aggregate-transit\|customize}` | Streaming .xlsx with friendly headers. |
| GET | `/api/edit/matrix` | `{zones[5], values[5][5]}`. |
| PUT | `/api/edit/matrix` | Replace the 5x5 matrix (validates 5x5, values 1 to 30). Future uploads only. |
| POST | `/api/edit/matrix/reset` | Restore the matrix from `matrix.csv`. |
| GET | `/api/edit/pincodes?page&per_page&search` | Paginated pincode master. |
| PUT | `/api/edit/pincode` | Toggle a single pincode's ODA (`{pincode, oda}`). |
| POST | `/api/edit/pincodes/reset` | Restore the pincode master from `pincode_master.xlsx`, returns `{rows_reset}`. |
| POST | `/api/edit/pincodes/upload` | Replace the master from a custom .xlsx (pincode/city/state/zone/oda, min 100 rows). |
| GET | `/api/insights/digest` | Cached What-Changed digest (5 bullets) + snapshot metadata. |
| GET | `/api/insights/patterns` | Cached pattern cards, sorted red, yellow, green, grey. |
| GET | `/api/insights/root-cause?company=X` | Cached per-company root-cause facts + narrative, or 404. |
| POST | `/api/assistant/chat` | `{messages[]}` to `{reply, offline, suggestions}`, grounded in cached insights. |
| POST | `/api/assistant/chat/stream` | Same answer delivered incrementally as NDJSON (`{"delta"}` lines, then `{"done", "offline"}`) so the UI types it out. What the chat actually uses. |
| GET | `/api/assistant/suggestions` | The four starter chips. |

### Column display names (`schemas.py`)

```python
COLUMN_DISPLAY_NAMES = {
  "_oda": "ODA",
  "_expected_tat_days": "Expected TAT",
  "_actual_tat_days": "Actual TAT",
  "_tat_variance_days": "TAT Variance",
  "_sla_status": "Delivery Status",
  "_origin_zone": "Origin Zone",
  "_destination_zone": "Destination Zone",
}
```
Used by exports and the frontend column picker. Caching is handled on the frontend by React
Query (5-minute stale time); the backend queries SQLite directly, which is fast given the
indexes and PRAGMAs above.

---

## 8. The AI Insights Engine

The Insights tab is "BI plus AI narration": deterministic SQL statistics find the patterns,
and one LLM call explains them in plain English. It runs after every upload (and once at
demo seed time), and page loads read from cache with zero live API calls.

### 8.1 The 8 detectors (`backend/insights/detectors.py`)

Each detector takes a SQLite connection and returns structured findings (numbers and company
names, never prose). `run_all_detectors()` isolates each one in its own try/except, so a
single failure never blocks the others.

| # | Detector | Fires when |
|---|---|---|
| 1 | Volume decline + late rise | A company's volume drops more than 30% and late rate rises more than 15 points (first half vs last half) |
| 2 | Client churned | A previously active company goes silent (zero in the final month after a real volume cliff) |
| 3 | Volume growth + improvement | Volume up more than 40% and late rate down more than 15 points |
| 4 | ODA structural lateness | ODA late rate is more than 1.5x the non-ODA late rate |
| 5 | Seasonal zone anomaly | Any 3-month window of East/NE lateness exceeds 1.8x the overall late rate |
| 6 | Bad lane | Any pincode with at least 10 adjudicated orders and a late rate above 60% |
| 7 | New client ramp | Zero orders in the first two months, then growing volume |
| 8 | Overall trend | Always fires; describes the month-over-month E+OT arc |

### 8.2 The single Groq call (`groq_narrator.py`)

One call to `llama-3.3-70b-versatile` (temperature 0.2, JSON response) receives all detector
output, per-company stats, the current and previous snapshot, and precomputed root-cause facts.
It returns one JSON document with three parts: `digest` (5 bullets), `patterns` (severity-coded
cards with headlines and stat bullets), and `root_causes` (per flagged company). The prompt
requires company names and specific numbers in every headline, forbids raw field names in
bullets, pins each finding's severity, and demands one card per fired finding.

**Offline fallback.** If `GROQ_API_KEY` is unset or the call fails, `generate_insights()` falls
back to a deterministic narrator that produces the same JSON shape from the detector output.
The tab stays fully functional offline, in CI, and during Groq outages. Set the key to get the
LLM path.

### 8.3 What-Changed digest and snapshots (`snapshot.py`)

After each upload, `write_upload_snapshot()` records a lightweight metrics snapshot (plus
per-company rows). The digest compares snapshot N to snapshot N-1. On the first demo seed, a
synthetic "snapshot zero" is written (slightly worse metrics, NEXUS still active, PRISM
healthier) so the very first digest has a meaningful comparison. `generate_and_cache_insights()`
is the shared orchestrator that both the upload route and the demo seed call.

### 8.4 Caching

Narrated results are stored in `insight_cache` keyed by `snapshot_id`. The three
`/api/insights/*` endpoints are pure cache reads. Regeneration happens only on upload (or the
demo seed), so page loads never wait on the LLM.

---

## 9. The AI Assistant

**Where:** the chat lives at the bottom of the Insights page (inline on desktop, a floating
overlay on mobile). **Endpoint:** `POST /api/assistant/chat/stream` (the blocking
`POST /api/assistant/chat` is kept for compatibility).

### Streaming

Replies are streamed so they type out rather than appearing all at once. The wire format is
NDJSON: `{"delta": "..."}` per fragment, then `{"done": true, "offline": bool}`. The frontend
reads it with a `ReadableStream` and appends each fragment to the live bubble.

Both delivery paths stream. With a working key, Groq tokens are forwarded as they are generated
(`stream: true`), so the first text lands in well under a second instead of after the whole
generation. The offline path has no upstream to stream, so its locally built answer is emitted
word-by-word for the same feel.

The system prompt, including the guardrails below, lives in one `_build_system()` used by both
the streaming and blocking paths, so they cannot drift apart.

### Architecture

The chat is grounded in the same cached insights: it loads the latest digest, patterns,
root-causes, and per-company stats, embeds them in a system prompt, and calls Groq
(`llama-3.3-70b-versatile`). It is context-stuffing, not function-calling: every request
carries a fresh data snapshot, so answers reflect the current upload.

### Topic guardrails

The system prompt includes strict rules: only answer questions about logistics, shipments,
delivery performance, companies, routes, and the given data. Anything off-topic (general
knowledge, math, coding) gets a fixed refusal: "I can only help with questions about your
logistics data." The guardrails are enforced on both the Groq path and the deterministic
offline path (a keyword check).

### Offline fallback

With no `GROQ_API_KEY`, the chat answers from the cached insights deterministically (at-risk
clients, a specific company, ODA impact, what improved), prefixed with an "offline" note. It
never returns a 500.

---

## 10. Sessions & Deployment

### 10.1 Per-session isolation (`backend/session.py` + `SessionMiddleware`)

The public demo is multi-tenant, so every visitor gets an isolated database:

- **`SessionMiddleware`** (pure ASGI, in `backend/main.py`) reads the `logi_session` cookie.
  A missing or invalid cookie mints a new UUID and copies `demo/demo.db` to
  `/tmp/sessions/<uuid>.db`.
- It points `app/store/db.get_conn()` at that file for the whole request via a ContextVar, so
  routers, the shared query helpers, and the ingest pipeline all read and write the visitor's
  own copy with no per-router changes.
- The cookie is `SameSite=None; Secure` in production (so it survives the cross-origin hop from
  Vercel to Render) and `Lax` on localhost. Frontend fetches send `credentials: 'include'`.
- Sessions expire after 24 hours; `cleanup_old_sessions()` sweeps them on startup.
- Raw ASGI (not `BaseHTTPMiddleware`) is deliberate: it guarantees the ContextVar is visible to
  downstream DB calls, including sync endpoints that run in the threadpool.

### 10.2 Session-exempt health endpoint

`EXEMPT_PATHS = ("/api/health", "/docs", "/openapi.json")` bypass the middleware entirely: no
cookie, no session DB, no Set-Cookie. `/api/health` is static (GET and HEAD) and does no DB
work. **Point uptime monitors at `/api/health`, not a data endpoint,** otherwise every
cookieless ping would mint a fresh session DB and fill the disk.

### 10.3 Deployment

- **Frontend on Vercel** (`vercel.json`): builds the Vite app and rewrites all routes to
  `index.html` for the SPA router. `VITE_API_URL` is empty in dev (the Vite proxy forwards
  `/api` to `:8000`) and the Render origin in production, so API calls go straight to the
  backend. Every API call is built through `apiUrl()`.
- **Backend on Render** (`render.yaml`): runs uvicorn, has a **persistent disk** so the DB
  survives redeploys, and takes `GROQ_API_KEY` from the dashboard.
- **Instant startup.** On a fresh disk the server copies the committed `demo/demo.db` (4,017
  shipments plus cached insights) instead of re-seeding, so cold starts finish in about 0.28s.

### 10.4 Demo login gate (client-side only, on purpose)

`Login.jsx` is a demo gate: correct credentials (demo@logisense.app / demo1234) set
`localStorage.logi_auth = "true"`, and a `ProtectedShell` layout route redirects unauthenticated
visitors to `/login`. This is **deliberately not real authentication.** There is no user data to
protect (the demo data is synthetic), and the true isolation boundary is the session database,
not the login. A productized build would add real server-side auth and API-level protection
(see Pending / Future Work).

---

## 11. Frontend (`frontend/`)

### Routing

`App.jsx` uses a `ProtectedShell` layout route: unauthenticated visitors are redirected to
`/login`; everyone else gets the sidebar and mobile-nav chrome around the matched page.

| Route | Page | Nav label / sublabel |
|---|---|---|
| `/login` | Login | (no chrome; the demo gate) |
| `/` | Landing | Landing, Overview |
| `/tat` | TAT | TAT Analysis, Delivered E+OT |
| `/transit` | Transit | Transit, In-flight |
| `/aggregate` | Aggregate | Aggregate, Company breakdown |
| `/aggregate-transit` | AggregateTransit | Aggregate Transit, Per-company in-flight |
| `/customize` | Customize | Customize, Ad-hoc query |
| `/insights` | Insights | AI Insights, Patterns and Chat |
| `/edit` | Edit | Edit, Reference data |
| `/assistant` | (redirect) | Redirects to `/insights` so old bookmarks work |

Sidebar active state: a 3px lavender left border plus a `rgba(177,138,255,0.06)` background.

### Mobile

Below 768px the desktop sidebar is hidden and `MobileNav.jsx` renders a fixed bottom bar
(Landing, Insights, Transit, Menu) plus a slide-up drawer for the rest, each with a Sign out
action. Tables scroll horizontally, KPI grids stack to one column, the Insights digest is
collapsed by default, and the chat becomes a floating button that opens a full-screen overlay.
The breakpoint is shared via `lib/useIsMobile.js` so the JS and CSS flip at the same width.

### The Insights page

`Insights.jsx` renders the `DigestCard` (What-Changed), a `PatternCard` grid (top six by
severity plus a "Show all" toggle) with inline `RootCausePanel` expanders, and the `ChatPanel`
at the bottom.

### Upload animation

`UploadModal.jsx` shows a `lottie-react` animation (`assets/airplane.json`) while a file
processes, then a success state with the row count, and auto-closes after 2.5 seconds.

### Data fetching pattern

```jsx
const { data, isLoading, isError, refetch } = useQuery({
  queryKey: ['landing', 'kpis'],
  queryFn: () => fetchJSON('/api/landing/kpis'),
})
// isError   → error message + Retry button (never stuck skeletons)
// isLoading → <Skeleton /> shimmer
// success && total === 0 → <EmptyState /> (only on confirmed zero rows)
// success && total > 0   → render content
```
QueryClient defaults: `staleTime: 5min`, `gcTime: 10min`, `retry: 1`,
`refetchOnWindowFocus: false`. Upload success calls `queryClient.invalidateQueries()`.

### Landing page KPI layout (12 cards, 4 rows)

```
Row 1 (3 cols): TOTAL ORDERS, DELIVERED (green+bar), IN TRANSIT (blue+bar)
Row 2 (3 cols): PENDING (amber+bar), RTO (red+bar), DATE RANGE (manifest-based)
Row 3 (4 cols): EARLY (green+bar), ON TIME (blue+bar), E+OT (lavender HERO), LATE (red+bar)
Row 4 (2 cols): ODA (lavender), NON-ODA (white)
```
IN TRANSIT counts everything that is not Delivered or RTO. All cards lift on hover only.

### Key component contracts

**KPICard:** `label, value, valueColor, subtext, showBar, barPercent, isDateCard, hero,
className`. `hero` renders the headline-metric treatment (46px value, brand-tinted surface,
resting glow); E+OT is the only card that uses it.

**DataTable:** `columns[{key,label,render?}], data, defaultSort, onExport, sort/onSortChange
(controlled), renderExpanded`. Zebra `#0F0F11`/`#131316`, sticky `#15151A` header, lavender sort
indicator, numeric right-align in JetBrains Mono, a search/download/expand toolbar.

**StatusPill:** value-driven color mapping (see 12). At Risk pills match even with a
"(Xd overdue)" suffix.

**ChartPair:** a fixed top chart plus type (Line/Bar/Pie) and dimension dropdowns, with a
fullscreen expand modal.

**UploadModal:** global singleton via `context/ui.jsx`. Drag-drop, file chips, a replace
warning, Lottie processing, success state.

**ColumnPicker:** lavender-tint pills with an x to hide, a dropdown to re-add, Show all / Reset,
plus Sort By and Asc/Desc. Used on TAT, Transit, Customize.

**icons.jsx:** the shared inline-SVG icon set (no icon library, no emoji). Built from one `ico()`
factory and consumed by the sidebar, mobile nav, chat, digest and upload modal.

---

## 12. Design System

### Colors (from `styles/tokens.js`)

| Token | Hex | Usage |
|---|---|---|
| bg | `#0B0C0D` | Page background |
| surface | `#0F0F11` | Cards, panels |
| surface2 | `#15151A` | Table headers, dropdowns, AI bubbles |
| surface3 | `#1A1A1F` | Hover states |
| border | `#27272A` | All 1px borders |
| borderSoft | `#1F1F23` | Internal dividers, chart gridlines |
| text | `#F8F8F8` | Primary text |
| textDim | `#A1A1AA` | Secondary |
| muted | `#8A8A93` | Labels, captions, axes (meets WCAG AA on every surface) |
| primary | `#B18AFF` | Lavender: brand, values, active nav, E+OT |
| early | `#4ADE80` | Green |
| onTime | `#60A5FA` | Blue |
| late | `#F87171` | Red |
| rto | `#94A3B8` | Grey |
| pending | `#FBBF24` | Amber (Pending status; distinct from the traffic-light yellow) |

Pill backgrounds are the status color at 15% opacity.
**Status colors are semantic and identical across every chart, pill, and cell.**

### Typography

| Role | Spec |
|---|---|
| Page title | 26px / 700 / Inter |
| Card label | 11px / 600 / uppercase / +0.08em / muted |
| Card value | 32px / 700 / JetBrains Mono |
| Table header | 11px / 600 / uppercase / muted |
| Table cell | 13px / Inter (numbers in JetBrains Mono) |

### Severity and status colors

- Pattern severity: red (churn/critical), yellow (watch/anomaly), green (growth), grey (info).
- E+OT % color coding (Aggregate table): `>= 85%` green, `>= 70%` yellow, `< 70%` red.
- TAT Variance: negative green, zero blue, positive red.
- Transit Days Remaining: negative red (overdue), zero amber (due today), positive green.

---

## 13. Performance Optimizations

| Optimization | Where | Effect |
|---|---|---|
| React Query staleTime 5min | `main.jsx` | No refetch on every navigation |
| refetchOnWindowFocus off | `main.jsx` | No refetch on tab switch |
| Insights cached per upload | `insight_cache` | Page loads read cache, zero live LLM calls |
| SQLite perf PRAGMAs | `app/store/db.py` | 64MB cache, MEMORY temp store, NORMAL sync |
| Indexes on hot columns | `SCHEMA_SQL` / `init_db()` | Fast GROUP BY and WHERE |
| Instant startup from demo.db | `main.py` lifespan | About 0.28s cold start instead of a full seed |
| Lazy-loaded Insights route | `App.jsx` | Smaller initial bundle |
| Parallel queries on Landing | `Landing.jsx` | KPIs, donut, and trend fetch simultaneously |

---

## 14. Running the Project

### Prerequisites
- Python 3.x with deps: `pip install -r requirements.txt` (PowerShell may need `python -m pip`).
- Node.js: `cd frontend && npm install`.
- Optional: a Groq key in `backend/.env` for the live LLM path (the app works without it via
  the deterministic fallback).

### Development (two PowerShell windows)

```powershell
# Window 1: backend  (run cd and the command on separate lines, PowerShell has no &&)
cd LogiSense
python -m uvicorn backend.main:app --reload --port 8000

# Window 2: frontend
cd LogiSense/frontend
npm run dev
```
Open **http://localhost:5173**. On first run the server seeds reference data (or copies
`demo/demo.db` if present).

### Production build
```powershell
cd frontend
npm run build        # to frontend/dist, served by FastAPI at :8000 if present
```

### Useful commands
```powershell
# Run tests
python -m pytest tests/ -q          # expect: 28 passed

# Check nothing sensitive is tracked
git ls-files | Select-String -Pattern "\.env$"

# Regenerate the demo upload file (reads real ODA pincodes)
python tools/generate_demo_upload.py

# Rebuild demo/demo.db from the current logisense.db (checkpoint first)
python -c "import sqlite3, shutil; c=sqlite3.connect('logisense.db'); c.execute('PRAGMA wal_checkpoint(TRUNCATE)'); c.close(); shutil.copy('logisense.db','demo/demo.db')"
```

---

## 15. Testing

- **28 pytest tests** in `tests/`, covering the `app/` pipeline: TAT math, delivery-performance
  classification, zone lookups, ODA normalization, dedup, and ingest behavior.
- They do not cover the FastAPI routers or the React frontend (those are browser-verified, and
  new backend features are checked with `TestClient` and curl).
- **Every change must end with `python -m pytest tests/ -q` giving 28 passed.** If a backend
  change breaks a test, the backend change is wrong; `app/` is truth.

---

## 16. Known Quirks & Gotchas

1. **Session DBs live in `/tmp/sessions`.** On Render `/tmp` is ephemeral, so sessions reset on
   redeploy or restart (fine; they expire in 24 hours and `demo/demo.db` is the template).

2. **Uptime monitors must ping `/api/health`, not a data endpoint.** `/api/health` is
   session-exempt and does no DB work. A cookieless ping to a data endpoint would mint a fresh
   6MB session DB each time and fill the disk.

3. **Groq key: one active key per account.** Generating a new key revokes the old one. If the
   chat and narration silently fall back to deterministic output, test the key directly against
   the Groq API (a 401 means the key is invalid, not a code bug). The app never crashes on a bad
   key; it degrades to the deterministic path.

4. **`backend/.env` must contain the whole assignment, not just the key.** Pasting only the raw
   `gsk_...` value is the single most common setup mistake here: dotenv then parses zero
   variables and the key arrives as an empty string (which surfaces as
   `Illegal header value b'Bearer '`, not a 401). The file needs:
   `GROQ_API_KEY=gsk_...` with no quotes and no spaces around `=`. On Render the name and value
   are separate fields, so do not paste `GROQ_API_KEY=` into the value box.

5. **A rejected Groq key is cached for the process lifetime.** A 401 means the key is bad rather
   than a transient error, and retrying it on every message cost a full round-trip before the
   fallback (felt directly as chat latency). The backend therefore stops trying Groq after the
   first 401, so **fixing a key always requires restarting the service**, locally and on Render.

6. **Cross-origin cookies need `SameSite=None; Secure` plus `credentials: 'include'`.** The demo
   frontend (Vercel) and backend (Render) are different origins, so a `Lax` cookie would never be
   sent on XHR and sessions would not persist. Do not "simplify" this to `Lax` in production.

7. **Reference edits are session-scoped.** Toggling a pincode's ODA or editing the matrix writes
   to the caller's session DB, never `logisense.db`. Verifying against `logisense.db` after an
   edit will correctly show no change.

8. **Vite proxy targets `127.0.0.1`, not `localhost`.** Windows resolves localhost to IPv6 while
   uvicorn binds IPv4, causing ECONNREFUSED. Do not change it back to localhost.

9. **`client` column is useless for grouping.** Real company names live in `order_id` (4.8).
   Any new company feature must use `order_id`.

10. **ODA = 0 can be correct.** If the dataset only reaches Normal Service pincodes, zero ODA is
   the truth. Verify with `SELECT _oda, COUNT(*) FROM shipments_latest GROUP BY _oda`.

11. **Every upload wipes previous data** by design (4.6). There is no merge mode.

12. **Two risk classifiers exist deliberately:** the shared `transit_risk.py` (RTO in its own
    bucket) for Transit and exports, and the original 4-bucket date-based classification inside
    aggregate-transit's company-detail.

13. **`.env` must never be committed.** It holds the Groq key. Verify `.gitignore` covers `.env`
    before any push.

---

## 17. Pending / Future Work

| Item | Priority | Notes |
|---|---|---|
| Policy-document RAG layer | High (v2) | Let the assistant answer from uploaded SLA/policy PDFs (a real vector store, not the current context-stuffing) |
| Problem-lanes table in TAT | Medium | A sortable worst-lane table on the TAT page (descriptive BI, kept off the Insights tab on purpose) |
| Sparklines on pattern cards | Nice-to-have | A small volume/late-rate trend inline on each company card |
| Real multi-tenant auth | If productized | Replace the client-side demo gate with server-side auth + API-level protection |

Dropped: the Electron desktop `.exe` (superseded by the hosted web demo).

---

## 18. Migration History

```
v1  Streamlit prototype: the original internal build
v2  FastAPI + React migration
     Phase 1: scaffold + Landing (12 KPI cards)
     Phase 2: all sections + shared components + exports
     UI parity pass: charts, tables, filters, modals
     Perf pass: caching, indexes, lazy loading
     Phase 3: the Groq AI assistant
v3  Insights + deployment (SHIPPED, current production)
     8 SQL detectors + one cached Groq narration
     What-Changed digest + inline root-cause panels
     Vercel (frontend) + Render (API) with a pre-built demo DB
     Per-session DB isolation, demo login gate, mobile responsive
     Editable reference data: matrix, pincode ODA, reset, custom upload
```

**Why the migration happened.** The prototype used a Python-only framework that re-ran the
whole script on every interaction: fragile UI, slow loads, session state lost on reload, poor
mobile support. FastAPI + React gives instant navigation, persistent views, a real API layer
for the AI features, and a clean hosted deployment. All business logic survived untouched: the
28 tests that passed on day one still pass today.

---

## 19. Glossary

| Term | Meaning |
|---|---|
| **LRN** | Unique shipment identifier in Delhivery's data. Primary key across the system. |
| **TAT** | Turn-Around Time in calendar days. |
| **Actual TAT** | `Delivered Date minus Manifest Date` (date-only). |
| **Expected TAT** | `matrix[origin_zone][dest_zone] + (1 if ODA)`. |
| **TAT Variance** | `Actual minus Expected` (signed days). Negative means early. |
| **Delivery Status** | Per-order Early / On Time / Late (internal: `_sla_status`). |
| **E+OT %** | `(Early + On Time) / Delivered × 100`, the headline metric. The UI never says "SLA". |
| **ODA** | Out of Delivery Area, a remote-pincode flag that adds +1 day to Expected TAT. |
| **5x5 matrix** | Zone-to-zone expected-TAT table. Zones: West, South, North, East, NE. |
| **Dedup** | Per-LRN merge of multiple snapshots into one winner via the lifecycle-rank ladder. |
| **Regression block** | A terminal status (Delivered/RTO) can never be downgraded by a later upload. |
| **Risk Status** | Transit classification: At Risk / Due Today / On Track / Pending (plus an RTO bucket). |
| **Manifest Date** | When Delhivery takes custody, the TAT clock start. |
| **RTO** | Return To Origin, a terminal outcome excluded from in-flight counts. |
| **Session isolation** | Each visitor gets a private SQLite DB (a copy of `demo/demo.db`), keyed by the `logi_session` cookie, so uploads and edits never affect other visitors. |
| **Pattern detector** | One of eight SQL functions that scan `shipments_latest` for a specific signal (churn, decline, ODA lateness, and so on) and return structured findings. |
| **Insight cache** | The `insight_cache` table storing narrated digest, patterns, and root-causes as JSON, keyed by snapshot, so page loads make zero live LLM calls. |
| **What-Changed digest** | Five plain-English bullets comparing the current upload's snapshot to the previous one. |

---

*This README is the single source of truth for the current architecture.*
