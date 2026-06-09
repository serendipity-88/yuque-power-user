---
name: yuque-power-user
description: >
  语雀文档自动化专家。提供 MCP API 避坑指南、YMD 高级语法（高亮块/多栏/折叠/标签卡片）、
  画板 DSL（flowchart/mindmap/architecturediagram）、CLI 图片上传、HTML 表格高级用法（colgroup列宽/背景色/合并）、
  Playwright 补充方案（全宽展示/拖拽精调）。当用户需要创建/更新语雀文档、插入画板、操作表格、
  上传图片、使用 YMD 扩展语法时触发。
  触发词：语雀、yuque、写文档、创建文档、更新文档、画板、流程图、思维导图、架构图、
  表格合并、表格背景色、表格列宽、colgroup、上传图片、高亮块、callout、折叠、多栏、YMD、
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
- "表格列宽不均匀，内容列太窄" → html-table-advanced（colgroup 列宽比例）+ playwright-workarounds（全宽展示）
- "我本地有几张截图要插到语雀文档里" → cli-guide
- "帮我写个周报模板，要有高亮块、折叠块、关键指标标红、加个日历卡片" → ymd-syntax
- "帮我创建一篇完整的周报，要画板、表格带底色、还要插截图" → end-to-end-example

## 🔴 7 条核心护栏（每次操作前回顾）

1. **读后写**：`skylark_doc_update` 是**全文替换**，不是增量更新。更新前必须先 `skylark_doc_detail` 读取完整内容，修改后整体写回
2. **4空格缩进**：画板 DSL 中 mindmap 和 architecturediagram 必须用 4 空格缩进，2 空格会导致解析失败且无错误提示
3. **Board ≠ Doc**：`skylark_resource_detail` 只能读取 Doc 类型文档中嵌入的 board resource。对 Board 类型文档（独立画板）调用会报 `unsupported doc type: Board`
4a. **colgroup 列宽比例**：HTML 表格可通过 `<colgroup><col width="N" />` 设定列宽比例（建议总和 ≤750px；超 750px 不会被剥离，但标准宽度下容器裁切溢出部分，开全宽展示可正常显示）。format=ymd 读写保留 colgroup，format=md 读回丢失（同 backgroundColor/colspan）
4b. **全宽展示**：让容器不再裁切表格，按 colgroup 比例自然展开。全宽展示与「自适应宽度」**独立**，可直接点击工具栏第一个无名 icon 开启，无需先开自适应宽度。全宽展示状态不反映在 YMD/MD 读回中
5. **创建 ≠ 覆盖**：同一文档多次调用 `skylark_resource_create` 会追加多个画板，不会覆盖已有的。如需替换，先删除旧的 board 引用再创建新的
6. **有毒标签禁用**：`<cardlink>`、`<mention>` 通过 MCP API 写入时不仅自身不渲染，还会**破坏后续所有段落**；`<todo>` 会被静默消除（自身消失但不破坏后续内容）。替代方案：链接用 `[标题](url)` 或 `<page-title-card>`、任务列表用 `- [ ]`、@人名用 `[@人名](url)` 视觉替代（不触发通知）
7. **表格不入容器**：表格（Markdown/HTML）不能嵌入 `:::colorN` 高亮块、`<details>` 折叠块、`<columns>` 多栏——表格内容会丢失，高亮块还会边界溢出吞掉后续段落

## 决策路由

根据用户需求，**按需加载**对应 reference 文件（不要一次全读）：

| 用户需求 | 加载 reference |
|---------|---------------|
| 创建/更新/搜索文档、知识库操作 | [mcp-api-guide.md](mcp-api-guide.md) |
| 上传图片到语雀文档 | [cli-guide.md](cli-guide.md) |
| 高亮块、多栏、折叠、标签卡片、日历、Mermaid、任务列表、页面引用、渐变色、上下标、段落对齐/缩进 | [ymd-syntax.md](ymd-syntax.md) |
| 插入画板（流程图/思维导图/架构图） | [board-dsl.md](board-dsl.md) |
| 表格合并单元格、背景色、列宽比例、富文本 | [html-table-advanced.md](html-table-advanced.md) |
| 表格列宽调整（拖拽精调）、全宽展示、Playwright 自动化 | [playwright-workarounds.md](playwright-workarounds.md) |
| 表格列宽（综合：colgroup + 全宽 + 拖拽） | [html-table-advanced.md](html-table-advanced.md) + [playwright-workarounds.md](playwright-workarounds.md) |
| 完整文档生产流程（画板+表格+图片组合） | [end-to-end-example.md](end-to-end-example.md) |

## ⚠️ 能力边界声明

本 Skill 中标注 ❌ 的能力是截至特定日期的验证结果。语雀和 MCP 持续更新，某些限制可能已被解除。

**自验证方法**：
1. 直接尝试（例如在 HTML 表格的 `<td>` 中设置 `width` 属性）
2. 如果不再报错，说明语雀/MCP 已更新，该限制不再适用
3. 更新对应的 reference 文件中的标注
