# Location 管理后台 — 产品需求文档（中文版）

**版本 8.0.0 ｜ 2026-11-25 ｜ 状态：待评审**

配套文件：可交互原型（`index.html`）· 判重规则 `Deplicate-check-v3.0-2026-11-23.md` · 数据库字段映射（v5.1.0，尚未按本模型更新）

---

## 1. 这一版改了什么，为什么改

5.x 把工作拆成三个页面：供应商列表、OC 列表、独立的重复检查页。实际工作不是这样。运营的每一个决定都从一条供应商记录开始，也落在同一条记录上，拆成三页意味着回答一个问题要点三次导航。

6.0 把控制台收成**一张列表**。同步状态、白名单开关、判重结论，全部挂在这一行上。

| | v5.1.0 | v6.0.x |
|---|---|---|
| 页面 | 供应商列表 + OC 列表 + 重复检查 | **只有 Vendor Location List** |
| 店铺身份 | 另一个号（`OC-…`），存在独立的店铺记录上 | **就是 vendor location ID 本身**；重复的两条靠标记关联 |
| 变更感知 | OC 页面上的 drift 徽章，需自己推断 | 对照上一次快照显式标记 **New / Changed** |
| 推送 eBay | 值自动流向下游 | **上线即发布**，没有单独的 apply 步骤 |
| 判重 | 独立页面 + 队列 | 选中两行，**Mark as duplicate** |
| 白名单 | 在 OC 页面逐店签收 | **行内 Live 开关**，加批量增删 |
| 筛选 | 三个文本条件 + facet | 新增 **Zipcode / Live / Change / Duplicates** |

判重规则（v3.0）仍然管「相似度怎么算」。本 PRD 管「运营看到什么、能做什么」。两者在身份问题上冲突时，以本文为准：原型就是这一版的契约。

---

## 2. 范围

### 2.1 本期做

- 一张列表承载全部供应商记录，带同步状态、上线状态、重复标记。
- 对照上一次同步快照做变更识别，精确到字段。
- 把新增/变更记录推给 eBay，单条或批量。
- 加入白名单（上线）与移出白名单，单条或批量。
- 把一条记录标成另一条的重复，两条各自保留自己的 ID，用标记关联。
- 列表的筛选、排序、导出。
- 记录详情页：显示哪些字段变了，以及变更历史。

### 2.2 本期不做

- 编辑供应商值。控制台不改地址，只标异常，由供应商改源头。
- 自动判重。相似度可以作提示，结论必须由人给。
- 自动上线。任何记录都不会自己进白名单。
- 按区域/活动做多店批量。批量操作只作用于勾选的行。
- 两个角色（Operations / Viewer）之外的权限体系。

---

## 3. 一页看懂模型

```
供应商同步（每天 23:30，单批次）
        │
        ▼
┌────────────────────────────────────────────────┐
│  vendor_location                               │
│  id            ← 唯一的身份                     │
│  name, address, city, state, zip               │
│  source, sweeper, qr_dropoff, status           │
│  live, blocked                                 │
│  duplicate_of（指向它重复的那条）              │
│  sync_state (new | changed | synced)           │
│  sync_fields, last_synced_at                   │
└────────────────────────────────────────────────┘
```

三句话决定全文所有行为：

1. **vendor location ID 就是身份。** 没有第二个键。一条记录就是一家店；两条记录其实是同一家店时，其中一条带 `duplicate_of` 指向另一条，两条各自保留自己的号。
2. **一家店只有一条 live 记录。** 重复对里的两条，同一时刻只能有一条上线；打开一条就把另一条撤下。
3. **有变更的记录不上线。** 供应商把值改了，记录就离开白名单，等人把它放回去。客户永远拿不到运营没看过的值。

---

## 4. 数据模型

### 4.1 vendor_location

| 字段 | 类型 | 说明 |
|---|---|---|
| `id` | varchar | `LOC-xxxxx`，入库时分配、永不改写。下游用的就是这个键 |
| `name`、`address_line1`、`address_line2`、`city`、`state`、`postal_code` | varchar | 供应商原样，此处不编辑 |
| `source` | enum | `PUDOpoint (PUDO)` / `packagehub (LMS)` / `M&MP` |
| `sweeper` | enum | 实际揽收方：`PUDO Inc` / `LMS` |
| `status` | enum | `active` / `inactive` / `removed`，来自供应商 |
| `qr_dropoff` | boolean | 供应商能力：柜台扫客户二维码完成投递 |
| `live` | boolean | 是否在白名单；上线就是把值送到 eBay 的那个动作 |
| `blocked` | boolean | 本控制台只读，由平台写入 |
| `list_removed_reason` | enum NULL | `UNSIGNED` / `PENDING_CHANGE` / `DUPLICATE_SWITCHED` / `PROVIDER_DISABLED` / `RECORD_VOIDED` |
| `duplicate_of` | varchar NULL | 本记录重复的那条；主记录为 `NULL` |
| `sync_state` | enum | `new` / `changed` / `synced` |
| `sync_fields` | json NULL | 变了哪些字段，供 Changed 徽章与详情页使用 |
| `last_synced_at` | timestamp | eBay 最后一次接受这条记录的时间 |

### 4.2 模型里**没有**什么

**没有 `oc_location` 表。** 8.0.0 把它删了，连带 `OC-{ST2}-{City3}-{Zip5}-{seq6}` 这个号。它存在的理由是「几条来源记录喂一家店时，给这家店一个身份」；一条记录本身就是一家店之后，那个身份只是同一行数据的第二个名字，围绕它的规则（发号、永不改写、合并时废弃）都只是在维护一个重复的键。

需要点明的连带结果：没有发号序列，「OC Location ID」筛选消失，没有独立于记录的店铺详情页，下游直接按 `vendor_location.id` 关联。

### 4.3 不变量

| # | 不变量 | 由什么保证 |
|---|---|---|
| I1 | 重复记录保留自己的号，绝不采用主记录的号 | 判重只写 `duplicate_of` |
| I2 | 一个重复对里最多一条记录处于上线 | 上线时把另一端撤下 |
| I3 | `blocked` 永远不由本控制台写入 | 不存在写路径 |
| I4 | `sync_state != synced` 的记录不上线 | 变更记录在入库时即被移出白名单 |
| I5 | 号一旦签发永不改写 | 发号只追加 |
| I6 | 供应商状态不是 `active` 的记录永远不上线 | 开关禁用、接口拒绝、批量跳过 |

---

## 5. 同步与变更识别

### 5.1 快照

每次同步先把供应商全量落一份快照，再应用。下一次与这份快照比对，而不是与行上残留的旧值比。

```
第 N 次：snapshot_N = 供应商原始数据
         diff(N, N-1) → 每条记录：new | changed(字段) | unchanged
第 N+1 次：snapshot_{N+1} = 供应商原始数据
         diff(N+1, N) → …
```

### 5.2 分类

| 条件 | `sync_state` | 列表表现 |
|---|---|---|
| 上一次快照里没有这条记录 | `new` | ID 下方 `New` 徽章 |
| 存在，且至少一个比对字段不同 | `changed` | `Changed` 徽章；差异在详情页逐字段列出 |
| 存在，且比对字段全部相同 | `synced` | 不显示任何徽章 |

比对字段：`name`、`address_line1`、`address_line2`、`city`、`state`、`postal_code`、`phone`、`opening_hours`、`services`、`sweeper`、`status`。坐标照存照显示，但不单独产生变更信号。

### 5.3 连带后果：变更记录立即掉线

记录被判为 `changed` 或 `new`，且它当前正在为某家店供数时，整店在入库阶段就下线：

```
oc_location.live = false
oc_location.live_source = null
记日志 PENDING_CHANGE_OFFLINE(oc_id, vendor_id)
记日志 WHITELIST_REMOVED(vendor_id, reason = PENDING_CHANGE)
```

号不动。只是这家店在有人处理之前不处于上线状态。这是本控制台唯一一处「故意不贴心」：客户绝不能收到运营没看过的值。

---

## 6. 列表

### 6.1 列

| # | 列 | 内容 |
|---|---|---|
| 1 | 勾选 | 批量操作用 |
| 2 | **Vendor Location ID** | 编号，点击进详情；有变更时下方挂 `New` / `Changed` 徽章 |
| 3 | Name | 供应商名称 |
| 4 | Address | 第一行；第二行（单元号）浅色显示 |
| 5 | City / Province | `Brooklyn, NY` |
| 6 | Zip | 前五位 |
| 7 | Source | `PUDOpoint (PUDO)` 等 |
| 8 | Sweeper | 徽章，或 `missing` |
| 9 | **QR** | 来源支持 QR 投递时显示 `QR` 徽章，不支持时显示 `—` |
| 10 | Status | 供应商状态徽章 |
| 11 | Live | 开关（`Live` / `Not live`）；`Blocked` 只读徽章 |
| 12 | Duplicate | 重复记录的该列标 `duplicate of LOC-xxxxx`，主记录显示 `—` |
| 13 | Issues | 未关闭异常数，可点击 |
| 14 | Updated | 供应商时间戳 |

QR 投递是**供应商的能力**，不是运营写的备注：客户拿到一个码，柜台扫一下完成投递。它和 Sweeper 放在一起，因为两者都是「这家店怎么运作」的事实；值只从上游读，控制台不猜，显示 `—` 就代表这个来源不支持。

> **命名规则**：列表列名与筛选项统一叫 **Vendor Location ID**。任何出现「Location ID」的地方都必须带上前缀。

### 6.2 徽章不占列

没有 Sync 列。与 eBay 一致的记录**不带任何徽章**，只有真的动过的才标记。每一行都在喊的列表，没人会去扫。

### 6.3 排序

按「配对」分组：一家店和它的重复记录上下相邻，主记录在前，组内顺序稳定；不成对的记录按供应商原顺序。

**Zipcode 筛选生效且命中多于一行时**，先按与该组第一条的文本相似度降序，再分组。查邮编本质是在找重复，最像的应该浮在最上面。

### 6.4 空态

筛选组合无命中时明确告知，并提供 Reset，不偷偷回落到全量。

---

## 7. Live 与白名单

### 7.1 开关

**每一行都有开关**，状态直接读记录上的 `live`。

| 行状态 | 渲染 |
|---|---|
| `status` 为 `inactive` 或 `removed` | **禁用态** `Not live` 开关：供应商说这家店没在营业，不能释放给 eBay |
| `blocked` | 红色 `Blocked` 徽章，不可点 |
| `live = true` | 绿色胶囊 `● Live` |
| `live = false` | 灰色胶囊 `○ Not live` |

点胶囊切换上线状态，需要 Operations 角色。开关只写这一行，没有要先发生的发号动作。

### 7.2 一店一源

重复对里的两条不能同时上线。号已经不同，所以这条规则不再由号兜底，而是落在配对关系上。打开一条，另一条在同一次点击里被撤下：

```
live(记录 A) = true
if 配对里的另一条正在上线:
    live(记录 B) = false, live_source(B) = null
    记日志 DUPLICATE_UNLISTED(B.oc_id, reason = DUPLICATE_SWITCHED)
    记日志 WHITELIST_REMOVED(B, reason = DUPLICATE_SWITCHED)
```

`LIVE_SOURCE_CHANGED` 下线了：一条记录一个号之后，同一个号下不可能再换供数方。换源就是上面的「配对操作」，`live_source` 永远指向持有该号的那一条。

关掉当前正在供数的那条，是**整店下线**，不是换条记录。

### 7.3 上线即应用

`sync_state` 不是 `synced` 的记录不能上线（I4）。所以点上线一次做两件事：把当前值写给 eBay，并翻转开关。

```
if sync_state != synced:
    sync_state = synced
    记日志 CHANGE_APPLIED(vendor_id, 原状态, synced)
live = true
```

这是刻意的：没有单独的「确认」步骤。只确认不发布，会留下一条标着干净、eBay 上却是旧值的记录。

### 7.4 批量

勾选任意行后出现批量条：`Add to whitelist & apply`、`Remove from whitelist`、`Mark as duplicate`、`Clear selection`。

批量加白名单按行执行同样的规则：缺号先发号、翻转开关、应用值；不能上线的行跳过并计数，结果提示里写明处理了几条、跳过几条。

---

## 8. 判重

### 8.1 动作

正好选中两行，点 `Mark as duplicate`。弹框并排显示两条记录（ID、名称、地址、城市、邮编），相似度分数**仅供参考**。两个控件决定结果：

- **Keep as the store** —— A / B 单选，默认选中当前在线的那条。
- **没有别的了。** 没有要选、要填、要合并的号：标记 `duplicate_of` 就是全部决定，两条各自保留自己的 ID。

### 8.2 四种结果

| 情形 | 结果 |
|---|---|
| 还没标记过 | 被弃那条写入 `duplicate_of = 保留方.id`，两条的 ID 都不变 |
| 已经标记过 | 重标只刷新方向，ID 永远不变 |
| 被弃那条正在上线 | 它被撤下，因为一对里只能有一条供数 |
| 保留方原本是别人的重复 | 标记转移到新的主记录 |

确认后，被弃那条写 `duplicate_of = 保留方.id`，pair 以 `merged` 状态写入 `duplicate_pair`。**没有任何一个号被改写、合并或废弃**——这个动作写的是关系，不是身份。如果被弃那条正在上线，它会被撤下，因为一对里只能有一条供数。

### 8.3 重复记录不是店

重复是一条**记录**，不是一家店。它保留自己的值、自己的异常、自己的历史。运营愿意的话，它可以成为 live 来源（见 7.2）。但它不能脱离主记录存在。

### 8.4 筛选重复

`Duplicates` 筛选：`Duplicated records` 返回每一对的两端，主记录与从属方一起返回、上下相邻；`No duplicate` 返回其余。筛选读的是关系，不是号。

---

## 9. 值怎么到 eBay

只有一条路：**上线**。记录被加进白名单的那一刻，它的值才写进 eBay，别的时候都不写。

这就是全部的发布模型。它也回答了以前那个按钮在问的问题：店没上线，eBay 没有理由持有它的值；店在上线状态而供应商改了字段，它在入库时就已经被撤下，再上线就是把新值写过去。

两种上线方式，逐行效果相同：

| 动作 | 写入 eBay | 是否上线 |
|---|---|---|
| 点行内 `Live` | 是，该条 | 是 |
| 批量 `Add to whitelist & apply`（先筛选） | 是，每条选中记录 | 是 |

所以操作路径就是：筛选 `Change = New` 或 `Changed` → 勾选 → `Add to whitelist & apply`。不需要别的动作，也不存在事后对账。

早期草稿里有 `Apply to eBay`，更早还有 `Push differences`。两者都是同一个效果的第二条路，而且都允许把记录标成「已同步」、eBay 上却还是旧值。只留一条路，它就不会和自己对不上。

---

## 10. 筛选

全部多选 + 内嵌搜索。同一筛选内取 OR，筛选之间取 AND。已选项以 chip 显示；Reset 清空所有控件（含本版新增的每一项）。

| 筛选 | 取值 | 说明 |
|---|---|---|
| **Vendor Location ID** | 最多 20 个条件，空格或逗号分隔 | 子串匹配，任一命中即可 |
| Name | 最多 20 个条件 | 一个条件是一个短语，词需全中 |
| Province | facet | 计数随其他筛选联动 |
| City | facet | 随 Province 级联 |
| Zipcode | 最多 20 个条件 | 取前五位；**命中后按相似度排序** |
| Source | facet | 三个值 |
| Status | facet | 供应商状态 |
| Live | facet | `Live` / `Not live` / `Blocked by system` |
| Change | facet | `New` / `Changed` / `No change` |
| Duplicates | facet | `Duplicated records` / `No duplicate` |
| Sweeper | facet | `PUDO Inc` / `LMS` |
| Issues | facet | 有未关闭异常 / 无 |

筛选区顺序：`Vendor Location ID · Name · Province · City · Zipcode · Source · Status · Live · Change · Duplicates · Sweeper · Issues`。

---

## 11. 记录详情页

点 Vendor Location ID 进入。五个标签页：Basic Info、Operations、Hours and Services、Issues、Change History。

### 11.1 字段级变更

Basic Info 里，自上一次快照以来变过的字段带 `changed` 徽章，并显示旧值：

```
Address        2918 Avenue I   [changed]
               was 2908 Avenue I
Postal code    11210           [changed]
               was 11210-2214
```

只有真的变了的字段才标记。没变过的详情页看起来就是一个普通详情页。

### 11.2 变更历史

历史页按时间倒序列出该记录的全部条目，包括同步产生的每字段一行：

| 时间 | 类型 | 字段 | 旧值 | 新值 |
|---|---|---|---|---|
| 2026-11-25 02:14 | PROVIDER_SYNC | addr1 | 2908 Avenue I | 2918 Avenue I |
| 2026-11-25 02:14 | PROVIDER_SYNC | postal | 11210-2214 | 11210 |

运营动作也在同一条时间线里：白名单变更、应用值、判重结论。

---

## 12. 状态机

两个互相独立的维度，这就是为什么「一个状态列」是错的。

**同步状态**

```
new ──────────(上线)─────────> synced
 │                                │
 │                     (供应商改了字段)
 │                                ▼
 └──────(供应商改了字段)──────> changed ──(上线)──> synced
```

**上线状态**

```
未上线 ──(开关打开，sync_state = synced)──> 上线
   ▲                                          │
   │                              (供应商改了字段)
   └────────────── live = false ──────────────┘
```

`blocked` 独立于两者之外：它是平台状态，此处只读。被 block 的店无法开启，尝试会明确报错而不是静默失败。

---

## 13. 接口草案

| 方法 | 路径 | 用途 |
|---|---|---|
| GET | `/vendor-locations` | 列表，带筛选、分页、排序 |
| GET | `/vendor-locations/{id}` | 详情，含 `sync_fields` 与旧值 |
| POST | `/vendor-locations/{id}/live` | `{ live: true }`，缺号先发号，再应用、上线 |
| POST | `/vendor-locations/live` | 批量，`{ ids, live }` |
| POST | `/duplicates` | `{ a, b, keep }`，写 `duplicate_of`，必要时把被弃那条撤下 |
| GET | `/duplicates/{id}` | pair 详情 |
| GET | `/oc-locations/{id}` | 店铺详情、live 来源、成员 |
| GET | `/change-log?loc_id=` | 单条记录的历史 |

需要专门设计的错误：给已有号的记录再发号、切换被 block 的店、`sync_state != synced` 时上线、以及标记两条已经各持不同号的记录（这种需要合并，不是建立关联）。

---

## 14. 验收标准

1. 侧栏只有一个业务页面，OC 列表与判重页不可达。
2. 新增或变更的记录在 ID 下方显示徽章；已同步的记录该位置没有东西。
3. 变更记录在入库后不处于上线，白名单记录显示为已移除，原因 `PENDING_CHANGE`。
4. 点带徽章的 ID 进入详情，变动字段带 `changed` 徽章并显示旧值。
5. Change History 每个变动字段一行 `PROVIDER_SYNC`，时间倒序。
6. 对 `sync_state = changed` 的记录点 `Live`，一次动作完成「发布当前值 + 清除徽章 + 上线」。
7. 重复对里的两条不能同时上线；开一条会撤下另一条，并记 `DUPLICATE_UNLISTED`。
8. 被 block 的店渲染为徽章，不是开关。
9. 没有 `Apply to eBay` 按钮：筛选 `Change = New`、勾选、点 `Add to whitelist & apply`，写值、清徽章、上线一次完成。
9b. 每一行都有开关；点一次即上线并把当前值写进 eBay，没有发号这一步。
10. 选中两条无号记录并确认 `Mark as duplicate`，产出一个号，两条都写它，被弃那条写 `duplicate_of`。
11. 标记两条各持不同号的记录，废弃其中一个号，两条都落到存活的号上。
12. `Duplicates` 筛选把一对记录一起返回、相邻、主记录在前。
13. 按 Zipcode 查询返回该邮编全部记录，最像的排最前。
14. Reset 清空文本条件、九个 facet 与邮编条件，行数回到全量。
15. 重复记录可以单独上线，上线后原来的供数方变为未上线。
16. 批量加白名单跳过不能上线的行，并在提示里报告跳过条数。
17. 导出每条供应商记录一行，带 ID、上线状态与重复标记。
18. **表头第二列与第一个筛选标签都写作 `Vendor Location ID`**，不出现无前缀的「Location ID」。
18b. **列表、筛选、详情页、导出、接口里都不出现 `OC-…` 编号**（只有版本历史里还会提到旧的号）。
19. **供应商状态为 `inactive`（或 `removed`）的记录渲染禁用开关**，任何路径都无法上线，也不会被发号。
20. **QR 列就在 Sweeper 旁边**，只有来源支持投递时才显示 `QR`，否则是横杠。

---

## 15. 待定

| # | 问题 | 归属 |
|---|---|---|
| 1 | 变更记录掉线，是即时生效还是等同步窗口结束？ | 运营 |
| 2 | 变更记录上线，要不要先弹窗列出差异让人确认，而不是一次点完？ | 产品 |
| 3 | 跨邮编合并的店，运营不指定时号里该用哪个邮编？ | 产品 / 数据 |
| 4 | 被废弃的号，对仍引用它的下游订单要保留多久的解析能力？ | 下游契约 |
| 5 | 重复标记能否撤销（不经过合并，直接解除）？ | 运营 |
| 6 | 是否需要批量判重（一次处理多对），还是两条一次就够？ | 运营 |

---

## 16. 从 5.x 迁移

| 步骤 | 动作 |
|---|---|
| 1 | 把 `oc_location` 上的 `live`、`blocked` 复制到当时供数的那条记录，然后删除这张表 |
| 2 | 由旧的共享号分组回填 `duplicate_of`：除上线方以外，其余成员都指向保留号的那条 |
| 3 | `vendor_location.id` 一律不动 —— 它现在是唯一的键，下游改按它关联 |
| 4 | 一次性告知下游：`MC_LEGACY_OC_ID(vendor_id, legacy_oc_id)`，让它们能换键而不是靠猜 |
| 5 | 最后一个消费方确认后，下线 `/oc-locations` 接口 |

---

*文档结束。版本 8.0.0，2026-11-25。*
