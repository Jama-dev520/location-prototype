# Location Management Admin Console — PRD Change Summary (v2.0)

> Document version v2.0 ｜ Change date 2026-10-30 ｜ Status: Draft — pending review
> **Live prototype (public)**: https://jama-dev520.github.io/location-prototype/

## 一、一句话总览

本次 PRD 的核心改动，是把控制台从 **单一合并门店列表**（v1.x）重构为 **两层模型**（v2.0）：**供应商层（Vendor Location）+ OC 层（OC Location）**，用两个菜单分别管理，让「供应商原始数据」和「OC 自有、面向下游的门店」彻底解耦。

---

## 二、关键变更摘要（v1.2 → v2.0）

### 1. 单一列表 → 两个菜单（两层模型）
- **Vendor Location List**（供应商视角）：只看供应商的原始门店信息，只读（唯一可改的是 **Sweeper**）。
- **OC Location List**（OC 视角）：基于供应商信息**回填**生成的 OC 自有门店，`enable` 逻辑绑定在这一层，支持编辑店铺信息。

### 2. Sweeper（原 Pickup 字段统一改名）
原 `pickup_provider` / "Pickup" 字段统一改名为 **Sweeper**，语义不变（谁实际做上门揽收），仍是 OC 可编辑、可跨供应商。

### 3. OC Location 自动生成（1 个 vendor → 1 个 OC location ID）
每个供应商 location（非疑似重复）自动生成一个对应的 OC location ID，回填名称/地址/营业时间，默认 `is_enable = false`。疑似重复的供应商 location **不**直接生成 OC ID，进入重复审核队列。

### 4. enable 逻辑迁移到 OC 层
`is_enable` 开关不再挂在供应商门店上，而是挂在 **OC location** 上。OC 层支持编辑店铺信息（地址、开业时间等，运营发现异常即可改），写 `STORE_DETAIL_EDIT` 日志。

### 5. 状态联动：供应商 inactive → OC 自动 disable
供应商门店状态变为 `inactive` 后，其关联的 OC location **自动 disable**，且**不允许同步给下游**；若强制打开会**报错**（锁定，直到供应商恢复 active）。

### 6. 映射关系：原则上 1:1，重复合并为 2:1
OC : Vendor 原则上 1:1（一个供应商 location 对应一个 OC location，即两个店铺）。跨供应商确认「同店」的重复对则 2:1（两个 vendor location 关联同一 OC location）。

---

## 三、对比表（v1.2 vs v2.0）

| 维度 | v1.2（旧） | v2.0（新） |
|---|---|---|
| **菜单结构** | 单一 Location List | Vendor Location List + OC Location List 两个菜单 |
| **数据模型** | 单一 location 表（供应商+OC 混合） | 两层：`vendor_location` + `oc_location` + `vendor_oc_mapping` |
| **揽收字段名** | Pickup / `pickup_provider` | **Sweeper** / `sweeper`（统一改名） |
| **enable 归属** | 挂在合并门店上 | 挂在 OC location 上 |
| **店铺信息编辑** | 供应商字段只读 | OC 层可编辑地址/开业时间等（回填+覆盖） |
| **OC location ID 生成** | 去重后分配 | 每个 vendor 自动生成 1 个 OC location ID（疑似重复除外） |
| **状态联动** | 无强制联动 | 供应商 inactive → OC 自动 disable + 阻断下游 + 强制启用报错 |
| **映射关系** | 未明确 | 原则 1:1；确认重复 2:1 |
| **可见性规则** | 有 OC location ID + active + enabled（三重门控） | OC 层 `status=active AND is_enable=true`（status 由供应商派生） |

---

## 四、PM 待确认项

- [ ] 高级权限（sweeper 编辑 / 店铺信息编辑 / 启用停用 / 去重确认）的授予在 v1 放到控制台外，由 IT/SSO 配置，本期**不做角色管理 UI**。
- [ ] OC 层「店铺信息可编辑」的具体字段范围（地址、开业时间是否含服务标签、坐标等）需最终确认。
- [ ] 供应商 inactive → OC auto-disable 后，若供应商短暂恢复再停用，是否需要「抑制频繁翻转」的防抖机制。
- [ ] 两家供应商是否暴露 `updated_since` 增量接口待确认，否则该供应商回退全量拉取。
- [ ] 确认重复对 2:1 合并后，两个 vendor location 的 sweeper / 状态不一致时，OC 层 `status` 如何派生（取交集还是任一 active）。
