# Location Management Admin Console — 数据库字段映射文档

> 配套 PRD：`Location-Management-Admin-PRD-2026-10-29`
> 目的：帮助研发/数据团队理解「前端筛选条件 ↔ 后端 API 参数 ↔ 数据库字段」及「各数据表字段」的对应关系。

---

## 一、筛选条件 ↔ 数据库字段映射

> 以下筛选条件均作用于 **P01 位置列表页（`GET /api/locations`）**，多个筛选条件之间按 **AND** 逻辑组合。

| 前端筛选条件（UI 标签） | API 查询参数（建议） | 数据库表 | 数据库字段 | 类型 | 匹配方式 | 说明 |
|---|---|---|---|---|---|---|
| Location ID | `location_id` | `location` | `location_id` | varchar(32) | 精确/前缀 | 字母数字，≤32 字符 |
| Name（名称关键词） | `name` | `location` | `name` | varchar | 模糊（LIKE %…%） | 命中后端索引 |
| Address（地址关键词） | `address_keyword` | `location` | `address_line1` / `address_line2` | varchar | 模糊（LIKE %…%） | 跨 address_line1/2 联合搜索 |
| Provider（供应商） | `provider` | `location` | `source_provider` | enum(PUDO,LMS) | 精确 | 级联 Type 的上级条件 |
| Status（业务状态） | `status` | `location` | `status` | enum(active,inactive,removed) | 精确 | 供应商来源字段，只读 |
| Enable（启用开关） | `is_enable` | `location` | `is_enable` | bool | 精确 | OC 内部开关 |
| Type（类型） | `location_type` | `location` | `location_type` | enum(PUDOpoint,Locker,packagehub) | 精确 | 与 Provider 互斥级联 |
| City（城市） | `city` | `location` | `city` | varchar | 精确 | — |
| State（州/省） | `state` | `location` | `state` | varchar | 精确 | — |
| Postal Code（邮编） | `postal_code` | `location` | `postal_code` | varchar | 精确 | — |
| Services（服务） | `services` | `location` | `services[]` | array | 包含（ANY） | FX/CP/TB（PUDO）、returns/platform_dropoff/retail（LMS） |
| Created from / to（创建时间范围） | `created_from` / `created_to` | `location` | `created_at` | timestamp | 范围（BETWEEN） | 秒级精度，跨度≤1年 |
| Updated from / to（更新时间范围） | `updated_from` / `updated_to` | `location` | `updated_at` | timestamp | 范围（BETWEEN） | 秒级精度，跨度≤1年 |
| 可见性（隐式，导出/统计用） | `visible` | `location` | `status` + `is_enable` | 组合 | `status=active AND is_enable=true` | 见「可见性规则」 |

> **可见性规则（核心）**：上游客户可见的门店 = `status = 'active' AND is_enable = true`。该规则在 customer-facing 查询 API 中强制执行。

---

## 二、数据表 ↔ 数据库字段

### 表 1：`location`（门店统一模型）

> 数据来源：PUDO Inc + LMS 每日同步；`location_type` 供应商独占，`is_enable` OC 独占。

| 字段名 | 类型 | 来源 | 只读？ | 说明 |
|---|---|---|---|---|
| `location_id` | varchar(32) PK | 系统内部 | 是 | 永久稳定，不随 provider_ref 变化 |
| `source_provider` | enum(PUDO,LMS) | 同步 | 是 | 来源供应商 |
| `provider_ref` | varchar | 同步 | 是 | dealerNo（PUDO）/ loc_id（LMS） |
| `store_code` | varchar | 同步 | 是 | 门店编码 |
| `name` | varchar | 同步 | 是 | 门店名称 |
| `location_type` | enum(PUDOpoint,Locker,packagehub) | 同步 | 是 | 供应商独占，互斥 |
| `address_line1` | varchar | 同步 | 是 | 地址行 1 |
| `address_line2` | varchar | 同步 | 是 | 地址行 2（可空） |
| `city` | varchar | 同步 | 是 | 城市 |
| `state` | varchar | 同步 | 是 | 州/省 |
| `postal_code` | varchar | 同步 | 是 | 邮编 |
| `country` | varchar | 同步 | 是 | 国家 |
| `latitude` | decimal | 同步 | 是 | 纬度 |
| `longitude` | decimal | 同步 | 是 | 经度 |
| `phone` | varchar | 同步 | 是 | 电话（可空） |
| `timezone` | varchar | 同步 | 是 | 时区 |
| `status` | enum(active,inactive,removed) | 同步 | 是 | 供应商业务状态 |
| `is_enable` | bool | OC | 否 | OC 内部开关，仅 F06/F07 可写 |
| `services[]` | array | 同步 | 是 | 服务列表 |
| `restrictions[]` | array | 同步 | 是 | 限制列表 |
| `hours_of_operation{}` | json | 同步 | 是 | 营业时间（7 天） |
| `cutoff_time` | time | 同步 | 是 | 截单时间（LMS） |
| `holiday_closures[]` | array | 同步 | 是 | 节假日停业 |
| `created_at` | timestamp | 同步 | 是 | 创建时间 |
| `updated_at` | timestamp | 同步 | 是 | 更新时间 |
| `activated_at` | timestamp | 同步 | 是 | 激活时间 |
| `deactivated_at` | timestamp | 同步 | 是 | 停用时间 |
| `last_synced_at` | timestamp | 系统 | 是 | 上次同步成功时间 |

### 表 2：`change_log`（变更日志）

> Append-only，不可变，不可删除。

| 字段名 | 类型 | 说明 |
|---|---|---|
| `log_id` | bigint PK | 日志主键 |
| `location_id` | varchar FK → location | 关联门店 |
| `occurred_at` | timestamp | 变更发生时间 |
| `operator` | varchar | 操作人，或 `System · PUDO Sync` / `System · LMS Sync` |
| `operation_type` | enum | ENABLE / DISABLE / BATCH_ENABLE / BATCH_DISABLE / PROVIDER_STATUS_CHANGE / PROVIDER_FIELD_UPDATE / PROVIDER_CREATED / PROVIDER_REMOVED |
| `field_name` | varchar | 变更字段名 |
| `old_value` | text | 旧值 |
| `new_value` | text | 新值 |
| `source` | enum(MANUAL,SYNC) | 变更来源：人工 / 同步 |
| `remark` | text | 备注 |

### 表 3：`sync_run`（同步运行记录）

> 每个供应商每天一条。

| 字段名 | 类型 | 说明 |
|---|---|---|
| `run_id` | bigint PK | 运行主键 |
| `provider` | enum(PUDO,LMS) | 供应商 |
| `started_at` | timestamp | 开始时间 |
| `finished_at` | timestamp | 结束时间 |
| `result` | enum(SUCCESS,FAILED) | 结果 |
| `records_pulled` | int | 拉取记录数 |
| `created_count` | int | 新增数 |
| `updated_count` | int | 更新数 |
| `deactivated_count` | int | 停用数 |
| `error_message` | text | 错误信息（失败时） |

### 表 4：`user`（用户）

> 由企业 SSO 提供，v1 无自助角色管理。

| 字段名 | 类型 | 说明 |
|---|---|---|
| `user_id` | varchar PK | 用户 ID |
| `name` | varchar | 姓名 |
| `email` | varchar | 邮箱 |
| `has_enable_permission` | bool | 是否拥有启用/停用高级权限 |

---

## 三、字段所有权速查

| 字段类别 | 拥有方 | 可写路径 | 只读？ |
|---|---|---|---|
| `status` 及所有业务属性（地址/营业时间/服务等） | 供应商 | 每日同步 | 是 |
| `is_enable` | OC 运营团队 | F06 单条 / F07 批量 | 否 |
| `location_id` / `created_at` 等系统字段 | 系统 | — | 是 |

> **关键规则**：供应商同步**永不覆盖** `is_enable`，只更新供应商拥有的字段；被供应商停用的门店保留其 `is_enable` 值。

---

## 四、API ↔ 表 快速对照

| API | 方法 | 关联表 | 说明 |
|---|---|---|---|
| `/api/locations` | GET | `location` | 列表 + 筛选（服务端分页） |
| `/api/locations/{id}` | GET | `location` | 详情 |
| `/api/locations/{id}/changes` | GET | `change_log` | 单店变更历史 |
| `/api/locations/{id}/enable` | PATCH | `location` + `change_log` | 单条启用/停用，写审计 |
| `/api/locations/batch-enable` | POST | `location` + `change_log` | Mode A 批量 |
| `/api/locations/batch-enable/upload` | POST | `location` + `change_log` | Mode B Excel 上传 |
| `/api/locations/batch-enable/template` | GET | — | 下载批量模板 |
| `/api/changes` | GET | `change_log` | 全局变更日志 |
| `/api/sync/status` | GET | `sync_run` | 同步状态 |
| `/api/stats/overview` | GET | `location` + `change_log` | 概览 KPI |
