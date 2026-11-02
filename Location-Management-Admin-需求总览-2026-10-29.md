# Location Management Admin Console — Requirement Overview (v2.6.1)

> **Live prototype (public)**: https://jama-dev520.github.io/location-prototype/
> **Demo video (2.5 min)**: https://jama-dev520.github.io/location-prototype/demo/location-admin-demo.mp4
> Companion: `Location-Management-Admin-PRD-2026-10-29` (v2.6.1)
> Purpose: a quick-reference doc answering three questions — ① what each field means; ② what this requirement does and its priority; ③ what the core flows look like.
> Detail view (v2.4.0): clicking a location ID opens a **full page** with tabs — Basic Info / Operations / Hours & Services / Linked Records / Change History.
> Sync protection (v2.6.0): provider data is a **proposal** — an enabled location keeps a signed-off baseline, changes are held until someone decides, and enabling is a sign-off checklist.

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
| `qr_dropoff` | bool | sync | yes | **QR-code dropoff capability（LMS 独有）**：顾客出示 QR code 而非打印面单即可投递；PUDO 不提供该能力 → N/A |
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
| `qr_dropoff` | bool | backfill (derived) | yes | QR-code dropoff：任一关联 vendor location 支持即为 `true` |
| `status` | enum(active/inactive/removed) | derived | yes | derived from linked vendor location(s) |
| `is_enable` | bool | OC | no | **OC switch**; new OC location defaults `false`; F08/F09 writable |
| `intake_status` | enum(pass/warn/fail/overridden) | intake check | yes | 数据质量准入结果；`fail` 表示根本没生成 OC 门店 |
| `baseline_at` | timestamp | system | yes | **人工核实过的那一版的时间** —— 判断"好数据"的参照物 |
| `propagated_at` | timestamp | system | yes | 最后一次推给下游的时间（缓存已失效）；`null` = 从未推送 |
| `created_at`/`updated_at` | timestamp | system | yes | created/updated |

### 1.4 Mapping / Change Log / Duplicate / Sync / User

| Table | Key fields | Meaning |
|---|---|---|
| `vendor_oc_mapping` | `mapping_id` / `vendor_location_id` / `oc_location_id` / `created_at` | links vendor ↔ OC (principle 1:1; duplicate 2:1) |
| `change_log` | `log_id` / `location_id` / `location_layer` / `occurred_at` / `operator` / `operation_type` / `field_name` / `old_value` / `new_value` / `source` / `remark` | append-only audit trail |
| `duplicate_pair` | `pair_id` / `vendor_location_a` / `vendor_location_b` / `matched_on` / `match_score` / `diff` / `status` / `detected_at` / `resolved_by` / `resolved_at` | cross-provider fuzzy duplicate pairs |
| `sync_run` | `run_id` / `provider` / `started_at` / `finished_at` / `result` / counts / `error_message` | one record per provider per day |
| `user` | `user_id` / `name` / `email` / `has_enable_permission` | SSO user; `has_enable_permission` gates enable/edit |
| **`oc_field_baseline`** (v2.6.0) | `baseline_id` / `oc_location_id` / `field_name` / `baseline_value` / `version_no` / `captured_at` / `capture_reason` | 每个字段"人工核实过"的历史值，保留最近 5 版，供单字段回退 |
| **`sync_change_set`** (v2.6.0) | `change_id` / `run_id` / `oc_location_id` / `field_name` / `old_value` / `new_value` / `risk_tier` / `decision` / `reason` | 上游变更的字段级记录；`pending` = 已记录但未写入 OC |
| **`field_lock`** (v2.6.0) | `lock_id` / `oc_location_id` / `field_name` / `locked_at` / `reason` | 字段锁：锁住后该字段的上游变更一律走 review |
| **`intake_issue`** (v2.6.0) | `issue_id` / `vendor_location_id` / `check_code` / `severity` / `status` | 数据质量检查项；`fail` 阻断生成 OC 门店 |

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
| `ENABLE_SIGNED_OFF` (v2.6.0) | 启用签收：完成核对并记录基线 |
| `DOWNSTREAM_PROPAGATED` (v2.6.0) | 人工编辑后立即推送下游（缓存失效） |
| `SYNC_CHANGE_ACCEPTED` / `SYNC_CHANGE_REJECTED` (v2.6.0) | 接受上游变更 / 拒绝并锁定字段 |
| `SYNC_ROLLBACK` (v2.6.0) | 按同步批次或单字段回退 |
| `INTAKE_OVERRIDDEN` (v2.6.0) | 数据质量不合格但人工强制生成 OC 门店（仍不可启用） |
| `SYNC_QUARANTINED` (v2.6.0) | 硬校验失败，整条记录隔离，不写 OC |

### 1.6 Key semantic distinctions

| Concept | Meaning | Key difference |
|---|---|---|
| `vendor_location` vs `oc_location` | provider raw record vs OC-owned entity | two layers; OC layer is downstream-facing |
| `sweeper` | who physically picks up packages | renamed from "Pickup"/`pickup_provider`; OC-editable |
| `status` vs `is_enable` | provider business status vs OC switch | independent; `is_enable` lives on OC layer |
| `removed` vs `inactive` | permanent delisting vs temporary closure | distinct semantics |
| `vendor_location_id` vs `oc_location_id` | internal vendor ID vs OC-owned ID | OC ID gates downstream sync |
| **baseline vs 当前值** (v2.6.0) | 人工核实过的版本 vs 库里的值 | 上游改了、但运营没点头时，两者**不相等**，OC 用 baseline 对外 |
| **held vs rejected** (v2.6.0) | 已记录但未写入（等决定） vs 已明确拒绝并锁字段 | held 可以随后接受；rejected 之后同字段不再自动应用 |
| **手改 vs 上游变更** (v2.6.0) | 人工裁决 vs 待裁决提案 | 手改立即生效并锁字段；上游变更按档位挂起 |
| **四个 review tab** (v2.6.0) | duplicates / data quality / sync changes / sign-off | 决策对象分别是「对 / 整条 / 字段 / 记录」，不能合成一个列表 |

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
| F18 | QR Dropoff | QR-code dropoff marker | P0 | LMS-only flag; shown on both lists + detail, filterable, exported |
| F19 | Review Center | Intake data-quality gate | P0 | 新门店地址/坐标/营业时间不合格 → **不生成 OC 门店**，进 Data Quality |
| F20 | Review Center | Field-level provider change review | P0 | 已上线门店被上游改动 → 挂起，保持签收值；逐字段接受/拒绝并锁定 |
| F21 | Enable | Enable sign-off (replaces one-click) | P0 | 启用前核对地址/坐标/营业时间 + 5 项检查 + 必须勾选，同时记基线 |
| F22 | Data Sync | Breaker & rollback | P1 | 单次 run 变更过多 → 降级干跑；按 run 一键回滚（7 天） |

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
| 11 | QR dropoff | LMS-only capability flag (`qr_dropoff`); shown as marker on both lists + detail; OC backfilled from linked vendor; read-only |
| 12 | Provider data (v2.6.0) | **提案，不是事实**：未上线门店直接写入；已上线门店按字段档位判定，非低风险字段一律挂起，OC 保持签收基线值 |
| 13 | Baseline (v2.6.0) | 「好数据」的参照物是**人工核实过的那一版**，不是"上次同步写进去的值"。签收/接受变更/人工编辑三处产生 |
| 14 | Review Center (v2.6.0) | 一个菜单四个 tab：Duplicates（一对记录）· Data Quality（整条记录）· Sync Changes（单个字段）· Enable Sign-off（能否上线）。决策粒度不同，状态机独立 |
| 15 | Enable (v2.6.0) | 启用 = 签收（核对 + 勾选 + 记基线），关闭 = 一键生效。批量启用逐条过检查并回报跳过项 |
| 16 | Operator edit (v2.6.0) | 人工编辑是**最终裁决**，不进 review：立即推下游 + 刷新基线 + 锁定字段 + 作废同字段的待审上游变更 |
| 17 | Fail-safe (v2.6.0) | 队列无人处理的结果永远是"不生成 / 不启用 / 保持旧值"，系统绝不替人决定接受上游值 |

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

### 3.7 上游变更审核（v2.6.0）— 挂起 → 接受 / 拒绝并锁定

![Provider Change Review](https://jama-dev520.github.io/location-prototype/diagrams/flow7-sync-change-review.png)

> 已上线门店的字段被上游改动时，先记账不动手：值挂在 `sync_change_set`，OC 继续用基线值对外。接受才会写并刷新基线；拒绝则保持旧值并把字段锁死。没人处理 = 什么都不变。

### 3.8 启用签收（v2.6.0）— 上线前的核对清单

![Enable Sign-off](https://jama-dev520.github.io/location-prototype/diagrams/flow8-enable-signoff.png)

> 启用是门店唯一变成「客户可见」的时刻，所以它是签收而不是开关：先过 4 项阻塞检查，再把地址/坐标/营业时间连同来源一起摆出来，勾选确认前按钮点不动。关闭则始终一键，因为下线是安全方向。

### 3.9 手改即裁决（v2.6.0）— 立即写入、锁字段、立即推下游

![Operator Edit](https://jama-dev520.github.io/location-prototype/diagrams/flow9-operator-edit.png)

> 运营手改被当作「决定」而不是「提案」：刷新基线、锁住改过的字段、把该字段上挂起的上游变更标记为 superseded，若门店已可见则立刻推下游。手改不进 review 队列——让人审批自己的修改只是多加一步。

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
| **上游数据是提案**（v2.6.0） | 已上线门店被上游改动 → 按字段档位判定，敏感字段挂起，客户继续看签收值 |
| **一个审查入口**（v2.6.0） | Review Center = Duplicates + Data Quality + Sync Changes + Enable Sign-off |
| **启用即签收**（v2.6.0） | 核对地址/坐标/营业时间 + 5 项检查 + 必须勾选，同时记录基线 |
| **人工编辑即生效**（v2.6.0） | 不进 review，立即推下游 + 刷基线 + 锁字段 + 作废同字段待审变更 |
| **兜底原则**（v2.6.0） | 没人处理 → 不生成 / 不启用 / 保持旧值，绝不自动接受上游错值 |
| **验收** | 见 `Location-Management-Admin-验收标准-2026-10-29.md`（通俗版，8 条快检 + 6 个故事） |
| **交付页面** | P01 Vendor List / P02 OC List / P03 详情 / **P04 Review Center（4 tab）** / P05 全局日志 / P06 同步状态 / P07 仪表盘 |
