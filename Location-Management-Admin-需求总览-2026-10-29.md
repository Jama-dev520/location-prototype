# Location Management Admin Console — Requirement Overview (v2.0)

> **Live prototype (public)**: https://jama-dev520.github.io/location-prototype/
> Companion: `Location-Management-Admin-PRD-2026-10-29` (v2.0)
> Purpose: a quick-reference doc answering three questions — ① what each field means; ② what this requirement does and its priority; ③ what the core flows look like.

---

## 一、字段解释（数据字典）

### 1.1 Two-Layer Model (v2.0)

The console is split into **two layers**:

| Layer | Table | Meaning | Who owns it |
|---|---|---|---|
| **Vendor Location** | `vendor_location` | Raw provider data (PUDO Inc / LMS) | Provider (read-only except `sweeper`) |
| **OC Location** | `oc_location` | OC-owned, downstream-facing store entity | OC (enable switch + editable store details) |

A **mapping table** (`vendor_oc_mapping`) links them — principle **1:1** (one vendor location → one OC location); a confirmed cross-provider duplicate links **2:1**.

### 1.2 `vendor_location`（供应商门店层）

| Field | Type | Source | Read-only | Meaning |
|---|---|---|---|---|
| `vendor_location_id` | varchar(32) PK | system | yes | permanent internal ID |
| `source_provider` | enum(PUDO/LMS) | sync | yes | source provider |
| `provider_ref` | varchar | sync | yes | dealerNo (PUDO) / loc_id (LMS) |
| `store_code` | varchar | sync | yes | store code |
| `name` | varchar | sync | yes | store name |
| `location_type` | enum | sync | yes | provider-exclusive: PUDO → PUDOpoint/Locker; LMS → packagehub |
| `sweeper` | enum(PUDO/LMS) | OC (editable) | no | **Sweeper** — who physically picks up packages (renamed from "Pickup"); default = source_provider; cross-provider allowed |
| `address_line1`/`address_line2` | varchar | sync | yes | address |
| `city`/`state`/`postal_code`/`country` | varchar | sync | yes | city/state/postal/country |
| `latitude`/`longitude` | decimal | sync | yes | coordinates |
| `phone` | varchar | sync | yes | phone (nullable) |
| `timezone` | varchar | sync | yes | timezone |
| `status` | enum(active/inactive/removed) | sync | yes | provider business status |
| `services[]`/`restrictions[]` | array | sync | yes | services/restrictions |
| `hours_of_operation{}` | json | sync | yes | weekly hours |
| `cutoff_time` | time | sync | yes | cutoff (LMS) |
| `holiday_closures[]` | array | sync | yes | holiday closures |
| `created_at`/`updated_at` | timestamp | sync | yes | created/updated (**second-level precision**) |
| `last_synced_at` | timestamp | system | yes | last successful sync |

### 1.3 `oc_location`（OC 门店层）

| Field | Type | Source | Read-only | Meaning |
|---|---|---|---|---|
| `oc_location_id` | varchar(32) PK | system | yes | OC-owned ID; gates downstream sync |
| `name` | varchar | backfill → OC-editable | no | store name (OC can override) |
| `address_line1`/`address_line2` | varchar | backfill → OC-editable | no | address (OC can override) |
| `city`/`state`/`postal_code`/`country` | varchar | backfill → OC-editable | no | city/state/postal/country |
| `latitude`/`longitude` | decimal | backfill → OC-editable | no | coordinates |
| `phone` | varchar | backfill → OC-editable | no | phone |
| `opening_hours{}` | json | backfill → OC-editable | no | opening hours (OC can override) |
| `status` | enum(active/inactive/removed) | derived | yes | derived from linked vendor location(s) |
| `is_enable` | bool | OC | no | **OC switch**; new OC location defaults `false`; F08/F09 writable |
| `created_at`/`updated_at` | timestamp | system | yes | created/updated |

### 1.4 Mapping / Change Log / Duplicate / Sync / User

| Table | Key fields | Meaning |
|---|---|---|
| `vendor_oc_mapping` | `mapping_id` / `vendor_location_id` / `oc_location_id` / `created_at` | links vendor ↔ OC (principle 1:1; duplicate 2:1) |
| `change_log` | `log_id` / `location_id` / `location_layer` / `occurred_at` / `operator` / `operation_type` / `field_name` / `old_value` / `new_value` / `source` / `remark` | append-only audit trail |
| `duplicate_pair` | `pair_id` / `vendor_location_a` / `vendor_location_b` / `matched_on` / `match_score` / `diff` / `status` / `detected_at` / `resolved_by` / `resolved_at` | cross-provider fuzzy duplicate pairs |
| `sync_run` | `run_id` / `provider` / `started_at` / `finished_at` / `result` / counts / `error_message` | one record per provider per day |
| `user` | `user_id` / `name` / `email` / `has_enable_permission` | SSO user; `has_enable_permission` gates enable/edit |

### 1.5 Audit operation types (`operation_type`)

| Type | Meaning |
|---|---|
| `ENABLE` / `DISABLE` | single enable/disable (OC layer) |
| `BATCH_ENABLE` / `BATCH_DISABLE` | batch enable/disable (OC layer) |
| `SWEEPER_CHANGE` | sweeper (pickup provider) change |
| `STORE_DETAIL_EDIT` | OC store detail edit (address/hours/etc.) |
| `OC_GENERATED` | OC location auto-generated from vendor |
| `OC_LINK_CONFIRMED` / `OC_LINK_REJECTED` | duplicate resolution: same-store / not-same |
| `PROVIDER_STATUS_CHANGE` | vendor status change (sync) |
| `PROVIDER_FIELD_UPDATE` | vendor field update (sync) |
| `PROVIDER_CREATED` / `PROVIDER_REMOVED` | vendor created/removed (sync) |

### 1.6 Key semantic distinctions

| Concept | Meaning | Key difference |
|---|---|---|
| `vendor_location` vs `oc_location` | provider raw record vs OC-owned entity | two layers; OC layer is downstream-facing |
| `sweeper` | who physically picks up packages | renamed from "Pickup"/`pickup_provider`; OC-editable |
| `status` vs `is_enable` | provider business status vs OC switch | independent; `is_enable` lives on OC layer |
| `removed` vs `inactive` | permanent delisting vs temporary closure | distinct semantics |
| `vendor_location_id` vs `oc_location_id` | internal vendor ID vs OC-owned ID | OC ID gates downstream sync |

**Visibility rule (core)**: an OC location is visible downstream **iff `status = active` AND `is_enable = true`**. When the linked vendor location is `inactive`, the OC location is auto-disabled and blocked.

---

## 二、需求总结 + 优先级

### 2.1 一句话概括

> Build an internal console with **two layers**: a **Vendor Location List** (raw provider data, read-only except sweeper) and an **OC Location List** (OC-owned, downstream-facing — enable switch + editable store details). OC locations are auto-generated from vendor records; when a vendor goes inactive, its OC location is auto-disabled.

### 2.2 Priority table

> P0 = must-have (MVP) / P1 = important / P2 = deferrable

| ID | Module | Feature | Priority | One-liner |
|---|---|---|---|---|
| F01 | Vendor List | Paginated vendor list | P0 | raw provider locations, read-only except sweeper |
| F02 | Vendor List | Filter & search | P0 | multi-condition AND |
| F03 | OC List | Paginated OC list | P0 | OC-owned locations with enable + linked vendor |
| F04 | OC List | Filter & search | P0 | multi-condition AND |
| F05 | Export | Export current view | P0 | both lists, ≤10k rows |
| F06 | Detail | Detail (vendor/OC) | P0 | tabs; OC detail shows editable fields |
| F07 | Change History | Per-location log | P0 | full timeline |
| F08 | Enable | Single enable/disable | P0 | OC layer; blocked if vendor inactive |
| F09 | Enable | Batch enable/disable | P0 | Mode A inline + Mode B Excel |
| F10 | Sweeper | Edit sweeper | P0 | vendor layer; writes SWEEPER_CHANGE |
| F11 | OC Edit | Edit OC store details | P0 | address/hours/etc.; writes STORE_DETAIL_EDIT |
| F12 | OC Generation | Auto-generate OC | P0 | new vendor (non-duplicate) → one OC (disabled) |
| F13 | Status Propagation | Vendor inactive → OC disable | P0 | auto-disable + block downstream |
| F14 | Duplicate | Fuzzy duplicate review | P0 | ≥75% flagged; no OC until resolved |
| F15 | Change History | Global log | P1 | cross-location search |
| F16 | Data Sync | Sync status | P1 | per-provider run + failure alerts |
| F17 | Overview | Dashboard | P2 | KPIs |

### 2.3 Key business decisions

| # | Decision | Conclusion |
|---|---|---|
| 1 | Architecture | Two-layer: Vendor Location (provider) + OC Location (OC-owned) |
| 2 | Menu | Two menus: "Vendor Location List" + "OC Location List" |
| 3 | OC generation | New vendor (non-duplicate) → auto-generate one OC location, `is_enable=false` |
| 4 | Mapping | Principle 1:1; confirmed duplicate 2:1 |
| 5 | Sweeper | Renamed from "Pickup"; OC-editable; cross-provider |
| 6 | OC store edit | OC can edit address, opening hours, and flagged fields |
| 7 | Status propagation | Vendor inactive → OC auto-disable + block; force-enable errors |
| 8 | Duplicate | Fuzzy match ≥75%; no OC until resolved |
| 9 | Visibility | OC visible iff `status=active AND is_enable=true` |
| 10 | Roles | Viewer (view/export) vs Location Manager (+sweeper/edit/enable/duplicate) |

---

## 三、流程图

### 3.1 Vendor Sync & OC Generation

![Vendor Sync & OC Generation](https://jama-dev520.github.io/location-prototype/diagrams/flow2-daily-sync.png)

### 3.2 Status Propagation — Vendor Inactive → OC Auto-Disable

![Status Propagation](https://jama-dev520.github.io/location-prototype/diagrams/flow6-status-propagation.png)

### 3.3 Operator Batch Disable (OC Layer)

![Operator Batch Disable](https://jama-dev520.github.io/location-prototype/diagrams/flow1-batch-disable.png)

### 3.4 Sweeper Edit

![Sweeper Edit](https://jama-dev520.github.io/location-prototype/diagrams/flow3-sweeper.png)

### 3.5 Duplicate Review (fuzzy)

![Duplicate Review](https://jama-dev520.github.io/location-prototype/diagrams/flow4-duplicate.png)

### 3.6 Exception

![Exception](https://jama-dev520.github.io/location-prototype/diagrams/flow5-exception.png)

---

## 附：一页速览（30 秒看懂）

| Dimension | Content |
|---|---|
| **做什么** | 两层模型内部后台：供应商层 + OC 层 |
| **两个菜单** | Vendor Location List（供应商数据）+ OC Location List（OC 自有门店） |
| **给谁用** | Viewer（查看/导出）、Location Manager（+sweeper/编辑/启用/去重） |
| **数据怎么来** | 每日增量同步；新供应商门店非重复则自动生成 OC 门店（默认 disabled） |
| **核心控制** | OC 层 `is_enable` 开关，决定下游可见性 |
| **状态联动** | 供应商 inactive → OC 自动 disable + 阻断下游；强制启用报错 |
| **去重** | 跨供应商地址模糊匹配 ≥75% 疑似重复；未确认不生成 OC |
| **审计** | 每次 sweeper/编辑/启用/去重/状态变化都留痕 |
| **交付页面** | P01 Vendor List / P02 OC List / P03 详情 / P04 重复审核 / P05 全局日志 / P06 同步状态 / P07 仪表盘 |
