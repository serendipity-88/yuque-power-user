# yuque-power-user

[Claude Code Skill](https://docs.anthropic.com/en/docs/claude-code) for Yuque (语雀) document automation. Teaches your AI agent how to correctly use Yuque's MCP API, CLI, and extended Markdown (YMD) syntax — including the pitfalls that cost you hours to discover on your own.

## What this Skill does

When you or your agent says something like:

- "帮我在语雀上建个文档"
- "给文档加个流程图和思维导图"
- "表格加个合并单元格、背景色，列宽能拉宽么"
- "本地截图要插到语雀文档里"
- "写个周报模板，要高亮块、折叠块、红色指标、日历卡片"

The skill automatically loads the right reference and **prevents common mistakes** before they happen.

## 6 Guardrails (the hard-won lessons)

1. **Read before write** — `skylark_doc_update` is a **full-text replacement**, not a patch. Always `doc_detail` first.
2. **4-space indent** — Board DSL for mindmap/architecturediagram silently fails with 2-space indent. No error message.
3. **Board ≠ Doc** — `skylark_resource_detail` only works on board resources embedded in Doc-type documents, not standalone Board documents.
4. **No API column width** — Table column width cannot be set via API or YMD. Only Playwright drag works.
5. **Create ≠ Replace** — Multiple `resource_create` calls append boards, they don't overwrite.
6. **Toxic tags** — `<cardlink>`, `<mention>`, `<todo>` written via MCP API will corrupt all subsequent paragraphs. Use `[title](url)`, `- [ ]`, and `[@name](profile-url)` instead.

## What's inside

```
yuque-power-user/
├── SKILL.md                              # Entry point: guardrails + decision router
└── references/
    ├── mcp-api-guide.md        (258 lines)  # skylark_* API usage + traps
    ├── ymd-syntax.md           (334 lines)  # Extended Markdown: callouts, columns, details, calendar, page refs...
    ├── cli-guide.md            (193 lines)  # yuque CLI for image upload
    ├── board-dsl.md            (183 lines)  # Flowchart / mindmap / architecture diagram DSL
    ├── html-table-advanced.md  (106 lines)  # colspan, rowspan, backgroundColor
    └── playwright-workarounds.md (118 lines) # Column width drag, image upload fallback
```

References are **lazy-loaded** — the agent only reads the file relevant to the current task, keeping context lean.

## Install

### Option A: From `.skill` package

```bash
# Download the .skill file from Releases, then:
cd ~/.claude/skills
unzip /path/to/yuque-power-user.skill
```

### Option B: From source

```bash
git clone https://github.com/<your-username>/yuque-power-user.git
cp -r yuque-power-user ~/.claude/skills/
```

### Verify

Start a new Claude Code conversation and say "帮我创建一个语雀文档". The skill should auto-trigger.

## Prerequisites

**This skill is a pure knowledge package — it does not install any runtime dependencies.**

You need at least one of:

| Dependency | What for | How to get |
|---|---|---|
| Yuque MCP Server (`skylark_*` tools) | Document CRUD, board creation, search | Auto-connected in Ant Group intranet environment |
| Yuque CLI (`yuque` command) | Image upload (MCP can't do this) | `npm i -g @antcli/yuque-ant-cli` (Node.js ≥ 20) |

## Capability coverage

| Category | Capabilities |
|---|---|
| MCP API | Create/update/search docs, board resources, book TOC |
| YMD Syntax | Callouts, columns, details/fold, label cards, calendar cards, page references, gradient text, sup/sub, task lists, @mention workaround |
| Board DSL | Flowchart, mindmap (4-space!), architecture diagram (4-space!) |
| HTML Tables | colspan, rowspan, backgroundColor, rich text in cells |
| CLI | Image upload via `--upload-images` |
| Playwright | Table column width drag, image upload fallback |

## Limitations

Items marked ❌ were verified as of 2026-06. Yuque evolves — if something marked unsupported now works, update the corresponding reference file.

**Self-verification method**: Try the thing (e.g., `<td width="200">`). If it no longer errors, the limitation has been lifted.

## License

MIT
