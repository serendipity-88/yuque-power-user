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

---

## 快速模板

### HTML 表格（跨列/跨行/背景色）

```html
<table>
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

## 已验证不支持的属性（2026-06 验证）

| 方案 | 结果 | 错误信息 |
|------|------|---------|
| `<td width="N">` | ❌ | `Unsupported <td> attribute "width"` |
| `<td style="width:N">` | ❌ | `Unsupported <td> attribute "style"` |
| `<colgroup><col>` | ❌ | `table block only allows <tr> direct children` |
| 不同长度内容自适应列宽 | ❌ | 列宽始终均分，不随内容变化 |

### 列宽调整

API/YMD 均无法控制列宽。Playwright 拖拽可持久化，详见 [playwright-workarounds.md](playwright-workarounds.md)。

---

## ✅ `<td>` 块级内容嵌套（官方确认）

> 参考：https://yuque.antfin.com/lark/ymd/lsegluq35m7psxhg
> "`td` 是 nested root，可承载段落、列表、代码块等块级内容"

### 无序列表

```plain
<table>
<tr>
<td>
+ 列表项一
+ 列表项二
</td>
</tr>
</table>
```

### 有序列表

```plain
<table>
<tr>
<td>
1. 确认需求
2. 评审方案
3. 跟进上线
</td>
</tr>
</table>
```

### 引用 + 多段落

```plain
<table>
<tr>
<td rowspan="2">
跨两行说明

> 引用内容
</td>
<td>
第一段

第二段
</td>
</tr>
</table>
```

### ⚠️ 注意事项
- 列表项必须**独立成行**，不要用 `<br>` 内联拼接
- `<td>` 内容前后需要换行，不要写在 `<td>` 同一行
- API 读回会降级为 Markdown 表格，列表格式丢失——仅影响读→改→写场景，不影响一次性写入的页面渲染
- Markdown `|...|` 表格不支持此特性（单元格只能单行）

---

## 🔴 表格选型决策（2026-06-05 实测验证）

### 互斥矩阵

| 能力 | Markdown 表格 | HTML 表格 |
|------|:---:|:---:|
| 背景色 | ❌ | ✅ |
| 合并单元格（colspan/rowspan） | ❌ | ✅ |
| 单元格内列表 | ❌ 不支持 | ✅ 块级嵌套（`+ item` 独立行） |
| 单元格内引用/代码块 | ❌ | ✅（`<td>` 是 nested root） |
| 读写安全（API 读→改→写不丢格式） | ✅ | ❌ 读回变 Markdown，样式全丢 |
| 彩色文字 `<font>` | ✅ | ✅ |
| 换行 `<br>` | ⚠️ 写入有效，读回变空格 | ⚠️ 同左 |

### 选型决策树

```
需要背景色/合并/列表？
├── 是 → HTML 表格（<table>/<tr>/<td>）
│   └── 后续需要 API 读→改→写？
│       ├── 是 → ⚠️ 每次更新必须重写完整 HTML
│       └── 否 → ✅ 一次性写入无问题
└── 否 → Markdown 表格（语法简洁、读写安全）
```

### 典型场景推荐

| 场景 | 推荐 | 原因 |
|------|------|------|
| 周报指标表（表头底色、红绿标记） | HTML | 背景色 + 列表都支持 |
| 策略矩阵（多行进展+列表） | HTML | `<td>` 块级嵌套列表 ✅ |
| 一次性发布的汇报 | HTML | 功能最全 |
| 需定期 API 更新的模板 | Markdown | 读写安全 |
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

`skylark_doc_detail` 读回时 HTML 表格被转为 Markdown 表格——backgroundColor、colspan、rowspan **全丢**。

**应对**：
- 修改非表格部分时：保留原始 HTML 表格代码不动
- 必须修改表格时：用完整 HTML 重新写入
