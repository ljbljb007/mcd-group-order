# 下单执行流程（MCP 调用链路）

本 Skill 用到的 `mcp__mcd-mcp__*` 工具共 11 个。下面按执行顺序列出调用时机、参数要点和踩坑点。

---

## 0. 参数配对铁律

`storeCode` / `beCode` / `orderType` / `beType` 四者必须一致，这是最容易报错的地方：

| 场景 | beType | orderType | beCode | 来源 |
|---|---|---|---|---|
| 到店自取 | 1 | 1 | **不传**（传了报错） | `query-nearby-stores`(beType=1) |
| 得来速 DT | 5 | 1 | 必传 | `query-nearby-stores`(beType=5) |
| 麦乐送 | 2 | 2 | 必传 | `delivery-query-stores`(beType=2) |
| 企业团餐 | 6 | 2 | 必传 | `delivery-query-stores`(beType=6) |

`reservationDate` 格式 `yyyy-MM-dd HH:mm`，**预约才传，不预约不要传**。

---

## 1. 定门店

### 到店：`query-nearby-stores`
- 必填 `beType`（1 或 5）、`searchType`
- `searchType=1` 查收藏（用户没收藏会失败）；`searchType=2` 按位置搜，需 `city` + `keyword`
- 聚餐常用 `searchType=2`：发起人说"陆家嘴附近"→ `city="上海市"`, `keyword="陆家嘴"`
- 返回门店后让用户选；DT 场景记得取 `beCode`

### 外送 / 团餐
1. `delivery-query-addresses()` → 选 `addressId`；没有就 `delivery-create-address`（`city`/`contactName`/`phone`/`address`/`addressDetail` 必填，**全部取用户真实输入，禁止生成**，phone 需 11 位 1 开头）
2. `delivery-query-stores(addressId, beType)` → 拿 `storeCode` + `beCode`

---

## 2. 拿菜单与营养

### `query-meals(storeCode, orderType, beType[, beCode])`
返回在售餐品与价格，取 `productCode`。搭配规则（人均档位）见 `menu-planning.md`。

### `list-nutrition-foods()`
拿热量/蛋白质/钠等。控热量需求时用。**静态常见餐品数据**，与门店在售可能不完全一致，搭配后用 `query-meals` 校验是否供应。

### `query-meal-detail(storeCode, orderType, beType, code)`
查组成与特调。用途：
- 确认某单品能否去辣酱、去生菜、少酱（服务"不吃辣""控热量"）
- 展示规则：`supportModify=true` 标【可特调】，但**不主动展开特调列表**，用户问"查看特调"再展开

**特调传参 key 规则**（高频错误）：
- 选中项：`key = selectedKey`
- 未选中项：若 `unselectedKey` 非空，`key = unselectedKey`，**也必须传入**
- 一个特调组里只要存在 `unselectedKey`，该组所有项都要传

---

## 3. 领券与选券

### `auto-bind-coupons()`
一键领取所有可领券，无入参。用户没领过就先调，不必询问——省下的钱是实打实的。

### `available-coupons()`
查看当前可领券列表。想向用户说明"这次帮你领了什么"时用。

### `query-store-coupons(orderType, beType, storeCode[, beCode])`
查**这家店这一单**真正能用的券。注意与 `query-my-coupons` 的区别：

| 工具 | 语义 |
|---|---|
| `query-my-coupons` | 卡包里有什么券（不做可用性校验，可能这单用不了） |
| `query-store-coupons` | 这家店这单能用哪些券（**下单看这个**） |

### 选券原则
1. **先满足硬约束**：用券不能把忌口单品换掉
2. 阶梯满减优先于单品券
3. 若最省钱的券要求加购（反而超预算），则放弃，选次优
4. 用券后必须重新 `calculate-price`，不要按券面金额自己减

---

## 4. 试算

### `calculate-price(storeCode, orderType, beType, items[, beCode][, couponId/couponCode][, gmServiceCode])`

- `items[]`：`productCode` + `quantity`（≥1）+ 可选 `couponId`/`couponCode`/`modification`/`roundList`
- 团餐场景传 `gmServiceCode`（来自 `query-meal-assistance`）
- **返回金额单位是「分」，展示必须 ÷100**
- 返回 `takeWayList[].code` → 下单时的 `takeWayCode`（仅 orderType=1 需要）

### 与预算比对

| 情况 | 动作 |
|---|---|
| 差额 ≤ 人均 5% | 通过 |
| 超预算 | 三级降级：砍加购/共享小食 → 饮料降档 → 主食降档 |
| 剩余 > 15% | 问发起人要不要加共享小食或升档 |

降级后**重新算价**，最多迭代 3 次。

---

## 5. 下单

### `create-order(storeCode, orderType, beType, items, ...)`
- `addressId`：orderType=2 必填
- `takeWayCode`：orderType=1 必填，来自 `calculate-price` 的 `takeWayList[].code`
- `remark`：≤50 字，**仅 orderType=2 引导填写**（如"无接触配送""放前台"）
- `gmServiceCode`：团餐必填
- `needTableware`：聚餐建议问一句要不要餐具（多人场景容易漏）

**执行前必须复述**：门店 + 全部商品 + 数量 + 实付金额，得到发起人明确肯定。这是真实扣款动作。

下单后：
- 给出支付链接
- 提示「支付完成后回复'支付完成'或'支付失败'，我帮你查最新状态」
- 收到反馈后用 `query-order(orderId)` 核实，**不凭用户口述下结论**

### `cancel-order(orderId, cancelReasonCode)`
原因码：1=改主意了，2=重复下单，3=点错了/点多了/点少了，4=地址/电话填错，5=送达时间选错，-1=其它。多人场景常见是 3（人数报错）或 1（有人临时不吃了）。

---

## 6. 团餐专属补充

若办公室场景走企业团餐（beType=6）：

1. `query-promotions(storeCode, orderType=2, beType=6, beCode)` → 满减/满折规则
   - `promotionType`：31=满额减、33=订单折扣
   - `ruleCategory`：30=满折、40=满减
   - 工具只给规则，**优惠解读与搭配由模型计算**
2. `query-meal-assistance(storeCode, orderType=2, beType=6, beCode)` → 选助餐服务，拿 `gmServiceCode`
3. 后续 `calculate-price` / `create-order` 都要带上 `gmServiceCode`

团餐搭配参考（来自官方规则）：20 元以下小食；20–30 汉堡+小食或汉堡+饮料；30–40 汉堡+薯条/小食+饮料；40–50 汉堡+薯条+小食+饮料；50 元以上丰富组合且小食不重复。

---

## 7. 工具调用全景

```
query-nearby-stores ──┐
delivery-query-addresses ─→ delivery-query-stores ─┤
                                                   ├─→ query-meals ─→ (query-meal-detail)
                                                   │        ↓
                                    list-nutrition-foods（控热量时）
                                                   │        ↓
                          auto-bind-coupons → query-store-coupons
                                                   │        ↓
                                              calculate-price  ←→ 三级降级迭代
                                                   │        ↓
                                    [团餐] query-promotions + query-meal-assistance
                                                   │        ↓
                                               create-order ─→ query-order
```

11 个工具：
`query-nearby-stores`、`delivery-query-addresses`、`delivery-query-stores`、`query-meals`、`query-meal-detail`、`list-nutrition-foods`、`auto-bind-coupons`、`query-store-coupons`、`calculate-price`、`create-order`、`query-order`（团餐追加 `query-promotions`、`query-meal-assistance`）。
