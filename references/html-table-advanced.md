# HTML 表格高级用法

## Markdown 表格 vs HTML 表格能力对比

| 能力 | Markdown 表格 | HTML 表格 |
|------|-------------|----------|
| 基本表格 | ✅ | ✅ |
| 单元格内换行 | ✅ `<br>` | ✅ |
| 加粗/斜体 | ✅ | ✅ |
| 单元格内列表 | ✅ `- ` | ✅ |
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

- ✅ 列表（`- ` 无序列表）
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

API/YMD 均无法控制列宽。如需调整，参见 [playwright-workarounds.md](playwright-workarounds.md)。
