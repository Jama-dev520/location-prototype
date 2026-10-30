# Location Management Admin Console — Product Requirements Document (PRD)

## 1. Document Information

### 1.1 Basic Information

| Field | Content |
|------|------|
| Document Version | v2.0 |
| Author | Product Team (drafted via requirement clarification session) |
| Created Date | 2026-10-29 |
| Status | Draft — pending review |
| Change Log | v1.0 Initial draft; v1.1 (2026-10-29) duplicate detection changed from exact address match to fuzzy match-score threshold (75%), added OC location ID filter; v1.2 (2026-10-29) published interactive prototype to public GitHub Pages + converted Mermaid flowcharts to rendered PNG images; **v2.0 (2026-10-30) restructured the console from a single merged location list into a two-layer model — Vendor Location List (provider view) + OC Location List (OC-owned view)** — see Change Summary |
| Live Prototype | https://jama-dev520.github.io/location-prototype/ (publicly accessible, no login required) |

### 1.2 Requirement Source Traceability

| Field | Content |
|------|------|
| Requirement Owner | Operations / Location Curation team (OC) |
| Requirement Source | Internal product request — location data management for multi-provider network |
| Source Description | OC manages location data synced daily from two providers (PUDO Inc and LMS) and needs an internal console to curate which locations are visible to downstream customers |
| Original Input Format | Text requirement + two provider API data samples (PUDO Inc DealerList JSON, LMS location list JSON) |
| Requirement Maturity | L2 (semi-formed → clarified) |
| Clarification Rounds | 4 rounds (key decisions resolved) |
| Key Clarification Records | ① Data synced from both providers once per day; ② `status` is provider-sourced business status, `is_enable` is OC's internal switch — a location is visible to downstream customers **only when status = active AND is_enable = true**; ③ Toggling enable requires a dedicated advanced permission; view/export unrestricted for now; ④ Change log must cover both provider status updates and internal enable operations; ⑤ (2026-10-29 follow-up) new location defaults `is_enable = false`; provider "removal" and "deactivation" are distinct; sync changed from full pull to incremental; `location_type` is provider-exclusive; ⑥ (2026-10-29 round 2) sweeper is OC-editable with audit logging; cross-provider address duplicates are flagged for ops confirmation and gate downstream sync via OC location ID; ⑦ (2026-10-29 round 3) duplicate detection is **fuzzy** (weighted match score ≥ 75%); OC location ID added as a list filter; ⑧ **(2026-10-30 round 4 — structural redesign)** the console splits into two menus: **Vendor Location List** (provider view — read-only except sweeper edit; every vendor location generates one OC location ID; the pickup field is renamed to **Sweeper**) and **OC Location List** (OC-owned view — populated by backfilling vendor data; the enable switch lives on the OC location; OC can edit store details such as address and opening hours; when a vendor location's status becomes `inactive`, its linked OC location is automatically disabled and blocked from downstream sync) |
| Requirement Scene Profile | D1: medium clarity · D2: 7+ feature points · D3: B-end internal admin console · D4: structure + operation flow · D5: PC Web · D6: English UI · D7: dual upstream data sources |
| Method Stack Applied | 5W2H + MoSCoW (clarification); User Story + Given-When-Then (PRD); High-fidelity prototype + walkthrough checklist (prototype) |
| Stakeholders | OC operations staff, Advanced-permission operators (enable/disable), Engineering, Downstream customer-facing API consumers |

## 2. Background & Goals

### 2.1 Business Background

OC's location network is aggregated from two upstream providers:

- **PUDO Inc** — supplies dealer locations (PUDOpoints and Lockers) with dealer numbers, addresses, coordinates, operating hours, supported services (FX/CP/TB), and activation lifecycle dates.
- **LMS** — supplies package hub locations with location codes, cutoff times, services (returns / platform_dropoff / retail), restrictions, and holiday closures.

In v1.x the console exposed a single merged catalog where provider locations and OC-managed attributes were mixed on one row. This conflated two different concerns that OC actually needs to manage separately:

1. **The provider's view** — what each provider actually supplied (their raw location records and business status), where the only OC action is assigning who does the physical pickup (the *sweeper*).
2. **OC's own view** — the store entity OC exposes downstream, whose enable/disable switch and edited store details OC owns and controls.

The v2.0 redesign splits these into two distinct layers and two menus, so that:
- Provider data stays clean and read-only (except the sweeper assignment), and
- OC's downstream-visible catalog is a separate, OC-owned layer that OC can edit and switch on/off independently.

### 2.2 Goals

- **Business Goal**: Establish full, auditable control over the downstream-visible location catalog (visible = `status = active AND is_enable = true` on the OC layer), with 100% of visibility changes traceable to an operator or sync event.
- **User Goal**: Operations staff can (a) inspect raw provider locations and set their sweeper, and (b) manage the OC-owned catalog — edit store details and enable/disable — within seconds using rich filters.
- **Product Goal**: Deliver a two-layer location management console: a **Vendor Location List** (provider data) and an **OC Location List** (OC-owned, downstream-facing), with a complete change history per layer.

## 3. Target Users & Scenarios

### 3.1 User Roles

| Role | Characteristics | Core Needs | Permissions |
|------|------|------|------|
| **Viewer** (default, all internal staff) | Ops/support/BD staff who need to look up location info | Search, view details, export, review change history | View both lists / view detail / export / view change log |
| **Location Manager** (advanced permission, granted to a subset of staff) | Trusted operators responsible for location curation | Set sweeper, edit OC store details, toggle `is_enable` for one or many OC locations | All Viewer permissions + sweeper edit + OC store detail edit + single/batch enable/disable |

> Permission provisioning (granting/revoking the advanced permission) is handled outside this console in v1 (e.g., by IT/admin configuration). No role management UI in this release. — *PM supplemented, confirm if acceptable*

### 3.2 Usage Scenarios

- **Scenario 1 — Set a location's sweeper**: A PUDO Inc location is physically picked up by LMS staff. The Location Manager opens the Vendor Location List, edits the Sweeper field from PUDO Inc to LMS, and the change is logged (SWEEPER_CHANGE). The field "Pickup" is now uniformly named **Sweeper** across the console.
- **Scenario 2 — Backfill & activate an OC location**: The nightly sync creates a new vendor location. Because it is not a suspected duplicate, the system auto-generates a corresponding OC location (backfilling name/address/hours from the vendor record) with `is_enable = false`. A Location Manager reviews it in the OC Location List and enables it, making it visible downstream.
- **Scenario 3 — Edit a store detail**: Ops notices an OC location's opening hours are wrong. The Location Manager edits the hours on the OC Location List; the change is stored as an OC-level override and logged (STORE_DETAIL_EDIT), without touching the underlying vendor record.
- **Scenario 4 — Vendor goes inactive → OC auto-disable**: The nightly sync flips a vendor location's status to `inactive`. The linked OC location is automatically disabled and blocked from downstream sync. If a Manager tries to force-enable it, the system raises an error ("vendor location is inactive"). When the provider later reactivates, the OC location can be enabled again.
- **Scenario 5 — Suspected duplicate does not generate an OC location**: The sync flags a new LMS location whose address fuzzy-matches an existing PUDO Inc location. Because it is a suspected duplicate, the system does **not** generate an OC location ID yet — it enters the Duplicate Review queue. Ops confirms "same store" (both link to one OC location) or "not same" (each gets its own OC location).
- **Scenario 6 — Batch suppression for a region**: OC decides to hide all Locker-type OC locations in Brooklyn. Staff filters the OC Location List, selects all results, clicks "Batch Disable", reviews the confirmation dialog, and confirms — each OC location gets an audit entry.
- **Scenario 7 — Daily reconciliation**: After the nightly sync, staff reviews the sync status page and the global change log to see which vendor locations changed and how OC locations were auto-updated (including auto-disable from vendor deactivation).

## 4. Feature Requirements List

> **Priority definitions**: P0 = must-have (MVP) / P1 = important / P2 = deferrable

| ID | Module | Feature | Description | Priority | Depends On | Page | API Ownership | Notes |
|----|--------|---------|-------------|----------|------------|------|---------------|-------|
| F01 | Vendor Location List | Paginated vendor location list | Display raw provider locations with columns: Location ID, Name, Address, Provider, Type, Sweeper, Status, Created Time, Updated Time (+ City/State, row actions) | P0 | — | P01 | New: `GET /api/vendor-locations` | Server-side pagination |
| F02 | Vendor Location List | Filter & search | Filter by Location ID, Name, Address keyword, Provider, Sweeper, Status, Type, City, State, Created/Updated time range (second-level precision) | P0 | F01 | P01 | New: `GET /api/vendor-locations` (query params) | Filters combine with AND logic |
| F03 | OC Location List | Paginated OC location list | Display OC-owned locations with columns: OC Location ID, Name, Address, Provider, Type, Sweeper, Status, Enable, Created/Updated Time, Linked Vendor Location(s) | P0 | — | P02 | New: `GET /api/oc-locations` | Server-side pagination |
| F04 | OC Location List | Filter & search | Filter by OC Location ID, Name, Address keyword, Provider, Sweeper, Status, Enable, Type, City, State, time range | P0 | F03 | P02 | New: `GET /api/oc-locations` (query params) | Filters combine with AND logic |
| F05 | Export | Export current view | Export filtered results (either list) to CSV, columns identical to the list; export follows active filters | P0 | F02/F04 | P01/P02 | New: `GET /api/vendor-locations/export` + `GET /api/oc-locations/export` | Max 10,000 rows per export |
| F06 | Detail | Location detail view | Full profile in tabs: Basic Info / Hours & Services / Change History. Vendor detail is read-only (except sweeper); OC detail additionally shows editable store fields and the linked vendor location(s) | P0 | — | P03 | New: `GET /api/vendor-locations/{id}` + `GET /api/oc-locations/{id}` | Provider-sourced fields read-only |
| F07 | Change History | Per-location change log | Timeline of every change for one location (vendor or OC): operator, time, type, field, old → new; covers provider sync changes, sweeper changes, OC store-detail edits, enable operations, OC link operations | P0 | F06 | P03 | New: `GET /api/locations/{id}/changes` | Chronological, latest first |
| F08 | Enable Control | Single enable/disable toggle | Location Manager toggles `is_enable` on/off on an **OC location**; requires advanced permission; writes audit entry; blocked (with error) when the linked vendor location is `inactive` | P0 | F03 | P02/P03 | New: `PATCH /api/oc-locations/{id}/enable` | Viewers see the control disabled/hidden |
| F09 | Enable Control | Batch enable/disable (two modes) | **Mode A — Inline selection**: select OC rows → "Batch Enable/Disable" → confirmation → execute. **Mode B — Excel bulk upload**: download template (`ocLocationID`, `is_enable`), upload, validate, preview, confirm; each changed OC location gets its own audit entry | P0 | F04/F08 | P02 | New: `POST /api/oc-locations/batch-enable` + `POST /api/oc-locations/batch-enable/upload` + `GET /api/oc-locations/batch-enable/template` | Mode A max 200/batch; Mode B max 5,000 rows, ≤ 5MB; advanced permission required |
| F10 | Sweeper Control | Edit sweeper on vendor location | Location Manager edits the **Sweeper** field from the Vendor Location List/detail; the field is uniformly named "Sweeper" (renamed from "Pickup"); every change writes `SWEEPER_CHANGE` | P0 | F01 | P01/P03 | New: `PATCH /api/vendor-locations/{id}/sweeper` | Advanced permission required |
| F11 | OC Store Edit | Edit OC store details | Location Manager edits OC store details (address, opening hours, and any field ops flags as incorrect) from the OC Location List/detail; stored as OC-level override, not touching the vendor record; writes `STORE_DETAIL_EDIT` | P0 | F03 | P02/P03 | New: `PATCH /api/oc-locations/{id}` | Advanced permission required |
| F12 | OC Generation | Auto-generate OC location from vendor | On sync, a new vendor location that is **not** a suspected duplicate auto-generates one OC location (backfilling name/address/hours from the vendor record) with `is_enable = false` | P0 | F01 | — | System (sync job) | One vendor location → one OC location ID (principle 1:1); cross-provider duplicate → no OC ID yet |
| F13 | Status Propagation | Vendor inactive → OC auto-disable | When a vendor location's status becomes `inactive`, its linked OC location is auto-disabled and blocked from downstream sync; force-enabling raises an error until the vendor is active again | P0 | F12 | P02/P03 | System (sync job) | Enforced on sync and on the enable endpoint |
| F14 | Duplicate Detection | Cross-provider address duplicate review (fuzzy) | Auto-detect cross-provider vendor locations whose addresses fuzzy-match (weighted match score ≥ 75%); flagged pairs do **not** generate OC locations until resolved; ops confirms "same store" (link to one OC location) or "not same" (each gets its own OC location) | P0 | F01 | P04 | New: `GET /api/duplicates` + `POST /api/duplicates/{id}/resolve` | OC location assignment gates downstream sync |
| F15 | Change History | Global change log | Cross-location searchable log with filters (time range, operator, provider, operation type); supplements per-location view | P1 | F07 | P05 | New: `GET /api/changes` | Ops reconciliation entry point |
| F16 | Data Sync | Daily sync status | Show last sync run per provider (time, result, records pulled/created/updated/deactivated/removed) and failure alerts | P1 | F01 | P06 | New: `GET /api/sync/status` | Sync daily 02:00 PUDO / 02:30 LMS; incremental pull |
| F17 | Overview | Dashboard | KPI cards: total vendor locations, total OC locations, visible OC locations, disabled OC locations, provider-deactivated count | P2 | F01/F03 | P07 | New: `GET /api/stats/overview` | Nice-to-have for v1.1 |

## 5. User Flows

### 5.1 Vendor Sync & OC Generation Path

![Vendor Sync & OC Generation Path](https://jama-dev520.github.io/location-prototype/diagrams/flow2-daily-sync.png)

> **Business decisions (confirmed 2026-10-30)**:
> - A newly synced vendor location that is not a suspected duplicate auto-generates **one** OC location (backfilled from the vendor record) with `is_enable = false` — it is NOT visible downstream until a Location Manager enables it.
> - A suspected duplicate (fuzzy match ≥ 75% across providers) does **not** generate an OC location — it enters the Duplicate Review queue.
> - Provider "removal" and "deactivation" are distinct semantics (see §8.2).

### 5.2 Status Propagation — Vendor Inactive → OC Auto-Disable

![Status Propagation Path](https://jama-dev520.github.io/location-prototype/diagrams/flow6-status-propagation.png)

### 5.3 Operator Batch Disable (OC Layer)

![Operator Batch Disable](https://jama-dev520.github.io/location-prototype/diagrams/flow1-batch-disable.png)

### 5.4 Sweeper Edit Path

![Sweeper Edit Path](https://jama-dev520.github.io/location-prototype/diagrams/flow3-sweeper.png)

### 5.5 Duplicate Review Path

![Duplicate Review Path](https://jama-dev520.github.io/location-prototype/diagrams/flow4-duplicate.png)

### 5.6 Exception Path

![Exception Path](https://jama-dev520.github.io/location-prototype/diagrams/flow5-exception.png)

## 6. Page Map

| Page ID | Page Name | Module | Core Functions | Entry | Exit | Data Passed | Feature IDs | Priority |
|---------|-----------|--------|----------------|-------|------|-------------|-------------|----------|
| P01 | Vendor Location List | Vendor Location List | Filter, list, export, sweeper edit | Sidebar "Vendor Location List" | P03 | Vendor Location ID (to detail) | F01/F02/F05/F10 | P0 |
| P02 | OC Location List | OC Location List | Filter, list, export, single/batch enable, store detail edit | Sidebar "OC Location List" (default landing) | P03 | OC Location ID (to detail) | F03/F04/F05/F08/F09/F11 | P0 |
| P03 | Location Detail | Detail (vendor / OC) | Full profile tabs, per-location change history, contextual actions (sweeper edit for vendor; enable + store edit for OC) | P01/P02 row click | P01, P02, P05 | Location ID | F06/F07/F08/F10/F11 | P0 |
| P04 | Duplicate Review | Duplicate Detection | Cross-provider duplicate queue, confirm/reject, OC location assignment | Sidebar "Duplicate Review" | P02/P03 | Pair ID → location IDs | F14 | P0 |
| P05 | Change Log (global) | Change History | Cross-location change log with filters | Sidebar "Change Log" | P03 | Optional location ID prefill | F15 | P1 |
| P06 | Sync Status | Data Sync | Last sync run per provider, results, failure alerts | Sidebar "Sync Status" | P05 | — | F16 | P1 |
| P07 | Overview Dashboard | Overview | KPI cards and trends | Sidebar "Overview" | P01 | — | F17 | P2 |

## 7. Interaction Specifications

### 7.1 P01 — Vendor Location List

- **Entry**: Sidebar "Vendor Location List".
- **Elements**: Filter panel (collapsible), data table (columns: checkbox, Location ID, Name, Address, City/State, Provider, Type, Sweeper, Status, Created Time, Updated Time, Actions), pagination bar (20/page, options 20/50/100), "Export CSV" button, selected-count indicator.
- **Filter fields**: Location ID, Name, Address keyword, City, State/Province, Provider, Sweeper, Status, Type, Created from/to, Updated from/to. All AND-combined.
- **Sweeper semantics**: The field formerly named "Pickup" is now uniformly named **Sweeper**. It identifies who performs the physical pickup for a location. It defaults to `source_provider` but is OC-editable (cross-provider sweep). On this list, an inline edit control (dropdown) is available to Location Managers; Viewers see it read-only.
- **Timestamp precision**: All `Created Time` / `Updated Time` values are displayed, filtered, and exported with **second-level precision** (`YYYY-MM-DD HH:MM:SS`).
- **Interactions**:
  - Filter apply on "Apply"; "Reset" clears all; active filter chips shown above table.
  - Click Location ID or Name → navigate to P03 (vendor detail).
  - Inline Sweeper dropdown (Manager only) → confirm → executes F10.
  - Export downloads `vendor_locations_YYYYMMDD_HHmmss.csv`.
- **States**: Default / loading / empty / error.
- **Data volume**: hundreds to low thousands; server-side pagination + filtering.
- **Validation rules**: Location ID alphanumeric, ≤ 32 chars; time range end ≥ start, max span 1 year.

### 7.2 P02 — OC Location List

- **Entry**: Sidebar "OC Location List"; default landing page after login.
- **Elements**: Filter panel (collapsible), data table (columns: checkbox, OC Location ID, Name, Address, City/State, Provider, Type, Sweeper, Status, Enable, Linked Vendor Location(s), Created Time, Updated Time, Actions), pagination bar, "Export CSV", "Batch Enable" / "Batch Disable", "Batch Upload (Excel)", selected-count indicator.
- **Filter fields**: OC Location ID, Name, Address keyword, City, State/Province, Provider, Sweeper, Status, Enable, Type, Created from/to, Updated from/to. All AND-combined.
- **Enable control**: The `is_enable` switch lives **only** on the OC layer. Toggling is gated by the linked vendor location's status — if the vendor location is `inactive`, the switch is disabled with a tooltip ("vendor location is inactive") and force-enabling raises an error.
- **Store detail edit**: Location Managers can edit store details (address, opening hours, and any field flagged as incorrect) directly from this list (inline) or from P03.
- **Interactions**:
  - Filter apply/reset same as P01.
  - Click OC Location ID or Name → navigate to P03 (OC detail).
  - Row toggle (switch) → confirm popover → executes F08; blocked if vendor inactive.
  - Checkbox selection → batch action bar; "Batch Disable/Enable" → confirmation modal (count + sample IDs + warning) → executes F09 Mode A.
  - "Batch Upload (Excel)" → upload dialog (download template, drag/drop, preview, invalid-row report) → executes F09 Mode B.
  - Inline store-detail edit (Manager only) → executes F11.
  - Export downloads `oc_locations_YYYYMMDD_HHmmss.csv`.
- **States**: Default / loading / empty / error.
- **Validation rules**: Batch upload file `.xlsx`/`.csv`, ≤ 5MB, ≤ 5,000 rows; header exactly `ocLocationID,is_enable`; `is_enable` values `T`/`F`.

### 7.3 P03 — Location Detail

- **Entry**: P01/P02 row click / Actions "View".
- **Vendor detail** (from P01): header (name, Location ID, provider badge, status pill), tabs **Basic Info** / **Hours & Services** / **Change History**; all fields read-only except Sweeper (Manager only).
- **OC detail** (from P02): header (name, OC Location ID, visibility pill "Visible to customers" / "Hidden"), action buttons (Enable/Disable for managers, "Edit"), tabs **Basic Info** / **Hours & Services** / **Change History**; Basic Info shows the linked vendor location(s) and editable store fields (address, opening hours, etc.) with OC-override badge; store edits write `STORE_DETAIL_EDIT`.
- **Change History**: reverse-chronological timeline; each entry: time, operator, operation type badge, field, old → new, remark.
- **States**: Loading / loaded / not-found.
- **Validation rules**: None (read-only except contextual actions).

### 7.4 P04 — Duplicate Review

- **Entry**: Sidebar "Duplicate Review"; badge shows pending pair count.
- **Elements**: Pending review table (columns: Pair, Location A, Location B, Match score, Address diff, Detected, Status, Action), "How it works" explainer, active match-score threshold.
- **Interactions**: Color-coded match score (≥ 95% green, ≥ threshold amber, below red); per-field address diff. Manager clicks "Same store" (link both to one OC location) or "Not same" (each gets its own OC location); both write audit entries (`OC_LINK_CONFIRMED` / `OC_LINK_REJECTED`). Until resolved, no OC location is generated.
- **States**: Pending / empty.

### 7.5 P05 — Change Log (Global) (P1)

- **Entry**: Sidebar "Change Log".
- **Elements**: Filters (time range, operator, provider, operation type, location ID), log table, pagination.
- **Interactions**: Click location → P03; filter apply/reset.

### 7.6 P06 — Sync Status (P1)

- **Entry**: Sidebar "Sync Status".
- **Elements**: One card per provider (PUDO Inc, LMS): last sync time, status, records pulled, created/updated/deactivated counts, error message; "Sync history" table (last 30 runs).
- **Interactions**: Refresh; failed run expands to show error detail.

### 7.7 P07 — Overview Dashboard (P2)

- **Entry**: Sidebar "Overview".
- **Elements**: KPI cards (Total Vendor Locations, Total OC Locations, Visible OC Locations, Disabled OC Locations, Provider-Deactivated), provider split, 30-day enable/disable trend.

## 8. Data Requirements

### 8.1 Core Data Objects

| Object | Key Fields | Source | Notes |
|--------|-----------|--------|-------|
| **Vendor Location** (provider layer) | vendor_location_id (internal), source_provider (PUDO/LMS), provider_ref (dealerNo or loc_id), store_code, name, location_type (PUDOpoint/Locker/packagehub), sweeper (PUDO/LMS — who performs the physical pickup), address_line1/2, city, state, postal_code, country, latitude, longitude, phone, timezone, status (active/inactive/removed), services[], restrictions[], hours_of_operation{}, cutoff_time, holiday_closures[], created_at, updated_at, activated_at, deactivated_at, last_synced_at | PUDO Inc + LMS daily sync | Provider-sourced; read-only except `sweeper`; `location_type` provider-exclusive; `sweeper` defaults to `source_provider` and is OC-editable |
| **OC Location** (OC layer) | oc_location_id (OC-owned, gates downstream sync), name, address_line1/2, city, state, postal_code, country, latitude, longitude, phone, opening_hours{}, store details (editable), is_enable (bool), status (derived), created_at, updated_at, linked_vendor_location_ids[] | Backfilled from Vendor Location; OC-editable store details | The downstream-visible entity; enable switch lives here; store details backfilled from vendor then OC-overridable; 1:N mapping to vendor locations (principle 1:1) |
| **Vendor ↔ OC Mapping** | mapping_id, vendor_location_id, oc_location_id, created_at | This system | Links vendor location(s) to the OC location; principle 1:1, but a confirmed duplicate pair links 2 vendor locations → 1 OC location |
| **Change Log Entry** | log_id, location_id, location_layer (VENDOR/OC), occurred_at, operator, operation_type (ENABLE / DISABLE / BATCH_ENABLE / BATCH_DISABLE / SWEEPER_CHANGE / STORE_DETAIL_EDIT / OC_LINK_CONFIRMED / OC_LINK_REJECTED / OC_GENERATED / PROVIDER_STATUS_CHANGE / PROVIDER_FIELD_UPDATE / PROVIDER_CREATED / PROVIDER_REMOVED), field_name, old_value, new_value, source (MANUAL/SYNC), remark | This system | Append-only, immutable |
| **Duplicate Pair** | pair_id, vendor_location_a, vendor_location_b, matched_on, match_score, diff, status (pending/confirmed/rejected), detected_at, resolved_by, resolved_at | This system | Cross-provider (LMS ↔ PUDO Inc) only |
| **Sync Run** | run_id, provider, started_at, finished_at, result, records_pulled, created_count, updated_count, deactivated_count, error_message | This system | One record per provider per day |
| **User** | user_id, name, email, has_enable_permission (bool) | Corporate SSO | No self-service role management in v1 |

### 8.2 Data Rules

- **Two-layer model (core, v2.0)**: The console has two distinct layers — **Vendor Location** (raw provider data) and **OC Location** (OC-owned, downstream-facing). Provider data is read-only (except sweeper); OC controls visibility and store details on its own layer.
- **Visibility rule (core)**: An OC location is synced downstream **iff `status = active` AND `is_enable = true`**. The `status` here is derived from the linked vendor location(s). When the vendor location is `inactive`, the OC location is auto-disabled and cannot be force-enabled.
- **Field ownership**:
  - **Vendor layer**: `status` and all business attributes owned by providers (read-only, refreshed by daily sync); `sweeper` owned by OC (writable via F10).
  - **OC layer**: `is_enable` owned by OC (writable via F08/F09); store details (address, opening hours, etc.) owned by OC (writable via F11); `oc_location_id` owned by system.
- **Sweeper (renamed from Pickup)**: The field formerly named "Pickup"/`pickup_provider` is now uniformly **Sweeper** (`sweeper`). It identifies who performs the physical pickup. Defaults to `source_provider`, may differ (cross-provider sweep). Writable on the vendor layer via F10; every change writes `SWEEPER_CHANGE`.
- **OC generation (v2.0)**: On sync, each new vendor location that is **not** a suspected duplicate auto-generates one OC location (backfilled from the vendor record) with `is_enable = false`, logged as `OC_GENERATED`. A suspected duplicate (fuzzy match ≥ 75% across providers) does **not** generate an OC location — it enters the Duplicate Review queue.
- **Mapping (1:N)**: Principle is 1 vendor location → 1 OC location (1:1). A confirmed cross-provider duplicate links 2 vendor locations → 1 OC location (2:1). A rejected duplicate assigns independent OC locations (1:1 each).
- **Status propagation (v2.0)**: When a vendor location's status becomes `inactive`, its linked OC location is auto-disabled and blocked from downstream sync. Force-enabling raises an error ("vendor location is inactive"). When the provider reactivates the vendor location (status → `active`), the OC location can be enabled again.
- **Removal vs deactivation**: Provider "removal" (location disappears from provider's list) = permanent delisting, `status = removed`, logged `PROVIDER_REMOVED`. Provider "deactivation" (`status = inactive`) = temporary closure, location remains, logged `PROVIDER_STATUS_CHANGE`.
- **Duplicate detection (fuzzy)**: Cross-provider duplicate = two vendor locations from different providers whose addresses fuzzy-match above a threshold (weighted match score: address_line1 50% + postal_code 25% + city 15% + state 10%, token Jaccard; threshold 75%).
- **Sync does not reset OC fields**: Provider sync never overwrites OC-owned fields (`is_enable`, OC store details, `oc_location_id`). It only updates vendor-owned fields and propagates status.
- **Change log completeness**: Every write produces exactly one log entry. Batch operations produce one entry per changed location; no-op rows skipped.
- **Batch upload rules**: Template has exactly 2 columns — `ocLocationID`, `is_enable`. `T` = enable, `F` = disable. Duplicate IDs: last wins. Unknown IDs rejected. Synchronous ≤ 5,000 rows with result summary.
- **Audit immutability**: Change log entries never edited or deleted.
- **ID stability**: Internal `vendor_location_id` and `oc_location_id` are permanent.

### 8.3 External System Integration

> **⚠️ Sync architecture is INCREMENTAL (confirmed 2026-10-29).** See §8.5.

| External System | Interface Purpose | Data Flow | Sync/Async | Integration Status | Notes |
|----------------|-------------------|-----------|------------|-------------------|-------|
| PUDO Inc Dealer API | Pull INCREMENTAL dealer changes (daily) | In: auth + `updated_since` cursor → Out: changed DealerList records | Async scheduled job (daily 02:00) | Existing API; incremental endpoint to be confirmed | Delta keyed on `LastUpdatedDate`; fallback full pull |
| LMS Location API | Pull INCREMENTAL location changes (daily) | In: auth + cursor + `updated_since` → Out: changed location records | Async scheduled job (daily 02:30) | Existing API; incremental endpoint to be confirmed | Iterate `has_more` for changed-record window |
| Customer-facing Location Query API (downstream) | Serves visible OC locations to downstream customers | Reads OC Location table with `status=active AND is_enable=true` filter | Sync (per request) | Existing | Enable/disable and status propagation take effect on next query |
| Corporate SSO | Staff authentication | In: credentials → Out: user identity + email | Sync | Existing | Groups → advanced permission mapping TBD |

### 8.5 Sync Failure Fallback & Incremental Strategy

**Incremental pull scope (confirmed 2026-10-29):**

- Each daily run pulls only records changed since the last successful sync (keyed on `updated_since` cursor).
- Scale bound: incremental window is a small fraction of the catalog.
- If a provider only supports full pull, fall back to full pull + local diff for that provider only.

**Sync failure fallback (confirmed 2026-10-29):**

- On failure, the local catalog is NOT cleared or partially updated — the last successful snapshot is preserved.
- Failed run marked `FAILED` on P06; `last_synced_at` not advanced.
- One automatic retry; then wait for next cycle.
- Incremental cursor advanced only on fully successful run.

### 8.4 Non-Functional Requirements

| Dimension | Requirement | Target | Priority |
|-----------|------------|--------|----------|
| Performance | List query p95 latency | ≤ 2s with filters, ≤ 5s export (10k rows) | P0 |
| Availability | Internal console SLA | 99.5% business hours; sync job must not block console | P0 |
| Security | Access control | SSO login mandatory; enable/disable + store edit gated by advanced permission; audit log immutable | P0 |
| Compatibility | Browser | Chrome / Edge latest two major versions, 1440px+ desktop | P0 |
| Scalability | Data growth | Support 100k vendor locations, 100k OC locations, 1M change log entries | P1 |
| Compliance | Data handling | Location data is business (non-PII); operator identity retained for audit | P1 |

## 9. Non-Goals

- ❌ No editing of provider-owned business data on the **vendor layer** (address, hours, services) — vendor layer is read-only except sweeper.
- ❌ No location creation/deletion by internal users on the vendor layer — vendor catalog is provider-driven (enter/exit only via sync).
- ❌ OC store-detail editing is limited to OC-owned fields (address, opening hours, and flagged fields) — it does **not** rewrite provider source records.
- ❌ No end-customer-facing UI — downstream customers consume locations through the existing query API.
- ❌ No real-time push/webhook to downstream customers.
- ❌ No role management UI in v1.
- ❌ No non-English localization in v1.
- ❌ No provider management UI in v1.

## 10. Acceptance Criteria

> Given-When-Then format; all criteria marked "suggest PM review before finalizing".

### 10.1 Functional Acceptance Mapping

| ID | Feature | Happy Path | Boundary | Exception Data | Verification | Priority |
|----|---------|-----------|----------|----------------|--------------|----------|
| F01/F03 | Vendor & OC lists | ①Open P01 → both-provider vendor rows shown; ②Open P02 → OC rows shown with all columns | 0 rows → empty state; 100/page renders cleanly | Missing address2/phone renders "—" | UI walkthrough + API comparison | P0 |
| F02/F04 | Filter & search | ①Filter Provider=LMS, Status=active → only matching rows; ②chips visible; ③Reset restores | Combined filters AND-logic; time range crossing year | Invalid Location ID rejected inline | UI walkthrough | P0 |
| F05 | Export | ①Apply filters → ②Export → CSV matches filtered set | No filter = full catalog (≤10k); 10,001 → prompt to narrow | Null fields export as empty cells | File open + row count | P0 |
| F06 | Detail | ①Vendor detail read-only (except sweeper); ②OC detail shows editable fields + linked vendor | Empty hours/restrictions → empty-state text | Non-existent ID → not-found + back | UI walkthrough | P0 |
| F07 | Change log | ①Entries reverse-chronological with operator/time/type/field/old→new | Zero changes → creation entry only; 20+ paginate | Sync entries show "System · {Provider} Sync" | Seeded records | P0 |
| F08 | Single enable | ①Manager toggles OC location → confirm → `is_enable` flips + audit entry; ②Vendor inactive → toggle blocked with error | Toggle on already-enabled = no-op toast | API failure → error toast, state unchanged | Permission A/B + audit check | P0 |
| F09 | Batch enable/disable | **Mode A**: select 3 OC rows → disable → modal → confirm → 3 flip + 3 entries. **Mode B**: upload 100-row file → preview → confirm → applied + entries | 200 max (Mode A); 5,000 rows (Mode B); duplicate IDs last-wins; no-op rows skipped | Unknown ID / invalid T/F / wrong header / oversized → row-level error report | Seeded batch test | P0 |
| F10 | Sweeper edit | ①Manager edits Sweeper on vendor location → confirm → saved + `SWEEPER_CHANGE` entry | Viewer sees read-only; selecting current value = no-op | API failure → value unchanged | Permission A/B + audit | P0 |
| F11 | OC store edit | ①Manager edits OC address/hours → saved + `STORE_DETAIL_EDIT` entry; vendor record unchanged | Viewer sees read-only | API failure → value unchanged | Permission A/B + audit | P0 |
| F12 | OC generation | ①New vendor location (non-duplicate) → one OC location backfilled with `is_enable=false` + `OC_GENERATED` entry | Suspected duplicate → no OC location, enters review queue | — | Trigger test sync | P0 |
| F13 | Status propagation | ①Vendor → `inactive` → OC auto-disabled + blocked; ②force-enable raises error; ③vendor → `active` → OC can enable again | — | Provider reactivation mid-cycle | Trigger test sync | P0 |
| F14 | Duplicate review | ①Pair at ≥75% → shown with score + diff; ②"Same store" → link to one OC location; ③"Not same" → independent OC locations | Score below threshold not surfaced; identical address = 100% | Unit-suffix/postal-extension differences still match | Seeded pair + audit | P0 |

### 10.2 End-to-End Flow Acceptance

- [ ] Core path walkable: filter → select OC rows → batch disable → confirm → audit entries → location hidden downstream.
- [ ] Sync path walkable: nightly sync → vendor status change → OC auto-disable → log entry → staff reviews.
- [ ] OC generation path: new vendor location → OC backfilled (disabled) → Manager enables → visible downstream.
- [ ] All P0 features clickable in the prototype.

## 11. Success Metrics

### 11.1 Metric Definitions

| Level | Metric | Definition | Baseline | Target | Cadence | Source |
|-------|--------|-----------|----------|--------|---------|--------|
| 🎯 North Star | Downstream catalog accuracy | % of OC locations whose effective visibility matches OC's intent | — | ≥ 99.5% | Monthly | Change log + spot audit |
| 📊 Core | Audit coverage of visibility changes | % of visibility changes with complete record | 0% | 100% | Monthly | Change log |
| 📊 Core | Daily sync success rate | % of days both provider syncs succeed | — | ≥ 99% | Monthly | Sync run records |
| 📊 Core | Time-to-locate | Median time to find a location via filters | ~minutes | < 30s | Quarterly | Usability sampling |
| 📊 Core | Batch operation adoption | % of bulk changes via batch vs one-by-one | — | ≥ 80% | Monthly | Change log |
| 📈 Auxiliary | Export usage | Exports per week | — | Track only | Weekly | Telemetry |
| 📈 Auxiliary | List query p95 latency | 95th percentile list API latency | — | ≤ 2s | Monthly | APM |

### 11.2 Measurement Plan

- T+7: verify audit coverage = 100% of visibility changes.
- T+30: first monthly reconciliation; review sync success rate.
- T+90: evaluate time-to-locate and batch adoption; decide v1.1 scope.

## 12. Risks & Dependencies

| Risk / Dependency | Type | Impact | Mitigation |
|-------------------|------|--------|------------|
| PUDO Inc / LMS API schema drift | Risk | Sync breaks, stale catalog | Contract tests; sync failure alerts (F16) |
| Provider incremental delta endpoint unavailable | Dependency | Fall back to full pull for that provider | Confirm `updated_since` capability; fallback flagged |
| Sync failure leaves stale catalog | Risk | Staff/customers act on outdated data | Last-success snapshot preserved; freshness indicator |
| SSO group → advanced permission mapping undefined | Dependency | Enable/edit control cannot ship | Confirm with IT before F08/F09/F10/F11 |
| Batch misuse (mass accidental disable) | Risk | Downstream catalog suddenly shrinks | Confirmation dialog; batch cap 200/5,000 |
| Malformed bulk upload file | Risk | Batch partially applied | Strict validation; row-level error report; dry-run preview |
| Data volume exceeds 10k export cap | Risk | Export blocked | Prompt to narrow filters |
| Timezone handling | Risk | Confusing displayed times | Normalize to console timezone (UTC) with labels |
| OC/vendor drift (OC overrides diverge from provider) | Risk | OC store details stale vs provider | Re-backfill on major provider field change (flagged for ops review) |

## 13. Appendix

### 13.1 Glossary

| Term | Definition |
|------|-----------|
| OC | Operations / Location Curation team — internal owner of this console |
| PUDO Inc | Upstream provider #1 — supplies dealer locations (PUDOpoints / Lockers) |
| LMS | Upstream provider #2 — supplies package hub locations |
| Vendor Location | The raw provider-sourced location record (PUDO Inc / LMS); read-only except Sweeper |
| OC Location | The OC-owned, downstream-facing store entity; backfilled from vendor, OC-editable, carries the enable switch |
| Sweeper | Who physically picks up packages from a location (renamed from "Pickup"/`pickup_provider`); defaults to source_provider, OC-editable |
| PUDOpoint / Locker | PUDO Inc location types |
| packagehub | LMS location type |
| status | Provider-sourced business status (active / inactive / removed) |
| is_enable | OC-controlled switch on the OC location; determines downstream visibility |
| removed | Terminal provider state — permanently delisted |
| inactive | Temporary provider state — store closed / on break |
| OC location ID | OC-owned identifier on the OC layer; gates downstream sync |
| Duplicate pair | Two cross-provider vendor locations whose normalized addresses fuzzy-match above threshold |
| Match score | Weighted 0–100% address similarity (addr1 50% + postal 25% + city 15% + state 10%); ≥ 75% flags duplicate |
| Visible to customers | OC location with `status = active` AND `is_enable = true` |
| Incremental sync | Daily pull of only changed records since last success |
| FX / CP / TB | PUDO Inc service codes |
| Cutoff time | LMS — latest drop-off time for same-day processing |

### 13.2 References

- PUDO Inc DealerList API sample (provided 2026-10-29)
- LMS Location List API sample (provided 2026-10-29)

### 13.3 Prototype Link

- `Location-Management-Admin-Prototype-2026-10-29.html` (generated after PRD confirmation)

### 13.4 Test Requirements

| Dimension | Requirement | Notes |
|-----------|------------|-------|
| External system test environments | PUDO Inc / LMS sandbox endpoints TBD | If unavailable, mock provider APIs |
| Test data source | Seed from samples + ~500 synthetic locations across both providers | Include edge cases: empty hours, missing address2, inactive status |
| Test accounts | 1 Viewer + 1 Location Manager | Per §3.1 |
| Mock requirements | Mock nightly sync trigger; mock partial batch failure; mock vendor→inactive transition | Verify F09 partial-failure, F13 auto-disable |
