# HTML 表格高级用法

## Markdown 表格 vs HTML 表格能力对比

| 能力 | Markdown 表格 | HTML 表格 |
|------|-------------|----------|
| 基本表格 | ✅ | ✅ |
| 单元格内换行 | ✅ `<br>` | ✅ |
| 加粗/斜体 | ✅ | ✅ |
| 单元格内列表 | ❌ 不支持 | ✅ 块级嵌套（`<td>` 内独立行写 `+ item`） |
| 跨列合并 | ❌ | ✅ `colspan` |
| 跨行合并 | ❌ | ✅ `rowspan` |
| 背景色 | ❌ | ✅ `backgroundColor` |
| 彩色文字 | ✅ `<font>` | ✅ |
| 列宽比例 | ❌ | ✅ `<colgroup><col width="N" />` |

---

## `<colgroup>` 列宽控制（2026-06-09 验证可用）

### 语法

在 `<table>` 首位放置 `<colgroup>`，为每列指定宽度（正整数 px）：

```html
<table>
<colgroup>
<col width="100" />
<col width="350" />
<col width="300" />
</colgroup>
<tr>
<td>策略</td>
<td>进展</td>
<td>效果</td>
</tr>
</table>
```

### 方向指引

根据每列内容类型分配宽度，标签列窄、内容列宽：

- 标签/分类列（策略名、状态、人名）：~100px
- 简短描述列（一句话说明、指标值）：~150-200px
- 列表/详情列（3+ bullet、多段文字）：~250-350px
- 富媒体列（截图、链接）：~150-200px

参考示例（3列）：`100 / 350 / 300`，参考示例（4列）：`80 / 200 / 270 / 200`

### 🔴 硬约束

1. **建议总和 ≤750px**：超过 750px 不会被 API 剥离，但标准宽度下容器（750px + `overflow: hidden`）会裁切溢出部分。开启全宽展示后可正常显示。建议总和 ≤750px 以避免标准宽度下溢出；如需更宽，配合全宽展示使用即可
2. **`<col>` 数量 = 表格列数**：colspan=2 的列占 2 个 `<col>` 位。数量不匹配会导致列宽错乱。**详见下方对照表**
3. **format=md 读回丢失**：colgroup 与 backgroundColor/colspan 一样，format=md 读回时丢失。含 HTML 表格的文档必须用 format=ymd 读取
4. **width 只接受正整数 px**：不支持百分比

### `<colgroup>` 与 `colspan` 对照表（2026-06-10 补充）

硬约束 #2 的规则容易按字面理解错：写 3 列 + 第 1 行 colspan=2 的表格时，常见误判是只写 2 个 `<col>`。实际上 colspan=2 占 2 个 `<col>` 位，`<col>` 总数仍 = 表格列数。

| 表格列数 | colspan / rowspan 分布 | `<col>` 个数 | 写法 |
|---|---|---|---|
| 3 | 无合并 | 3 | `<col>×3` |
| 3 | 第 1 行 colspan=2 | **3** | `<col>×3`（colspan 占 2 个位） |
| 3 | 第 2 行 rowspan=2 | 3 | `<col>×3`（rowspan 不影响 col 数量） |
| 4 | 第 1 行 colspan=2 + 第 2 行 colspan=2 | 4 | `<col>×4`（两个 colspan 各占 2 个位） |
| 4 | 跨 2 行 + 跨 2 列混合 | 4 | `<col>×4` |
| 5 | colspan/rowspan 混合 | 5 | `<col>×5` |

#### ❌ 错误示例（3 列 colspan=2 写 2 个 col）

```html
<table>
<colgroup>
<col width="200" />  <!-- 错！3 列表格应写 3 个 <col> -->
<col width="200" />
</colgroup>
<tr>
  <td colspan="2">合并两列</td>
  <td>独立</td>
</tr>
</table>
```

**实际结果**：语雀可能回退为 2 列渲染，colspan 行为异常。

#### ✅ 正确示例（3 列 colspan=2 写 3 个 col）

```html
<table>
<colgroup>
<col width="200" />  <!-- 第 1 列 -->
<col width="300" />  <!-- 第 2 列（与第 1 列共同被 colspan=2 占用） -->
<col width="200" />  <!-- 第 3 列 -->
</colgroup>
<tr>
  <td colspan="2">合并两列</td>
  <td>第 3 列</td>
</tr>
<tr>
  <td>第 1 列</td>
  <td>第 2 列</td>
  <td>第 3 列</td>
</tr>
</table>
```

### 与其他能力组合

colgroup 可与 backgroundColor、colspan、rowspan 自由组合：

```html
<table>
<colgroup>
<col width="80" />
<col width="200" />
<col width="200" />
<col width="270" />
</colgroup>
<tr>
<td backgroundColor="#D6E4FF">**序号**</td>
<td backgroundColor="#D6E4FF">**模块**</td>
<td backgroundColor="#D6E4FF">**负责人**</td>
<td backgroundColor="#D6E4FF">**说明**</td>
</tr>
<tr>
<td>2</td>
<td colspan="2">合并两列</td>
<td>剩余内容</td>
</tr>
</table>
```

### 列宽控制方案选择

```
创建表格时就知道列宽比例？
├── 是 → API 写 <colgroup>（比例写入 body，建议总和 ≤750px）
│       └── 列多空间紧或总和 >750px？→ 追加 Playwright 全宽展示
└── 否 → 创建后 Playwright 拖拽 + 全宽展示

两种方案可组合：colgroup 设初始比例 → 标宽下拖拽精调 → 全宽展示扩展总宽
详见 playwright-workarounds.md
```

---

## 快速模板

### HTML 表格（跨列/跨行/背景色/列宽）

```html
<table>
<colgroup>
<col width="100" />
<col width="350" />
<col width="300" />
</colgroup>
<tr>
<td colspan="2" backgroundColor="#f5f5f5">**合并两列标题**</td>
<td rowspan="2">跨两行</td>
</tr>
<tr>
<td>普通单元格</td>
<td backgroundColor="#E8F7CF">绿色背景</td>
</tr>
</table>
```

### Markdown 表格（换行 + 列表）

```markdown
| 策略 | 进展 | 结论 |
|------|------|------|
| **策略名**<br>描述 | - 数据点1<br>- 数据点2 | 结论 |
```

---

## 详细说明

### 支持的 `<td>` 属性

| 属性 | 语法 | 说明 |
|------|------|------|
| 跨列合并 | `colspan="N"` | 合并 N 列 |
| 跨行合并 | `rowspan="N"` | 合并 N 行 |
| 背景色 | `backgroundColor="#hex"` | 单元格背景色 |

### 单元格内富文本支持

- ✅ 列表：`<td>` 是 nested root，支持块级列表。写法：每项独立一行（`+ item1` 换行 `+ item2`），不要用 `<br>` 内联。同时支持有序列表（`1. 2. 3.`）、引用（`>`）、代码块
- ✅ 代码块
- ✅ 引用
- ✅ 加粗/斜体/链接
- ✅ 图片（`![](url)`）
- ✅ `<br>` 换行

### 常用背景色参考（2026-06-03 实测验证）

| 色系 | Hex | 用途 | 实测 |
|------|-----|------|------|
| 浅蓝 | `#D6E4FF` | 信息/说明/表头 | ✅ |
| 浅灰 | `#F5F5F5` | 标题行/分组行 | ✅ |
| 亮黄 | `#FEF6D0` | 重点关注/高亮 | ✅ |
| 浅绿 | `#E8F7CF` | 正常/达标 | ✅ |
| 浅红 | `#FDDEDE` | 异常/风险 | ✅ |

### 表头加粗 + 整行底色模板

```html
<table>
<colgroup>
<col width="100" />
<col width="350" />
<col width="300" />
</colgroup>
<tr>
<td backgroundColor="#D6E4FF">**列标题1**</td>
<td backgroundColor="#D6E4FF">**列标题2**</td>
<td backgroundColor="#D6E4FF">**列标题3**</td>
</tr>
<tr>
<td>内容</td>
<td><span backgroundColor="#E8F7CF">达标</span></td>
<td><font style="color:#DF2A3F;">风险项</font></td>
</tr>
</table>
```

支持在单元格内混合使用：`backgroundColor`（底色）+ `**bold**`（加粗）+ `<span backgroundColor>`（文字高亮）+ `<font style="color:">`（彩色文字）+ `[链接](url)`。

---

## ✅ 已验证可用（2026-06 验证）

| 方案 | 用途 | 限制 |
|------|------|------|
| `<colgroup><col width="N" />` | 设置列宽比例 | 建议 ≤750px；超 750px 不剥离但标准宽度下容器裁切；format=md 读回丢失 |
| `colspan="N"` | 跨列合并 | — |
| `rowspan="N"` | 跨行合并 | — |
| `backgroundColor="#hex"` | 单元格背景色 | format=md 读回丢失 |
| `<td>` 块级嵌套 | 单元格内列表/引用/代码 | format=md 读回丢失 |

## ❌ 已验证不可用（2026-06 验证）

| 方案 | 结果 | 错误信息 |
|------|------|---------|
| `<td width="N">` | ❌ | `Unsupported <td> attribute "width"` |
| `<td style="width:N">` | ❌ | `Unsupported <td> attribute "style"` |
| 不同长度内容自适应列宽 | ❌ | 列宽始终均分，不随内容变化 |

---

## 🔴 表格选型决策（2026-06-09 更新）

### 互斥矩阵

| 能力 | Markdown 表格 | HTML 表格 |
|------|:---:|:---:|
| 背景色 | ❌ | ✅ |
| 合并单元格（colspan/rowspan） | ❌ | ✅ |
| 列宽比例（colgroup） | ❌ | ✅ |
| 单元格内列表 | ❌ 不支持 | ✅ 块级嵌套（`+ item` 独立行） |
| 单元格内引用/代码块 | ❌ | ✅（`<td>` 是 nested root） |
| 读写安全（API 读→改→写不丢格式） | ✅ | ❌ 读回变 Markdown，样式全丢 |
| 彩色文字 `<font>` | ✅ | ✅ |
| 换行 `<br>` | ⚠️ 写入有效，读回变空格 | ⚠️ 同左 |

### 选型决策树

```
需要背景色/合并/列宽控制/列表？
├── 是 → HTML 表格（<table>/<tr>/<td>）
│   ├── 需要列宽比例？→ 加 <colgroup>
│   └── 后续需要 API 读→改→写？
│       ├── 是 → ⚠️ 必须用 format=ymd 读取，否则 colgroup/背景色/合并全丢
│       └── 否 → ✅ 一次性写入无问题
└── 否 → Markdown 表格（语法简洁、读写安全）
```

### 典型场景推荐

| 场景 | 推荐 | 原因 |
|------|------|------|
| 周报指标表（表头底色、列宽控制、红绿标记） | HTML + colgroup | 背景色 + 列宽 + 列表都支持 |
| 策略矩阵（多行进展+列表） | HTML + colgroup | `<td>` 块级嵌套列表 ✅ |
| 一次性发布的汇报 | HTML + colgroup | 功能最全 |
| 需定期 API 更新的数据表 | Markdown | 读写安全 |
| 简单对比表（无背景色无列表） | Markdown | 语法简洁 |

---

## 🔴 容器嵌套限制（2026-06-05 实测验证）

**表格不能嵌入以下容器，否则内容会丢失或边界溢出：**

| 容器 | 嵌入表格 | 表现 |
|------|:---:|---------|
| `:::colorN` 高亮块 | ❌ | 表格丢失 + callout 边界溢出，吞掉后续段落 |
| `<details>` 折叠块 | ❌ | 只剩 `<summary>`，表格内容全丢 |
| `<columns>` 多栏 | ❌ | 表格完全丢失 |
| 同文档并列 | ✅ | 两种表格可共存 |

安全做法：表格放文档顶层，用 `## 标题` 分组。

---

## HTML 表格读写降级（2026-06-05 实测验证）

`skylark_doc_detail` 读回时 HTML 表格被转为 Markdown 表格——backgroundColor、colspan、rowspan、**colgroup** 全丢。

**应对**：
- 修改非表格部分时：保留原始 HTML 表格代码不动
- 必须修改表格时：用完整 HTML 重新写入
- **含 HTML 表格的文档必须用 format=ymd 读取**，否则 colgroup/backgroundColor/colspan 一次性全丢