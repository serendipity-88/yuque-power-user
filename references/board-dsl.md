# 语雀画板 DSL 语法速查

通过 MCP `skylark_resource_create` / `skylark_resource_update` 使用。

> **最重要的规则：Text DSL 缩进必须用 4 空格（不是 2 空格）**

## 快速模板

```python
# 创建流程图
skylark_resource_create(
    resource_type="board",
    doc_id=DOC_ID,
    type="flowchart",
    dsl="A([开始]) --> B[处理] --> C{判断}\nC -->|是| D[执行]\nC -->|否| E([结束])"
)

# 创建思维导图（注意 4 空格缩进）
skylark_resource_create(
    resource_type="board",
    doc_id=DOC_ID,
    type="mindmap",
    dsl="- 中心主题\n    - 分支1\n        - 子节点1.1\n    - 分支2\n        - 子节点2.1"
)

# 更新画板（注意用 text 参数，不是 dsl）
skylark_resource_update(
    resource_type="board",
    doc_id=DOC_ID,
    resource_id=RESOURCE_ID,
    text="A[新开始] --> B[新结束]"
)
```

---

## 支持的画板类型

| 类型 | MCP type 值 | DSL 特点 | 稳定性 |
|------|------------|---------|-------|
| 流程图 | `flowchart` | 节点+连线，无缩进要求 | 最稳定 |
| 思维导图 | `mindmap` | `- ` 列表，**4 空格缩进** | 缩进敏感 |
| 架构图 | `architecturediagram` | `+`/`-`/`*` 层级，**4 空格缩进** | 缩进敏感 |

---

## flowchart 语法

最稳定的画板类型，推荐优先使用。

```
A([开始]) -->|步骤1| B[处理]
B --> C{判断}
C -->|是| D[执行]
C -->|否| E([结束])
```

**节点形状**：

| 语法 | 形状 |
|------|------|
| `A[文本]` | 矩形 |
| `A{文本}` | 菱形（判断） |
| `A([文本])` | 体育场形（圆角矩形） |
| `A((文本))` | 双圆（圆形） |

**连线**：

| 语法 | 效果 |
|------|------|
| `A --> B` | 普通连线 |
| `A -->\|标签\| B` | 带标签连线 |

**换行**：节点内换行用 `\n`

**禁止事项**：

- 不要用双引号包裹节点文本 — `A["文本"]` 会显示引号，改为 `A[文本]`
- 不要用 `subgraph` 语法 — 会导致解析失败

---

## mindmap 语法

必须使用 4 空格缩进，否则只显示根节点。

```
- 中心主题
    - 分支1
        - 子节点1.1
        - 子节点1.2
    - 分支2
        - 子节点2.1
```

每级缩进 4 个空格。`- ` 后跟节点文本。

---

## architecturediagram 语法

必须使用 4 空格缩进，否则解析失败。

```
+ 接入层
    - Web 接入
        * API Gateway
        * BFF 聚合
+ 应用层
    - 核心服务
        * 用户服务
        * 订单服务
+ 数据层
    - 在线存储
        * MySQL
        * Redis
```

**符号含义**：

| 符号 | 层级 | 说明 |
|------|------|------|
| `+` | 顶层分组 | 最外层架构域 |
| `-` | 中间层模块 | 功能模块 |
| `*` | 叶子节点 | 具体组件/服务 |

---

## MCP 调用详解

### 创建画板

```
skylark_resource_create(
    resource_type="board",
    doc_id=<doc-id>,
    type="flowchart",       # 或 "mindmap" / "architecturediagram"
    dsl="A[开始] --> B[结束]"
)
```

返回值包含 `resource.id`（如 `"ki2ue"`），用于后续更新和读取。

### 更新画板

```
skylark_resource_update(
    resource_type="board",
    doc_id=<doc-id>,
    resource_id="<resource-id>",
    text="A[新文本] --> B[新文本]"    # 注意：更新用 text 参数，不是 dsl
)
```

### 读取画板

```
skylark_resource_detail(
    resource_type="board",
    doc_id=<doc-id>,
    resource_id="<resource-id>"
)
```

### 注意事项

- `resource_create` 是追加操作，不会覆盖已有画板。要替换需先删除旧引用再创建新画板。
- 更新时用 `text` 参数传入 DSL 内容，不是 `dsl` 参数。
- 只能在 Doc 类型文档中操作画板，不能对 Board 类型文档调用 `resource_detail`。

---

## 常见错误排查

| 错误现象 | 原因 | 解决方法 |
|---------|------|---------|
| mindmap 只显示根节点 | 使用了 2 空格缩进 | 改为 4 空格缩进 |
| architecturediagram 解析失败 | 使用了 2 空格缩进 | 改为 4 空格缩进 |
| 节点文本显示引号 | 用了 `A["文本"]` | 改为 `A[文本]`，去掉双引号 |
| `unsupported doc type: Board` | 对 Board 类型文档调用了 resource_detail | 只在 Doc 类型文档中操作画板 |
| 旧画板没被替换 | resource_create 是追加不是覆盖 | 先删除文档中的旧画板引用，再创建新画板 |
| 更新无效 | 用了 `dsl` 参数 | 更新时用 `text` 参数，不是 `dsl` |
| mindmap 节点丢失 | 缩进混用 tab 和空格 | 统一使用 4 个空格字符 |
