# End-to-End 示例：从零创建一篇带画板+表格+图片的文档

> 本示例覆盖语雀文档生产的完整链路，串联所有 reference 文件的核心知识。
> 每个步骤标注了对应的护栏和 reference 来源。

## 场景

用户说：「帮我在 xxx 知识库里创建一篇项目周报，要有高亮块总结、一个流程图、一个带背景色的表格、还要插几张本地截图。」

---

## Step 1：定位知识库 → 获取 book_id

```python
# 方式 A：按名称搜索
skylark_user_book_list(query="项目周报")
# → 返回 book_id=12345678

# 方式 B：从 URL 解析
skylark_resolve_url(url="https://yuque.antfin.com/guannan.gan/kzxqb4")
# → 返回 book_id=12345678
```

### URL 与 namespace / slug 的对应关系（2026-06-10 补充）

| 写法 | 含义 | 适用场景 |
|---|---|---|
| `https://yuque.antfin.com/guannan.gan/kzxqb4/abc123` | 完整 URL | 浏览器访问、分享链接 |
| `guannan.gan/kzxqb4` | **namespace**（user-login + book-slug） | CLI `yuque create doc --namespace`、MCP `skylark_search(scope=...)` |
| `guannan.gan/kzxqb4/abc123` | namespace + doc-slug | CLI `yuque update doc <namespace>/<slug>` |
| `12345678` | book_id（数字 ID） | MCP `skylark_doc_create(book_id=...)` |
| `87654321` | doc_id（数字 ID） | MCP `skylark_doc_detail/update(doc_id=...)` |

#### 从 URL 提取 namespace 和 slug

URL: `https://yuque.antfin.com/guannan.gan/kzxqb4/abc123`
- 域名后第 1 段 `guannan.gan` = user-login
- 第 2 段 `kzxqb4` = book-slug
- 最后一段 `abc123` = doc-slug
- **namespace = user-login + "/" + book-slug** = `guannan.gan/kzxqb4`
- **CLI 完整路径 = namespace + "/" + doc-slug** = `guannan.gan/kzxqb4/abc123`

#### 常见错误

```bash
# ❌ 错误：把 doc-slug 漏了，只传 namespace
yuque update doc guannan.gan/kzxqb4 --body-file ./doc.md
# → 报错：namespace 缺少 doc-slug

# ✅ 正确：namespace + doc-slug
yuque update doc guannan.gan/kzxqb4/abc123 --body-file ./doc.md
```

**📎 来源**：[mcp-api-guide.md] 快速参考表

---

## Step 2：组装文档正文（YMD + HTML 表格）

在本地组装完整 Markdown 正文，一次性传入。

```markdown
# W23 项目周报（0602-0608）

:::color2
**本周总结：** 核心指标环比 +12%，AB 实验完成第二轮验证，下周进入全量评审。
:::

## 关键指标

<table>
<colgroup>
<col width="100" />
<col width="150" />
<col width="250" />
<col width="250" />
</colgroup>
  <tr>
    <td backgroundColor="#f5f5f5">**指标**</td>
    <td backgroundColor="#f5f5f5">**本周**</td>
    <td backgroundColor="#f5f5f5">**环比**</td>
    <td backgroundColor="#f5f5f5">**状态**</td>
  </tr>
  <tr>
    <td>DAU</td>
    <td>128.5w</td>
    <td><font style="color:#59A869;">+12.3%</font></td>
    <td backgroundColor="#E8F7CF">达标</td>
  </tr>
  <tr>
    <td>转化率</td>
    <td>3.2%</td>
    <td><font style="color:#DF2A3F;">-0.5pp</font></td>
    <td backgroundColor="#FDDEDE">关注</td>
  </tr>
</table>

## 数据趋势

![DAU趋势](./images/dau-trend.png)

![转化漏斗](./images/funnel.png)

## 下周计划

- [ ] 全量评审材料准备
- [ ] AB 实验第三轮方案确认

## 附录：详细实验数据

| 实验组 | 样本量 | 转化率 | p值 |
|--------|--------|--------|-----|
| 对照组 | 50w | 3.7% | — |
| 实验A | 50w | 3.2% | 0.03 |
```

**⚠️ 注意事项**：
- `:::colorN` 前后必须有空行，否则不渲染（color2=绿色正向、color4=红色危险，按语义选色） → [ymd-syntax.md] 常见错误
- **表格选型**：本示例用 HTML 表格（需要背景色）。HTML `<td>` 是 nested root，支持块级列表嵌套（`+ item` 独立行写法）。后续 API 读→写会丢样式，如需定期 API 更新改用 Markdown 表格 → [html-table-advanced.md] 选型决策树
- **colgroup 列宽**：`<colgroup>` 控制列宽比例，建议总和 ≤750px（超 750px 不剥离但标准宽度下容器裁切）。标签列窄（~100px），内容列宽（~250px）。如果表格仍显窄或总和 >750px，后续用 Playwright 开全宽展示 → [html-table-advanced.md] colgroup 章节
- `<font style="color:#DF2A3F;">` 必须用 HEX 颜色值，不能用 `color:red` → [ymd-syntax.md] 文本格式增强
- **不要使用** `<cardlink>`、`<mention>`、`<todo>` 标签 → 护栏 6：有毒标签禁用
- **表格不入容器**：本示例中实验数据表放在文档顶层（`## 标题`分隔），不嵌入 `<details>` → 护栏 7

---

## Step 3：创建文档（MCP API）

```python
skylark_doc_create(
    book_id=12345678,
    title="W23 项目周报（0602-0608）",
    body=上面组装的完整 Markdown,
    format="markdown"
)
# → 返回 doc_id=87654321
```

**📎 来源**：[mcp-api-guide.md] 快速创建文档模板

---

## Step 4：插入流程图画板

```python
skylark_resource_create(
    resource_type="board",
    doc_id=87654321,
    type="flowchart",
    dsl="A([需求评审]) -->|通过| B[AB实验设计]\nB --> C{实验结论}\nC -->|显著| D[全量上线]\nC -->|不显著| E[迭代优化]\nE --> B"
)
# → 返回 resource_id="ki2ue"
```

**⚠️ 注意事项**：
- 创建用 `dsl` 参数，更新用 `text` 参数——两个不一样！ → [board-dsl.md] MCP 调用详解
- 再次调用 `resource_create` 会**追加**新画板，不会覆盖 → 护栏 5
- 如果要插思维导图，必须用 **4 空格**缩进 → 护栏 2

```python
# 思维导图示例（注意缩进）
skylark_resource_create(
    resource_type="board",
    doc_id=87654321,
    type="mindmap",
    dsl="- 本周工作\n    - 实验\n        - AB实验第二轮\n        - 数据分析\n    - 产品\n        - PRD更新\n        - 评审准备"
)
```

---

## Step 5：上传本地图片（CLI）

MCP API **不支持**图片上传，必须用 CLI。

```bash
# 1. 准备目录结构
mkdir -p /tmp/weekly-report/images
cp ~/screenshots/dau-trend.png /tmp/weekly-report/images/
cp ~/screenshots/funnel.png /tmp/weekly-report/images/

# 2. 将 Step 2 的 Markdown 保存为文件
#    确保图片引用用相对路径：![](./images/dau-trend.png)

# 3. 用 CLI 更新文档（带图片上传）
cd /tmp/weekly-report
yuque update doc guannan.gan/kzxqb4/<slug> \
  --body-file ./doc.md \
  --upload-images \
  --yes
```

**⚠️ 注意事项**：
- 必须 `cd` 到 Markdown 文件所在目录，否则相对路径解析失败 → [cli-guide.md] 常见错误 #1
- 必须用 `--body-file`，不能用 `--body` 内联 → [cli-guide.md] 常见错误 #2
- CLI 更新也是全文替换，会覆盖 Step 3 创建的内容——所以 **Step 2 的 Markdown 必须是完整正文**（包含画板引用 `board://ki2ue`）

### CLI 更新时保留画板引用

如果 Step 4 已经插入了画板，CLI 更新前需要先读取文档拿到画板引用：

```python
# 1. 先读取当前文档内容
skylark_doc_detail(doc_id=87654321)
# → body 中包含 board://ki2ue

# 2. 在本地 Markdown 的对应位置插入画板引用
# 在 "## 数据趋势" 之前加一行：
# board://ki2ue
```

这是**护栏 1（读后写）**的典型应用——即使用 CLI 更新，也要先读取完整内容，确保不丢失已有的画板。

---

## Step 6（可选）：调整表格列宽（colgroup + Playwright）

Step 2 已通过 `<colgroup>` 设置了列宽比例（建议 ≤750px）。如果表格仍显窄或列宽需要微调，按优先级操作：

1. **colgroup 已设比例** → 检查效果，通常已够用
2. **需要更宽** → Playwright 开全宽展示（标宽→全宽，容器不再裁切，表格按 colgroup 比例自然展开）
3. **个别列需精调** → Playwright 拖拽手柄（建议标宽下操作，再开全宽）

```python
# Playwright 全宽展示代码详见 playwright-workarounds.md
```

**📎 来源**：[html-table-advanced.md] colgroup 章节 + [playwright-workarounds.md] 全宽展示

---

## 完整操作链总结

```
定位知识库 (book_id)
    │
    ▼
组装完整 Markdown（YMD + HTML 表格 + 图片占位）
    │
    ▼
MCP 创建文档 (doc_id)  ──护栏1: 读后写──┐
    │  护栏4a: colgroup 建议≤750px          │
    │                                    │
    ▼                                    │
MCP 插入画板 (resource_id)               │
    │  护栏2: 4空格缩进                   │
    │  护栏5: 追加不覆盖                   │
    │                                    │
    ▼                                    │
读取文档 ← 获取画板引用 board://xxx ──────┘
    │
    ▼
合并到本地 Markdown
    │
    ▼
CLI 更新文档 + 上传图片
    │  护栏6: 不用有毒标签
    │
    ▼
（可选）Playwright 全宽展示/拖拽精调
```

## 常见变体

### 变体 A：更新已有文档（非新建）

区别在于 Step 1 和 Step 3：

```python
# Step 1: 用 URL 解析获取 doc_id
skylark_resolve_url(url="https://yuque.antfin.com/guannan.gan/kzxqb4/xxx")
# → doc_id=87654321

# Step 3: 先读后改再写（护栏 1！）
existing = skylark_doc_detail(doc_id=87654321)  # 读
modified_body = 修改 existing.body 的对应部分     # 改
skylark_doc_update(doc_id=87654321, body=modified_body, format="markdown")  # 写
```

### 变体 B：只用 MCP，不需要图片

跳过 Step 5（CLI），Step 2 中不引用本地图片即可。整个流程只需 MCP API。

### 变体 C：只用 CLI，不需要画板

跳过 Step 4（画板），直接在 Step 5 用 CLI 创建文档：

```bash
cd /tmp/weekly-report
yuque create doc \
  --namespace guannan.gan/kzxqb4 \
  --title "W23 项目周报" \
  --body-file ./doc.md \
  --upload-images \
  --yes
```
