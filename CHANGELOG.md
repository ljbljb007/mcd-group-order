# 更新日志

## 2026-10-30

### 文档

- **README 重写**：新增中英双语简介（可折叠 English 区）、目录导航、Before/After 对比表、30 秒上手、完整对话示例、适用场景矩阵、常见问题（含「能否分开付钱」「与官方点餐助手是否冲突」）、设计取舍。
- 徽章更新：接入动态 Star 数，参赛状态更新为「已参赛」。

### Skill

- `SKILL.md` 新增「与 `mcdonald-assistant` 的路由分工」说明：按人数 ≥ 2 判定，避免两个麦当劳 Skill 抢触发。

## 2026-10-23

### 首发

- 项目开源并完成参赛报名（Issue [#1120](https://github.com/M-China/mcd-developer-innovation-challenge/issues/1120)，官方已回复「您的作品已成功参赛」）。
- 交付 Skill 主体 `SKILL.md`：五阶段协调流程（立案 → 定门店 → 首版提案 → 异议收敛 → 用券算价 → 确认下单）。
- 交付参考文件 `references/`：`menu-planning.md`（双池配额算法）、`conflict-resolution.md`（冲突矩阵）、`ordering-flow.md`（MCP 调用链路）。
- 交付参赛必需文件：`README.md`、`MCP_INTEGRATION.md`、`CONTEST_DECLARATION.md`、`mcp-config.example.json`、`workbuddy.md`。
- 交付 `examples/sample-session.md` 完整对话样例。
- 补 `LICENSE`（MIT）与 `.gitignore`（拦截真实 Token 与凭证文件）。
