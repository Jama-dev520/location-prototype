# Location Management Admin Console — Database Field Mapping (v5.0.0)

> Companion to PRD `Location-Management-Admin-PRD-2026-10-29` (v5.0.0)
> Feature overview (English, for readers outside the team): `https://jama-dev520.github.io/location-prototype/feature-overview.html`
>
> **v3.0.0 model change.** The console no longer writes location fields. `field_lock` is replaced by `data_issue`; `sync_change_set` decisions and `oc_field_baseline` versions used for revert are retired; downstream values are a per-field merge with **LMS first**. The console writes exactly four things: `oc_location.is_enable`, `data_issue` rows, baseline captures, and `change_log`.
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
| Provider | `provider` | `vendor_location` | `source_provider` | enum(PUDO,LMS,M&amp;MP) | exact | **hard-cascades Type** (v3.2.0) |
| Sweeper | `sweeper` | `vendor_location` | `sweeper` | enum(PUDO,LMS) | exact | OC-editable |
| Status | `status` | `vendor_location` | `status` | enum(active,inactive,removed) | exact | provider-sourced, read-only |
| Type | `location_type` | `vendor_location` | `location_type` | enum(PUDOpoint,Locker,packagehub,M&amp;MP) | exact | provider-exclusive; options derived from Provider |
| QR Dropoff | `qr_dropoff` | `vendor_location` | `qr_dropoff` | bool | exact | LMS-only store capability; PUDO = N/A |
| City | `city` | `vendor_location` | `city` | varchar | exact | — |
| State | `state` | `vendor_location` | `state` | varchar | exact | — |
| Created from/to | `created_from`/`created_to` | `vendor_location` | `created_at` | timestamp | BETWEEN | ≤1yr span |
| Updated from/to | `updated_from`/`updated_to` | `vendor_location` | `updated_at` | timestamp | BETWEEN | ≤1yr span |

### 1.2 OC Location List (`GET /api/oc-locations`)

> v2.7.0 added the **Downstream** filter and the `⏳ Not live yet (N)` preset, which maps to `is_enable = false` (N = `COUNT(*) WHERE is_enable = false`). This preset is the sign-off queue that used to be the fourth Review Center tab. **v4.1.0** renames the filter to **Live** (`Live` / `Not live`) so it matches the column it filters, and adds `live_source_vendor_id` as a filter param (`live_source`) — the row that answers «which record feeds this store» is a query an operator actually asks during an incident.

| UI Filter | API Param | Table | Field | Type | Match | Notes |
|---|---|---|---|---|---|---|
| OC Location ID | `oc_location_id` | `oc_location` | `oc_location_id` | varchar(32) | exact/prefix | — |
| Vendor Location ID (forward lookup) | `vendor_location_id` | `vendor_oc_mapping` → `vendor_location` | `vendor_location_id` | varchar(32) | exact/prefix | cross-layer: find OC locations fed by a given vendor |
| Name | `name` | `oc_location` | `name` | varchar | LIKE %…% | OC-editable |
| Address keyword | `address_keyword` | `oc_location` | `address_line1`/`address_line2` | varchar | LIKE %…% | OC-editable |
| Provider | `provider` | `vendor_location` (via mapping) | `source_provider` | enum | exact | join through mapping |
| Sweeper | `sweeper` | `vendor_location` (via mapping) | `sweeper` | enum | exact | join through mapping |
| Status | `status` | `oc_location` | `status` | enum(active,inactive,removed) | exact | derived from linked vendor |
| **Live** (was *Enable*, v4.1.0) | `live` | `oc_location` + `vendor_location` + list | `live_source_vendor_id IS NOT NULL` AND source `is_enable = 1` AND source `is_active = 1` | derived | — | **v4.3.1:** the console writes none of those three flags — it writes the whitelist entry. A blocked store (`list_type = black`) is reported as `blocked`, not as `not live` |
| **Live source** (v4.1.0) | `live_source` | `oc_location` | `live_source_vendor_id` | varchar(32) | exact | which linked record the store reads from; `null` when the store has never been signed off |
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
| `location_type` | enum(PUDOpoint,Locker,packagehub,M&MP) | sync | yes | provider-exclusive: PUDO Inc → PUDOpoint/Locker; LMS → packagehub; M&MP → M&MP store |
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
| **`addr_fingerprint`** | varchar(16) | computed | yes | **(v5.0.0)** SHA-256[:16] over the cleaned record — the candidate bucket key. Recomputed when cleaning changes; carries no verdict |
| **`match_confidence`** | int | computed | yes | **(v5.0.0)** the 0–100 text score, recomputed on every judgement |
| **`duplicate_tag`** | enum | computed | yes | **(v5.0.0)** `HIGH_SUSPECT` / `MEDIUM_SUSPECT` / `LOW_SUSPECT` |
| **`review_status`** | enum | human | yes | **(v5.0.0)** `UNREVIEW` / `APPROVED` / `REJECTED` — the only thing that can produce an OC ID |

### Table 2 — `oc_location` (OC layer)

> Backfilled from vendor; OC-editable store details; carries `is_enable`.

| Field | Type | Source | Read-only? | Notes |
|---|---|---|---|---|
| `oc_location_id` | **varchar(22)** PK | system | yes | **(v5.0.0)** `OC-{ST2}-{City3}-{Zip5}-{seq6}` — a permanent cluster identity, e.g. `OC-NY-NYC-10036-000001`. Minted **only on human confirmation**, never rewritten: not by a re-run, not by an address correction, not by a cleaning upgrade. The state / city / zip inside the string are a **snapshot** at issuance — query `primary_*` instead. Split mints a new ID, merge keeps the older, a retired ID is marked `is_discarded` and never reused |
| **`primary_state` / `primary_city` / `primary_city_code` / `primary_zip5`** | varchar(2/64/3/5) | system | yes | **(v5.0.0)** the real address of the cluster, kept beside the ID because the ID's segments are a snapshot. **Every filter, report and API query reads these, never parses the ID** |
| **`is_discarded`** | bool | system | yes | **(v5.0.0)** set when a cluster is emptied by a split or absorbed by a merge; the ID is never reissued |
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
| `is_enable` | bool | **provider (read-only here)** | no | **The provider's business status — 1 open, 0 closed. The console never writes it (v4.3.1).** What the console controls is the whitelist entry: sign-off sets `live_source_vendor_id` and writes `list_type = white`. `is_active` is our own row flag (0 = void) and `is_primary` is unusable (1 on every record) |
| **`live_source_vendor_id`** (v4.1.0) | varchar(32) FK → vendor_location | OC (chosen at sign-off) | no | the one record the store reads from. **Default `null`** — written only by a sign-off, never by the sync, never by a merge, never by a ranking. Must be a record linked through `vendor_oc_mapping`; at most one per `oc_location`. **Cleared by un-signing, by a source switch, and by the system** when that record stops qualifying (its `is_enable` or `is_active` goes to 0) — the store then reads as `Not live` with `list_removed_reason = PROVIDER_DISABLED | RECORD_VOIDED`, and reopening does not restore the pointer |
| **`baseline_source_vendor_id`** (v4.1.0) | varchar(32) FK → vendor_location | system | yes | which record the current baseline was captured from — the missing half of «what was verified», since with 2:1 the values alone no longer say whose they are |
| **`intake_status`** (v2.6.0) | enum(pass,warn,fail,overridden) | intake check | yes | result of the quality gate; `fail` means no OC record; `overridden` means it exists but cannot be enabled |
| **`baseline_at`** (v2.6.0) | timestamp | system | yes | **when the currently-stored values were human-verified** — the reference for good data |
| **`propagated_at`** (v2.6.0) | timestamp | system | yes | last push to the downstream (cache invalidated); `null` = never pushed |
| `created_at` | timestamp | system | yes | created time |
| `updated_at` | timestamp | system | yes | updated time |

### Table 2b — `oc_sequence` and `city_code_mapping` (v5.0.0)

> The ID allocator. Two small tables, and together they are why an ID can be permanent.

| `oc_sequence` | Type | Notes |
|---|---|---|
| `seq_key` | varchar PK | `OC_{ST2}_{City3}_{Zip5}`, e.g. `OC_NY_NYC_10036` |
| `next_val` | int | read under a row lock, then incremented — two clusters confirmed in the same second must not receive the same number |

| `city_code_mapping` | Type | Notes |
|---|---|---|
| `state2` | varchar(2) | part of the unique key `(state2, city3_code)` |
| `city_name` | varchar | normalised upper case |
| `city3_code` | varchar(3) | IATA where one exists (`NYC`, `AUS`, `BNA`), otherwise allocated by hand. **Never derived from the first three letters** — `SAN` alone means San Antonio, Santa Ana or San Jose |
| `source` | enum(iata, manual) | so a reviewer can tell a standard code from an invented one |

> A city with no code yet gets `XXX`; the record is accepted, the code is filled later, and **the ID keeps `XXX`**. Reissuing it would break the only promise the ID makes.

### Table 3 — `vendor_oc_mapping` (mapping table)

> Links vendor location(s) to OC location. Principle 1:1; confirmed duplicate → 2:1. **v4.1.0:** a store may hold several **active** linked records and exactly one of them is the live source (`oc_location.live_source_vendor_id`); the mapping table itself stays a plain link table — the choice lives on the OC row, not on the link, so it survives a re-link unchanged. Inserting a row here (a new or merged record) writes **no** live source: a record joins the store as a candidate.

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
| `operation_type` | enum | ENABLE / DISABLE / BATCH_ENABLE / BATCH_DISABLE / ENABLE_SIGNED_OFF / DATA_ISSUE_RAISED (v3.0.0) / DATA_ISSUE_REPORTED (v3.0.0) / DATA_ISSUE_CLOSED (v3.0.0) / DATA_ISSUE_AUTO_CLOSED (v3.3.0) / OC_LINK_CONFIRMED / OC_LINK_REJECTED / OC_GENERATED / PROVIDER_STATUS_CHANGE / PROVIDER_FIELD_UPDATE / PROVIDER_CREATED / PROVIDER_REMOVED / DOWNSTREAM_PROPAGATED / **LIVE_SOURCE_CHANGED** (v4.1.0: store stayed live, source moved) / **OC_MAPPING_CHANGED** (v4.0.0: a `source_key` was re-pointed after a person confirmed it) — retired in v3.0.0: SWEEPER_CHANGE, STORE_DETAIL_EDIT, SYNC_CHANGE_ACCEPTED, SYNC_CHANGE_REJECTED, FIELD_UNLOCKED, FIELD_REVERTED |
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
| `text_score` (v4.0.0, recomputed per v4.2.0) | int | 0–100, **proportional**: each field pays `weight ×` the share of sorted adjacent character pairs the two values share (address 40 · name 10 · postal 25 · province 15 · city 10); a different Zip5 zeroes it. A stored value is a cache — recompute on read. Replaces `match_score` |
| `distance_m` (v4.0.0) | int | coordinate evidence, not a score: ≤30 same point / 30–100 grey / >100 different |
| `label` (v4.0.0) | enum(HIGH_SUSPECT, MEDIUM_SUSPECT) | derived from `text_score` + `distance_m`; a label, never a percentage |
| `diff` | json | per-field differences |
| `status` (v4.0.0) | enum(pending,merged,separate,fix) | pending / same station (one disabled) / two tenants / address to fix |
| `keep_vendor_location_id` (v4.0.0) | varchar FK | the record that keeps the OC location |
| `merged_into` (v4.0.0) | varchar FK | written on the record that lost; cleared when that record leaves the store again |
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

> **The reference point for "good data".** It records what a human verified, not what the last sync happened to write. Each field keeps its 5 most recent versions — this is what F24 (field revert) reads when an operator undoes a manual edit.

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

### Table 10 — `data_issue` (data anomalies, replaces `field_lock` in v3.0.0)

> A flag, not an edit. It records **what is wrong, at what granularity, and how bad** — on the source record. The value itself is never touched. A **blocking** issue stops new sign-offs; a **warning** does not.

| Field | Type | Notes |
|---|---|---|
| `issue_id` | varchar PK | `ISS-0001` |
| `vendor_location_id` | varchar FK → vendor_location | the record that has to be fixed |
| `field_name` | varchar | top-level field (`address_line1`, `postal_code`, `opening_hours`, `sweeper`, …) |
| `sub_field` | varchar NULL | **granularity** — which address line, which weekday, which coordinate (`address_line2`, `SATURDAY`, `latitude`) |
| `issue_type` | enum | `MISSING` · `INCORRECT` · `INCONSISTENT` · `OUT_OF_RANGE` · `DUPLICATE` · `STALE` · `FORMAT` |
| `severity` | enum(BLOCKING, WARNING) | blocking = cannot be signed off; warning = live but wrong for customers. Defaults from `issue_type`, can be raised |
| `reason` | text NOT NULL | **mandatory**; what exactly is wrong, in the operator's words |
| `status` | enum(OPEN, REPORTED, FIXED) | open → reported (chased) → fixed (source corrected) |
| `owner_type` | enum(PROVIDER, STORE) | who fixes it: PUDO Inc / LMS, or the **M&MP store owner** internally |
| `raised_by` / `raised_at` | varchar / timestamp | who, when (to the second) |
| `closed_at` | timestamp NULL | when it reached FIXED |
| `closed_by` | enum(AUTO_SYNC) NULL | always the provider run (v3.3.0); the v3.4.0 manual close was removed, so `OPERATOR` only appears on rows written before that |
| `verified_value` | text NULL | the value the run delivered that closed it — the evidence, kept with the issue |
| `auto_close_run` | varchar NULL | `sync_run.run_id` that closed it |
| `last_checked_at` | timestamp NULL | last run that re-checked this flag, closed or not |
| `check_note` | varchar NULL | what that run saw: `unchanged` / `still invalid` / `needs a human re-check` |

#### 10.1 Issue types, with one example each

| Type | Default severity | What it means | Example (from the prototype) |
|---|---|---|---|
| `MISSING` | blocking | the owner sent no value at all | `LOC-00181 · sweeper` — sweeper was not supplied, downstream routing cannot resolve who picks up |
| `INCORRECT` | blocking | a value exists but it is the wrong one | `LOC-00182 · sweeper` — set to PUDO Inc on an LMS packagehub with no PUDO agreement |
| `INCONSISTENT` | blocking | two fields contradict each other | `LOC-00183 · postal_code` — 92058 belongs to Oceanside, the street line is in 92054 |
| `OUT_OF_RANGE` | blocking | placeholder or impossible value | `LOC-00177 · latitude/longitude` — (0,0)-style drift, the pin lands 297 km away in the water |
| `DUPLICATE` | warning | this record looks like another one | `LOC-00186 · unit / suite` — suffix appears twice and fuzzy-matches LOC-12194 at 75% |
| `STALE` | warning | unchanged in a way that no longer matches reality | `LOC-00184 · MONDAY` — hours unchanged since 2024, the store now closes at 16:00 |
| `FORMAT` | warning | unparseable or truncated | `LOC-00179 · phone` — 7 digits, cannot be dialled from another area code |

> Granularity ladder: **field → sub-field → type → severity → free-text reason.** The sub-field is what makes an issue actionable: "opening hours are wrong" cannot be fixed by a provider, "SATURDAY has no range" can.

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

### Table 12 — `mapping_change` (source_key → OC re-point, added v4.0.0)

> One **source**, never a pair. Written when a record's identity fields change and its mapping is re-judged.

| Field | Type | Notes |
|---|---|---|
| `change_id` | varchar PK | e.g. `MAP-001` |
| `round` | int | which re-check produced it |
| `source_key` | varchar | `PUDO_INC:<vendor_location_id>` / `LMS:<vendor_location_id>` — the key downstream actually uses |
| `changed_field` | varchar | which identity field moved (`address_line1`, `name`, `postal_code`, `lat`, `lng`, `city`, `state`) |
| `old_oc_location_id` | varchar FK → oc_location | where the source is mapped now |
| `new_oc_location_id` | varchar FK → oc_location | proposed target; `null` when a new OC has not been allocated yet |
| `basis` | varchar | the re-check that produced it, e.g. `NO_SUSPECT · text 0 · 119169 m` |
| `status` | enum(pending,effective,manual) | `pending` = nothing written yet; `manual` = both sides moved in one run (M6) |
| `actor` | varchar | who confirmed it |
| `decided_at` | timestamp | when confirmed |

---

## 3. Field Ownership Quick Reference

### 3.1 Who owns what (v3.0.0)

| Layer | Fields | Who writes them | Can the console change them? |
|---|---|---|---|
| `vendor_location` | every field, **including `sweeper`** | the provider sync | **no** — a wrong value is flagged (`data_issue`) |
| `oc_location` | `is_enable` only | the operator (sign-off, F21) | yes, and that is the only OC write |
| derived (not stored) | name / address / hours / coordinates / sweeper as shown downstream | computed from the linked vendor records | no |

### 3.2 Downstream merge rule — one record supplies the store (F25, v3.13.0; named source since v4.1.0)

When one OC location is fed by several vendor records (2:1), the console picks **one** record and takes every field from it:

```
0. if oc_location.live_source_vendor_id is set and still linked -> that record wins,
   whatever the ranking says (a merge must not re-point a signed-off store)
1. candidates = linked vendor records with status = 'active'
2. winner     = first by priority: M&MP (self-operated) > LMS > PUDO Inc
3. if no candidate is active -> same priority applied to all linked records,
   and the row is flagged as sourced from an inactive record
4. row.f      = winner.f   for every field f   (an empty winner field stays empty)
5. only when the OC has no linked record at all -> the value stored on oc_location
```

| Situation | Result |
|---|---|
| M&MP active, LMS active | every field from **M&MP** — first-party data outranks a partner's |
| LMS active, PUDO active | every field from **LMS** |
| LMS active but `phone` is empty, PUDO has one | **`phone` stays empty** — the gap belongs to LMS and is shown as one |
| LMS inactive, PUDO active | every field from **PUDO** — the whole record, not the half LMS was missing |
| all linked records inactive | the same priority still applies; the row is flagged as sourced from an inactive record |

> **Record-level on purpose (changed in v3.13.0).** The earlier rule resolved each field independently and let the next source fill the gap. That is convenient for whoever reads the row and wrong for the process: an empty LMS `phone` was filled from PUDO, so nothing ever reached LMS as something to fix, and "who owns this field" had no answer. Whole-record merging makes the owner of the store the owner of its gaps. The cost is explicit and accepted: a live row can show `—` for a field its owner never sent. Do **not** "fix" that by reintroducing a per-field fallback in the read model — surface the gap instead.

### 3.3 What the console writes (and what it no longer writes)

| Write | Trigger | Table |
|---|---|---|
| `is_enable = true` + baseline | sign-off (F21) | `oc_location` + `oc_field_baseline` |
| `is_enable = false` | disable, batch disable, vendor inactive | `oc_location` |
| `data_issue` row | operator flags a field (F11) | `data_issue` |
| `data_issue.status` | chased / fixed | `data_issue` |
| **`data_issue.status = FIXED`** (v3.3.0) | **provider run delivers a valid value** — not a human click | `data_issue` (`closed_by = AUTO_SYNC`) + `change_log` |
| `last_checked_at` / `check_note` (v3.3.0) | every run that touches the flagged field, whether it closes or not | `data_issue` |
| audit row | every action above | `change_log` |

Retired in v3.0.0: `STORE_DETAIL_EDIT`, `SWEEPER_CHANGE`, `FIELD_UNLOCKED`, `FIELD_REVERTED`, `SYNC_CHANGE_ACCEPTED`, `SYNC_CHANGE_REJECTED`, and the `field_lock` / `sync_change_set` decision columns. `oc_field_baseline` keeps one row per sign-off (no 5-version stack — nothing is reverted any more).

### 3.4 Type and source are one value in the UI (v3.10.0, supersedes the v3.2.0 filter cascade)

`location_type` is not a free field: it is determined by `source_provider`. Because the mapping is 1:1, the console prints the pair as **one value** — `packagehub (LMS)`, `PUDOpoint (PUDO)`, `M&MP` — in the list and in the filter.

| `source_provider` | `location_type` | Shown as | In the Source filter |
|---|---|---|---|
| `LMS` | `packagehub` | `packagehub (LMS)` | yes |
| `PUDO Inc` | `PUDOpoint` | `PUDOpoint (PUDO)` | yes |
| `PUDO Inc` | `Locker` | `Locker (PUDO)` | **not in this release** |
| `M&MP` | `M&MP` | `M&MP` (printed once — both halves are the same word) | yes |

Rules:

1. **One filter, three values.** The Source filter is the type, annotated with its owner; there is no separate Type filter and therefore no cascade to keep in step. Selecting `packagehub (LMS)` compiles to `location_type=packagehub`.
2. **The columns stay two.** `location_type` and `source_provider` remain separate in `vendor_location`; the merged string is a rendering, not a stored value (see the no-new-column note above).
3. **No mismatch badge.** With one field there is no place for `type ≠ source` to show, and the `Issues → Type ≠ source` filter is gone. Detection, if the feed ever contradicts itself, happens in the ingest job (§3.5) and surfaces as a `data_issue` on the record.
4. **`Locker` is modelled, not surfaced.** `PUDO Inc` can still deliver a `Locker`; the console simply does not offer it as a Source this release, because a value that always filters to zero rows is worse than an absent one. Adding it is one entry in the front-end source table.
5. **On the OC list** the type is the **merged** type (source priority M&amp;MP → LMS → PUDO Inc), and a 2:1 location prints both owners in the same order: `packagehub (LMS / PUDO)`.
6. **Server side**: `location_type=<value>` is still validated against `provider`; a request pairing `provider=LMS` with `location_type=Locker` returns `400 INVALID_TYPE_FOR_PROVIDER` instead of an empty page. The UI cannot build such a request, but the API still refuses one.

### 3.4a Sweeper on a self-operated store (v3.10.0)

`M&MP` stores are our own shops, but the pickup is not: `sweeper` on an `M&MP` record is **always `LMS`**.

- The Sweeper filter has no `M&MP` option, and no row carries that value.
- It is **not** a CHECK constraint: the provider still owns the field, and an `M&MP` record that arrives with another sweeper is a data issue to flag back (§3.5), not a value the console rejects at write time — rejecting it would silently drop the record instead of surfacing the problem.

### 3.5 Issue auto-close rule (v3.3.0)

A run closes an issue only when all four conditions hold:

```sql
-- evaluated per issue, after the run has written the provider values
issue.status <> 'FIXED'
AND issue.issue_type IN ('MISSING','FORMAT','INCONSISTENT','OUT_OF_RANGE')
AND current_value(issue.vendor_location_id, issue.field_name, issue.sub_field) <> issue.value_at_flag
AND is_valid(issue.field_name, issue.sub_field, current_value(...))
```

`value_at_flag` is captured when the flag is raised (`issue_base` in the prototype); comparing against the *current* value would make every issue look closed on the next read.

| `issue_type` | Auto-closable | Why |
|---|---|---|
| `MISSING` | yes | presence is objective — a value arrived or it did not |
| `FORMAT` | yes | the format rule is fixed (5-digit postal, 10-digit phone) |
| `INCONSISTENT` | yes | the contradiction is re-evaluated against the same rule |
| `OUT_OF_RANGE` | yes | placeholder / impossible values are detectable |
| `INCORRECT` | **no** | “wrong name” needs the business fact, which the feed cannot supply |
| `STALE` | **no** | “no longer true” is a judgement about the real world |
| `DUPLICATE` | **no** | whether two records are one store is a decision (F26) |

Validity per field: `sweeper` ∈ {PUDO Inc, LMS, M&MP} · `postal_code` = 5 digits · `phone` ≥ 10 digits · `latitude`/`longitude` ≠ placeholder · `opening_hours[day]` has at least one range with `start < end` · anything else: non-empty.

Failure modes the rule deliberately produces:

- **changed but still invalid** → stays open, `check_note` records the attempted value, no status change.
- **unchanged** → stays open, `check_note = value unchanged`, and **no** `PROVIDER_FIELD_UPDATE` row is written (nothing was delivered).
- **closed** → `DATA_ISSUE_AUTO_CLOSED` row carries `open → fixed`, the run id, and both values.

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
| ~~`/api/review/sync-changes/{id}/decide`~~ | — | — | **retired in v3.0.0**: there is no per-field accept/reject, the provider value already applies |
| **`/api/oc-locations/{id}/enable`** (v2.6.0, extended) | PATCH | `oc_location` + `oc_field_baseline` + `change_log` | enable now requires the sign-off payload (checklist acknowledged); writes `baseline_at`, `propagated_at`, `ENABLE_SIGNED_OFF` |
| **`/api/oc-locations/{id}/live-source`** (v4.1.0) | PATCH | `oc_location` + `oc_field_baseline` + `change_log` | set `live_source_vendor_id` and `is_enable = true` in one write, with the same checklist payload as sign-off; `LIVE_SOURCE_CHANGED` (old and new source) when the store was already live, `ENABLE_SIGNED_OFF` when it was not. `DELETE` on the same path drops the store to `is_enable = false` — it never promotes another record |
| **`/api/vendor-locations/{id}/issues`** (v3.0.0) | POST | `data_issue` + `change_log` | flag a field: `field_name` + `reason` (both required); never writes a value |
| **`/api/issues/{id}`** (v3.0.0) | PATCH | `data_issue` + `change_log` | move status OPEN → REPORTED → FIXED; closing requires the provider value to have changed or an explicit confirmation |
| **`list_entry`** (v4.3.1) | — | `whitelist_entry` + `change_log` | — | **One row per (vendor_location, entry), never updated in place.** A record enters the whitelist on sign-off and leaves it on un-sign, on a source switch, or **when the system finds it no longer qualifies** (`is_enable = 0` / `is_active = 0`). Re-entry inserts a new row instead of reviving the old one, so "who put this store in front of customers, when, and who took it out" is a chain, not an overwrite. `removed_reason` is `UNSIGNED` / `SOURCE_SWITCHED` / `BLOCKED` / `PROVIDER_DISABLED` / `RECORD_VOIDED`; the first two are written by an operator, the last two by the sync. `list_type` is mutually exclusive per record — a row that is both white and black is a bug, not a policy. |
| **OC identity** (v5.0.0) | — | `vendor_location` + `oc_location` + `oc_sequence` + `city_code_mapping` | — | **The ID stops being a counter.** `oc_location_id` becomes `OC-{ST2}-{City3}-{Zip5}-{seq6}`, and the migration is one-way: existing ids keep their value (they are identities, not numbers to renumber). `addr_fingerprint` votes for nothing — it is an index, so it may be recomputed at will; `oc_location_id` may not. The sequence is per `(ST2, City3, Zip5)` and must be allocated under a lock; `City3` must come from the dictionary, and a missing code is written `XXX` rather than guessed. Add `primary_*` beside the ID so no query parses the string, and `is_discarded` so a retired number is never handed out again. |
| **`oc_location.live_source_vendor_id`** / `baseline_source_vendor_id` (v4.1.0) | — | `oc_location` + `change_log` | — | **The live source is a stored pointer, not a computed ranking.** `is_enable = true` must name a record (`live_source_vendor_id`), and that name beats the merge ranking — otherwise a merge into a live store silently re-points what customers read. Constrain it to a record reachable through `vendor_oc_mapping` (a CHECK or a trigger, not application code alone), and keep at most one per `oc_location` — there is no join table for «which of the two is live», because that question has one answer. `baseline_source_vendor_id` records whose values the baseline holds: with 2:1 the baseline values alone no longer say which record was verified. `live_source` is a filter param, `is_enable` keeps its name, and the UI word everywhere is **Live**. |
| **`duplicate_pair.text_score` / `distance_m` / `label`** + new `mapping_change` (v4.0.0) | — | `duplicate_pair` + `mapping_change` | — | **The score stops being a percentage and the pair stops being the unit.** `text_score` is proportional over five fields (sorted adjacent pairs, v4.2.0) with Zip5 as a gate; `distance_m` is evidence stored beside it, never folded into it; `label` is derived, so a stored copy is a cache, not a fact. `match_score` is superseded — keep it only as history. The disposition set changes from `confirmed/rejected` to `merged/separate/fix`, with `merged_into` written on the record that lost. The new `mapping_change` table exists because the second layer judges **one source at a time**: without it, a re-point would be a silent write to `vendor_oc_mapping`, and the confirmation step (M5) would have nowhere to live. |
| *(none)* (v3.14.0) | — | — | — | **The re-check adds one column and removes a habit.** `duplicate_pair.status` gains `split`, and `duplicate_pair.decided_identity` (JSON: score + both identity snapshots) is written at decision time — that snapshot is what a later change is compared with; without it the console can only say "these two differ", not "this one moved". The pair `score` becomes **derived, not trusted**: recompute it from the two `vendor_location` rows on read, and never treat the stored value as evidence — a stored score is exactly what made the old decision look permanent |
| *(none)* (v3.13.0) | — | — | — | **The record-level merge adds nothing to the schema.** The winner is computed on read from `status` + `source_provider`; do not add `merged_*` columns to `oc_location`, and do not keep a "primary vendor" pointer — a cached winner would go stale the moment a source is deactivated, and the whole point of v3.13.0 is that the owner of the store is always recomputed |
| *(none)* (v3.12.0) | — | — | — | **Province is a label, not a column.** The filter, the column header and the CSV header all say `Province`; the field behind them is `state` — the same column for a US state and a province elsewhere. Do **not** rename `state` to `province`: it is a migration, a backfill and a downstream contract change to buy a word. Do **not** add a derived `province_code` either. The filter takes `state=NY` and `city=Brooklyn`; the city list is a `SELECT DISTINCT city WHERE state = :state` |
| *(none)* (v3.11.0) | — | — | — | **The OC issue split is derived, not stored.** `n blocking` is `COUNT(data_issue WHERE severity='blocking' AND status<>'fixed')` over the linked `vendor_id`s; `n drift` is a comparison against `oc_location.baseline`; `2:1` is `COUNT(vendor_oc_mapping) > 1`; `intake` is the `intake_override` flag. Do **not** add `blocking_issue_count` to `oc_location`: it would be a fourth copy of a number that three other tables already know, and it would go stale the moment an issue closes automatically |
| *(none)* (v3.10.0) | — | — | — | **Merging type and source adds nothing either — and must not.** `vendor_location` keeps `location_type` and `source_provider` as two columns: the UI prints them as one string because they are one fact, but the fact still has two owners in the feed and the export still writes both. Do **not** add a `source_label` column: it would be a third copy of two existing values, and it would have to be recomputed whenever either changes |
| *(none)* (v3.9.0) | — | — | — | **Refresh adds nothing to the schema.** A manual pull writes the same `vendor_location` columns a nightly run writes, logs the same `change_log` rows and inserts the same `sync_run` row — it is the same job, triggered by a person rather than by cron. Do **not** add `last_pulled_by` or `pull_source` to `vendor_location`: who pressed the button is already `change_log.actor`, and a column on the record would need maintaining on every write for no reader |
| *(none)* (v3.8.0) | — | — | — | **QR and the merged type column add nothing to the schema.** `qr_dropoff` was already on `vendor_location` (v2.3.0); `packagehub (LMS)` is `type` plus `source` rendered together. Do not create a `type_with_source` column — it would duplicate two fields that already have owners |
| *(none)* (v3.7.0) | — | — | — | **The short-day check adds nothing to the schema either.** It is `SUM(end - start)` per `opening_hours.day`, compared with a display constant (`3h`). No `short_day` flag, no `check_type` — a stored copy would drift from the ranges it describes, and 3h differs per market, which is configuration rather than data |
| *(none)* (v3.6.0) | — | — | — | **The three checks add nothing to the schema.** A check is a set of `data_issue.field` values (`addr1/addr2/city/state`, `postal`, `hoursText`), evaluated when the ledger renders. Do not store a `check_type` column — a stored copy can drift from the field it is supposed to describe, and the counter would then disagree with the list it filters |
| *(none)* (v3.5.0) | — | — | — | **The language switch adds nothing to the schema.** Interface strings live in the client bundle; every displayed value keeps the column, type and casing documented here. Do not translate into new columns — a translated store name or address would be a second source of truth. If a language is ever needed server-side (an email to a provider, say), it belongs in a `locale` on the *recipient*, not on the location |
| **`/api/issues/export`** (v3.4.0) | GET | `data_issue` + `vendor_location` + `vendor_oc_mapping` | CSV of open issues (filterable by type / severity / owner) — the work list handed to the provider |
| **`/api/oc-locations/export`** (v3.4.0) | GET | `oc_location` + `vendor_location` + `data_issue` | CSV of the current filter, with `open_issues` and `drift_fields` columns |
| ~~`/api/oc-locations/{id}/resign`~~ | — | — | **retired in v3.4.0**: there is no approval step, so there is nothing to re-sign |
| **`/api/oc-locations/{id}/resolved`** (v3.0.0) | GET | `vendor_location` + `vendor_oc_mapping` | the merged, one-record view with a `source` per field (F25) — what downstream receives |
| **`/api/oc-locations/{id}`** (v2.6.0, extended) | PATCH | `oc_location` + `oc_field_baseline` + `field_lock` + `sync_change_set` + `change_log` | operator edit: refresh baseline, lock fields, supersede pending changes, set `propagated_at` if visible |
| **`/api/sync/runs/{id}/rollback`** (v2.6.0) | POST | `oc_location` + `change_log` | revert a run's OC rows to the stored before-image (7-day window) |
| **`/api/sync/safe-mode`** (v2.6.0) | PATCH | `sync_run` (flag) | freeze / unfreeze all propagation |
| `/api/stats/overview` | GET | `vendor_location` + `oc_location` + `change_log` | overview KPIs |
