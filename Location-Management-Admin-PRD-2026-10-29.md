# Location Management Admin Console — Product Requirements Document (PRD)

## 1. Document Information

### 1.1 Basic Information

| Field | Content |
|------|------|
| Document Version | v1.1 |
| Author | Product Team (drafted via requirement clarification session) |
| Created Date | 2026-10-29 |
| Status | Draft — pending review |
| Change Log | v1.0 Initial draft; v1.1 (2026-10-29) duplicate detection changed from exact address match to fuzzy match-score threshold (75%), added OC location ID filter; v1.2 (2026-10-29) published interactive prototype to public GitHub Pages + converted Mermaid flowcharts to rendered PNG images |
| Live Prototype | https://jama-dev520.github.io/location-prototype/ (publicly accessible, no login required) |

### 1.2 Requirement Source Traceability

| Field | Content |
|------|------|
| Requirement Owner | Operations / Location Curation team (OC) |
| Requirement Source | Internal product request — location data management for multi-provider network |
| Source Description | OC manages location data synced daily from two providers (PUDO Inc and LMS) and needs an internal console to curate which locations are visible to upstream customers |
| Original Input Format | Text requirement + two provider API data samples (PUDO Inc DealerList JSON, LMS location list JSON) |
| Requirement Maturity | L2 (semi-formed → clarified) |
| Clarification Rounds | 1 round (4 key decisions resolved) |
| Key Clarification Records | ① Data synced from both providers once per day; ② `status` is provider-sourced business status, `is_enable` is OC's internal switch — a location is visible to upstream customers **only when status = active AND is_enable = true**; ③ Toggling enable requires a dedicated advanced permission; view/export unrestricted for now; ④ Change log must cover both provider status updates (via daily sync) and internal enable operations; ⑤ (2026-10-29 follow-up) new location defaults `is_enable = false`; provider "removal" and "deactivation" are distinct semantics; sync changed from full pull to incremental; `location_type` is provider-exclusive; ⑥ (2026-10-29 round 2) sweeper is OC-editable with audit logging; cross-provider address duplicates are flagged for ops confirmation and gate upstream sync via OC location ID (upstream sync = has OC ID + active + enabled); ⑦ (2026-10-29 round 3) duplicate detection is **fuzzy** — addresses never match exactly, so a weighted match score (≥ 75%) flags potential duplicates instead of exact equality; OC location ID added as a list filter |
| Requirement Scene Profile | D1: medium clarity · D2: 7+ feature points · D3: B-end internal admin console · D4: structure + operation flow · D5: PC Web · D6: English UI · D7: dual upstream data sources |
| Method Stack Applied | 5W2H + MoSCoW (clarification); User Story + Given-When-Then (PRD); High-fidelity prototype + walkthrough checklist (prototype) |
| Stakeholders | OC operations staff, Advanced-permission operators (enable/disable), Engineering, Upstream customer-facing API consumers |

## 2. Background & Goals

### 2.1 Business Background

OC's location network is aggregated from two upstream providers:

- **PUDO Inc** — supplies dealer locations (PUDOpoints and Lockers) with dealer numbers, addresses, coordinates, operating hours, supported services (FX/CP/TB), and activation lifecycle dates.
- **LMS** — supplies package hub locations with location codes, cutoff times, services (returns / platform_dropoff / retail), restrictions, and holiday closures.

Today there is no unified internal tool to manage this merged catalog. Operations staff cannot efficiently search locations, cannot control which locations are exposed to upstream customers, and has no audit trail when a location's visibility changes (either because a provider flipped its status during the nightly sync, or because an internal operator toggled the enable switch). This creates two concrete pain points:

1. **No controllable visibility switch** — upstream customers can see everything the providers mark active, even locations OC wants to suppress.
2. **No accountability** — when a location unexpectedly appears or disappears upstream, nobody can answer "who changed what, when".

### 2.2 Goals

- **Business Goal**: Establish full, auditable control over the upstream-visible location catalog (visible = `status = active AND is_enable = true`), with 100% of visibility changes traceable to an operator or sync event.
- **User Goal**: Operations staff can find any location within seconds using rich filters, inspect its full details, and enable/disable single or multiple locations in one confirmed action.
- **Product Goal**: Deliver a unified location management console consolidating PUDO Inc + LMS data into one normalized model with a complete change history per location.

## 3. Target Users & Scenarios

### 3.1 User Roles

| Role | Characteristics | Core Needs | Permissions |
|------|------|------|------|
| **Viewer** (default, all internal staff) | Ops/support/BD staff who need to look up location info | Search, view details, export, review change history | View list / view detail / export / view change log |
| **Location Manager** (advanced permission, granted to a subset of staff) | Trusted operators responsible for location curation | Toggle `is_enable` for one or many locations | All Viewer permissions + single enable/disable + batch enable/disable |

> Permission provisioning (granting/revoking the advanced permission) is handled outside this console in v1 (e.g., by IT/admin configuration). No role management UI in this release. — *PM supplemented, confirm if acceptable*

### 3.2 Usage Scenarios

- **Scenario 1 — Investigating an upstream complaint**: A customer reports a location appears in their app but is permanently closed. Staff searches by location ID/name/address, opens the detail page, checks the change history, sees the provider deactivated it in last night's sync, and disables it in one click if needed.
- **Scenario 2 — Batch suppression for a region**: OC decides to temporarily hide all Locker-type locations in Brooklyn. Staff filters provider = PUDO Inc, type = Locker, city = Brooklyn, selects all results, clicks "Batch Disable", reviews the confirmation dialog (count + affected IDs), and confirms. Each location gets an audit entry.
- **Scenario 3 — Daily data reconciliation**: After the nightly sync, a staff member reviews the sync status page and the global change log to see which locations the providers activated/deactivated/updated.
- **Scenario 4 — Export for offline analysis**: Finance requests a full list of enabled locations in NY. Staff filters state = NY, enable = enabled, exports the current view to CSV.
- **Scenario 5 — Reassign pickup (sweeper)**: A PUDO Inc location is physically picked up by LMS staff. The Location Manager opens the detail page, edits the pickup provider to LMS, and the change is logged with operator/time/old→new.
- **Scenario 6 — Resolve a cross-provider duplicate**: The nightly sync flags a new LMS location whose address fuzzy-matches an existing PUDO Inc location at 75% similarity (differing only in a unit suffix and postal extension). Ops opens Duplicate Review, inspects the highlighted field differences, verifies they are the same physical store, and confirms — both locations are linked to one OC location ID and become eligible for upstream sync.

## 4. Feature Requirements List

> **Priority definitions**: P0 = must-have (MVP) / P1 = important / P2 = deferrable

| ID | Module | Feature | Description | Priority | Depends On | Page | API Ownership | Notes |
|----|--------|---------|-------------|----------|------------|------|---------------|-------|
| F01 | Location List | Paginated location list | Display merged locations from both providers with columns: Location ID, OC ID, Name, Address, Provider, Pickup Provider, Status, Enable, Created Time, Updated Time (+ supplemented: Type, City/State, row actions) | P0 | — | P01 | New: `GET /api/locations` | Server-side pagination |
| F02 | Location List | Filter & search | Filter by Location ID, OC location ID, Name, Address keyword, Provider, Pickup Provider, Status, Enable, Created/Updated time range (second-level precision); supplemented: Type, City, State, Postal Code, Services | P0 | F01 | P01 | New: `GET /api/locations` (query params) | Filters combine with AND logic; `created_at`/`updated_at` are full timestamps |
| F03 | Location List | Export current view | Export filtered results to CSV, columns identical to list; export follows active filters | P0 | F02 | P01 | New: `GET /api/locations/export` | Max 10,000 rows per export |
| F04 | Location Detail | Location detail view | Full profile in tabs: Basic Info (identity, address, coordinates, timezone, lifecycle dates), Hours & Services (operating hours, services, restrictions, cutoff, holiday closures), Change History | P0 | — | P02 | New: `GET /api/locations/{id}` | Provider-sourced fields are read-only |
| F05 | Change History | Per-location change log | Timeline of every change for one location: operator, operation time, operation type, changed field, old value → new value; covers provider sync changes, internal enable operations, and sweeper/pickup-provider changes | P0 | F04 | P02 | New: `GET /api/locations/{id}/changes` | Chronological, latest first |
| F06 | Enable Control | Single enable/disable toggle | Location Manager toggles `is_enable` on/off from the list row or detail page; requires advanced permission; writes audit entry | P0 | F01 | P01/P02 | New: `PATCH /api/locations/{id}/enable` | Viewers see the control disabled/hidden |
| F07 | Enable Control | Batch enable/disable (two modes) | **Mode A — Inline selection**: select rows via checkboxes → "Batch Enable/Disable" → confirmation dialog (affected count, sample IDs, warning) → execute. **Mode B — Excel bulk upload**: download template (2 columns: `locationID`, `is_enable` with values T/F), fill and upload, system validates the file, shows a parsed-row preview, user confirms execution; each changed location gets its own audit entry | P0 | F02/F06 | P01 | New: `POST /api/locations/batch-enable` (Mode A) + `POST /api/locations/batch-enable/upload` (Mode B) + `GET /api/locations/batch-enable/template` | Mode A max 200/batch; Mode B max 5,000 rows/file, file ≤ 5MB; advanced permission required for both modes |
| F08 | Change History | Global change log | Cross-location searchable log of all changes with filters (time range, operator, provider, operation type); supplements per-location view | P1 | F05 | P03 | New: `GET /api/changes` | Ops reconciliation entry point |
| F09 | Data Sync | Daily sync status | Show last sync run per provider (time, result, records pulled/created/updated/deactivated/removed) and failure alerts | P1 | F01 | P04 | New: `GET /api/sync/status` | Sync runs daily (02:00 PUDO / 02:30 LMS); incremental pull with fallback (§8.5) |
| F10 | Overview | Dashboard | KPI cards: total locations, visible (has OC ID + active + enabled) count, disabled count, provider-deactivated count, trend of enable/disable operations | P2 | F01 | P05 | New: `GET /api/stats/overview` | Nice-to-have for v1.1 |
| F11 | Pickup Control | Edit sweeper / pickup provider | Location Manager edits the pickup provider (`pickup_provider`) from the detail page; the field defaults to `source_provider` but can be set to the other provider (cross-provider sweep); every change writes an audit entry (`SWEEPER_CHANGE`) | P0 | F04 | P02 | New: `PATCH /api/locations/{id}/pickup-provider` | Advanced permission required; audit entry per change |
| F12 | Duplicate Detection | Cross-provider address duplicate review (fuzzy) | Auto-detect locations from different providers whose addresses are **fuzzy-matched** — because provider addresses never match exactly, the system normalizes `address_line1 + city + state + postal_code` and computes a weighted **match score (0–100%)**; pairs scoring at/above a configurable threshold (default **75%**) are surfaced in a Duplicate Review queue with the differing fields highlighted; ops confirms "same store" (both link to one OC location ID) or "not same" (independent OC IDs); both actions logged (`OC_LINK_CONFIRMED` / `OC_LINK_REJECTED`) | P0 | F01 | P06 | New: `GET /api/duplicates` + `POST /api/duplicates/{id}/resolve` | OC location ID assignment gates upstream sync (§8.2) |

## 5. User Flows

### 5.1 Core Path — Operator Batch Disable

![Core Path — Operator Batch Disable](https://jama-dev520.github.io/location-prototype/diagrams/flow1-batch-disable.png)

### 5.2 Daily Sync Path — Provider Status Change

![Daily Sync Path — Provider Status Change](https://jama-dev520.github.io/location-prototype/diagrams/flow2-daily-sync.png)

> **Business decisions (confirmed 2026-10-29)**:
> - New-location default `is_enable = false` — a newly synced location is NOT visible to upstream customers until a Location Manager explicitly enables it.
> - Provider "removal" (location disappears from provider's list) and "deactivation" (`status = inactive`) are **distinct semantics** — removal is a permanent delisting (stop cooperation), deactivation is a temporary closure (store closed / on break). See §8.2.

### 5.3 Exception Path

![Exception Path](https://jama-dev520.github.io/location-prototype/diagrams/flow5-exception.png)

### 5.4 Pickup Provider (Sweeper) Edit Path

![Pickup Provider (Sweeper) Edit Path](https://jama-dev520.github.io/location-prototype/diagrams/flow3-sweeper.png)

### 5.5 Duplicate Review Path

![Duplicate Review Path](https://jama-dev520.github.io/location-prototype/diagrams/flow4-duplicate.png)

## 6. Page Map

| Page ID | Page Name | Module | Core Functions | Entry | Exit | Data Passed | Feature IDs | Priority |
|---------|-----------|--------|----------------|-------|------|-------------|-------------|----------|
| P01 | Location List | Location List | Filter, list, export, single toggle, batch operations | Sidebar "Locations" (default landing) | P02 | Location ID (to detail) | F01/F02/F03/F06/F07 | P0 |
| P02 | Location Detail | Location Detail | Full profile tabs, per-location change history, single toggle, sweeper edit | P01 row click / action | P01, P03 | Location ID | F04/F05/F06/F11 | P0 |
| P03 | Change Log (global) | Change History | Cross-location change log with filters | Sidebar "Change Log"; P02 "View all changes" | P02 | Optional: location ID prefill | F08 | P1 |
| P04 | Sync Status | Data Sync | Last sync run per provider, results, failure alerts | Sidebar "Sync Status" | P03 | — | F09 | P1 |
| P05 | Overview Dashboard | Overview | KPI cards and trends | Sidebar "Overview" | P01 | — | F10 | P2 |
| P06 | Duplicate Review | Duplicate Detection | Cross-provider address duplicate queue, confirm/reject, OC location ID assignment | Sidebar "Duplicate Review" | P02 | Pair ID → location IDs | F12 | P0 |

## 7. Interaction Specifications

### 7.1 P01 — Location List

- **Entry**: Sidebar "Locations"; default landing page after login.
- **Elements**: Filter panel (collapsible), data table (columns: checkbox, Location ID, Name, Address, City/State, Provider, Type, Status, Enable, Created Time, Updated Time, Actions), pagination bar (20/page, page-size options 20/50/100), "Export CSV" button, "Batch Enable" / "Batch Disable" buttons, "Batch Upload (Excel)" button, selected-count indicator.
- **Filter fields**: Location ID, OC location ID, Name, Address keyword, City, State/Province, Provider, Pickup (sweeper), Status, Enable, Type, Created from/to, Updated from/to. All fields AND-combined; the OC location ID field is an exact/prefix text filter over the location's `oc_location_id` (locations without an OC ID are excluded when a value is entered).
- **Timestamp precision**: All `Created Time` / `Updated Time` values are displayed, filtered, and exported with **second-level precision** (`YYYY-MM-DD HH:MM:SS`). The filter panel provides `Created from` / `Created to` and `Updated from` / `Updated to` range inputs (datetime pickers) that filter on full timestamp.
- **Interactions**:
  - Filter inputs apply on click "Apply"; "Reset" clears all; active filter chips shown above table.
  - Click Location ID or Name → navigate to P02.
  - Row toggle (switch component) → immediate confirm popover ("Disable this location? It will become invisible to upstream customers") → executes F06.
  - Checkbox selection updates batch action bar; "select all on page" + hint when filtered result spans multiple pages.
  - Batch Disable/Enable → confirmation modal: affected count, first 5 sample IDs, warning line, "Cancel" / "Confirm" buttons → executes F07 Mode A → result toast.
  - **Batch Upload (Excel)** → upload dialog: ① "Download Template" link (CSV/XLSX with header row `locationID,is_enable`); ② drag-and-drop or click-to-select file; ③ file parsed client+server side → preview table (row count to enable / to disable / skipped duplicates); ④ invalid rows listed with row number + reason (unknown location ID, invalid T/F value, malformed row) with option "Skip invalid rows and process valid ones" or "Cancel"; ⑤ confirm → executes F07 Mode B → result summary toast.
  - Export downloads CSV named `locations_YYYYMMDD_HHmmss.csv`.
- **States**: Default with data / loading (skeleton rows) / empty (no results match filters + "Clear filters" action) / error (banner + retry).
- **Data volume**: TBD — assume hundreds to low thousands of locations (*baseline to be confirmed with ops*). Strategy: server-side pagination + server-side filtering; keyword search hits backend index. Max export 10,000 rows.
- **Validation rules**:
  - Location ID: alphanumeric, ≤ 32 chars, optional, validated on apply — "Location ID accepts letters and digits only".
  - Time range: end ≥ start, max span 1 year — "Time range cannot exceed 1 year".
  - Batch upload file: `.xlsx` or `.csv`, ≤ 5MB, ≤ 5,000 data rows; header row must be exactly `locationID,is_enable` (order fixed) — "Template header must be locationID,is_enable"; `is_enable` values must be `T` or `F` (case-insensitive) — "is_enable accepts T or F only (row {n})".

### 7.2 P02 — Location Detail

- **Entry**: P01 row click / Actions "View".
- **Elements**: Header (name, Location ID, provider badge, visibility status pill showing effective state: "Visible to customers" / "Hidden"), action buttons (Enable/Disable toggle for managers), tabs: **Basic Info** / **Hours & Services** / **Change History**.
  - Basic Info: provider ref numbers (dealerNo / loc_id / store code), address (full), coordinates, phone, timezone, lifecycle dates (created / activated / deactivated / last updated / last synced).
  - Hours & Services: weekly hours grid (7 days, time ranges), 24h flag, services chips, restrictions chips, cutoff time + timezone, holiday closures list.
  - Change History: reverse-chronological timeline; each entry: time, operator (person with avatar, or "System · PUDO Sync" / "System · LMS Sync"), operation type badge, changed field, old value → new value, remark.
- **Interactions**: Tab switching; toggle enable from header (F06, same confirm popover); "View all changes for this location" link → P03 pre-filtered.
- **States**: Loading / loaded / not-found ("Location does not exist or was removed" + back to list).
- **Data volume**: Single record; change history paginated (20/page).
- **Validation rules**: None (read-only page apart from the toggle).

### 7.3 P03 — Change Log (Global) (P1)

- **Entry**: Sidebar "Change Log"; P02 link.
- **Elements**: Filters (time range, operator, provider, operation type, location ID), log table (Time, Operator, Operation Type, Location ID + Name, Field, Old → New Value, Source), pagination.
- **Interactions**: Click location → P02; filter apply/reset same as P01.
- **States**: Default / empty / error.
- **Data volume**: Grows daily with sync + manual ops; assume < 5,000 entries/month initially. Server-side pagination.
- **Validation rules**: Time range end ≥ start, max span 1 year.

### 7.4 P04 — Sync Status (P1)

- **Entry**: Sidebar "Sync Status".
- **Elements**: One card per provider (PUDO Inc, LMS): last sync time, status (Success/Failed/Running), records pulled, locations created / updated / deactivated counts, error message if failed, "Sync history" table (last 30 runs).
- **Interactions**: Refresh button; failed run row expands to show error detail.
- **States**: Success / Failed (red highlight + alert banner) / Never run.
- **Data volume**: 30 runs per provider.

### 7.5 P05 — Overview Dashboard (P2)

- **Entry**: Sidebar "Overview".
- **Elements**: KPI cards (Total Locations, Visible to Customers, Disabled by OC, Deactivated by Provider), 30-day trend chart of enable/disable operations, provider split.
- **States**: Loading / loaded / error.

### 7.6 P06 — Duplicate Review

- **Entry**: Sidebar "Duplicate Review"; badge shows the pending pair count.
- **Elements**: Pending review table (columns: Pair, Location A, Location B, Match score, Address diff, Detected, Status, Action), a "How it works" explainer, and the active match-score threshold.
- **Interactions**: Each pending pair shows a color-coded **match score** (≥ 95% green, ≥ threshold amber, below red) and a per-field **address diff** listing `address_line1`, `postal`, `city`, `state` for both locations with differing fields highlighted. A Location Manager clicks "Same store" to link both to one OC location ID, or "Not same" to assign independent IDs; both write audit entries (`OC_LINK_CONFIRMED` / `OC_LINK_REJECTED`). Viewers see the actions disabled with a "Manager only" note.
- **States**: Pending (with pairs) / empty ("No pending duplicates").
- **Data volume**: Small — only above-threshold cross-provider pairs accumulate per sync.

## 8. Data Requirements

### 8.1 Core Data Objects

| Object | Key Fields | Source | Notes |
|--------|-----------|--------|-------|
| **Location** (unified) | location_id (internal), oc_location_id (OC-owned, gates upstream sync), source_provider (PUDO/LMS), provider_ref (dealerNo or loc_id), store_code, name, location_type (PUDOpoint/Locker/packagehub), pickup_provider (PUDO/LMS — who performs the physical sweep/pickup), address_line1/2, city, state, postal_code, country, latitude, longitude, phone, timezone, status (active/inactive/removed), is_enable (bool), services[], restrictions[], hours_of_operation{}, cutoff_time, holiday_closures[], created_at, updated_at, activated_at, deactivated_at, last_synced_at | PUDO Inc + LMS daily sync | Unified model normalizing both provider schemas; `location_type` is provider-exclusive (PUDO: PUDOpoint/Locker, LMS: packagehub); `pickup_provider` defaults to `source_provider` but is OC-editable (cross-provider sweep); `oc_location_id` links same-store provider locations and gates upstream sync |
| **Change Log Entry** | log_id, location_id, occurred_at, operator (user or "System · {Provider} Sync"), operation_type (ENABLE / DISABLE / BATCH_ENABLE / BATCH_DISABLE / SWEEPER_CHANGE / OC_LINK_CONFIRMED / OC_LINK_REJECTED / PROVIDER_STATUS_CHANGE / PROVIDER_FIELD_UPDATE / PROVIDER_CREATED / PROVIDER_REMOVED), field_name, old_value, new_value, source (MANUAL / SYNC), remark | This system | Append-only, immutable |
| **Duplicate Pair** | pair_id, location_a, location_b, matched_on (address_line1+city+state+postal), match_score (0–100%), diff (per-field differing values), status (pending/confirmed/rejected), detected_at, resolved_by, resolved_at | This system | Cross-provider (LMS ↔ PUDO Inc) only |
| **Sync Run** | run_id, provider, started_at, finished_at, result (SUCCESS/FAILED), records_pulled, created_count, updated_count, deactivated_count, error_message | This system | One record per provider per day |
| **User** | user_id, name, email, has_enable_permission (bool) | Corporate SSO | No self-service role management in v1 |

### 8.2 Data Rules

- **Visibility rule (core, updated 2026-10-29)**: A location is synced to upstream customers **iff it has an OC location ID AND `status = active` AND `is_enable = true`**. All three conditions must hold; any failing condition excludes the location from the upstream sync. The OC location ID requirement is the new gate — locations without an OC ID are never synced upstream, regardless of status/enable.
- **Field ownership**: `status` and all business attributes (address, hours, services, etc.) are owned by providers — read-only in this console, refreshed by daily sync. `is_enable` is owned by OC — writable only via F06/F07. `pickup_provider` is owned by OC — writable only via F11 (Location Manager).
- **OC location ID (new, 2026-10-29)**: A unique OC-owned identifier that links provider locations representing the same physical store and gates upstream sync. Assignment rules:
  - **No duplicate match** → the location is assigned its own OC location ID automatically at creation (one-to-one).
  - **Cross-provider duplicate** → the pair is queued for review; ops confirmation links both to **one** OC location ID; rejection assigns independent IDs. Until resolved, the location(s) stay **without** an OC ID and are not synced upstream.
  - Removed/terminal locations need no OC ID.
- **Duplicate detection (fuzzy, updated 2026-10-29)**: Cross-provider duplicate = two locations from different providers whose addresses are **fuzzy-matched** above a threshold. Addresses never match exactly across providers, so the system: ① normalizes `address_line1 + city + state + postal_code` (lowercase, trim, punctuation/whitespace-insensitive, unit/suite suffix stripped); ② computes a weighted **match score (0–100%)** — `address_line1` 50%, `postal_code` 25%, `city` 15%, `state` 10% (token Jaccard similarity per field); ③ flags pairs scoring **≥ 75% (configurable threshold)** as potential duplicates, surfacing the per-field differences for ops to judge. Detection runs on every sync for new/changed locations only; the match scope is **LMS ↔ PUDO Inc only** (not intra-provider).
- **New location default**: A location appearing in a sync for the first time is created with `is_enable = false` and a PROVIDER_CREATED log entry. **A new location is NOT visible to upstream customers until a Location Manager explicitly enables it.** — *Confirmed 2026-10-29*
- **Provider ↔ Type exclusivity (confirmed 2026-10-29)**: `location_type` is mutually exclusive per provider — PUDO Inc locations are only ever `PUDOpoint` or `Locker`; LMS locations are only ever `packagehub`. The console filter UI enforces this as a cascade (selecting Provider narrows the Type options); the data model treats `(provider, type)` as a constrained combination and rejects invalid pairs on sync.
- **Pickup provider (updated 2026-10-29)**: `pickup_provider` identifies who performs the physical sweep/pickup for a location. It **defaults to `source_provider`** but is an independent, OC-editable field and **may differ** — e.g. a PUDO Inc location whose pickup is actually handled by LMS. The list, detail, filter, and CSV export all surface this field; when `pickup_provider ≠ source_provider`, the UI flags it as a cross-provider sweep. A Location Manager can edit it via F11; every change writes an audit entry (`SWEEPER_CHANGE`).
- **Removal vs deactivation (confirmed 2026-10-29)**: A provider can affect a location's visibility in two **distinct** ways:
  - **Deactivation** (`status` → `inactive`): temporary closure — store closed for break, short-term outage. Location remains in the catalog; `is_enable` is preserved; it can be reactivated later by the provider.
  - **Removal** (location disappears from the provider's list): permanent delisting — stop cooperation / end of partnership. The location is marked `status = removed` (a distinct state, NOT `inactive`), logged as `PROVIDER_REMOVED`. Removed locations are treated as terminal in v1: they are no longer visible and are excluded from the active catalog, though their history and audit trail remain queryable.
- **Sync does not reset is_enable**: Provider sync never overwrites `is_enable`; it only updates provider-owned fields. Deactivated-by-provider locations keep their `is_enable` value (relevant if the provider reactivates them later).
- **Change log completeness**: Every write to `status`, `is_enable`, or material provider fields must produce exactly one log entry. Manual batch operations (inline or Excel upload) produce one entry per location whose value actually changes; no-op rows are skipped and reported in the result summary without an audit entry.
- **Batch upload rules**: Template has exactly 2 columns — `locationID`, `is_enable`. `T` = enable, `F` = disable, case-insensitive. Duplicate locationIDs in one file: last occurrence wins, earlier ones reported as skipped. Unknown location IDs are rejected as error rows and never silently created. Processing is synchronous for ≤ 5,000 rows with a result summary (applied / skipped-no-change / error counts).
- **Audit immutability**: Change log entries are never edited or deleted.
- **ID stability**: Internal `location_id` is permanent; provider_ref changes do not break history linkage.

### 8.3 External System Integration

> **⚠️ Sync architecture changed to INCREMENTAL (confirmed 2026-10-29).** The previous "full pull + daily diff" design is replaced by an incremental (delta) sync strategy to avoid a full-list pull of up to 100k locations conflicting with the daily-diff approach. See §8.5 below.

| External System | Interface Purpose | Data Flow | Sync/Async | Integration Status | Notes |
|----------------|-------------------|-----------|------------|-------------------|-------|
| PUDO Inc Dealer API | Pull INCREMENTAL dealer changes (daily) | In: auth + `updated_since` cursor → Out: changed DealerList records (dealerNo, OperatingAs, address, lat/lng, Active, DealerSupports, hoursOfOperation, lifecycle dates) | Async scheduled job (daily 02:00) | Existing API, contract available from sample; **incremental delta endpoint to be confirmed with provider** | Delta keyed on `LastUpdatedDate`; fallback to full pull if provider lacks incremental support |
| LMS Location API | Pull INCREMENTAL location changes (daily) | In: auth + cursor (`has_more`) + `updated_since` → Out: changed location records (loc_id, name, type, address, locationCode, cutoff, services, restrictions, holiday_closures) | Async scheduled job (daily 02:30) | Existing API, supports cursor pagination; **incremental delta endpoint to be confirmed with provider** | Iterate `has_more` only for the changed-record window, not the full catalog |
| Customer-facing Location Query API (upstream) | Serves visible locations to upstream customers | Reads Location table with `has_oc_id = true AND status=active AND is_enable=true` filter | Sync (per request) | Existing | Enable/disable and OC-ID assignment changes take effect on next query; no push notification |
| Corporate SSO | Staff authentication | In: credentials → Out: user identity + email | Sync | Existing | Groups → advanced permission mapping TBD |

### 8.5 Sync Failure Fallback & Incremental Strategy

**Incremental pull scope (confirmed 2026-10-29):**

- Each daily run pulls only records changed since the last successful sync (keyed on provider `LastUpdatedDate` / `updated_since` cursor), instead of the full catalog.
- Scale bound: incremental window is expected to be a small fraction of the catalog (typically hundreds to low thousands of changed records per day), decoupling sync cost from total catalog size (target ceiling 100k locations).
- **Provider capability to be confirmed**: both providers must expose a `updated_since`-style delta endpoint. If a provider only supports full pull, the system falls back to full pull + local diff for that provider **only**, and this is flagged in the sync config. This is the only remaining full-pull path.

**Sync failure fallback (confirmed 2026-10-29):**

- On any sync failure (network timeout, 5xx, malformed payload, delta cursor error), the local catalog is **NOT cleared and NOT partially updated** — the last successfully synced snapshot is preserved intact.
- The failed run is marked `FAILED` on P04 with the error detail, and the location's `last_synced_at` is **not** advanced (so staff can see the data is stale).
- The system performs **one automatic retry** shortly after failure; if it still fails, it waits for the next scheduled cycle. No manual recovery is required for transient failures.
- Data freshness is surfaced on P04 so staff can distinguish "up to date" from "last success at X, subsequent runs failed".
- Incremental cursor is only advanced on a fully successful run, guaranteeing no changes are silently skipped.

### 8.4 Non-Functional Requirements

| Dimension | Requirement | Target | Priority |
|-----------|------------|--------|----------|
| Performance | List query p95 latency | ≤ 2s with filters, ≤ 5s export (10k rows) | P0 |
| Availability | Internal console SLA | 99.5% business hours; sync job must not block console | P0 |
| Security | Access control | SSO login mandatory; enable/disable gated by advanced permission; audit log immutable | P0 |
| Compatibility | Browser | Chrome / Edge latest two major versions, 1440px+ desktop | P0 |
| Scalability | Data growth | Support 100k locations and 1M change log entries without redesign | P1 |
| Compliance | Data handling | Location data is business (non-PII) data; operator identity retained for audit per internal policy | P1 |

## 9. Non-Goals

- ❌ No editing of provider-owned business data (address, hours, services) — v1 is read-only on provider fields; manual data overrides are out of scope.
- ❌ No location creation or deletion by internal users — the catalog is provider-driven; locations enter/exit only via sync.
- ❌ No end-customer-facing UI — upstream customers consume locations through the existing query API, not this console.
- ❌ No real-time push/webhook to upstream customers when visibility changes — effect applies on next query.
- ❌ No role management UI in v1 — advanced permission provisioned via IT/SSO configuration.
- ❌ No non-English localization in v1.
- ❌ No provider management (adding/removing providers) UI in v1.

## 10. Acceptance Criteria

> Given-When-Then format; all criteria marked "suggest PM review before finalizing".

### 10.1 Functional Acceptance Mapping

| ID | Feature | Happy Path | Boundary | Exception Data | Verification | Priority |
|----|---------|-----------|----------|----------------|--------------|----------|
| F01 | Location list | ①Open P01 → ②Table shows both PUDO and LMS locations with all specified columns ③Pagination works (20/page) | ①0 locations match → empty state with "Clear filters"; ②100/page renders without layout break | Location with missing address2/phone/dealerHours renders gracefully ("—" placeholders) | UI walkthrough + API response comparison | P0 |
| F02 | Filter & search | ①Set Provider=LMS, Status=active, Enable=enabled → ②only matching rows shown ③active filter chips visible ④Reset restores full list | ①Combined 7 filters AND-logic correct; ②Time range crossing year boundary | Invalid Location ID (special chars) rejected with inline error | UI walkthrough with seeded data | P0 |
| F03 | Export | ①Apply filters → ②Export → ③CSV downloads reflecting exactly the filtered set and list columns | ①Export with no filters = full catalog (≤10k); ②10,001 matches → prompt to narrow filters | Corrupted row (null fields) exports as empty cells without breaking file | File open + row count + spot-check values | P0 |
| F04 | Location detail | ①Click row → ②all three tabs populated for both a PUDO and an LMS location ③visibility pill matches status×enable logic | ①Location with empty hours/restrictions shows empty-state text; ②Locker vs PUDOpoint vs packagehub types all render | Non-existent ID → "Location does not exist" + back link | UI walkthrough both providers | P0 |
| F05 | Per-location change log | ①Open Change History tab → ②entries reverse-chronological ③each shows operator/time/type/field/old→new | ①Location with zero changes shows creation entry only; ②20+ entries paginate | Sync-generated entries show "System · {Provider} Sync" operator | Seeded change records + UI comparison | P0 |
| F06 | Single toggle | ①Manager clicks toggle → ②confirm popover → ③confirm → ④is_enable flips, list refreshes, audit entry created | ①Toggle on already-enabled location = no-op with toast "No change"; ②Viewer sees toggle disabled with tooltip | API failure → error toast, state unchanged, no audit entry | Permission A/B test + audit log check | P0 |
| F07 | Batch enable/disable | **Mode A**: ①Select 3 rows → ②Batch Disable → ③modal shows count + sample IDs + warning → ④confirm → ⑤all 3 flip + 3 audit entries + success toast with count. **Mode B**: ①Download template → ②upload valid 100-row file (mix of T/F) → ③preview shows N to enable / M to disable → ④confirm → ⑤all applied + audit entries + result summary | ①Mode A: 200 (max) succeeds, 201 blocked; ②Mode B: 5,000 rows OK, 5,001 rejected with message; ③Duplicate locationIDs → last wins, earlier reported skipped; ④Rows already in target state → no-op, reported, no audit entry | ①Mode B: unknown location ID / invalid T/F value / wrong header / oversized file → row-level error report with reasons; "Skip invalid rows" processes valid ones only; ②Partial failure → result dialog N ok / M failed + retry-failed; ③Session expires mid-flow → re-login returns to confirmation step | Seeded batch test + audit entries count | P0 |
| F08 | Global change log | ①Open P03 → ②filter by operator + time range → ③matching entries across all locations | ①Filter operator="System" isolates sync entries; ②Empty range → empty state | Location deleted upstream still shows its historical entries | Data comparison vs DB | P1 |
| F09 | Sync status | ①Open P04 → ②both provider cards show last run + counts | ①Failed run shows red banner + error detail; ②Never-run state | Malformed provider payload → job marked FAILED, console unaffected | Trigger test sync | P1 |
| F10 | Dashboard | ①Open P05 → ②KPI numbers match list-derived counts | ①Zero locations → zero-state cards | — | KPI vs list count reconciliation | P2 |
| F11 | Edit sweeper / pickup provider | ①Manager opens P02 → ②edits pickup provider PUDO Inc → LMS → ③saved → ④audit entry `SWEEPER_CHANGE` with operator/time/old→new → ⑤cross-provider badge shown | ①Viewer sees control disabled with tooltip; ②Selecting the current value = no-op, no audit entry | API failure → error toast, value unchanged, no audit entry | Permission A/B test + audit log check | P0 |
| F12 | Fuzzy duplicate review | ①Sync flags a pair at ≥75% score → ②P06 shows pair with score + per-field diff → ③Manager confirms "same store" → ④both linked to one OC ID + `OC_LINK_CONFIRMED` entry → ⑤both eligible for upstream sync | ①Reject path assigns independent OC IDs + `OC_LINK_REJECTED`; ②Score below threshold is not surfaced; ③Identical address → 100% score; ④Viewer sees "Manager only" | Address with unit-suffix / postal-extension differences still matches above threshold (not treated as exact-equality failure) | Seeded cross-provider pair + audit entries count | P0 |

### 10.2 End-to-End Flow Acceptance

- [ ] Core path walkable: filter → select → batch disable → confirm → audit entries → location hidden upstream.
- [ ] Sync path walkable: nightly sync → provider status change → log entry → staff reviews on P03/P04.
- [ ] All P0 features clickable in the prototype.

## 11. Success Metrics

### 11.1 Metric Definitions

| Level | Metric | Definition | Baseline | Target | Cadence | Source |
|-------|--------|-----------|----------|--------|---------|--------|
| 🎯 North Star | Upstream catalog accuracy | % of locations whose effective visibility (active × enabled) matches OC's business intent, measured by audit reconciliation | — (no tool today) | ≥ 99.5% | Monthly | Change log + spot audit |
| 📊 Core | Audit coverage of visibility changes | % of visibility changes with complete operator/time/old→new record | 0% (no tool) | 100% | Monthly | Change log |
| 📊 Core | Daily sync success rate | % of days both provider syncs complete successfully | — | ≥ 99% | Monthly | Sync run records |
| 📊 Core | Time-to-locate | Median time for staff to find a specific location via filters | ~minutes (manual) | < 30s | Quarterly | Internal usability sampling |
| 📊 Core | Batch operation adoption | % of bulk enable/disable changes done via batch feature vs one-by-one | — | ≥ 80% | Monthly | Change log (BATCH_* vs single ops) |
| 📈 Auxiliary | Export usage | Exports per week | — | Track only | Weekly | Console telemetry |
| 📈 Auxiliary | List query p95 latency | 95th percentile list API latency | — | ≤ 2s | Monthly | APM |

### 11.2 Measurement Plan

- T+7: verify audit coverage = 100% of all visibility changes since launch.
- T+30: first monthly reconciliation of catalog accuracy; review sync success rate.
- T+90: evaluate time-to-locate and batch adoption; decide v1.1 scope (dashboard, role management UI).

## 12. Risks & Dependencies

| Risk / Dependency | Type | Impact | Mitigation |
|-------------------|------|--------|------------|
| PUDO Inc / LMS API schema drift | Risk | Sync breaks, stale catalog | Contract tests on sample payload; sync failure alerts (F09) |
| Provider incremental delta endpoint unavailable | Dependency | Must fall back to full pull for that provider | Confirm `updated_since` capability with both providers before sync build; fallback full-pull path flagged per-provider (§8.5) |
| Sync failure leaves stale catalog | Risk | Staff or customers act on outdated data | Last-success snapshot preserved on failure; `last_synced_at` not advanced; P04 freshness indicator (§8.5) |
| SSO group → advanced permission mapping undefined | Dependency | Enable control cannot ship | Confirm with IT before F06/F07 development |
| Batch misuse (mass accidental disable) | Risk | Upstream catalog suddenly shrinks | Confirmation dialog with count + sample IDs; batch cap 200 (inline) / 5,000 (upload); consider audit-based recovery drill |
| Malformed bulk upload file (wrong header, invalid T/F, unknown IDs) | Risk | Batch partially applied or confusing failures | Strict header/format validation before preview; row-level error report; dry-run preview before execution; "skip invalid rows" is explicit user choice |
| Data volume exceeds 10k export cap | Risk | Export blocked for full-catalog requests | Prompt to narrow filters; v1.1 async export job if needed |
| Timezone handling (dealerTimeZone vs cutoff_timezone vs console TZ) | Risk | Confusing displayed times | Normalize all timestamps to console timezone (UTC) with explicit labels |

## 13. Appendix

### 13.1 Glossary

| Term | Definition |
|------|-----------|
| OC | Operations / Location Curation team — internal owner of this console |
| PUDO Inc | Upstream provider #1 — supplies dealer locations (PUDOpoints / Lockers) |
| LMS | Upstream provider #2 — supplies package hub locations |
| PUDOpoint / Locker | PUDO Inc location types (dealerLocationTypeID 1 / 2) |
| pickup_provider | Who performs the physical sweep/pickup for a location; defaults to source_provider but is OC-editable and may differ (cross-provider sweep) |
| packagehub | LMS location type |
| status | Provider-sourced business status (active / inactive / removed) |
| is_enable | OC-controlled internal switch; combined with status and OC location ID determines upstream visibility |
| removed | Terminal provider state — location permanently delisted (stop cooperation); distinct from `inactive` |
| inactive | Temporary provider state — store closed / on break; location remains in catalog |
| OC location ID | OC-owned identifier linking provider locations of the same physical store; gates upstream sync (locations without it are never synced) |
| Sweeper | The provider that physically picks up packages from a location (same as `pickup_provider`) |
| Duplicate pair | Two cross-provider locations whose normalized addresses fuzzy-match above the score threshold |
| Match score | Weighted 0–100% address similarity (address_line1 50% + postal_code 25% + city 15% + state 10%, token Jaccard); ≥ 75% flags a potential duplicate |
| Visible to customers | has OC location ID AND `status = active` AND `is_enable = true` |
| Visibility rule | The core gating logic above, enforced in the customer-facing query API |
| Incremental sync | Daily pull of only changed records since last success (vs. full-list pull), keyed on `updated_since` |
| FX / CP / TB | PUDO Inc service codes (DealerSupports field) |
| Cutoff time | LMS — latest drop-off time for same-day processing |

### 13.2 References

- PUDO Inc DealerList API sample (provided in requirement input, 2026-10-29)
- LMS Location List API sample (provided in requirement input, 2026-10-29)

### 13.3 Prototype Link

- `Location-Management-Admin-Prototype-2026-10-29.html` (generated after PRD confirmation)

### 13.4 Test Requirements

| Dimension | Requirement | Notes |
|-----------|------------|-------|
| External system test environments | PUDO Inc / LMS sandbox endpoints TBD | If unavailable, mock provider APIs with the two provided sample payloads |
| Test data source | Seed from the two provided samples + generate ~500 synthetic locations across both providers | Include edge cases: empty hours, missing address2, inactive status, 24h locations |
| Test accounts | 1 Viewer account + 1 Location Manager account | Per §3.1 role table |
| Mock requirements | Mock nightly sync trigger for on-demand runs; mock partial batch failure (some rows 500) | Verify F07 partial-failure dialog; prepare test upload files: valid mixed T/F file, file with unknown IDs / invalid values / wrong header / >5,000 rows / >5MB |
