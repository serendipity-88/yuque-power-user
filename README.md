# yuque-power-user

[Claude Code Skill](https://docs.anthropic.com/en/docs/claude-code)，教你的 AI agent 正确使用语雀 MCP API、CLI 和扩展 Markdown（YMD）语法——包括那些你得踩几小时坑才能发现的陷阱。

## 这个 Skill 做什么

当你或你的 agent 说类似这样的话：

- "我写了个方案，帮我发到语雀上，放在 xxx 知识库里"
- "帮我在文档里加个流程图和思维导图"
- "表格要好看一点，表头加底色，有几个单元格要合并，表格太窄了能拉宽么"
- "我本地有几张截图要插到语雀文档里"
- "帮我写个周报模板，要有高亮块、折叠块、关键指标标红、加个日历卡片"

Skill 会自动加载对应的参考文档，并**在错误发生之前拦截常见问题**。

## 6 条护栏（血泪教训）

1. **读后写** — `skylark_doc_update` 是**全文替换**，不是增量更新。更新前必须先 `doc_detail` 读取完整内容。
2. **4 空格缩进** — 画板 DSL 中 mindmap/architecturediagram 用 2 空格会静默失败，没有任何报错。
3. **Board ≠ Doc** — `skylark_resource_detail` 只能读取 Doc 类型文档中嵌入的 board resource，不支持独立的 Board 类型文档。
4. **列宽不可控** — 表格列宽无法通过 API 或 YMD 设置，只有 Playwright 拖动可行。
5. **创建 ≠ 覆盖** — 多次调用 `resource_create` 会追加画板，不会覆盖已有的。
6. **有毒标签** — `<cardlink>`、`<mention>`、`<todo>` 通过 MCP API 写入会破坏后续所有段落。用 `[标题](url)`、`- [ ]`、`[@人名](profile-url)` 替代。

## 目录结构

```
yuque-power-user/
├── SKILL.md                              # 入口：护栏 + 决策路由
└── references/
    ├── mcp-api-guide.md        (258 行)  # skylark_* API 用法 + 陷阱
    ├── ymd-syntax.md           (334 行)  # 扩展 Markdown：高亮块、多栏、折叠、日历、页面引用...
    ├── cli-guide.md            (193 行)  # 语雀 CLI 图片上传
    ├── board-dsl.md            (183 行)  # 流程图 / 思维导图 / 架构图 DSL
    ├── html-table-advanced.md  (106 行)  # colspan、rowspan、backgroundColor
    └── playwright-workarounds.md (118 行) # 列宽拖动、图片上传备选
```

Reference 文件**按需加载** — agent 只读取当前任务相关的文件，保持上下文精简。

## 安装

### 方式 A：从 `.skill` 包安装

```bash
# 从 Releases 下载 .skill 文件，然后：
cd ~/.claude/skills
unzip /path/to/yuque-power-user.skill
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
| HTML 表格 | colspan、rowspan、backgroundColor、单元格内富文本 |
| CLI | 通过 `--upload-images` 上传图片 |
| Playwright | 表格列宽拖动、图片上传备选方案 |

## 局限性

标注 ❌ 的能力截至 2026-06 验证。语雀在持续迭代 — 如果某个标注为不支持的功能现在可用了，请更新对应的 reference 文件。

**自验证方法**：直接尝试（如在 `<td>` 中设置 `width="200"`）。如果不再报错，说明限制已解除。

## License

MIT
