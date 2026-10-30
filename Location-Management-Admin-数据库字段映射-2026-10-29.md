# Location Management Admin Console — Database Field Mapping (v2.0)

> Companion to PRD `Location-Management-Admin-PRD-2026-10-29` (v2.0)
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

## 一、筛选条件 ↔ 数据库字段映射

### 1.1 Vendor Location List (`GET /api/vendor-locations`)

| UI Filter | API Param | Table | Field | Type | Match | Notes |
|---|---|---|---|---|---|---|
| Location ID | `location_id` | `vendor_location` | `vendor_location_id` | varchar(32) | exact/prefix | alphanumeric, ≤32 |
| Name | `name` | `vendor_location` | `name` | varchar | LIKE %…% | hits index |
| Address keyword | `address_keyword` | `vendor_location` | `address_line1`/`address_line2` | varchar | LIKE %…% | joint search |
| Provider | `provider` | `vendor_location` | `source_provider` | enum(PUDO,LMS) | exact | cascades Type |
| Sweeper | `sweeper` | `vendor_location` | `sweeper` | enum(PUDO,LMS) | exact | OC-editable |
| Status | `status` | `vendor_location` | `status` | enum(active,inactive,removed) | exact | provider-sourced, read-only |
| Type | `location_type` | `vendor_location` | `location_type` | enum(PUDOpoint,Locker,packagehub) | exact | provider-exclusive |
| City | `city` | `vendor_location` | `city` | varchar | exact | — |
| State | `state` | `vendor_location` | `state` | varchar | exact | — |
| Created from/to | `created_from`/`created_to` | `vendor_location` | `created_at` | timestamp | BETWEEN | ≤1yr span |
| Updated from/to | `updated_from`/`updated_to` | `vendor_location` | `updated_at` | timestamp | BETWEEN | ≤1yr span |

### 1.2 OC Location List (`GET /api/oc-locations`)

| UI Filter | API Param | Table | Field | Type | Match | Notes |
|---|---|---|---|---|---|---|
| OC Location ID | `oc_location_id` | `oc_location` | `oc_location_id` | varchar(32) | exact/prefix | — |
| Name | `name` | `oc_location` | `name` | varchar | LIKE %…% | OC-editable |
| Address keyword | `address_keyword` | `oc_location` | `address_line1`/`address_line2` | varchar | LIKE %…% | OC-editable |
| Provider | `provider` | `vendor_location` (via mapping) | `source_provider` | enum | exact | join through mapping |
| Sweeper | `sweeper` | `vendor_location` (via mapping) | `sweeper` | enum | exact | join through mapping |
| Status | `status` | `oc_location` | `status` | enum(active,inactive,removed) | exact | derived from linked vendor |
| Enable | `is_enable` | `oc_location` | `is_enable` | bool | exact | OC switch |
| Type | `location_type` | `vendor_location` (via mapping) | `location_type` | enum | exact | join through mapping |
| City | `city` | `oc_location` | `city` | varchar | exact | OC-editable |
| State | `state` | `oc_location` | `state` | varchar | exact | OC-editable |
| Created from/to | `created_from`/`created_to` | `oc_location` | `created_at` | timestamp | BETWEEN | ≤1yr span |
| Updated from/to | `updated_from`/`updated_to` | `oc_location` | `updated_at` | timestamp | BETWEEN | ≤1yr span |
| Visibility (implicit) | `visible` | `oc_location` | `status` + `is_enable` | combined | `status=active AND is_enable=true` | see visibility rule |

> **Visibility rule (core)**: an OC location is visible downstream **iff `status = 'active'` AND `is_enable = true`**. `status` is derived from the linked vendor location(s); when the vendor is `inactive`, the OC location is auto-disabled and blocked.

---

## 二、数据表 ↔ 数据库字段

### 表 1：`vendor_location`（供应商门店层）

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
| `hours_of_operation{}` | json | sync | yes | weekly hours |
| `cutoff_time` | time | sync | yes | cutoff (LMS) |
| `holiday_closures[]` | array | sync | yes | holiday closures |
| `created_at` | timestamp | sync | yes | created time |
| `updated_at` | timestamp | sync | yes | updated time |
| `activated_at` | timestamp | sync | yes | activated time |
| `deactivated_at` | timestamp | sync | yes | deactivated time |
| `last_synced_at` | timestamp | system | yes | last successful sync |

### 表 2：`oc_location`（OC 门店层）

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
| `status` | enum(active,inactive,removed) | derived | yes | derived from linked vendor location(s) |
| `is_enable` | bool | OC | no | OC switch; only F08/F09 writable |
| `created_at` | timestamp | system | yes | created time |
| `updated_at` | timestamp | system | yes | updated time |

### 表 3：`vendor_oc_mapping`（映射表）

> Links vendor location(s) to OC location. Principle 1:1; confirmed duplicate → 2:1.

| Field | Type | Notes |
|---|---|---|
| `mapping_id` | bigint PK | mapping primary key |
| `vendor_location_id` | varchar FK → vendor_location | vendor location |
| `oc_location_id` | varchar FK → oc_location | OC location |
| `created_at` | timestamp | created time |

### 表 4：`change_log`（变更日志）

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

### 表 5：`duplicate_pair`（跨供应商重复对）

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

### 表 6：`sync_run`（同步运行记录）

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

### 表 7：`user`（用户）

| Field | Type | Notes |
|---|---|---|
| `user_id` | varchar PK | user ID |
| `name` | varchar | name |
| `email` | varchar | email |
| `has_enable_permission` | bool | advanced permission |

---

## 三、字段所有权速查

| Field Category | Owner | Writable Path | Read-only? |
|---|---|---|---|
| Vendor `status` + business attributes (address/hours/services) | Provider | daily sync | yes |
| Vendor `sweeper` | OC (Location Manager) | F10 vendor list/detail | no |
| OC `is_enable` | OC (Location Manager) | F08 single / F09 batch | no |
| OC store details (address, opening hours, etc.) | OC (Location Manager) | F11 OC list/detail | no |
| `oc_location_id` | System | OC generation (F12) + duplicate resolution (F14) | yes |
| `vendor_location_id` / `created_at` | System | — | yes |

> **Key rules**:
> - Provider sync **never overwrites** `sweeper`, OC `is_enable`, OC store details, or `oc_location_id`.
> - When vendor `status` → `inactive`, the linked OC location is auto-disabled and blocked from downstream sync; force-enable raises an error.

---

## 四、API ↔ 表 快速对照

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
| `/api/stats/overview` | GET | `vendor_location` + `oc_location` + `change_log` | overview KPIs |
