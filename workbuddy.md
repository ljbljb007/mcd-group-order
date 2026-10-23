# workbuddy.md

使用腾讯 WorkBuddy 开发本项目的对话上下文记录，用于核验 WorkBuddy 联动活动奖励条件。

---

## 一、项目信息

| 项 | 内容 |
|---|---|
| 项目名称 | mcd-group-order · 麦当劳聚餐点餐协调器 |
| 开发工具 | 腾讯 WorkBuddy（Agent 模式） |
| MCP 连接器 | `mcd-mcp`（麦当劳中国官方 MCP Server，`https://mcp.mcd.cn`） |
| 参赛活动 | 麦当劳程序员创意开发大赛（2026） |

---

## 二、需求定义

**用户原始需求（原话）**：

> 帮我做一个 麦当劳聚餐点餐协调器的 skill，这个 skill 是面向办公室、宿舍、朋友聚会的。
> 用户只需对 AI 说："6个人，预算150元，有人不吃辣，有人想控制热量，帮我们点一单。"
> AI 就会自动负责收集每个人的偏好、搭配菜单、使用优惠券、计算总价，并在确认后统一下单。
> 价值在于解决"多人点麦当劳，最麻烦的不是选什么，而是协调"。

**需求拆解**：

1. 面向三类场景：办公室工作餐 / 宿舍夜宵 / 朋友聚会
2. 一句话立案即可启动，不要把参数问成表格
3. 全流程覆盖：收集偏好 → 搭配菜单 → 用券 → 算价 → 统一下单
4. 差异化必须落在「协调」上，而不是「推荐菜单」

---

## 三、开发过程

### 阶段 1：摸清 MCP 能力边界

在 WorkBuddy 中通过工具检索，把 `mcd-mcp` 的 35 个工具 schema 全部拉取核对，明确哪些能力可用于多人场景：

- 门店：`query-nearby-stores` / `delivery-query-stores`
- 菜单：`query-meals` / `query-meal-detail`
- 营养：`list-nutrition-foods`
- 券：`available-coupons` / `auto-bind-coupons` / `query-store-coupons`
- 价格：`calculate-price`
- 订单：`create-order` / `query-order` / `cancel-order`

同时确认了关键约束：`beType`/`orderType`/`beCode` 配对规则、金额单位为「分」、`takeWayCode` 只能来自算价结果、特调 `selectedKey`/`unselectedKey` 传参规则。

### 阶段 2：确定差异化设计

判断：如果只做「按预算推荐菜单」，与单人推荐器没有区别，也体现不出 MCP 的价值。因此把设计重心放在协调机制上：

- **提案先行 + 异议收敛**：不做 N 轮偏好问询，先给方案再收偏差，默认同意制
- **双池模型**：个人池（主食+饮料按人头）+ 共享池（小食按人数折算），这是聚餐特有的成本结构
- **忌口分级**：L0 硬禁忌 / L1 强约束 / L2 软偏好，不同级别不同处理强度
- **三级降级**：超预算时给可执行的取舍路径，而不是报错
- **确认卡**：把"协调结果"固化成一张可读可转发的卡片

### 阶段 3：编写 Skill 与配套文档

在 WorkBuddy 中直接产出：

- `SKILL.md`：frontmatter + 六条铁律 + 五阶段流程 + 冲突消解速查 + 确认卡模板
- `references/menu-planning.md`：双池配额算法、人均预算档位映射、热量控制清单
- `references/conflict-resolution.md`：冲突矩阵、异议话术模板、收敛失败兜底
- `references/ordering-flow.md`：MCP 调用链路、参数配对表、踩坑点

### 阶段 4：按官方规则补齐参赛文件

依据活动仓库 `README.md` 与 `activityGuidelines.md` 的要求补齐：

- `README.md`（项目介绍、安装方法、使用示例、目标用户）
- `CONTEST_DECLARATION.md`（官方原文，未做任何改动）
- `MCP_INTEGRATION.md`（MCP Server、Tool 清单、调用流程、业务价值）
- `mcp-config.example.json`（脱敏配置，仅 `${MCD_MCP_TOKEN}` 占位符）
- `examples/sample-session.md`（完整对话样例）
- `workbuddy.md`（本文件）

---

## 四、开发过程中的关键决策记录

| 决策点 | 选择 | 理由 |
|---|---|---|
| 偏好收集方式 | 提案先行，而非逐个询问 | 逐个问在多人场景必然退化，且答案互相冲突 |
| 小食配置 | 共享池按人数折算 | 聚餐实际消耗低于人均一份；共享池也是预算缓冲层 |
| 预算超标处理 | 三级降级并告知砍了什么 | 报错等于把问题丢回给用户，等于没解决协调 |
| 冲突回应方式 | 永远给代价与抵消方案 | 让发起人做决策，AI 只负责算账 |
| 收敛轮次 | 上限 3 轮 | 协调器的价值是缩短决策，不是无限讨论 |
| 价格来源 | 只用 `calculate-price` | 自行加总必然与真实券后价不符 |

---

## 五、WorkBuddy 在开发中承担的工作

1. **MCP 工具能力探测**：批量检索并读取 35 个工具 schema，形成准确的调用约束
2. **Skill 架构设计**：主文件 + 按需加载的 references 分层，避免上下文膨胀
3. **文档生成**：SKILL.md、README.md、MCP_INTEGRATION.md 等全部在 WorkBuddy 内产出与迭代
4. **规则对齐**：读取官方活动规则，逐项核对必需文件与命名要求
5. **样例构造**：编写完整对话样例，验证流程自洽

---

## 六、声明

- 本项目由参赛者使用腾讯 WorkBuddy 开发，为参赛作品，非麦当劳官方产品。
- 文件中不含真实 Token、密钥、账号凭证或他人个人信息，MCP 配置仅使用环境变量占位符。
- 餐品信息、价格及供应状态以麦当劳官方渠道实时结果为准；热量数据仅供参考，不构成营养或医疗建议。
