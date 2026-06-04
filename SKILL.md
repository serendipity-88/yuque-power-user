---
name: yuque-power-user
description: >
  语雀文档自动化专家。提供 MCP API 避坑指南、YMD 高级语法（高亮块/多栏/折叠/标签卡片）、
  画板 DSL（flowchart/mindmap/architecturediagram）、CLI 图片上传、HTML 表格高级用法、
  Playwright 补充方案（表格列宽调整）。当用户需要创建/更新语雀文档、插入画板、操作表格、
  上传图片、使用 YMD 扩展语法时触发。
  触发词：语雀、yuque、写文档、创建文档、更新文档、画板、流程图、思维导图、架构图、
  表格合并、表格背景色、表格列宽、上传图片、高亮块、callout、折叠、多栏、YMD、
  skylark、知识库、页面引用、嵌入文档、日历卡片、渐变色、上标、下标、段落对齐
---

# 语雀 Power User

语雀 MCP 工具 + CLI 的高级使用指南。核心价值：避坑 + 高级语法 + Playwright 补充。

## 前置条件（二选一或都有）

- **MCP 工具**（`skylark_*` 系列）：蚂蚁内网环境下 Claude Code 自动连接，无需手动安装
- **CLI 命令**（`yuque`）：`npm i -g @antcli/yuque-ant-cli`，Node.js ≥20。用于图片上传等 MCP 不支持的操作

两者独立，不冲突，互补使用。

> **本 Skill 是纯知识型参考，不包含任何运行时组件。** MCP Server 需在蚂蚁内网环境下自动连接（非蚂蚁内网不可用），CLI 需手动 `npm install`。Skill 安装后只是让 agent 知道"怎么正确使用这些工具"，不会帮你安装工具本身。

## 触发示例

以下是真实用户的典型说法，帮助判断何时触发本 Skill：

- "我写了个方案，帮我发到语雀上，放在 xxx 知识库里" → mcp-api-guide
- "这个语雀文档内容过时了，帮我把第二段改一下" → mcp-api-guide（护栏：读后写）
- "帮我在文档里加个流程图和思维导图" → board-dsl
- "表格要好看一点，表头加底色，有几个单元格要合并，表格太窄了能拉宽么" → html-table-advanced + playwright-workarounds
- "我本地有几张截图要插到语雀文档里" → cli-guide
- "帮我写个周报模板，要有高亮块、折叠块、关键指标标红、加个日历卡片" → ymd-syntax

## 🔴 6 条核心护栏（每次操作前回顾）

1. **读后写**：`skylark_doc_update` 是**全文替换**，不是增量更新。更新前必须先 `skylark_doc_detail` 读取完整内容，修改后整体写回
2. **4空格缩进**：画板 DSL 中 mindmap 和 architecturediagram 必须用 4 空格缩进，2 空格会导致解析失败且无错误提示
3. **Board ≠ Doc**：`skylark_resource_detail` 只能读取 Doc 类型文档中嵌入的 board resource。对 Board 类型文档（独立画板）调用会报 `unsupported doc type: Board`
4. **表格列宽不可控**：API 和 YMD 均无法设置列宽。唯一可行方案是 Playwright 拖动编辑器中的列分隔线
5. **创建 ≠ 覆盖**：同一文档多次调用 `skylark_resource_create` 会追加多个画板，不会覆盖已有的。如需替换，先删除旧的 board 引用再创建新的
6. **有毒标签禁用**：`<cardlink>`、`<mention>`、`<todo>` 通过 MCP API 写入时会被解析器破坏——不仅自身不渲染，还会吞掉后续段落。替代方案：链接用 `[标题](url)`、任务列表用 `- [ ]`、@人名只能编辑器手动操作

## 决策路由

根据用户需求，**按需加载**对应 reference 文件（不要一次全读）：

| 用户需求 | 加载 reference |
|---------|---------------|
| 创建/更新/搜索文档、知识库操作 | [references/mcp-api-guide.md](references/mcp-api-guide.md) |
| 上传图片到语雀文档 | [references/cli-guide.md](references/cli-guide.md) |
| 高亮块、多栏、折叠、标签卡片、日历、Mermaid、任务列表、页面引用、渐变色、上下标、段落对齐/缩进 | [references/ymd-syntax.md](references/ymd-syntax.md) |
| 插入画板（流程图/思维导图/架构图） | [references/board-dsl.md](references/board-dsl.md) |
| 表格合并单元格、背景色、富文本 | [references/html-table-advanced.md](references/html-table-advanced.md) |
| 表格列宽调整、Playwright 自动化 | [references/playwright-workarounds.md](references/playwright-workarounds.md) |

## ⚠️ 能力边界声明

本 Skill 中标注 ❌ 的能力是截至特定日期的验证结果。语雀和 MCP 持续更新，某些限制可能已被解除。

**自验证方法**：
1. 直接尝试（例如在 HTML 表格的 `<td>` 中设置 `width` 属性）
2. 如果不再报错，说明语雀/MCP 已更新，该限制不再适用
3. 更新对应的 reference 文件中的标注
