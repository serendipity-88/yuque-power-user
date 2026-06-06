# 语雀 CLI 速查指南（`yuque` 命令）

## 快速模板

**创建文档并上传图片：**
```bash
cd <文档所在目录>
yuque create doc \
  --namespace <your-namespace> \
  --title "文档标题" \
  --body-file ./doc.md \
  --upload-images \
  --yes
```

**更新文档并上传图片：**
```bash
cd <文档所在目录>
yuque update doc <your-namespace>/<slug> \
  --body-file ./doc.md \
  --upload-images \
  --yes
```

**预览模式（不执行）：**
```bash
yuque create doc \
  --namespace <your-namespace> \
  --title "文档标题" \
  --body-file ./doc.md \
  --upload-images \
  --plan
```

---

## 安装与配置

```bash
# 安装
npm i -g @antcli/yuque-ant-cli

# 验证安装
yuque --version

# 首次登录 / 验证身份
yuque whoami --json
```

---

## 参数说明

| 参数 | 说明 |
|------|------|
| `--upload-images` | 启用图片上传，自动将 Markdown 中引用的本地图片上传到语雀 |
| `--yes` | 跳过确认提示，直接执行 |
| `--plan` | 预览模式，显示将要执行的操作但不实际执行 |
| `--body-file` | 指定 Markdown 文件路径作为文档正文 |
| `--json` | 以 JSON 格式输出结果 |
| `--namespace` | 知识库 namespace，格式如 `user/book-slug` |
| `--title` | 文档标题 |

---

## 图片上传详细说明

### 前置条件

1. Markdown 文件和图片在**同一目录**（或可通过相对路径访问）
2. 图片使用**相对路径**引用：`![描述](./images/xxx.png)`
3. CLI 根据 `--body-file` 的位置解析相对路径

### 工作原理

```
项目目录/
  ├── doc.md              ← --body-file 指向这里
  └── images/
      ├── screenshot.png  ← doc.md 中引用 ![](./images/screenshot.png)
      └── diagram.png
```

CLI 执行时会：
1. 解析 `doc.md` 中的图片引用
2. 找到本地图片文件
3. 上传到语雀图床
4. 将 Markdown 中的本地路径替换为语雀图床 URL
5. 创建/更新文档

### 典型工作流

```bash
# 1. 准备目录
mkdir -p /tmp/yuque-doc/images

# 2. 将图片复制到目录
cp ~/screenshots/*.png /tmp/yuque-doc/images/

# 3. 编写 Markdown（引用图片用相对路径）
cat > /tmp/yuque-doc/doc.md << 'EOF'
# 项目周报

## 数据概览
![数据趋势](./images/trend.png)

## 架构图
![系统架构](./images/architecture.png)
EOF

# 4. 上传
cd /tmp/yuque-doc
yuque create doc \
  --namespace <your-namespace> \
  --title "项目周报" \
  --body-file ./doc.md \
  --upload-images \
  --yes
```

---

## 限制与注意事项

### 全文替换

`yuque update doc` 是全文替换，会覆盖文档全部内容——和 `skylark_doc_update` 一样。更新前必须先读取完整内容，修改后整体写回。详见 mcp-api-guide.md 陷阱 #1。

### 图片上传限制
- **仅支持 Doc 类型文档**，Sheet / HtmlDoc 不支持
- 必须配合 `--body-file` 使用，不能通过 `--body` 内联参数上传图片
- **MCP API 无图片上传能力**，这是 CLI 的独占功能

### CLI vs MCP API 能力对比

| 能力 | CLI (`yuque`) | MCP API (`skylark_*`) |
|------|:---:|:---:|
| 创建/更新文档 | 支持 | 支持 |
| 图片上传 | 支持 | 不支持 |
| 画板操作 | 不支持 | 支持 |
| 文档搜索 | 不支持 | 支持 |
| 目录管理 | 不支持 | 支持 |
| 批量操作 | 适合 | 适合 |
| 交互式编辑 | 不适合 | 适合 |

### 最佳实践
- 纯文本文档：直接用 MCP API，更灵活
- 含图片文档：用 CLI 创建/更新，利用 `--upload-images`
- 先用 `--plan` 预览确认无误后再执行

---

## 常见错误

### 1. 图片路径解析失败

**错误原因：** 没有 `cd` 到 Markdown 文件所在目录，或图片用了绝对路径。

```bash
# 错误：在其他目录执行
yuque create doc --body-file /tmp/doc/doc.md --upload-images
# 图片 ./images/xxx.png 会从当前目录找，而不是 /tmp/doc/

# 正确：先 cd 到文档目录
cd /tmp/doc
yuque create doc --body-file ./doc.md --upload-images
```

### 2. 不带 --body-file 使用 --upload-images

```bash
# 错误：--body 内联内容不支持图片上传
yuque create doc --body "![](./img.png)" --upload-images

# 正确：必须用 --body-file
yuque create doc --body-file ./doc.md --upload-images
```

### 3. namespace 格式错误

```bash
# 错误：用 book_id
yuque create doc --namespace 12345678

# 正确：用 namespace 字符串
yuque create doc --namespace <your-namespace>
```

### 4. 更新时忘记指定 slug

```bash
# 错误：只传 namespace
yuque update doc <your-namespace>

# 正确：namespace/slug 完整路径
yuque update doc <your-namespace>/<slug>
```
