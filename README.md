# yuque-power-user

[Claude Code Skill](https://docs.anthropic.com/en/docs/claude-code)，让你的 AI agent 像人一样全面、精准地使用语雀，致力于让写文档写材料写方案写周报写 PRD……等等一切事务 0 人工。

## 解决什么问题

- 团队周报/月报文档要用超宽表格 —— MCP 不支持
- 流程图 / 思维导图 / 架构图要好看 —— Agent 控制不精准
- 进展要 @ 对应同学 —— Agent 写入会破坏后续段落
- 汇报材料素材多种富媒体，要上传图片，还要插入到表格里 —— MCP 不支持
- Agent 告诉我这个功能语雀暂不支持，要我手动操作 —— 能力幻觉
- ……

Skill 自动加载对应的参考文档，**在错误发生之前拦截常见问题**，把上面这些"做不到"变成"自动做"。

## 7 条护栏（血泪教训）

1. **读后写** — `skylark_doc_update` 是**全文替换**，不是增量更新。更新前必须先 `doc_detail` 读取完整内容。
2. **4 空格缩进** — 画板 DSL 中 mindmap/architecturediagram 用 2 空格会静默失败，没有任何报错。
3. **Board ≠ Doc** — `skylark_resource_detail` 只能读取 Doc 类型文档中嵌入的 board resource，不支持独立的 Board 类型文档。
4. **colgroup 列宽 + Playwright 配合** — `<colgroup><col width="N" />` 可通过 API 直接设定列宽比例（总和≤750px，超出静默剥离）；表格总宽扩展和单列微调仍需 Playwright。操作顺序：colgroup 设比例 → 标宽下拖拽精调 → 全宽展示
5. **创建 ≠ 覆盖** — 多次调用 `resource_create` 会追加画板，不会覆盖已有的。
6. **有毒标签** — `<cardlink>`、`<mention>`、`<todo>` 通过 MCP API 写入会破坏后续所有段落。用 `[标题](url)`、`- [ ]`、`[@人名](profile-url)` 替代。
7. **表格不入容器** — 表格不能嵌入 `:::colorN` 高亮块、`<details>` 折叠块、`<columns>` 多栏——表格内容会丢失，高亮块还会边界溢出吞掉后续段落。

## 目录结构

```
yuque-power-user/
├── SKILL.md                              # 入口：护栏 + 决策路由
└── references/
    ├── mcp-api-guide.md        (262 行)  # skylark_* API 用法 + 陷阱
    ├── ymd-syntax.md           (280 行)  # 扩展 Markdown：高亮块、多栏、折叠、日历、页面引用...
    ├── cli-guide.md            (197 行)  # 语雀 CLI 图片上传
    ├── board-dsl.md            (183 行)  # 流程图 / 思维导图 / 架构图 DSL
    ├── html-table-advanced.md  (228 行)  # colspan、rowspan、backgroundColor、块级嵌套、选型决策
    ├── playwright-workarounds.md (382 行) # 列宽拖动、权重分配、工具栏5概念、全宽展示
    └── end-to-end-example.md   (256 行)  # 完整文档生产流程示例
```

Reference 文件**按需加载** — agent 只读取当前任务相关的文件，保持上下文精简。

## 安装

### 方式 A：下载安装包（推荐）

从 [Releases](https://github.com/serendipity-88/yuque-power-user/releases) 下载 `yuque-power-user.zip`，然后：

```bash
unzip yuque-power-user.zip -d ~/.claude/skills/
```

### 方式 B：从源码安装

```bash
git clone https://github.com/serendipity-88/yuque-power-user.git
cp -r yuque-power-user ~/.claude/skills/
```

### 验证

新开一个 Claude Code 对话，输入"帮我创建一个语雀文档"。Skill 应自动触发。

## 前置条件

**本 Skill 是纯知识包 — 不会安装任何运行时依赖。**

你至少需要以下其中一项：

| 依赖 | 用途 | 获取方式 |
|---|---|---|
| 语雀 MCP Server（`skylark_*` 工具） | 文档 CRUD、画板创建、搜索 | 蚂蚁内网环境自动连接 |
| 语雀 CLI（`yuque` 命令） | 图片上传（MCP 不支持此功能） | `npm i -g @antcli/yuque-ant-cli`（Node.js ≥ 20） |

## 能力覆盖

| 分类 | 能力 |
|---|---|
| MCP API | 创建/更新/搜索文档、画板资源、知识库目录 |
| YMD 语法 | 高亮块、多栏、折叠、标签卡片、日历卡片、页面引用、渐变文字、上下标、任务列表、@人名替代方案 |
| 画板 DSL | 流程图、思维导图（4空格！）、架构图（4空格！） |
| HTML 表格 | colspan、rowspan、backgroundColor、**colgroup 列宽比例**、单元格内富文本、**块级列表嵌套** |
| CLI | 通过 `--upload-images` 上传图片 |
| Playwright | 表格列宽拖动、**全宽展示**（与 colgroup 配合）、图片上传备选方案 |

## 局限性

标注 ❌ 的能力截至 2026-06 验证。语雀在持续迭代——如果某个标注为不支持的功能现在可用了，请更新对应的 reference 文件。

**自验证方法**：直接尝试（如在 `<td>` 中设置 `width="200"`），如果不再报错，说明限制已解除。

## 更新日志

### v2.2 (2026-06-09)

1. 新增 `<colgroup>` 列宽比例：HTML 表格可通过 API 直接设定列宽比例，无需 Playwright
2. 修正全宽展示与自适应宽度的关系：自适应宽度是全宽展示的前置开关，不是互斥项
3. 明确列宽操作顺序：先 colgroup 设比例，再标宽下拖拽精调，最后全宽展示扩展总宽

### v2.1 (2026-06-06)

- 修复高亮块颜色映射，有毒标签描述，补充了 @mention 的视觉替代方案
- 修复多栏描述支持表格、交叉引用指向章节标题缺失、callout 读写不对称问题，补充了 `<callout kind>` → `:::colorN` 的映射关系
- 新增 CLI `yuque update doc` 的全文替换警告
- 精简 ymd-syntax，包括快速模板、重复章节、错误条目，减少 context token 消耗

### v2.0 (2026-06-06)

- 新增 HTML 表格单元格内写列表、有序列表、引用的能力（官方确认 td 是 nested root）
- 新增超宽表格列宽权重分配规则（label=1, desc=2, list=3, media=2），Playwright 拖拽时按内容类型自动计算目标宽度
- 新增表格工具栏 5 个概念的完整说明（自适应宽度、列等宽、全宽/标宽展示、全屏），搞清楚依赖关系
- 修复 API 创建的表格"自适应宽度"默认关闭的问题，Playwright 操作前必须先开启
- 修复表格放进高亮块/折叠块/多栏里会丢失内容的问题，明确表格只能放文档顶层
- 修复 HTML 表格通过 API 读回后样式全丢（backgroundColor/colspan/rowspan 降级为普通 Markdown 表格），给出应对方案
- 新增端到端示例文档，覆盖从创建文档到画板到图片上传到调列宽的完整流程

### v1.0 (2026-06-04)

- 初始发布

## 许可证

MIT
