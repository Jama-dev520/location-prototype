# Location Management Admin Console — Requirement Overview (v4.5.0)

> **Live prototype (public)**: https://jama-dev520.github.io/location-prototype/
> Companion: `Location-Management-Admin-PRD-2026-10-29` (v4.5.0 — 状态机、名单口径、筛选与上线节奏已同步，F08/F21 仍待同步)
> 给外部同事看的英文功能概览：https://jama-dev520.github.io/location-prototype/feature-overview.html
> 演示视频（3 分 25 秒，36 条英文字幕，**录的是 v3.8.0** —— v4.0.0 的重复检查与 v4.1.0 的多源展示都还没演，需要时再重录：https://jama-dev520.github.io/location-prototype/demo/location-admin-demo.mp4
> Purpose: a quick-reference doc answering three questions — ① what each field means; ② what this requirement does and its priority; ③ what the core flows look like.
> Detail view (v2.4.0): clicking a location ID opens a **full page** with tabs — Basic Info / Operations / Hours & Services / Linked Records / Change History.
> **同步节奏（2026-11-19 确认）**：OC 每天**晚上 23:30** 从供应商拉一次店铺信息（单一批次，不分供应商），先写入供应商层，再做入库校验与去重判定，最后才轮到「上线」。
> **字段口径（2026-11-19 与技术确认）**：`is_enable` 是**供应商的店铺状态**（1 活跃 / 0 关闭），`is_active` 是**我们数据库自己的标记**（不是供应商推的），`is_primary` **不可参考**。**下游能不能看到由名单决定**：白名单可见 · 黑名单不可见（且禁止上线）· 不在名单不可见。**签收 = 写白名单**，名单由运营在 OC location 维度自己开关。本文件与**原型（v4.3.1）**已按此口径修正；其余五份文档待同步（见 §2.4 决策 43）。
>
> **v3.0.0 — 供应商之外，自营 M&MP 小店也走同一套。** M&MP = 自营小店（美国叫法），是门店目录里的第三类来源，数据由门店自己维护，异常找门店负责人；规则与供应商门店完全一致。
>
> **v3.0.0 — provider data is the only data.** The OC layer no longer edits anything: it has exactly two verbs, **enable** and **flag an anomaly**. A wrong sweeper or address is flagged back to the provider with a reason (on the vendor record, no separate page). Duplicate checking is its own page and is marked 2:1 in the OC list. Downstream values are merged **per field with LMS first**. M&MP works the same data under the same rules.

---

## 零、上线节奏（2026-11-20 会议结论）

| 期 | 做什么 | 为什么是这个顺序 |
|---|---|---|
| **一期 P1** | **Vendor Location List**（列表、筛选、标记异常、导出 CSV） | 它只处理「刚同步进来的数据」，下面没有任何东西还没建好，所以它不可能错 |
| **二期 P2** | 先 **Duplicate Check**，再 **OC Location List** | 重复判定决定一家店**挂几条记录**。这件事没定之前，OC 的一行就是「它读哪条记录」的猜测 —— 先做 OC 列表等于做两遍 |

**两期都不含、但两期都要用的**：Sync Status（23:30 那次跑成什么样）、Change Log（谁改了什么）、Data Issues（标记异常落库的台账）。这三个是只读的基础设施，不参与上面两个决定。

> 标识标在工具上：侧栏菜单与页面标题都带 `P1` / `P2`，这样截图、评审意见、bug 单都自带范围。它不是权限、也不是开关 —— 原型里每一页都能点，因为原型要展示整套模型，包括晚一点才做的那部分。

> **一期要提前说清的一件事**：P1 的列表里 `OC Location ID` 一列基本是空的（映射还没建）。这一列应该显示 `—` 加提示「二期分配」，而不是 `none` —— `none` 读起来像数据有问题，而在一期它不是问题。

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
| `location_type` | enum | sync | yes | provider-exclusive：PUDO Inc → PUDOpoint/Locker；LMS → packagehub；M&MP → `M&MP`。**筛选器里由 Source 决定可选项**（v3.2.0） |
| `sweeper` | enum(PUDO/LMS) | OC (editable) | no | **Sweeper** — who physically picks up packages (renamed from "Pickup"); default = source_provider; cross-provider allowed |
| `address_line1`/`address_line2` | varchar | sync | yes | address |
| `city`/`state`/`postal_code`/`country` | varchar | sync | yes | city/state/postal/country |
| `latitude`/`longitude` | decimal | sync | yes | coordinates |
| `phone` | varchar | sync | yes | phone (nullable) |
| `timezone` | varchar | sync | yes | timezone |
| **`is_enable`** | tinyint(0/1) | sync | yes | **供应商的店铺状态**（2026-11-19 与技术确认）：`1` = 活跃，`0` = 关闭。这是**营业状态**，不是我们的开关 |
| **`is_active`** | tinyint(0/1) | **DB（我们的标记）** | yes | **数据库记录状态**（不是供应商推的）：`0` = 该行已作废（被替代 / 软删除），只说明"库里还有没有这条"，与门店开不开门无关 |
| **`is_primary`** | tinyint(0/1) | sync | yes | **不具备参考性**（技术确认）：每条记录都是 1，不参与任何判定、不用于挑保留方 |
| ~~`status`~~ | derived view | — | — | 界面上的 `active` / `inactive` / `removed` 是上面两个标记的**组合视图**，不是独立字段：`is_active=0` → removed；`is_enable=0` → inactive；否则 active |
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
| `status` | enum(active/inactive/removed) | derived | yes | 界面用的**组合视图**（不是独立字段）：`is_active=0` → `removed`；`is_enable=0` → `inactive`；否则 `active` |
| `live_source_vendor_id` | varchar(32) FK | OC（签收时选定） | no | **这家店由哪条记录供数**，也是"上线"的唯一开关：有值 = 上线候选，无值 = 未上线。清空它就是主动下线 |
| **`is_enable`**（继承） | tinyint(0/1) | 取自 live source | yes | **供应商的店铺状态，原样继承**，不是 OC 自己改的。`0` 时这家店自动下线（供应商关店），界面显示 `inactive` |
| **`is_active`**（继承） | tinyint(0/1) | 取自 live source | yes | 数据库记录状态。`0` 时记录已作废，OC 不再对外 |
| **`list_type`** | enum(white/black/none) | 名单表 | yes | **名单状态**（三态互斥）：`white` = 下游可见；`black` = 明确禁止上线；`none` = 不在名单。**签收就是写 `white`**，所以它与 `live_source_vendor_id` 同时产生 |
| `baseline_source_vendor_id` | varchar(32) FK | system | yes | **v4.1.0：当前基线是从哪条记录取的。** 2:1 时只看值已经说不清"核实的是谁" |
| `intake_status` | enum(pass/warn/fail/overridden) | intake check | yes | 数据质量准入结果；`fail` 表示根本没生成 OC 门店 |
| `baseline_at` | timestamp | system | yes | **人工核实过的那一版的时间** —— 判断"好数据"的参照物 |
| `propagated_at` | timestamp | system | yes | 最后一次推给下游的时间（缓存已失效）；`null` = 从未推送 |
| `created_at`/`updated_at` | timestamp | system | yes | created/updated |

### 1.4 Mapping / Change Log / Duplicate / Sync / User

| Table | Key fields | Meaning |
|---|---|---|
| `vendor_oc_mapping` | `mapping_id` / `vendor_location_id` / `oc_location_id` / `created_at` | links vendor ↔ OC (principle 1:1; duplicate 2:1)。**v4.1.0：** 一个 OC 可挂多条 active 记录，其中恰好一条是 live source；这个选择存在 `oc_location` 上，不在映射行上 |
| `change_log` | `log_id` / `location_id` / `location_layer` / `occurred_at` / `operator` / `operation_type` / `field_name` / `old_value` / `new_value` / `source` / `remark` | append-only audit trail |
| `duplicate_pair` | `pair_id` / `vendor_location_a` / `vendor_location_b` / **`text_score`** / **`distance_m`** / **`label`** / `diff` / **`status`** / **`keep_vendor_location_id`** / **`merged_into`** / `detected_at` / `resolved_by` / `resolved_at` | **v4.0.0：** 站点层打分（0–100，**v4.2.0 起按相邻字符对重合比例**）+ 坐标距离，输出 `HIGH_SUSPECT` / `MEDIUM_SUSPECT` 标签；处置三种 `merged` / `separate` / `fix` |
| **`mapping_change`** (v4.0.0) | `change_id` / `round` / `source_key` / `changed_field` / `old_oc_location_id` / `new_oc_location_id` / `basis` / `status` / `actor` / `decided_at` | 映射层：**只针对一个来源**重判，`pending` 表示还没写任何东西，等人确认 |
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
| `LIVE_SOURCE_CHANGED` (v4.1.0) | 换供数来源：店保持上线，字段整条替换、基线重记一次，记 old → new 来源 |
| `OC_MAPPING_CHANGED` (v4.0.0) | 映射变更：某条 source_key 在人工确认后被指到另一个 OC |
| `UNSIGNED_OFF` (2026-11-19) | 撤签：运营把店从白名单撤下（业务动作） |
| `WHITELIST_ADDED` (2026-11-19) | 名单条目写入白名单：谁写的、哪条 vendor location、由哪次签收/换源触发 |
| `WHITELIST_REMOVED` (2026-11-19) | 名单条目移出白名单：必须带 `removed_reason`（UNSIGNED / SOURCE_SWITCHED / BLOCKED / PROVIDER_DISABLED） |
| `LOCATION_BLOCKED` (2026-11-19) | 系统拉黑：名单转为 `black`，下游立即不可见。控制台只读，解铃要找系统方 |
| `SOURCE_CLOSED` (2026-11-19) | **系统自动下线**：供应商关店（`is_enable=0`）或记录作废（`is_active=0`），系统把该记录移出白名单并清空 live source，门店回到 Not live |

### 1.6 Key semantic distinctions

| Concept | Meaning | Key difference |
|---|---|---|
| `vendor_location` vs `oc_location` | provider raw record vs OC-owned entity | two layers; OC layer is downstream-facing |
| `sweeper` | who physically picks up packages | renamed from "Pickup"/`pickup_provider`; OC-editable |
| **`is_enable`** | 供应商的店铺状态：1 活跃 / 0 关闭 | **营业状态**。OC 不能改它，只能跟着它下线；"这家店开不开门"由供应商说 |
| **`is_active`** | 数据库记录状态：1 有效 / 0 作废 | **数据是否还在**。不代表营业：一条 `is_active=0` 的记录连供应商都不再维护了 |
| **`is_primary`** | 每条记录都是 1（技术确认） | **不可参考**。任何"挑哪条保留 / 谁优先"的规则都不许用它，改用创建时间或完整性 |
| **OC 的"上线"** | 有没有 `live_source_vendor_id` | 签收 = 选一条记录 + 写白名单，两件事一次完成；撤签 = 两件一起撤销。**所以 live 与"在白名单里"是同一个事实的两种说法** |
| **名单三态** | `white` / `black` / `none`，互斥 | 只有白名单可见；黑名单是「明确不许上线」，不是「暂时关了」。签收 = 写白名单，撤签 = 回到 `none` |
| `removed` vs `inactive` | **`is_active=0`**（记录作废 / 供应商下架） vs **`is_enable=0`**（门店关门） | 两件事：一条还在库里但关了门，与一条已经不在库里了。日志与筛选必须分开 |
| `vendor_location_id` vs `oc_location_id` | internal vendor ID vs OC-owned ID | OC ID gates downstream sync |
| **baseline vs 当前值** (v2.6.0) | 人工核实过的版本 vs 库里的值 | 上游改了、但运营没点头时，两者**不相等**，OC 用 baseline 对外 |
| **held vs rejected** (v2.6.0) | 已记录但未写入（等决定） vs 已明确拒绝并锁字段 | held 可以随后接受；rejected 之后同字段不再自动应用 |
| **标记 vs 改值** (v3.0.0) | 报告问题 vs 修改数据 | OC 只能标记，值只由供应商给；标记不改值、不下线 |
| **四个 review tab** (v2.6.0) | duplicates / data quality / sync changes / sign-off | 决策对象分别是「对 / 整条 / 字段 / 记录」，不能合成一个列表 |

**Visibility rule (core，2026-11-19 二次修订)**: 一家门店对下游可见，当且仅当 ——

1. 它的 **vendor location 在白名单里**（`list_type = white`）；黑名单与「不在名单里」都不可见；
2. 这条记录 **`is_active = 1`**（我们的库里没把它作废）；
3. 这条记录 **`is_enable = 1`**（供应商说这家店还在营业）。

**第 2、3 条不是读的时候临时判断，而是由系统主动维护名单**：任一条掉到 0，系统立刻把这条记录移出白名单（`removed_reason = PROVIDER_DISABLED` / `RECORD_VOIDED`），清空 `live_source_vendor_id`，门店回到 **Not live**。所以「在白名单里」就等于「两条都成立」，不会出现"名单里有它、供应商却已经关门"这种自相矛盾的状态。

> **签收 = 进白名单，两者不是两道门。** 运营在 OC location 上点「签收」，落到数据上是把**被选中的那条 vendor location** 写进白名单（同时写 `live_source_vendor_id`）；取消签收 = 把它移出名单。所以「签收了但白名单没开」这种状态**不存在** —— 有 `live_source_vendor_id` 就意味着在白名单里。

**名单是三态互斥的**，一个 vendor location 只可能处于其中之一：

| 名单状态 | 下游 | 怎么产生 | 控制台 |
|---|---|---|---|
| **白名单**（`white`） | **可见** | 运营签收 | 签收按钮 |
| **黑名单**（`black`） | 不可见，且明确禁止上线 | **开发层面系统控制**（不是运营操作） | **只读**：显示 `blocked` 与原因，运营解不开，要找系统方 |
| **不在名单里**（`none`） | 不可见（默认状态） | 新记录进来时、或撤签之后 | 撤签按钮回到这里 |

三态互斥是有意的：如果一家店「既在白名单又在黑名单」，谁都说不清它该不该显示，下游只能靠实现顺序决定 —— 那是把业务问题留给代码。

---

### 1.7 「移出白名单」与「撤签」不是同一层的东西

两个词经常被混用，但一个是**数据层的一次写操作**，一个是**运营的一个意图**。分开定义，日志才能回答「这家店为什么不在名单里」。

| | 撤签（un-sign） | 移出白名单（whitelist removal） |
|---|---|---|
| **是什么** | 运营的意图：这家店我不再让它上线 | 名单表上的一次写操作：某条 vendor location 从 `white` 变成 `none` |
| **谁触发** | 只有运营 | 四种来源：**撤签** · **换源**（旧源被替换） · **系统拉黑** · **供应商关店 / 记录作废**（系统自动） |
| **数据变化** | 清空 `live_source_vendor_id` + 名单条目删除 | 删名单条目；**换源与系统下线同时清 `live_source_vendor_id`**（换源随即写新的，系统下线则门店回到 Not live） |
| **日志** | `UNSIGNED_OFF` + 一条 `WHITELIST_REMOVED(UNSIGNED)` | `WHITELIST_REMOVED`，带 `removed_reason` 说明是谁移的；**不记 `UNSIGNED_OFF`** |
| **下游** | 立即不可见 | 立即不可见 |

**为什么名单表要带 `removed_reason`。** 只删一行的话，三个月后没人说得清这家店当初是「运营主动下线」「换了个源」还是「被系统拉黑」——三种情况的处理方式完全不同（第一种要问运营，第二种要去查新源，第三种要找开发）。所以移出时不删行，而是打标记：

| `removed_reason` | 谁做的 | 典型问题 |
|---|---|---|
| `UNSIGNED` | 运营撤签 | 「这家为什么下线了」→ 问运营 |
| `SOURCE_SWITCHED` | 运营换源 | 「旧记录为什么不在名单了」→ 因为换成了另一条 |
| `BLOCKED` | 系统 | 「为什么可见开关打不开」→ 被系统拉黑，找开发 |
| `PROVIDER_DISABLED` | 供应商（系统执行） | 「为什么突然看不见了」→ 供应商 `is_enable=0`，店关门了 |
| `RECORD_VOIDED` | 我们（系统执行） | 「为什么这家也不见了」→ 这条记录的 `is_active=0`，库里已作废 |

**日志记在哪个 ID 上，也要分清。** 名单是 **vendor location** 维度的，运营的动作是 **OC location** 维度的，所以：

- `UNSIGNED_OFF` / `ENABLE_SIGNED_OFF` / `LIVE_SOURCE_CHANGED` 记在 **OC location**（`location_layer = OC`）；
- `WHITELIST_ADDED` / `WHITELIST_REMOVED` 记在 **vendor location**（`location_layer = VENDOR`），并在 remark 里带上 OC 号。

查一家店的历史时，这两类要按当时的映射合并显示 —— 否则运营在 OC 详情页只会看到"撤签了"，看不到"哪条记录的名单条目被移走"。

**三个动作分别落几条日志**（一次名单变动必然有记录，否则查不了账）：

| 动作 | 日志 | 说明 |
|---|---|---|
| 签收 | `ENABLE_SIGNED_OFF` + `WHITELIST_ADDED(white)` | 业务动作 + 名单写入 |
| 换源 | `LIVE_SOURCE_CHANGED` + `WHITELIST_REMOVED(SOURCE_SWITCHED)` + `WHITELIST_ADDED(white)` | 三条：换了谁、旧条目为什么走、新条目进来了 |
| 撤签 | `UNSIGNED_OFF` + `WHITELIST_REMOVED(UNSIGNED)` | 两条：意图 + 结果 |
| 系统拉黑 | `LOCATION_BLOCKED` + `WHITELIST_REMOVED(BLOCKED)` | 两条，都由系统写 |
| **供应商关店 / 记录作废** | `SOURCE_CLOSED` + `WHITELIST_REMOVED(PROVIDER_DISABLED / RECORD_VOIDED)` | 两条，都由系统写，**没有人参与** |

> **为什么换源要三条而不是一条。** 只记 `LIVE_SOURCE_CHANGED` 的话，旧记录的名单条目什么时候没的、为什么没的，日志里查不到 —— 而这正是"这条记录为什么从客户那儿消失了"的答案。三条日志各自回答一个问题，合并成一条就只剩"换过源"，查不到旧条目的去向。

> **供应商关店，门店自己离名单（v4.3.1 修正）。** `is_enable → 0` 或 `is_active → 0` 时，系统**立刻**把这条记录移出白名单、清空 live source，门店回到 **Not live** —— 不需要任何人记得去关。之前写的是"名单不动、只是不可见"，那等于门店既在白名单里、客户又看不到，两个事实从来没被对齐过；一旦供应商悄悄重开，店会在没人核实的情况下自己回到客户面前。
>
> **重开不会自动回来。** 关着的那段时间，店可能搬了、营业时间也改了，所以重开必须**重新签收**（走 5 项检查 + 记基线）。代价要说清楚：装修、盘点这类短期关店也会掉线，重开要人点一次 —— 这是有意选的，因为"没核实过的数据不上线"是这套系统唯一的硬规则。

---

### 1.8 三个 ID 各自是什么：谁当键，谁是对外身份

这是最容易混的地方，先说结论：**去重比较的是 vendor 记录，收敛出来的是 OC；对外可见的开关落在 vendor 记录上。** 三个东西各有职责。

| ID | 标识什么 | 谁产生 | 稳定性 | 用在哪 |
|---|---|---|---|---|
| **`vendor_location_id`** | 供应商的一条记录（一个 source_key） | 同步（各供应商自己的编号） | 稳定，同一家供应商不会复用 | 去重的**比较单位** · 名单条目的 key · `source_key` 的组成 |
| **`oc_location_id`** | 一家店（物理门店实体） | 我们：生成 / 合并 / 拆分 | **合并时不变；拆分时新建** | live 的**决策单位** · 下游看到的门店 · 详情页入口 |
| **名单条目**（`vendor_location_id` + `list_type`） | 下游能不能看到这条记录 | 运营签收 / 撤签，系统拉黑 | 随记录走 | 下游可见性的**开关** |

**所以你那句理解，对一半、差一半：**

- ✅ **对的一半**：`oc_location_id` 是去重后的**实体身份** —— 几条 vendor 记录判定为同一家店，就收敛到同一个 OC，之后的 live 决策（哪家店上线、由哪条记录供数）都是在这个 OC 上做的。
- ⚠️ **差的一半**：它**不是去重的比对键**。真正被比较的是两条 vendor 记录的地址、名称、邮编、坐标；OC ID 只是"这些记录属于同一家店"这个结论的**容器**。把 OC ID 当 key 的话，会绕回"先有 OC 才能判重"的循环。
- ⚠️ **另一半**：对外 live 的**执行**也不在 OC 上 —— 名单条目挂在 **vendor location** 上（上一节定的）。OC 上没有"可见"字段，可见性落在具体那条记录。

> **一句话**：OC ID 是"这家店是谁"，vendor location ID 是"这条数据从哪来"，名单是"让不让客户看见"。判重看中间那层的数据，做决定在 OC 上，开关落在记录上。

> **一个提醒**：OC ID **不是永久稳定的对外键** —— 拆分（映射层判定某条记录身份变了）会**新建一个 OC**，原来的 OC 留给另一条记录。所以下游如果拿 OC ID 做缓存键，拆分时必须失效；这条要与技术确认下游实际用的是哪个 ID（见 §2.4 决策 45）。

---

## 二、需求总结 + 优先级

### 2.1 背景与目的：运营每天要做的四件事

这套后台不是为了「管理数据」，是为了让运营**在几秒内**做完四件具体的活。从 v3.0.0 起，这四件事里已经没有「改数据」——数据归供应商，OC 只判断能不能给客户看，以及把错的地方标回去。

![整体全貌](https://workbuddy-space-static.codebuddy.work/image/M9nBRbD9eQJBNFivHPpu8l.png)

*一张图看全：供应商 → 供应商层 → 两道闸门 → OC 层 → 下游；右侧是三个决策队列，底部是贯穿全链路的变更记录。J1–J4 标记的四件事，就是运营每天真正要做的动作。*

| # | 运营要做什么 | 做完是什么样 | 在哪儿做 |
|---|---|---|---|
| **J1** | **查哪些门店是上线的** | 问「现在有哪些店在客户那儿可见」，界面上直接得到答案：按 `Live` 筛一下（= 在白名单、该源 `is_active=1` 且 `is_enable=1`），看到条数，把这一批导出来。不用提 SQL、不用找研发、不用导全量再自己筛 | P02 · `Live` 筛选 + 导出 CSV |
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
| F02 | Vendor List | Filter & search | P0 | **v4.4.0：每个下拉都是多选 + 内嵌搜索**（输入 `broo` 找 Brooklyn）。**v4.5.0：Location ID / OC Location ID / Name 各收最多 20 个条件**（空格或逗号分隔），每个条件子串匹配；语义仍是同筛选内 OR、筛选之间 AND |
| F03 | OC List | Paginated OC list | P0 | OC 门店列表；**v4.1.0：一条供应商记录一行**，同一家店两条记录就两行，各显示自己的值，列表列头叫 **Live** |
| F04 | OC List | Filter & search | P0 | 同一套 facet 组件；`Live` 也是多选（`Live + Blocked` 一次查完），一家店只算一次。三个文本字段同样收 20 个条件 |
| F05 | Export | Export current view | P0 | both lists, ≤10k rows |
| F06 | Detail | Detail (vendor/OC) | P0 | tabs; OC detail shows editable fields |
| F07 | Change History | Per-location log | P0 | full timeline |
| F08 | Live | One live source per store | P0 | **v4.1.0：开关是「哪条记录供数」**，一个 OC 只有一个；开备用行 = 带换源的签收；关 live 行 = 整店下线，不自动切到备用 |
| F09 | Enable | Batch enable/disable | P0 | Mode A inline + Mode B Excel |
| F10 | Sweeper | Sweeper sync + flag | P0 | sweeper comes with the sync, read-only; a wrong one is flagged (`DATA_ISSUE_RAISED`) |
| F11 | Data issue | Flag an anomaly on a vendor field | P0 | field + reason, no value change; open issues block enable |
| F12 | OC Generation | Auto-generate OC | P0 | new vendor (non-duplicate) → one OC (disabled) |
| F13 | Status Propagation | Vendor inactive → OC disable | P0 | auto-disable + block downstream |
| F14 | Duplicate | Station check + mapping layer | P0 | **v4.0.0：** 站点层打分 + 坐标证据 → `HIGH_SUSPECT` / `MEDIUM_SUSPECT`；处置三种（合并停用 / 同址两家 / 地址待纠正）；映射层一次只判一个来源。**v4.2.0：** 打分改成按相邻字符对重合比例给分 |
| F15 | Change History | Global log | P1 | cross-location search |
| F16 | Data Sync | Sync status | P1 | per-provider run + failure alerts |
| F17 | Overview | Dashboard | P2 | KPIs |
| F18 | QR Dropoff | QR-code dropoff marker | P0 | LMS-only flag; shown on both lists + detail, filterable, exported |
| F19 | Review Center | Intake data-quality gate | P0 | 新门店地址/坐标/营业时间不合格 → **不生成 OC 门店**，进 Data Quality |
| F20 | Drift | Baseline drift + re-sign | P0 | 已上线门店被上游改动 → 下游用新值，台账显示漂移，可重新签收或标记异常 |
| F21 | Enable | Enable sign-off | P0 | 前置 5 项检查（其中之一是**该记录 `is_enable=1`**）+ 必须勾选 + 记基线；缺时间/重复未决/有未关问题都不能启用。**v4.1.0：签收的是「一条记录」**，换源时弹框顶部说明已上线在哪条，按钮变 `Switch live source & sign off` |
| F22 | Data Sync | Breaker & rollback | P1 | 单次 run 变更过多 → 降级干跑；按 run 一键回滚（7 天） |
| F23 | *(retired)* | Field lock release | — | v3.0.0 删除：没有手改，就没有需要解锁的锁 |
| F24 | *(retired)* | Field-level revert | — | v3.0.0 删除：值只能由供应商改，回退等于改数据 |
| F25 | Merge | One record supplies the store | P0 | **整条记录**由一个来源供数：先看 `live_source_vendor_id`（**v4.1.0**），没有才按 M&MP → LMS → PUDO Inc（active 优先）；赢的那条供全部字段，<b>空的也算空的</b>，不跨记录借值 |
| F31 | OC List | One line per source record | P0 | **v4.1.0：** 2:1 的店一行一条记录，live 在前，备用行 `↳` 起头；勾选框、异常徽章、关联供应商只在第一行，一次决定只对一家店 |
| F32 | Live Source | Which record feeds the store | P0 | **v4.1.0：** `live_source_vendor_id` 存指针而不是算排名；供应商列表标 live / standby；详情页 `Live source` 挨着**供应商的 `is_enable`**；导出带 `Live source` 列 |
| F26 | Data Issues | One ledger for all anomalies | P0 | 供应商问题 + 签收后漂移 + 入库失败合成一页，替代 Review Center |

### 2.4 Key business decisions

| # | Decision | Conclusion |
|---|---|---|
| 1 | Architecture | Two-layer: Vendor Location (provider) + OC Location (OC-owned) |
| 2 | Menu | Two menus: "Vendor Location List" + "OC Location List" |
| 3 | OC generation | New vendor (non-duplicate) → auto-generate one OC location, **无 `live_source_vendor_id`**（即未上线，不是靠 OC 自己的 `is_enable`） |
| 4 | Mapping | Principle 1:1; confirmed duplicate 2:1 |
| 5 | Sweeper | Renamed from "Pickup"; OC-editable; cross-provider |
| 6 | OC store edit | OC can edit address, opening hours, and flagged fields |
| 7 | Status propagation | Vendor inactive → OC auto-disable + block; force-enable errors |
| 8 | Duplicate | 按 **`HIGH_SUSPECT` / `MEDIUM_SUSPECT`** 两个标签判断（不是 75% 阈值）；未裁决前不生成 OC |
| 9 | Visibility | OC 可见 iff **有 live source，且该源 `is_active=1` 且 `is_enable=1`**（2026-11-19 口径：`is_enable` 是供应商营业状态，不是我们的开关） |
| 10 | Roles | Viewer (view/export) vs Location Manager (+sweeper/edit/enable/duplicate) |
| 11 | QR dropoff | LMS-only capability flag (`qr_dropoff`); shown as marker on both lists + detail; OC backfilled from linked vendor; read-only |
| 12 | Provider data (v2.6.0) | **提案，不是事实**：未上线门店直接写入；已上线门店按字段档位判定，非低风险字段一律挂起，OC 保持签收基线值 |
| 13 | Baseline (v2.6.0) | 「好数据」的参照物是**人工核实过的那一版**，不是"上次同步写进去的值"。签收/接受变更/人工编辑三处产生 |
| 14 | No review center (v3.0.0) | 只保留一个 **Data Issues 台账**（供应商问题 / 漂移 / 入库失败）；重复裁决独立成页。不再有多个 review tab |
| 15 | Enable (v2.6.0) | 启用 = 签收（核对 + 勾选 + 记基线 + **指定 live source**），关闭 = 一键（**清空 live source**）。批量启用逐条过检查并回报跳过项 |
| 16 | No OC editing (v3.0.0) | OC 不再改任何值。数据错 → **标记异常**（字段 + 原因），推供应商修；OC 只保留 enable 与标记两个动作 |
| 17 | Fail-safe (v2.6.0) | 队列无人处理的结果永远是"不生成 / 不启用 / 保持旧值"，系统绝不替人决定接受上游值 |
| 18 | Sweeper (v3.0.0) | Sweeper 随同步进来，OC 只读。跨供应商揽收仍允许（值本身来自供应商），但 OC 不能改，只能标记 |
| 19 | Duplicate (v3.0.0) | 独立 **Duplicate Check** 页；同一家 → 2:1 并在 OC 列表标出；不是同一家 → 1:1。未裁决的重复对阻塞启用 |
| 20 | Merge priority (v3.0.0 → v3.13.0 改为整条记录) | 来源优先级 **M&MP（自营）→ LMS → PUDO Inc**。取值单位由「字段」改成「记录」，见决策 37 |
| 21 | Drift (v3.0.0) | 签收后供应商改了值：下游直接用新值，台账显示「与基线不同」，运营可**重新签收**或**标记异常**。基线是参照物，不是冻结器 |
| 48 | 先做 P1，再做 P2 (2026-11-20 会议) | 一期只做 **Vendor Location List**（含筛选、标记、导出）；二期做 **Duplicate Check → OC Location List**。理由是依赖而不是工作量：重复判定决定一家店挂几条记录，OC 列表在它之前做不准。Sync Status / Change Log / Data Issues 是两期共用的只读基础设施，不单独排期。标识打在侧栏与页面标题上（`P1` / `P2`），截图与 bug 单自带范围 |
| 49 | 三个字段各收 20 个条件 (v4.5.0) | 跨字段那个搜索框**去掉**了：它一次读六列，等于在筛选项旁边偷偷用另一套规则，同一行数据可能被它捞出来又被 facet 排掉。现在只有 **Location ID / OC Location ID / Name** 三个字段负责「找」，每个支持**最多 20 个条件**（空格或逗号分隔），每个条件是子串匹配。**ID 是清单**（`LOC-1, LOC-2` = 任意一条，粘贴工单里的 ID 才有用）；**名字是短语**（`postal annex` 是一家店，逗号才是第二家 —— 否则会捞出所有 "Postal *"）。条件数量印在标签旁，超过 20 变橙写明，不静默截断 |
| 47 | 筛选升级为 facet (v4.4.0) | 单选下拉撑不住运营真实会问的问题（「纽约或佛罗里达的店」）。每个下拉改成**多选 + 内嵌搜索**，按钮上写清选了什么（`NY +1`）；语义不变：**同一筛选内 OR、筛选之间 AND**。两条约束：① 省份变了之后不存在的城市**被摘掉并在工具栏说明**，不能静默保留（否则空列表看起来像数据没了）；② 徽章、预设、列表读同一份状态，数字不允许各算各的 |
| 46 | 关店自动离名单 (v4.3.1) | **`is_enable=0`（供应商关店）或 `is_active=0`（记录作废）会自动把该记录移出白名单**，并清空 live source，门店回到 Not live —— 没有人需要记得去关。之前「关店时名单不动」的写法作废：那会让门店既在白名单里、客户又看不到，供应商一重开，店就在没人核实的情况下自己回到客户面前。代价是重开必须重新签收（装修、盘点这类短期关店也会掉线），这是有意选的：没核实过就不上线。 |
| 45 | 下游用哪个 ID 查 (2026-11-19，待技术确认) | 控制台内部：判重比 vendor 记录、decision 落在 OC、可见性落在名单（vendor location）。但**下游实际拿哪个 ID 做查询/缓存键**需要确认：若用 `oc_location_id`，拆分后新建 OC 时下游缓存必须失效；若用 `source_key`（vendor 侧），OC 只是内部管理实体。这决定了「拆分」对外是"换了一个门店"还是"同一条记录换了归属" |
| 44 | 名单三态；白名单归运营，黑名单归系统 (2026-11-19) | 一个 vendor location 的名单状态只有三种：**白名单**（下游可见）· **黑名单**（下游不可见，且明确禁止上线）· **不在名单里**（默认，不可见）。**签收 = 写白名单**，不是另一道门；撤签 = 回到「不在名单」。名单由**运营在 OC location 维度自己开关**，不走别的系统。三态互斥是硬约束：同时命中黑白名单时，下游只能靠实现顺序决定显示与否，那是把业务问题留给代码。**名单变动必须独立记日志**：`WHITELIST_ADDED` 与 `WHITELIST_REMOVED`（带 `removed_reason`）各自一条，撤签再额外记 `UNSIGNED_OFF`。**黑名单由开发层面系统控制，控制台不提供拉黑按钮** —— 运营只有签收与撤签两个动作。但控制台必须**显示** `blocked`（列表第三态、详情徽章、可筛选），否则运营看到一家「怎么都上不了线」的店会以为是数据问题，反复签收。 |
| 43 | 三个标记各管一件事 (2026-11-19 与技术确认) | **`is_enable` = 供应商的店铺状态**（1 活跃 / 0 关闭），**`is_active` = 我们数据库自己的标记**（0 表示这行作废，不是供应商推的），**`is_primary` 不可参考**（每条都是 1）。所以：供应商关店 → 我们跟着下线；记录作废 → 不再对外；而「我们主动下线」只能靠**取消 live source** 表达 —— 三种下线原因在日志里必须分开记，否则出事查不出是谁关的。界面上的 `active / inactive / removed` 是前两个标记的**组合视图**，不是独立字段。 |
| 42 | 打分按重合的字符对算 (v4.2.0) | 「全等或 0」把最该看见的情况判成了 0：同一栋楼里换了个单元（`2918 Avenue I` vs `2918 Avenue I Ste 2`），在那个占 40 分的字段上一分不得，只能靠坐标通道捞回来。现在每个字段**按比例**给分：两个值切成**相邻字符对**、排序（词的先后不影响结果），重合比例就是得分（Dice 系数）。同一对从 50 分变成 89 分，真正对不上的仍接近 0；对比表每行直接印这个比例，评审看得见「为什么是 89」而不只是一个勾。 |
| 41 | 新记录只是候选，不是来源 (v4.1.0) | 同步进来的 vendor location **默认不 live**：新建 OC 只有供应商的 `is_enable`，没有 `live_source`；把一条记录合并进已上线的店，指针不动（哪怕它排名更高）；两条记录没人选过源 = 这家店不 live。排名只回答「选它之后客户会读到什么」，它本身不是「选」。 |
| 40 | 一家店多条记录，只有一个供数 (v4.1.0) | 2:1 的店以前只打一行合并值：排名高的那条供全部字段，另一条**看不见** —— 于是没人知道两家在喂一家店，更没法挑。现在**一条记录一行**，各显示各自的值，而且**只有一行能是 live**。开关的含义跟着变：从「这家店开没开」变成「这条店由谁供数」。关掉 live 行 = 门店下线（备用记录从没被签收过，没有可回退的对象）；打开备用行 = **带换源的签收**，字段整条替换、基线重记。列名与筛选统一叫 **Live / Not live** |
| 39 | 重复检查按 v2.0 规则重建 (v4.0.0) | 从「加权相似度 ≥75%」改成两层：**站点层**打分（地址 40 · 名称 10 · 邮编 25 · 州 15 · 城市 10，邮编不同直接 0；v4.2.0 起按相邻字符对重合比例给分）+ **坐标证据通道**（≤30 m 同点 / 30–100 m 灰区 / >100 m 不同），输出 `HIGH_SUSPECT` / `MEDIUM_SUSPECT` 两个标签，不再是百分比；**映射层**一次只判一个来源（source_key），变更先提案、等人确认，两边同一次都变了进人工。处置从两种变三种：合并停用一条（记 `merged_into`）/ 同址两家各自保留 / 地址待纠正。**没有一步是自动的** |
| 38 | 配对会被重读 (v3.14.0) | 2:1 是「这两条记录**在那一刻**是同一家店」的判断，但以前没人再读过它：分数是入池时存下的，决定是终态。现在分数**按现值算**，判定时**留一份快照**（分数 + 两条身份）；之后每次同步只要改了某条的身份（地址/城市/州/邮编），就重算它所在的配对：<br>· 仍 ≥75 → 保持 2:1，无需动作<br>· 掉到 75 以下 → **拆分**：**变的那条**退出去，另给一个 OC（默认不启用、要签收），另一条留着原 OC 和它签收时的身份<br>· 这条不在配对里 → 就是一次普通更新，值与漂移照旧<br>「变的那条走」而不是「优先级低的那条走」—— 因为失效的是它的主张。**注意**：OC 不因此改 ID；新 OC 建好后会立刻跟其他记录再判一次重，免得它一出生就是半个重复 |
| 37 | 整条记录供数 (v3.13.0) | 2:1 的店不再「逐字段拼」：**挑一条记录，它供全部字段**。以前 LMS 没填电话，就拿 PUDO 的补上 —— 界面看着完整，但「LMS 缺电话，谁去催」这个问题被系统自己回答成了「不用催，已经有值了」，于是 LMS 永远不会被要求补齐。<br>现在没填就是没填：OC 那一行显示 `—`，来源标 LMS，缺口留在能被看见、也就能被标出去的地方。选赢家仍然先看 **active**：LMS 下架，是**整店**交给 PUDO，而不是交一半。代价要说清：已上线的店可能出现 `—`，这是故意的 —— 宁可露缺口，也不要一家看起来完整、实际没人负责的店 |
| 36 | 省市筛选 (v3.12.0) | 两个列表都加 **Province** / **City** 两个筛选，每个选项带实时条数。城市跟着州走：选 NY 只剩纽约那三个 —— 不做收窄的城市下拉就是一堵只会变长的名字墙。**切换州时会先清掉不属于新城范围的市**，否则表里会显示 0 行而下拉写着「全部」。供应商列表的城市列补上州（`Brooklyn, NY`）：能筛的东西得能在行上看见。CSV 表头统一叫 `Province`，背后仍是 `state` 列 |
| 35 | OC 只数挡路的 (v3.11.0) | OC 行的 Issues 列原来复制供应商那边的异常总数（提醒级也算），2:1 的店还要把两条记录加起来 —— 那是**供应商列表的问题**，而且几乎从来不是让人改变决定的那个数。现在只带 **blocking**，按后果命名；旁边三个 `drift` / `2:1` / `intake` 才是 OC 独有的。颜色也跟着语义走：**红=挡路，橙=要看一眼**（原来漂移是红的、阻塞是橙的，正好反了）。筛选从「有没有异常」变成三个理由：`Blocking data issue` / `Changed since sign-off` / `Duplicate 2:1`；`Intake overridden` 只留徽章 —— 数据里没有这种记录，一个永远筛出 0 行的选项比没有更糟 |
| 34 | 一个 Source，不是两个 (v3.10.0) | 列表和筛选都只留一个 **Source**：`PUDOpoint (PUDO)`、`packagehub (LMS)`、`M&MP`（M&MP 不重复写两遍）。Type 列、Type 下拉级联、`type ≠ source` 徽章与筛选项一起下线 —— 不可能的组合不再是「被标出来」，而是拼不出来。`Locker` 仍是合法 PUDO 类型但本版不暴露（加回来只是 SOURCES 里一行）。**M&MP 的揽收方固定是 LMS**：店是我们的，揽收不是 —— 数据里没有 M&MP 这个揽收方，筛选里也没有这个选项 |
| 33 | 供应商改完再拉一次 (v3.9.0) | 供应商列表加 **Refresh from providers**，补上闭环缺的那一脚：**标异常 → 导出 → 供应商改 → 我们拉一次**，不用等夜间同步。拉取只写供应商这次推来的值（沿用 v3.4.0：没有接受/拒绝），逐字段拿「标记那一刻的值」重判，**只有变合法**才自动关闭；不动 enable、不动签收、不动基线（所以 Viewer 不能点 —— 写值是操作员动作）。工具栏下留一行 **上次拉取 + run 号 + 关闭/更新/待人工 三个数**，空拉就明说「没有新数据」 |
| 31 | 类型带来源 (v3.8.0) | 供应商列表原来的 Provider 列并进 Type，写成 `packagehub (LMS)` —— 一句话说清「是什么」和「谁的数」。**Source 筛选保留**：它驱动类型级联（选 LMS 只剩 packagehub）和合并优先级（2:1 时 LMS 优先），删了筛选就断了两条规则。2:1 门店显示 `(LMS / PUDO)`，顺序就是合并优先级 |
| 30 | 营业不足 3 小时 (v3.7.0) | 一条「明显不对」但语法完全合法的数据：某天只开两小时。它不需要人先标 —— 系统按 `hoursOfOp` 逐天累加，**不足 3 小时**就进必查项，并在 Hours 标签页点名是哪一天、几点到几点、合计多久。3 小时是展示常量，不是字段；要按市场区分就变成配置，绝不落库成数据 |
| 32 | 异常按性质分组 + 说明 (v3.8.0) | 台账按**性质**分两组：**关键字段缺失**（该有的没给，`MISSING`，默认阻塞）与**数据不准确**（值有但是错的）。两组要的责任方不同、关闭规则也不同，混在一起会让人以为「都等供应商补一下」。Issues **筛选标签**旁一个 **?**（v3.10.1 从列头移过来），点开讲清：什么算异常、什么阻塞、什么会自己关闭 |
| 29 | 三个必查项 (v3.6.0) | 运营每天真正在追的只有三个缺口：**地址不全 / 没邮编 / 没营业时间**。它们放在台账最上面按未关闭异常计数，点一下只看这一类，再点取消。归类**靠字段**（addr1/addr2/city/state、postal、hoursText），所以走 F11 标记的任何一条会自动落格，不需要运营再选一个「类别」，也不存在计数和列表对不上的可能 |
| 28 | 中英双语 (v3.5.0) | 顶部右上角 `EN / 中文` 换的是**界面**：导航、筛选、表头、页面说明、弹框、详情标签页、演示脚本、提示、异常类型名、版本说明，连 `<html lang>` 和浏览器标题一起换。**数据不翻**：门店名、地址、供应商、ID、字段名、事件码、时间戳一律保持供应商推来的原样 —— 翻了就等于造出第二个真相来源。选择存在 `localStorage`，也可以用 `?lang=zh` / `?lang=en` 按链接锁定，方便把英文版直接发给外国同事 |
| 27 | 上线能力保留 (v3.4.0) | 控制台只放弃「数据是否应用」这一层，**不放弃「门店是否上线」**。两个决定分属两处：值是否给下游——不由控制台决定，永远给；门店是否对客户可见——我们选谁供数来决定。注意：控制台没有 `is_enable` 的写权限，那个字段是供应商的营业状态（2026-11-19）由运营 `enable` 决定，走签收（5 项检查 + 勾选 + 记基线）。如果连签收也去掉，入库通过即自动上线，就没有人确认客户会开到的那个地址了 |
| 26 | 不决定是否应用 (v3.4.0) | 供应商推来的值**永远直接给下游**，控制台里没有「接受 / 拒绝 / 重新签收」。所以「签收后漂移」只是**信息**：告诉你自签收以来哪些字段变了；发现变错了就标记异常。同理，issue 也不该由人点「已修复」——只有同步跑完、值确实变了且合格，才由系统关闭。控制台的出口是**导出**：把异常清单导成 CSV 发给供应商当工单，而不是在这里做流程 |
| 25 | 修复后自动关闭 (v3.3.0) | 供应商下一次同步带来新值时，系统自动比对「标记时的值」：**变了且合格** → 自动关闭（`closed_by = AUTO_SYNC`，记 run id 与验证值）；**变了但仍不合格** → 保持 open 并写明原因；**没变** → 保持 open，记为「未变化」。只有**机器可判定**的类型能自动关闭（缺失 / 格式 / 不一致 / 越界）；`INCORRECT` / `STALE` / `DUPLICATE` 永远回到人——同步能带来新值，但证明不了它是对的 |
| 24 | Source 决定 Type (v3.2.0 → v3.10.0 合并) | 一个来源只可能产生它自己的类型：**LMS → packagehub；PUDO Inc → PUDOpoint；M&MP → 自营门店**。v3.10.0 起两者直接合成一个 **Source** 字段：`packagehub (LMS)`、`PUDOpoint (PUDO)`、`M&MP` —— 列表和筛选都只有这一个值，所以「不存在的组合」不是被拦下来，而是**根本拼不出来**。原来的 Type 下拉、级联、`type ≠ source` 徽章与筛选项一并下线 |
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

### 3.4 Duplicate Check — 两层，各问一个问题（v4.0.0）

**站点层**：这两条记录是不是同一个物理站点？

![Duplicate Check 2.0 — 站点层](https://workbuddy-space-static.codebuddy.work/image/rGWDRa0VC1YNMYnMZuvv4I.png)

**映射层**：这个 source_key 现在对应哪个 OC？

![Duplicate Check 2.0 — 映射层](https://workbuddy-space-static.codebuddy.work/image/3VT8csckDQBI9Bs3zQSB76.png)

对应 PRD §5.21：<br/>
· **站点层**：这两条记录是不是同一个物理站点？打分**按比例** —— 地址 40 · 名称 10 · 邮编 25 · 州 15 · 城市 10，每个字段给的是两个值**相邻字符对**的重合比例（先排序，词的先后不影响结果）；邮编是门，不同直接 0。坐标不折算成分数，而是**并排的证据通道**：≤30 m 同点位 / 30–100 m 灰区 / >100 m 不同。输出是标签：`HIGH_SUSPECT`（≥80）/ `MEDIUM_SUSPECT`（50–79）；<br/>
· **三种处置**（都要人点）：同一站点 → 停用一条并记 `merged_into`，留下的那条保住 OC；同址两家 → 都保留、各自一个 OC、都标「同址」；地址有误 → 挂起等下一次同步送来正确地址；<br/>
· **映射层**：某个 `source_key` 现在对应哪个 OC？**一次只判一个来源，不按配对连带**。身份字段变了就重判，变更会**先提案**（附打分与距离依据）等人确认；两边在同一次 run 都变了 → 进人工；<br/>
· **没有任何一步是自动的**：不自动合并、不自动停用、不自动阻断。

### 3.4a 一家店多条记录，只有一个供数（v4.1.0）

对应 PRD §5.22：<br/>
· **展示**：2:1 的店在 OC 列表里**一条记录一行**，live 在前，备用行以 `↳` 起头，各显示自己的名称/地址/来源/揽收方/状态/更新时间；勾选框、异常徽章、关联供应商只在第一行；<br/>
· **live 只有一个**：签收会把**选中的那条记录**写进白名单并记下 `live_source_vendor_id`；该记录还必须 `is_active=1`、`is_enable=1`（供应商没关店）。打开备用行 → 走签收（5 项检查针对**那条记录**），弹框顶部说明这家店现在在哪条上，按钮是 `Switch live source & sign off`；确认后字段整条替换、基线重记、记 `LIVE_SOURCE_CHANGED`；<br/>
· **关掉 live 行 = 整店下线**：不会自动切到备用记录 —— 备用记录从没被签收过，等于拿未核实的数据顶上，这是这套系统唯一不肯做的事；<br/>
· **排序让位给选择**：`live_source_vendor_id` 优先于 M&MP → LMS → PUDO 的排名。否则一条新记录合并进来，会把一家已签收的店悄悄换个数据源；排名只在「这家店还从没选过源」时兜底。

### 3.5 Sweeper 随同步进来（v3.0.0）

sweeper 由供应商随门店一起提供，和其它字段一样只读。发现不对（没给、给了错的揽收方）→ 在供应商记录上**标记异常**，写明原因，推供应商改；下次同步把新值带进来。原「Sweeper 编辑」流程（flow3）已下线。

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

### 3.10 Location 状态机（v4.3.1）

![Location 状态机](https://workbuddy-space-static.codebuddy.work/image/alYzxdfuemEOghH61ckKpN.png)

两行、一根轴：**门店能是什么状态**，**一条记录被允许做什么**。它们只在一个地方相遇 —— 签收，因为那是唯一同时移动两行的动作。

- **门店**（状态就是名单状态）：`No record`（入库失败 / 重复未决）→ `Not live`（**不在名单里**；两条记录没人选也落在这里）→ `Live`（**在白名单**，恰好一个供数源，基线已记）→ `Blocked`（**在黑名单**，由平台写，运营既不能签收也不能解除）→ `Retired`（被合并）。撤签把店送回 `Not live`；`Offline` 不再是状态 —— 关店就是撤签；供应商关门（`is_enable=0`）也落到 `Not live`，只是这一步由系统自己完成，不需要人点。
- **记录**：`Candidate`（同步刚送来的，永远不是来源；待判重复和入库失败也停在这里 —— 此时**没有 OC**）→ `Standby`（挂在一家 live 店上，但**不在名单里**，名单是按记录写的，只有被选中的那条才有条目）或 `Live source`（这家店读它，一店一个），出口是 `Withdrawn`（`merged_into`）与 `Removed`（供应商下架）。
- **四条不变量画在图上**：① 同步本身进不了任何状态；② **待判重复不是状态**（记录停在 Candidate、没有 OC）；③ **Live 只维持到记录还开着为止** —— 供应商关门（`is_enable=0`）或记录作废（`is_active=0`）时系统自己把这条记录移出白名单、清空 live source，门店回到 `Not live`；重开也不会自动回来，要重新签收；④ 下线一键、上线一检，黑名单两边都不由我们动。

对应 PRD §5.23；图为 `diagrams/flow13-location-state.png`。

## 四、交付物清单（v4.4.0）

| 交付物 | 文件 | 语言 | 读者 | 说明 |
|---|---|---|---|---|
| 需求总览（本文） | `Location-Management-Admin-需求总览-2026-10-29.md` | 中文 | 全员 | 字段字典、需求与优先级、流程图、交付物清单 |
| 产品需求文档 | `Location-Management-Admin-PRD-2026-10-29.md` | 英文 | 研发 / QA | F01–F32、页面规格、验收标准（§10） |
| 数据库字段映射 | `Location-Management-Admin-数据库字段映射-2026-10-29.md` | 英文 | 后端 / 数据 | 12 张表、筛选↔API↔字段、来源→类型约束、`live_source_vendor_id` 与 `mapping_change` |
| 变更摘要 | `Location-Management-Admin-PRD-变更摘要-2026-10-29.md` | 中文 | 全员 | 每一版改了什么、为什么改 |
| 验收标准（通俗版） | `Location-Management-Admin-验收标准-2026-10-29.md` | 中文 | 业务 / 测试 | 13 条快检、分页面清单、5 个故事、不该发生的事 |
| **英文功能概览** | `feature-overview-en.html` / `Location-Management-Admin-Feature-Overview-EN.md` | 英文 | **外部同事** | 一页讲清做什么、为什么这样设计；可直接转发或打印 |
| 交互原型 | `Location-Management-Admin-Prototype-2026-10-29.html` | 中英双语（右上角切换） | 全员 | 单文件高保真，覆盖 6 个菜单 + 详情 5 个标签页 |
| 流程图 | `diagrams/flow1..13-*.png` | 英文 | 全员 | 13 张，2x 渲染，飞书可直接嵌入。v4 新增：`flow4` 站点层、`flow12` 映射层（重复检查 2.0），`flow13` Location 状态机 |
| 演示视频 | `Location-Management-Admin-Demo-2026-10-29.mp4` | 英文字幕 | 全员 | 3 分 25 秒、36 条字幕，按 v3.8.0 重录；十段：两层 → 详情 → 标记 → 列表与来源/类型 → 签收 → 重复 → 自动关闭 → 漂移 → 只读角色 → 中英切换 |

**对外分享用哪一个**：给外国同事发 `feature-overview.html`（英文单页），需要展开细节时再给 PRD 和字段映射；中文团队内部用本文件 + 验收标准。

## 附：一页速览（30 秒看懂）

| Dimension | Content |
|---|---|
| **做什么** | 两层模型的内部后台：供应商层（只读镜像）+ OC 层（客户可见记录） |
| **两个列表** | Vendor Location List + OC Location List（含 `⏳ Not live yet` 签收队列） |
| **三类来源** | PUDO Inc（PUDOpoint / Locker）· LMS（packagehub）· **M&MP（自营小店）** |
| **给谁用** | Viewer（只读 + 导出）· Location Manager（+ enable/签收、标记异常、裁决重复、重新签收） |
| **数据怎么来** | **每晚 23:30** 从供应商拉一次；新记录过入库校验与去重（`HIGH_SUSPECT` / `MEDIUM_SUSPECT`）后生成或关联 OC 记录，默认不上线 |
| **筛选**（v4.4.0 / v4.5.0） | 每个下拉是**多选 + 内嵌搜索**（输入 `broo` 找 Brooklyn）；**Location ID / OC Location ID / Name 各收最多 20 个条件**（空格或逗号分隔，子串匹配）；同筛选内 OR、筛选之间 AND；被省份变化挤掉的城市会明说 |
| **上线节奏**（2026-11-20） | **P1** Vendor Location List · **P2** Duplicate Check → OC Location List（先定「一家店几条记录」，OC 一行才不是猜的） |
| **核心控制** | OC 层只有两个动作：`enable`（签收）和 `flag an issue`；**不改任何值** |
| **可见性** | 三个条件全过才可见：**名单在白名单**（= 已签收）· 记录 `is_active=1` · 供应商 `is_enable=1` |
| **状态联动** | 供应商 inactive → 关联 OC 自动 disable + 阻断下游；强制启用报错 |
| **取值规则** | **整条记录**合并：先看 `live_source`，没有才按 **M&MP → LMS → PUDO Inc**（active 优先），赢的那条供全部字段，**它空着的字段也照空**，不拿另一家补 |
| **一家店多条记录**（v4.1.0） | 2:1 的店**一条记录一行**：各自的值都看得见，**只有一个 live**；开备用行 = 带换源的签收，关 live 行 = 整店下线（不自动顶替） |
| **Live 与 enable**（v4.1.0 / v4.2.0 口径修正） | 列表列头、筛选、行内标签统一叫 **Live / Not live**，被系统拉黑的店显示 **`blocked`**（只读）；`is_enable` 是**供应商的营业状态**，OC 只读；**签收 = 进白名单 = 下游可见**，撤签 = 出名单 |
| **一个来源**（v3.10.0，原「来源决定类型」v3.2.0） | `packagehub (LMS)` · `PUDOpoint (PUDO)` · `M&MP` —— 类型是值、归属是括号，列表一列、筛选一项，没有需要同步的第二个下拉 |
| **去重**（v4.0.0 / v4.2.0） | 独立 Duplicate Check 页，两层：站点层按相邻字符对重合比例打分 + 坐标证据 → `HIGH_SUSPECT` / `MEDIUM_SUSPECT`；映射层一次只判一个来源；处置三种；**未裁决阻塞启用** |
| **异常** | 四层粒度（字段 → 子字段 → 7 类 → 严重度）；`blocking` 阻塞签收、`warning` 不阻塞；open → reported → fixed |
| **OC 列表数什么**（v3.11.0） | 只数**阻塞级**（`n blocking`），加上三个 OC 自己的状态：`n drift`（签收后变化）、`2:1`、`intake`。**不再显示供应商的异常总数** —— 那是供应商列表的问题；提醒级不进这个决定。筛选跟着变成三个理由：阻塞 / 签收后变化 / 重复 2:1 |
| **签收即基线** | 5 项检查 + 核对客户可见值 + 必须勾选；记基线，写 `ENABLE_SIGNED_OFF` |
| **签收后漂移** | 上游改动照常生效；与基线比对显示 drift；**重新签收**或**标记异常**，都不改值 |
| **修复后自动关闭**（v3.3.0） | 同步带来「变了且合格」的值 → issue 自动 fixed；变了但还不合格 → 保持 open 并记下尝试值 |
| **QR code dropoff**（v3.8.0 恢复） | 供应商列表加 `QR` 列：LMS 且支持扫码投递的显示徽章，其余显示 `—`；详情页 Basic Info 也有这一项。只会出现在 LMS（PUDO 不暴露这个能力） |
| **异常清单可导出**（v3.4.0） | Data Issues 顶部 `Export open issues (CSV)`：字段·子字段·类型·严重度·原因，直接贴进给供应商的工单；OC 列表导出带「阻塞级异常数 / 其余未关闭数 / 签收后变化数」三列（v3.11.0 拆开：单元格里的 `3 blocking 2 drift` 进表格没法算）。**v3.7.0 起每行还带门店本身**：店名、地址、城市、州、邮编、来源、类型 —— 供应商拿到 CSV 就知道是哪家店，不用再回查单号 |
| **审计** | 每个动作一条：谁、何时（到秒）、字段、旧值 → 新值 |
| **兜底原则** | 没人处理 → 不启用 / 保持旧值 / 不生成 OC ID，绝不自动接受错值 |
| **交付页面** | Vendor List · OC List · Duplicate Check · Data Issues · Change Log · Sync Status + 详情（5 tab） |
| **验收** | 见 `Location-Management-Admin-验收标准-2026-10-29.md`（13 条快检 + 5 个故事） |
