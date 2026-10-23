# MCP 集成说明

## 1. 使用的 MCP Server

| 项 | 值 |
|---|---|
| 名称 | `mcd-mcp` |
| 提供方 | 麦当劳中国（M-China）官方 MCP Server |
| 端点 | `https://mcp.mcd.cn` |
| 协议 | `streamablehttp` |
| 鉴权 | `Authorization: Bearer ${MCD_MCP_TOKEN}`（Token 申请方式见 [mcd-mcp-server](https://github.com/M-China/mcd-mcp-server)） |
| 配置示例 | [`mcp-config.example.json`](./mcp-config.example.json)（仅含环境变量占位符，不含真实 Token） |

本项目为纯 Skill 形态（Markdown 编排 + MCP 工具调用），不含本地服务端代码，所有业务数据均通过 MCP 实时获取。

---

## 2. 实际调用的 Tool 清单

共 13 个工具，按流程阶段组织：

| # | Tool | 阶段 | 在本项目中的作用 |
|---|---|---|---|
| 1 | `query-nearby-stores` | 定门店 | 到店自取/得来速场景查可点餐门店，取 `storeCode` 与 `beCode` |
| 2 | `delivery-query-addresses` | 定门店 | 麦乐送/团餐场景取用户配送地址 `addressId` |
| 3 | `delivery-create-address` | 定门店 | 地址缺失时按发起人提供的信息建档 |
| 4 | `delivery-query-stores` | 定门店 | 按地址查可配送门店，取外送场景的 `storeCode` + `beCode` |
| 5 | `query-meals` | 出提案 | 拿门店实时在售菜单与价格 → 生成个人池/共享池提案 |
| 6 | `query-meal-detail` | 满足忌口 | 查餐品组成与特调项，实现「去辣酱 / 少酱 / 换配料」 |
| 7 | `list-nutrition-foods` | 控热量 | 取餐品热量与营养数据，支撑低卡搭配与热量标注 |
| 8 | `available-coupons` | 用券 | 查看当前可领券（向用户说明领了什么） |
| 9 | `auto-bind-coupons` | 用券 | 一键领取所有可领券，聚餐场景下自动省钱 |
| 10 | `query-store-coupons` | 用券 | 查**该门店该订单类型**真实可用券，避免"卡包有券用不了" |
| 11 | `calculate-price` | 试算 | 真实算价（含优惠），驱动预算比对与三级降级迭代 |
| 12 | `create-order` | 下单 | 把协调结果落成一张真实订单，返回支付链接 |
| 13 | `query-order` | 收尾 | 支付后核实订单真实状态，不凭用户口述下结论 |
| 附 | `query-promotions` / `query-meal-assistance` | 团餐 | 企业团餐场景查满减规则与助餐服务（`gmServiceCode`） |

---

## 3. 调用流程

```
[Phase 1 定门店]
  query-nearby-stores  ──→  storeCode / beCode（到店）
  delivery-query-addresses ─→ delivery-query-stores  ──→  addressId / storeCode / beCode（外送）

[Phase 2 出提案]
  query-meals ─────────────→ 在售菜单 + 价格 + productCode
  list-nutrition-foods ───→ 热量数据（控热量需求时）
  query-meal-detail ──────→ 特调项（不吃辣 / 少酱 / 换配料）
        ↓
     双池配额算法 → 首版提案

[Phase 3 异议收敛]  最多 3 轮，纯对话层，不调工具

[Phase 4 用券与试算]
  auto-bind-coupons ──────→ 领取可领券
  query-store-coupons ───→ 本单可用券
  calculate-price ───────→ 真实总价（分）
        ↓
     超预算？ → 三级降级 → 重新 calculate-price（≤3 次）

[Phase 5 确认下单]
  确认卡 → 发起人确认 → create-order ─→ 支付链接
        ↓
  query-order(orderId) ──→ 核实支付状态
```

### 关键参数配对

| 场景 | beType | orderType | beCode | 金额单位 |
|---|---|---|---|---|
| 到店自取 | 1 | 1 | 不传 | 分（÷100 转元） |
| 得来速 | 5 | 1 | 必传 | 分 |
| 麦乐送 | 2 | 2 | 必传 | 分 |
| 企业团餐 | 6 | 2 | 必传 | 分 |

`create-order` 的 `takeWayCode` 必须取自 `calculate-price` 返回的 `takeWayList[].code`（仅到店场景需要）。

---

## 4. 业务价值：为什么必须用真实 MCP

多人聚餐协调器的可信度，来自**每一个数字都是真的**。

| 环节 | 硬编码/估算的问题 | MCP 实时数据带来的价值 |
|---|---|---|
| 菜单 | 门店在售随时变化，抄来的菜单会点不到 | `query-meals` 保证方案里的东西**一定点得到** |
| 价格 | 门店定价、时段活动价差异大 | `calculate-price` 给出真实券后价，预算控制才成立 |
| 优惠券 | 券包里的券不等于这单能用 | `query-store-coupons` 只返回本单可用券，避免"以为省了其实没省" |
| 忌口 | 能否去辣酱只有商品详情知道 | `query-meal-detail` 的特调项让「不吃辣」真正落地 |
| 热量 | 网上抄的热量值对不上具体单品 | `list-nutrition-foods` 提供官方营养数据 |
| 下单 | —— | `create-order` 直接落单，从协调到支付闭环，不是"纸上谈兵的推荐" |

**核心价值闭环**：协调（对话层）→ 试算（真实算价）→ 落单（真实订单）。少了 MCP 任何一环，本 Skill 都退化成"聊天机器人给建议"，而不是"帮你把饭订了"。

### 相比单人点餐的差异化

MCP 提供的是**单会员、单订单**能力。本项目的创新在于在其上叠加多人协调层：

- 双池模型把"一人一份"改造成"个人池 + 共享池"，在同样预算下提升丰富度
- 异议收敛机制把 N×M 次偏好问询压缩成 1 次提案 + ≤3 轮偏差修正
- 三级降级把"超预算报错"变成"给出可执行的取舍路径"
- 硬约束逐条打勾，让忌口这种高风险事项在下单前被显式确认

---

## 5. 约束与降级

| 情况 | 处理 |
|---|---|
| `calculate-price` 金额超预算 | 三级降级并重新算价，最多 3 次；仍超则请发起人调整预算或人数 |
| 用户无配送地址 | `delivery-create-address`，所有字段取自用户真实输入 |
| 查不到门店 | `query-nearby-stores` 从 `searchType=1`（收藏）切 `searchType=2`（位置） |
| 券用不上 | 以 `query-store-coupons` 为准，卡包有券不等于本单可用 |
| 餐品不支持去辣 | 明说并给出不辣替代，不为了凑单让忌口者将就 |
| 营养数据缺失 | 标注「热量未知」，不编造数字 |
| 下单类接口失败 | 原样展示接口返回的失败原因，不改写、不假装成功 |

---

## 6. 合规

- 所有凭证仅通过环境变量占位符 `${MCD_MCP_TOKEN}` 引用，仓库中不含真实 Token、密钥或个人信息。
- 调用 `create-order` 等真实扣款接口前，必须向发起人复述订单关键信息并获得明确确认。
- 热量数据仅供参考，不构成医疗或营养建议；餐品、价格、供应状态以麦当劳官方渠道实时结果为准。
