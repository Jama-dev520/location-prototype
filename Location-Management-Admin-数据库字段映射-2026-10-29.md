# Location Management Admin Console — Database Field Mapping (v2.6.1)

> Companion to PRD `Location-Management-Admin-PRD-2026-10-29` (v2.6.1)
> Purpose: help engineering/data teams understand "front-end filter ↔ API param ↔ DB field" and the table schemas for the **two-layer model**.

---

## 0. Two-Layer Model Overview (v2.0)

The console has two layers:

1. **Vendor Location** (`vendor_location` table) — raw provider data (PUDO Inc / LMS). Read-only except `sweeper`.
2. **OC Location** (`oc_location` table) — OC-owned, downstream-facing. Backfilled from vendor; carries `is_enable` and OC-editable store details.

A **mapping table** links them (principle 1:1; a confirmed cross-provider duplicate links 2 vendor → 1 OC).

```
vendor_location (provider data)
        │  1:N mapping (vendor_oc_mapping)
        ▼
oc_location (OC-owned, downstream-facing)
```

---

## 1. Filter ↔ Database Field Mapping

### 1.1 Vendor Location List (`GET /api/vendor-locations`)

| UI Filter | API Param | Table | Field | Type | Match | Notes |
|---|---|---|---|---|---|---|
| Location ID | `location_id` | `vendor_location` | `vendor_location_id` | varchar(32) | exact/prefix | alphanumeric, ≤32 |
| OC Location ID (reverse lookup) | `oc_location_id` | `vendor_oc_mapping` → `oc_location` | `oc_location_id` | varchar(32) | exact/prefix | cross-layer: find vendors mapped to a given OC location |
| Name | `name` | `vendor_location` | `name` | varchar | LIKE %…% | hits index |
| Address keyword | `address_keyword` | `vendor_location` | `address_line1`/`address_line2` | varchar | LIKE %…% | joint search |
| Provider | `provider` | `vendor_location` | `source_provider` | enum(PUDO,LMS) | exact | cascades Type |
| Sweeper | `sweeper` | `vendor_location` | `sweeper` | enum(PUDO,LMS) | exact | OC-editable |
| Status | `status` | `vendor_location` | `status` | enum(active,inactive,removed) | exact | provider-sourced, read-only |
| Type | `location_type` | `vendor_location` | `location_type` | enum(PUDOpoint,Locker,packagehub) | exact | provider-exclusive |
| QR Dropoff | `qr_dropoff` | `vendor_location` | `qr_dropoff` | bool | exact | LMS-only store capability; PUDO = N/A |
| City | `city` | `vendor_location` | `city` | varchar | exact | — |
| State | `state` | `vendor_location` | `state` | varchar | exact | — |
| Created from/to | `created_from`/`created_to` | `vendor_location` | `created_at` | timestamp | BETWEEN | ≤1yr span |
| Updated from/to | `updated_from`/`updated_to` | `vendor_location` | `updated_at` | timestamp | BETWEEN | ≤1yr span |

### 1.2 OC Location List (`GET /api/oc-locations`)

| UI Filter | API Param | Table | Field | Type | Match | Notes |
|---|---|---|---|---|---|---|
| OC Location ID | `oc_location_id` | `oc_location` | `oc_location_id` | varchar(32) | exact/prefix | — |
| Vendor Location ID (forward lookup) | `vendor_location_id` | `vendor_oc_mapping` → `vendor_location` | `vendor_location_id` | varchar(32) | exact/prefix | cross-layer: find OC locations fed by a given vendor |
| Name | `name` | `oc_location` | `name` | varchar | LIKE %…% | OC-editable |
| Address keyword | `address_keyword` | `oc_location` | `address_line1`/`address_line2` | varchar | LIKE %…% | OC-editable |
| Provider | `provider` | `vendor_location` (via mapping) | `source_provider` | enum | exact | join through mapping |
| Sweeper | `sweeper` | `vendor_location` (via mapping) | `sweeper` | enum | exact | join through mapping |
| Status | `status` | `oc_location` | `status` | enum(active,inactive,removed) | exact | derived from linked vendor |
| Enable | `is_enable` | `oc_location` | `is_enable` | bool | exact | OC switch |
| Type | `location_type` | `vendor_location` (via mapping) | `location_type` | enum | exact | join through mapping |
| QR Dropoff | `qr_dropoff` | `oc_location` | `qr_dropoff` | bool | exact | backfilled from linked vendor location |
| **Sync status** (v2.6.0) | `sync_state` | `sync_change_set` (derived) | `decision` | enum | `pending` / `none` / `all_resolved` | derived: does this OC location have held provider changes |
| **Intake status** (v2.6.0) | `intake_status` | `oc_location` | `intake_status` | enum(pass,warn,fail,overridden) | exact | failed intake never became an OC location |
| City | `city` | `oc_location` | `city` | varchar | exact | OC-editable |
| State | `state` | `oc_location` | `state` | varchar | exact | OC-editable |
| Created from/to | `created_from`/`created_to` | `oc_location` | `created_at` | timestamp | BETWEEN | ≤1yr span |
| Updated from/to | `updated_from`/`updated_to` | `oc_location` | `updated_at` | timestamp | BETWEEN | ≤1yr span |
| Visibility (implicit) | `visible` | `oc_location` | `status` + `is_enable` | combined | `status=active AND is_enable=true` | see visibility rule |

> **Visibility rule (core)**: an OC location is visible downstream **iff `status = 'active'` AND `is_enable = true`**. `status` is derived from the linked vendor location(s); when the vendor is `inactive`, the OC location is auto-disabled and blocked.

---

## 2. Tables & Columns

### Table 1 — `vendor_location` (vendor layer)

> Source: PUDO Inc + LMS daily sync. Read-only except `sweeper`.

| Field | Type | Source | Read-only? | Notes |
|---|---|---|---|---|
| `vendor_location_id` | varchar(32) PK | system | yes | permanent internal ID |
| `source_provider` | enum(PUDO,LMS) | sync | yes | source provider |
| `provider_ref` | varchar | sync | yes | dealerNo (PUDO) / loc_id (LMS) |
| `store_code` | varchar | sync | yes | store code |
| `name` | varchar | sync | yes | store name |
| `location_type` | enum(PUDOpoint,Locker,packagehub) | sync | yes | provider-exclusive |
| `sweeper` | enum(PUDO,LMS) | OC | no | **who performs the physical pickup** (renamed from `pickup_provider`); default = source_provider; cross-provider allowed; writable via F10 |
| `address_line1` | varchar | sync | yes | address line 1 |
| `address_line2` | varchar | sync | yes | address line 2 (nullable) |
| `city` | varchar | sync | yes | city |
| `state` | varchar | sync | yes | state/province |
| `postal_code` | varchar | sync | yes | postal code |
| `country` | varchar | sync | yes | country |
| `latitude` | decimal | sync | yes | latitude |
| `longitude` | decimal | sync | yes | longitude |
| `phone` | varchar | sync | yes | phone (nullable) |
| `timezone` | varchar | sync | yes | timezone |
| `status` | enum(active,inactive,removed) | sync | yes | provider business status |
| `services[]` | array | sync | yes | services list |
| `restrictions[]` | array | sync | yes | restrictions |
| `qr_dropoff` | bool | sync | yes | **QR-code dropoff capability** (LMS only): customer presents a QR code instead of a printed label. PUDO does not expose it → `NULL`/`false` |
| `hours_of_operation{}` | json | sync | yes | weekly hours |
| `cutoff_time` | time | sync | yes | cutoff (LMS) |
| `holiday_closures[]` | array | sync | yes | holiday closures |
| `created_at` | timestamp | sync | yes | created time |
| `updated_at` | timestamp | sync | yes | updated time |
| `activated_at` | timestamp | sync | yes | activated time |
| `deactivated_at` | timestamp | sync | yes | deactivated time |
| `last_synced_at` | timestamp | system | yes | last successful sync |

### Table 2 — `oc_location` (OC layer)

> Backfilled from vendor; OC-editable store details; carries `is_enable`.

| Field | Type | Source | Read-only? | Notes |
|---|---|---|---|---|
| `oc_location_id` | varchar(32) PK | system | yes | OC-owned ID; gates downstream sync |
| `name` | varchar | backfill → OC-editable | no | store name (OC can override) |
| `address_line1` | varchar | backfill → OC-editable | no | address line 1 (OC can override) |
| `address_line2` | varchar | backfill → OC-editable | no | address line 2 |
| `city` | varchar | backfill → OC-editable | no | city |
| `state` | varchar | backfill → OC-editable | no | state |
| `postal_code` | varchar | backfill → OC-editable | no | postal code |
| `country` | varchar | backfill → OC-editable | no | country |
| `latitude` | decimal | backfill → OC-editable | no | latitude |
| `longitude` | decimal | backfill → OC-editable | no | longitude |
| `phone` | varchar | backfill → OC-editable | no | phone |
| `opening_hours{}` | json | backfill → OC-editable | no | opening hours (OC can override) |
| `qr_dropoff` | bool | backfill (derived) | yes | QR-code dropoff; `true` if **any** linked vendor location supports it |
| `status` | enum(active,inactive,removed) | derived | yes | derived from linked vendor location(s) |
| `is_enable` | bool | OC | no | OC switch; only F08/F09 writable |
| **`intake_status`** (v2.6.0) | enum(pass,warn,fail,overridden) | intake check | yes | result of the quality gate; `fail` means no OC record; `overridden` means it exists but cannot be enabled |
| **`baseline_at`** (v2.6.0) | timestamp | system | yes | **when the currently-stored values were human-verified** — the reference for good data |
| **`propagated_at`** (v2.6.0) | timestamp | system | yes | last push to the downstream (cache invalidated); `null` = never pushed |
| `created_at` | timestamp | system | yes | created time |
| `updated_at` | timestamp | system | yes | updated time |

### Table 3 — `vendor_oc_mapping` (mapping table)

> Links vendor location(s) to OC location. Principle 1:1; confirmed duplicate → 2:1.

| Field | Type | Notes |
|---|---|---|
| `mapping_id` | bigint PK | mapping primary key |
| `vendor_location_id` | varchar FK → vendor_location | vendor location |
| `oc_location_id` | varchar FK → oc_location | OC location |
| `created_at` | timestamp | created time |

### Table 4 — `change_log` (audit trail)

> Append-only, immutable.

| Field | Type | Notes |
|---|---|---|
| `log_id` | bigint PK | log primary key |
| `location_id` | varchar FK | the location ID (vendor or OC) |
| `location_layer` | enum(VENDOR,OC) | which layer the change belongs to |
| `occurred_at` | timestamp | occurred time |
| `operator` | varchar | operator, or `System · PUDO Sync` / `System · LMS Sync` |
| `operation_type` | enum | ENABLE / DISABLE / BATCH_ENABLE / BATCH_DISABLE / SWEEPER_CHANGE / STORE_DETAIL_EDIT / OC_LINK_CONFIRMED / OC_LINK_REJECTED / OC_GENERATED / PROVIDER_STATUS_CHANGE / PROVIDER_FIELD_UPDATE / PROVIDER_CREATED / PROVIDER_REMOVED |
| `field_name` | varchar | changed field |
| `old_value` | text | old value |
| `new_value` | text | new value |
| `source` | enum(MANUAL,SYNC) | manual / sync |
| `remark` | text | remark |

### Table 5 — `duplicate_pair` (cross-provider duplicate pair)

> Cross-provider (LMS ↔ PUDO Inc) only; fuzzy match score; manual confirmation.

| Field | Type | Notes |
|---|---|---|
| `pair_id` | varchar PK | pair ID (e.g. DUP-001) |
| `vendor_location_a` | varchar FK → vendor_location | location A (one provider) |
| `vendor_location_b` | varchar FK → vendor_location | location B (other provider) |
| `matched_on` | varchar | address_line1 + city + state + postal_code |
| `match_score` | int | 0–100% (addr1 50% + postal 25% + city 15% + state 10%) |
| `diff` | json | per-field differences |
| `status` | enum(pending,confirmed,rejected) | pending / confirmed same-store / rejected |
| `detected_at` | timestamp | detected time |
| `resolved_by` | varchar | resolver |
| `resolved_at` | timestamp | resolved time |

### Table 6 — `sync_run` (sync run record)

| Field | Type | Notes |
|---|---|---|
| `run_id` | bigint PK | run primary key |
| `provider` | enum(PUDO,LMS) | provider |
| `started_at` | timestamp | start time |
| `finished_at` | timestamp | finish time |
| `result` | enum(SUCCESS,FAILED) | result |
| `records_pulled` | int | pulled records |
| `created_count` | int | created |
| `updated_count` | int | updated |
| `deactivated_count` | int | deactivated |
| `error_message` | text | error (if failed) |

### Table 7 — `user` (console user)

| Field | Type | Notes |
|---|---|---|
| `user_id` | varchar PK | user ID |
| `name` | varchar | name |
| `email` | varchar | email |
| `has_enable_permission` | bool | advanced permission |

### Table 8 — `oc_field_baseline` (field baseline, added in v2.6.0)

> **The reference point for "good data".** It records what a human verified, not what the last sync happened to write. Each field keeps its 5 most recent versions so a single field can be reverted.

| Field | Type | Notes |
|---|---|---|
| `baseline_id` | bigint PK | primary key |
| `oc_location_id` | varchar FK → oc_location | which OC location |
| `field_name` | varchar | which field (`address_line1`, `latitude`, `opening_hours`, …) |
| `baseline_value` | text | the verified value |
| `version_no` | int | 1..N, keep last 5 |
| `captured_at` | timestamp | when verified |
| `captured_by` | varchar | operator, or `System` |
| `capture_reason` | enum(SIGNOFF, ACCEPTED_CHANGE, OPERATOR_EDIT) | why this version became the baseline |

> **Write timing**: only three events create a baseline — enable sign-off (F21), accepting a provider change (F20), and an operator edit. A provider sync never writes it.

### Table 9 — `sync_change_set` (provider change set, added in v2.6.0)

> One row per changed field per run. `pending` means **held** — recorded on the vendor layer, deliberately not written to the OC layer.

| Field | Type | Notes |
|---|---|---|
| `change_id` | varchar PK | e.g. `SC-0013` |
| `run_id` | varchar FK → sync_run | which sync run produced it |
| `oc_location_id` | varchar FK → oc_location | affected OC location |
| `vendor_location_id` | varchar FK → vendor_location | where the value came from |
| `field_name` | varchar | single field |
| `old_value` | text | baseline (current) value |
| `new_value` | text | provider's proposed value |
| `risk_tier` | enum(auto,review,block) | decided by §8.6 policy matrix |
| `decision` | enum(pending,accepted,rejected,superseded) | `superseded` = an operator edited the same field |
| `reason` | varchar | why it was held (e.g. "coordinate moved 297 km — exceeds the 1 km limit") |
| `decided_by` / `decided_at` | varchar / timestamp | who decided, when |

### Table 10 — `field_lock` (field lock, added in v2.6.0)

> A permanent "ask me". Once locked, a provider change to that field always goes to review, **even if its base tier is `auto`**.

| Field | Type | Notes |
|---|---|---|
| `lock_id` | bigint PK | primary key |
| `oc_location_id` | varchar FK → oc_location | which OC location |
| `field_name` | varchar | which field |
| `locked_at` | timestamp | when |
| `locked_by` | varchar | operator |
| `reason` | enum(OPERATOR_EDIT, REJECTED_CHANGE) | operator fixed it, or a change was rejected |

### Table 11 — `intake_issue` (intake check, added in v2.6.0)

> Runs on every newly synced vendor location, before an OC location is generated.

| Field | Type | Notes |
|---|---|---|
| `issue_id` | bigint PK | primary key |
| `vendor_location_id` | varchar FK → vendor_location | which vendor record |
| `check_code` | varchar | `ADDR1_EMPTY` / `POSTAL_CITY_MISMATCH` / `COORD_PLACEHOLDER` / `COORD_OUT_OF_RANGE` / `HOURS_INVALID` |
| `severity` | enum(warn,fail) | `fail` → no OC location generated; `warn` → created but cannot be enabled |
| `detected_at` | timestamp | when detected |
| `status` | enum(open,overridden,resolved) | `overridden` = ops forced an OC record (still `is_enable=false`) |

---

## 3. Field Ownership Quick Reference

| Field Category | Owner | Writable Path | Read-only? |
|---|---|---|---|
| Vendor `status` + business attributes (address/hours/services) | Provider | daily sync | yes |
| Vendor `qr_dropoff` | Provider (LMS) | daily sync | yes |
| Vendor `sweeper` | OC (Location Manager) | F10 vendor list/detail | no |
| OC `is_enable` | OC (Location Manager) | F08 single / F09 batch | no |
| OC store details (address, opening hours, etc.) | OC (Location Manager) | F11 OC list/detail | no |
| `oc_location_id` | System | OC generation (F12) + duplicate resolution (F14) | yes |
| `vendor_location_id` / `created_at` | System | — | yes |

> **Key rules**:
> - Provider sync **never overwrites** `sweeper`, OC `is_enable`, OC store details, or `oc_location_id`.
> - When vendor `status` → `inactive`, the linked OC location is auto-disabled and blocked from downstream sync; force-enable raises an error.

### 3.1 Sync tiers (introduced in v2.6.0; applies only to OC locations that are already visible)

| Tier | Fields | What happens when the provider changes it |
|---|---|---|
| `auto` | `qr_dropoff`, `services`, `restrictions`, `cutoff_time`, `phone`, `timezone` | Written straight through, logs `PROVIDER_FIELD_UPDATE` |
| `conditional auto` | `opening_hours` | Auto if the structure is valid and no day moves more than 2 h; otherwise `review` |
| `review` | `name`, `address_line1/2`, `city`, `state`, `postal_code`, `latitude`, `longitude` | Held (`sync_change_set.decision = pending`); the OC location keeps its baseline value |
| `immediate (one-way)` | `status` | `active → inactive` applies at once; a batch of `inactive` trips the breaker |
| **forced review** | **any field present in `field_lock`** | Held regardless of its base tier — a human must decide |

> Two extra gates: **two-run confirmation** (an `auto` field is applied only when two consecutive runs agree) and **magnitude thresholds** (coordinate drift > 1 km, any postal/city change, opening hours moving > 2 h on a day → forced review).

### 3.2 The three write paths (v2.6.0)

| Path | Trigger | Enters the review queue? | Pushed downstream? | Baseline |
|---|---|---|---|---|
| Provider sync, `auto` tier | daily job | no | yes | unchanged |
| Provider sync, `review` tier | daily job | **yes** (held) | **no** — keeps the old value | unchanged |
| Operator edit | human saves | **no** — an operator edit is the final decision | yes if visible, logs `DOWNSTREAM_PROPAGATED` | **refreshed**, and the edited fields are written into `field_lock` |

---

## 4. API ↔ Table Quick Reference

| API | Method | Tables | Notes |
|---|---|---|---|
| `/api/vendor-locations` | GET | `vendor_location` | vendor list + filter (server pagination) |
| `/api/vendor-locations/{id}` | GET | `vendor_location` | vendor detail |
| `/api/vendor-locations/{id}/sweeper` | PATCH | `vendor_location` + `change_log` | edit sweeper, write SWEEPER_CHANGE |
| `/api/vendor-locations/export` | GET | `vendor_location` | export vendor view |
| `/api/oc-locations` | GET | `oc_location` + `vendor_oc_mapping` + `vendor_location` | OC list + filter |
| `/api/oc-locations/{id}` | GET | `oc_location` + mapping | OC detail |
| `/api/oc-locations/{id}/enable` | PATCH | `oc_location` + `change_log` | single enable/disable (blocked if vendor inactive) |
| `/api/oc-locations/{id}` | PATCH | `oc_location` + `change_log` | edit OC store details, write STORE_DETAIL_EDIT |
| `/api/oc-locations/batch-enable` | POST | `oc_location` + `change_log` | Mode A batch |
| `/api/oc-locations/batch-enable/upload` | POST | `oc_location` + `change_log` | Mode B Excel upload |
| `/api/oc-locations/batch-enable/template` | GET | — | download template |
| `/api/oc-locations/export` | GET | `oc_location` + mapping | export OC view |
| `/api/changes` | GET | `change_log` | global change log |
| `/api/duplicates` | GET | `duplicate_pair` | duplicate review queue (pending) |
| `/api/duplicates/{id}/resolve` | POST | `duplicate_pair` + `oc_location` + `vendor_oc_mapping` + `change_log` | confirm/reject; assign OC location IDs |
| `/api/sync/status` | GET | `sync_run` | sync status |
| **`/api/review/summary`** (v2.6.0) | GET | `duplicate_pair` + `intake_issue` + `sync_change_set` | Review Center badge counts for the three queue tabs |
| **`/api/review/data-quality`** (v2.6.0) | GET | `intake_issue` + `vendor_location` | Data Quality list (failed intake, no OC record) |
| **`/api/review/data-quality/{id}/override`** (v2.6.0) | POST | `oc_location` + `intake_issue` + `change_log` | force-create an OC record with `is_enable=false`, `intake_status=overridden` |
| **`/api/review/sync-changes`** (v2.6.0) | GET | `sync_change_set` + `oc_field_baseline` | held changes grouped by OC location, with baseline value |
| **`/api/review/sync-changes/{id}/decide`** (v2.6.0) | POST | `sync_change_set` + `oc_field_baseline` + `field_lock` + `oc_location` + `change_log` | accept → write + refresh baseline; reject → keep baseline + lock field |
| **`/api/oc-locations/{id}/enable`** (v2.6.0, extended) | PATCH | `oc_location` + `oc_field_baseline` + `change_log` | enable now requires the sign-off payload (checklist acknowledged); writes `baseline_at`, `propagated_at`, `ENABLE_SIGNED_OFF` |
| **`/api/oc-locations/{id}`** (v2.6.0, extended) | PATCH | `oc_location` + `oc_field_baseline` + `field_lock` + `sync_change_set` + `change_log` | operator edit: refresh baseline, lock fields, supersede pending changes, set `propagated_at` if visible |
| **`/api/sync/runs/{id}/rollback`** (v2.6.0) | POST | `oc_location` + `change_log` | revert a run's OC rows to the stored before-image (7-day window) |
| **`/api/sync/safe-mode`** (v2.6.0) | PATCH | `sync_run` (flag) | freeze / unfreeze all propagation |
| `/api/stats/overview` | GET | `vendor_location` + `oc_location` + `change_log` | overview KPIs |
