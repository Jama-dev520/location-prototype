# Location Management Admin Console — Requirement Overview (v3.2.1)

> **Live prototype (public)**: https://jama-dev520.github.io/location-prototype/
> Companion: `Location-Management-Admin-PRD-2026-10-29` (v3.2.1)
> 给外部同事看的英文功能概览：https://jama-dev520.github.io/location-prototype/feature-overview.html
> Purpose: a quick-reference doc answering three questions — ① what each field means; ② what this requirement does and its priority; ③ what the core flows look like.
> Detail view (v2.4.0): clicking a location ID opens a **full page** with tabs — Basic Info / Operations / Hours & Services / Linked Records / Change History.
> **v3.0.0 — 供应商之外，自营 M&MP 小店也走同一套。** M&MP = 自营小店（美国叫法），是门店目录里的第三类来源，数据由门店自己维护，异常找门店负责人；规则与供应商门店完全一致。
>
> **v3.0.0 — provider data is the only data.** The OC layer no longer edits anything: it has exactly two verbs, **enable** and **flag an anomaly**. A wrong sweeper or address is flagged back to the provider with a reason (on the vendor record, no separate page). Duplicate checking is its own page and is marked 2:1 in the OC list. Downstream values are merged **per field with LMS first**. M&MP works the same data under the same rules.

---

## 一、字段解释（数据字典）

### 1.1 Two-Layer Model (v2.0)

The console is split into **two layers**:

| Layer | Table | Meaning | Who owns it |
|---|---|---|---|
| **Vendor Location** | `vendor_location` | Raw provider data (PUDO Inc / LMS), **including sweeper**; plus our own **M&MP self-operated stores** | Provider (PUDO Inc / LMS) or the store itself (M&MP) — fully read-only for OC |
| **OC Location** | `oc_location` | Downstream-facing store, assembled per field from its vendor records | OC — **enable + flag only**, no editing |

A **mapping table** (`vendor_oc_mapping`) links them — principle **1:1**; a duplicate confirmed as the same store links **2:1** and the OC record is merged field by field (**LMS wins**, PUDO fills the gaps).

### 1.2 `vendor_location`（供应商门店层）

| Field | Type | Source | Read-only | Meaning |
|---|---|---|---|---|
| `vendor_location_id` | varchar(32) PK | system | yes | permanent internal ID |
| `source_provider` | enum(PUDO/LMS) | sync | yes | source provider |
| `provider_ref` | varchar | sync | yes | dealerNo (PUDO) / loc_id (LMS) |
| `store_code` | varchar | sync | yes | store code |
| `name` | varchar | sync | yes | store name |
| `location_type` | enum | sync | yes | provider-exclusive：PUDO Inc → PUDOpoint/Locker；LMS → packagehub；M&MP → M&MP store。**筛选器里由 Source 决定可选项**（v3.2.0） |
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
| **`oc_field_baseline`** (v2.6.0) | `baseline_id` / `oc_location_id` / `field_name` / `baseline_value` / `captured_at` / `capture_reason` | 签收时的值，用来判断「签收后变了哪些字段」；v3.0 起不再用于回退 |
| **`sync_change_set`** (v2.6.0) | `change_id` / `run_id` / `oc_location_id` / `field_name` / `old_value` / `new_value` / `risk_tier` / `decision` / `reason` | 上游变更的字段级记录；`pending` = 已记录但未写入 OC |
| **`data_issue`** (v3.0.0，v3.1.0 细化) | `issue_id` / `vendor_id` / `field_name` / **`sub_field`** / **`issue_type`** / **`severity`** / `reason` / `status` / `owner_type` / `raised_by` / `raised_at` | 数据异常标记：只记录问题，不改值。粒度 = 字段 → 子字段 → 类型 → 严重度；`blocking` 阻止签收，`warning` 不阻止 |
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
| `FIELD_UNLOCKED` (v2.7.0) | 解锁字段：手理解锁或"接受并解锁"，必带理由 |
| `DATA_ISSUE_RAISED` (v3.0.0) | 标记数据异常：字段 + 原因，推供应商改 |
| `RE_SIGNED_OFF` (v3.0.0) | 重新签收：认可漂移后的新值，刷新基线 |
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
| **标记 vs 改值** (v3.0.0) | 报告问题 vs 修改数据 | OC 只能标记，值只由供应商给；标记不改值、不下线 |
| **四个 review tab** (v2.6.0) | duplicates / data quality / sync changes / sign-off | 决策对象分别是「对 / 整条 / 字段 / 记录」，不能合成一个列表 |

**Visibility rule (core)**: an OC location is visible downstream **iff `status = active` AND `is_enable = true`**. When the linked vendor location is `inactive`, the OC location is auto-disabled and blocked.

---

## 二、需求总结 + 优先级

### 2.1 背景与目的：运营每天要做的四件事

这套后台不是为了「管理数据」，是为了让运营**在几秒内**做完四件具体的活。从 v3.0.0 起，这四件事里已经没有「改数据」——数据归供应商，OC 只判断能不能给客户看，以及把错的地方标回去。

![整体全貌](https://workbuddy-space-static.codebuddy.work/image/M9nBRbD9eQJBNFivHPpu8l.png)

*一张图看全：供应商 → 供应商层 → 两道闸门 → OC 层 → 下游；右侧是三个决策队列，底部是贯穿全链路的变更记录。J1–J4 标记的四件事，就是运营每天真正要做的动作。*

| # | 运营要做什么 | 做完是什么样 | 在哪儿做 |
|---|---|---|---|
| **J1** | **查哪些门店是 enabled / 在线的** | 问「现在有哪些店在客户那儿可见」，界面上直接得到答案：按 `status = active 且 is_enable = true` 筛一下，看到条数，把这一批导出来。不用提 SQL、不用找研发、不用导全量再自己筛 | P02 · `Enable` / `Downstream` 筛选 + 导出 CSV |
| **J2** | **确认谁去揽收（Sweeper）对不对** | sweeper 现在随同步一起进来，OC 不再改它。不对就**标记异常 + 写原因**，推供应商改，下次同步进来 | 供应商门店详情 → **Flag a data issue**（field = sweeper） |
| **J3** | **确认门店地址对不对** | 地址、城市、邮编、坐标错 → **标记异常**，理由写在供应商记录上；改由供应商做，OC 不做二次录入 | 供应商门店详情 → **Flag a data issue**，或在 Data Issues 里点「Flag as an issue」 |
| **J4** | **及时查前后变化** | 任意门店、任意字段：谁改的、什么时候、旧值 → 新值。查一家店看它自己的时间线；和供应商对账时看全站日志 | P03 · **Change History** / P05 · **Change Log** |

> **为什么这四件事决定了架构。** 它们作用在**同一条门店**上，但归属不同：J2/J3 是供应商那条记录的事实（所以只能标记、不能改），J1 问的是「下游目录里有没有这家」（`status AND is_enable` 的组合，既不是供应商记录也不是 OC 自己的值），J4 要把这三种都覆盖。所以有两个列表而不是一个，启用开关只在 OC 层，而变更记录是流程的一部分。

两条推论也先说在前面，它们解释了后面很多规则的由来：

- **J1 是频率最高、也最容易答错的一件。** 每次投放、每次故障、每次和供应商复盘，第一个问题都是「哪些店在线」。如果答案要靠导出全量再自己筛，那答案永远慢半拍——所以开关、筛选、条数、导出这四样必须永远互相对得上。
- **J3 和 J4 是一对。** 标记异常之所以有意义，前提是**同一个问题不会被标两次、修好了看得见**。所以一条 issue 有 open → reported → fixed 的状态，修复结果由下次同步带进来，而不是靠人再点一次。

### 2.2 一句话概括（架构层面）

> Build an internal console with **two layers**: a **Vendor Location List** (provider data, fully read-only — a wrong value is flagged with a reason and sent back to the provider) and an **OC Location List** (what customers see — **enable + flag only**). OC records are auto-generated from vendor records; when every linked vendor is inactive the OC record is auto-disabled. Downstream values are merged per field from the linked records, **LMS first**.

### 2.3 Priority table

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
| F10 | Sweeper | Sweeper sync + flag | P0 | sweeper comes with the sync, read-only; a wrong one is flagged (`DATA_ISSUE_RAISED`) |
| F11 | Data issue | Flag an anomaly on a vendor field | P0 | field + reason, no value change; open issues block enable |
| F12 | OC Generation | Auto-generate OC | P0 | new vendor (non-duplicate) → one OC (disabled) |
| F13 | Status Propagation | Vendor inactive → OC disable | P0 | auto-disable + block downstream |
| F14 | Duplicate | Standalone duplicate check | P0 | own page; same store → 2:1 (marked in OC list), not same → 1:1 |
| F15 | Change History | Global log | P1 | cross-location search |
| F16 | Data Sync | Sync status | P1 | per-provider run + failure alerts |
| F17 | Overview | Dashboard | P2 | KPIs |
| F18 | QR Dropoff | QR-code dropoff marker | P0 | LMS-only flag; shown on both lists + detail, filterable, exported |
| F19 | Review Center | Intake data-quality gate | P0 | 新门店地址/坐标/营业时间不合格 → **不生成 OC 门店**，进 Data Quality |
| F20 | Drift | Baseline drift + re-sign | P0 | 已上线门店被上游改动 → 下游用新值，台账显示漂移，可重新签收或标记异常 |
| F21 | Enable | Enable sign-off | P0 | 前置 5 项检查 + 必须勾选 + 记基线；无在营供应商/缺时间/重复未决/有未关问题都不能启用 |
| F22 | Data Sync | Breaker & rollback | P1 | 单次 run 变更过多 → 降级干跑；按 run 一键回滚（7 天） |
| F23 | *(retired)* | Field lock release | — | v3.0.0 删除：没有手改，就没有需要解锁的锁 |
| F24 | *(retired)* | Field-level revert | — | v3.0.0 删除：值只能由供应商改，回退等于改数据 |
| F25 | Merge | LMS-first field merge | P0 | 同一条 OC 记录逐字段取源：LMS 有值用 LMS，缺了用 PUDO，详情页标来源 |
| F26 | Data Issues | One ledger for all anomalies | P0 | 供应商问题 + 签收后漂移 + 入库失败合成一页，替代 Review Center |

### 2.4 Key business decisions

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
| 14 | No review center (v3.0.0) | 只保留一个 **Data Issues 台账**（供应商问题 / 漂移 / 入库失败）；重复裁决独立成页。不再有多个 review tab |
| 15 | Enable (v2.6.0) | 启用 = 签收（核对 + 勾选 + 记基线），关闭 = 一键生效。批量启用逐条过检查并回报跳过项 |
| 16 | No OC editing (v3.0.0) | OC 不再改任何值。数据错 → **标记异常**（字段 + 原因），推供应商修；OC 只保留 enable 与标记两个动作 |
| 17 | Fail-safe (v2.6.0) | 队列无人处理的结果永远是"不生成 / 不启用 / 保持旧值"，系统绝不替人决定接受上游值 |
| 18 | Sweeper (v3.0.0) | Sweeper 随同步进来，OC 只读。跨供应商揽收仍允许（值本身来自供应商），但 OC 不能改，只能标记 |
| 19 | Duplicate (v3.0.0) | 独立 **Duplicate Check** 页；同一家 → 2:1 并在 OC 列表标出；不是同一家 → 1:1。未裁决的重复对阻塞启用 |
| 20 | Merge priority (v3.0.0) | 同一条 OC 记录逐字段取源：**M&MP（自营）→ LMS → PUDO Inc 依次取，先有值的赢**。详情页与列表都标出来源 |
| 21 | Drift (v3.0.0) | 签收后供应商改了值：下游直接用新值，台账显示「与基线不同」，运营可**重新签收**或**标记异常**。基线是参照物，不是冻结器 |
| 24 | Source 决定 Type (v3.2.0) | 一个来源只可能产生它自己的类型：**LMS → packagehub；PUDO Inc → PUDOpoint / Locker；M&MP → 自营门店**。所以筛选不能把 Source 和 Type 当两个独立条件：选 Source 后 Type 只列该来源能产生的值（带条数），只有一个选项时自动选中；清空 Source 也清空 Type。列表里若出现「来源不可能产生的类型」，标 `type ≠ source` 并可用 `Issues → Type ≠ source` 一键筛出——这是发现供应商传错的手段，不是在这里改值 |
| 23 | Issue 粒度 (v3.1.0) | 一个 issue 必须说清四层：哪个**字段**（address_line1）→ 哪个**子字段**（address_line2 / SATURDAY / latitude）→ 哪类**问题**（缺失 / 错误 / 不一致 / 越界 / 重复 / 过期 / 格式）→ 多**严重**（blocking 阻止签收 / warning 只提示）。子字段是让 issue 可执行的关键：「营业时间不对」供应商没法改，「周六没有时间段」可以 |
| 22 | M&MP (v3.0.0) | **M&MP = 自营小店**（美国叫法），是第三类数据来源，与 PUDO Inc / LMS 并列，不是角色或团队。同样的目录、同样的 enable 与签收、同样的异常台账；差别是 M&MP 的数据由门店自己维护，标记异常找**门店负责人（内部）**而不是供应商 |

---

## 三、流程图

### 3.1 Vendor Sync & OC Generation

![Vendor Sync & OC Generation](https://workbuddy-space-static.codebuddy.work/image/tQa1reB86SbeW5E7dHCEvt.png)

### 3.2 Status Propagation — Vendor Inactive → OC Auto-Disable

![Status Propagation](https://workbuddy-space-static.codebuddy.work/image/F1Z3SApwjlM6stfEn0tCnp.png)

### 3.3 Operator Batch Disable (OC Layer)

![Operator Batch Disable](https://workbuddy-space-static.codebuddy.work/image/bRWhlIpawHygvuqvunNgMg.png)

### 3.4 Sweeper 随同步进来（v3.0.0）

sweeper 由供应商随门店一起提供，和其它字段一样只读。发现不对（没给、给了错的揽收方）→ 在供应商记录上**标记异常**，写明原因，推供应商改；下次同步把新值带进来。原「Sweeper 编辑」流程（flow3）已下线。

### 3.5 Duplicate Check（独立页）

![Duplicate Review](https://workbuddy-space-static.codebuddy.work/image/THq85hNgrIbu1LIbM5Pwrz.png)

> 独立页面只回答一个问题：这两条供应商门店是不是同一家。是 → 2:1（两条记录喂一个 OC 门店，下游按字段合并，LMS 优先）；不是 → 1:1。OC 列表里 2:1 的门店带 `2:1` 标记。

### 3.6 Exception

![Exception](https://workbuddy-space-static.codebuddy.work/image/NkJGL93Qos7exTRSPNYWNK.png)

### 3.7 签收后漂移（v3.0.0）— 下游用新值，台账提示

供应商改了已上线门店的值：下游立刻用新值（数据归供应商，OC 不拦），同时台账多一条「与基线不同」。运营两个选择：

- **重新签收**：认可新值，基线刷新，写 `RE_SIGNED_OFF`；
- **标记异常**：不认可，写 `DATA_ISSUE_RAISED`，推供应商修，修好后下次同步回来。

基线在这里的作用变了：v2.x 它**冻结**下游值，v3.0 它**只做参照**——回答「这条门店自签收以来变了哪些字段」。

### 3.8 启用签收（v2.6.0，v3.0.0 收紧）

![Enable Sign-off](https://workbuddy-space-static.codebuddy.work/image/UggYn65vVXnsaDXfZaUvPM.png)

> 启用仍是唯一让门店变成客户可见的动作：先过 5 项前置检查（无在营供应商 / 缺营业时间 / 入库被绕过 / 重复未裁决 / 有未关问题），再把地址、坐标、营业时间连同**每字段来源**摆出来，勾选确认后才可点，确认时记基线。关闭始终一键。

### 3.9 异常闭环（v3.0.0）— 标记 → 催 → 修 → 同步

1. 运营在供应商记录上标记：字段 + 原因 → `data_issue` 状态 `open`。
2. 运营跟进后标 `reported`（已催供应商），或供应商已改则标 `fixed`。
3. 下一次同步把源数据带进来；问题关闭、漂移消失。
4. 该门店在问题关闭前**不能启用**；已经上线的门店不受影响（不会被自动下线）。

原来的「手改即裁决」「字段锁」「字段回退」三条流程（flow9/10/11）在 v3.0.0 全部下线——没有改，就没有锁和回退。

## 四、交付物清单（v3.2.1）

| 交付物 | 文件 | 语言 | 读者 | 说明 |
|---|---|---|---|---|
| 需求总览（本文） | `Location-Management-Admin-需求总览-2026-10-29.md` | 中文 | 全员 | 字段字典、需求与优先级、流程图、交付物清单 |
| 产品需求文档 | `Location-Management-Admin-PRD-2026-10-29.md` | 英文 | 研发 / QA | F01–F26、页面规格、验收标准（§10） |
| 数据库字段映射 | `Location-Management-Admin-数据库字段映射-2026-10-29.md` | 英文 | 后端 / 数据 | 11 张表、筛选↔API↔字段、来源→类型约束 |
| 变更摘要 | `Location-Management-Admin-PRD-变更摘要-2026-10-29.md` | 中文 | 全员 | 每一版改了什么、为什么改 |
| 验收标准（通俗版） | `Location-Management-Admin-验收标准-2026-10-29.md` | 中文 | 业务 / 测试 | 13 条快检、分页面清单、5 个故事、不该发生的事 |
| **英文功能概览** | `feature-overview-en.html` / `Location-Management-Admin-Feature-Overview-EN.md` | 英文 | **外部同事** | 一页讲清做什么、为什么这样设计；可直接转发或打印 |
| 交互原型 | `Location-Management-Admin-Prototype-2026-10-29.html` | 英文界面 | 全员 | 单文件高保真，覆盖 6 个菜单 + 详情 5 个标签页 |
| 流程图 | `diagrams/flow1..11-*.png` | 英文 | 全员 | 11 张，2x 渲染，飞书可直接嵌入 |
| 演示视频 | `Location-Management-Admin-Demo-2026-10-29.mp4` | 英文字幕 | 全员 | ⚠️ 录制于 v2.7.0，仍演示“改地址”，**不要用于 v3 验收**，待重录 |

**对外分享用哪一个**：给外国同事发 `feature-overview.html`（英文单页），需要展开细节时再给 PRD 和字段映射；中文团队内部用本文件 + 验收标准。

## 附：一页速览（30 秒看懂）

| Dimension | Content |
|---|---|
| **做什么** | 两层模型的内部后台：供应商层（只读镜像）+ OC 层（客户可见记录） |
| **两个列表** | Vendor Location List + OC Location List（含 `⏳ Not live yet` 签收队列） |
| **三类来源** | PUDO Inc（PUDOpoint / Locker）· LMS（packagehub）· **M&MP（自营小店）** |
| **给谁用** | Viewer（只读 + 导出）· Location Manager（+ enable/签收、标记异常、裁决重复、重新签收） |
| **数据怎么来** | 每日同步；新记录过入库校验与重复校验后生成或关联 OC 记录，默认未启用 |
| **核心控制** | OC 层只有两个动作：`enable`（签收）和 `flag an issue`；**不改任何值** |
| **可见性** | `status = active` **且** `is_enable = true`，下游才可见 |
| **状态联动** | 供应商 inactive → 关联 OC 自动 disable + 阻断下游；强制启用报错 |
| **取值规则** | 逐字段合并：**M&MP → LMS → PUDO Inc**，先有值者胜，缺的由下一个补 |
| **来源决定类型**（v3.2.0） | LMS → packagehub · PUDO Inc → PUDOpoint / Locker · M&MP → 自营；筛选器按此级联 |
| **去重** | 独立 Duplicate Check 页；≥75% 疑似；**未裁决阻塞启用**；确认 2:1 后两条记录喂一条 OC 记录 |
| **异常** | 四层粒度（字段 → 子字段 → 7 类 → 严重度）；`blocking` 阻塞签收、`warning` 不阻塞；open → reported → fixed |
| **签收即基线** | 5 项检查 + 核对客户可见值 + 必须勾选；记基线，写 `ENABLE_SIGNED_OFF` |
| **签收后漂移** | 上游改动照常生效；与基线比对显示 drift；**重新签收**或**标记异常**，都不改值 |
| **审计** | 每个动作一条：谁、何时（到秒）、字段、旧值 → 新值 |
| **兜底原则** | 没人处理 → 不启用 / 保持旧值 / 不生成 OC ID，绝不自动接受错值 |
| **交付页面** | Vendor List · OC List · Duplicate Check · Data Issues · Change Log · Sync Status + 详情（5 tab） |
| **验收** | 见 `Location-Management-Admin-验收标准-2026-10-29.md`（13 条快检 + 5 个故事） |
