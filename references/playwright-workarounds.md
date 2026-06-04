# Playwright 补充方案

用于解决语雀 MCP/CLI 无法完成的操作。

**适用场景**：
1. 表格列宽调整（API/YMD 无法控制）
2. 图片上传的备选方案（当 CLI 不可用时）

---

## 一、表格列宽调整

> ✅ 已验证可行，列宽持久化

### 快速模板

```javascript
async (page) => {
  // 1. 进入编辑模式
  await page.click('button:has-text("编辑")');
  await page.waitForTimeout(3000);
  
  // 2. 点击表格单元格激活工具栏
  await page.click('table.ne-table td');
  await page.waitForTimeout(500);
  
  // 3. 开启全宽模式（"自适应宽度"按钮）
  await page.getByTestId('ne-card-toolbar-item-columnAdaptation').click();
  await page.waitForTimeout(500);
  
  // 4. 获取拖拽手柄位置
  const handle = await page.evaluate(() => {
    const h = document.querySelector('.ne-ui-table-resize-right');
    const rect = h.getBoundingClientRect();
    return { x: rect.x + rect.width / 2, y: rect.y + rect.height / 2 };
  });
  
  // 5. 拖动列分隔线（正数=右移拉宽，负数=左移缩窄）
  const dragOffset = 150;
  await page.mouse.move(handle.x, handle.y);
  await page.mouse.down();
  for (let i = 1; i <= 10; i++) {
    await page.mouse.move(handle.x + (dragOffset * i / 10), handle.y);
    await page.waitForTimeout(50);
  }
  await page.mouse.up();
  await page.waitForTimeout(1000); // 等待自动保存
};
```

### 完整步骤说明

1. 进入编辑模式（点击"编辑"按钮）
2. 点击表格内单元格，激活表格工具栏
3. 点击"自适应宽度"按钮开启全宽模式
4. 定位拖拽手柄 `.ne-ui-table-resize-right`
5. 拖动手柄调整列宽
6. 等待自动保存

### 关键说明

- 拖拽手柄 `.ne-ui-table-resize-right` 只在选中单元格后才出现
- 每次只能看到当前选中列的拖拽手柄
- 调整多列时需要依次点击不同列的单元格，重新获取手柄坐标
- ✅ 列宽变化在页面刷新后保持（已验证持久化）
- ✅ 全宽模式（`ne-full-width` class）刷新后保留

### "自适应宽度"按钮说明

| 属性 | 值 |
|------|-----|
| data-testid | `ne-card-toolbar-item-columnAdaptation` |
| 作用 | 开启全宽模式（表格 maxWidth 从 750px 变为 100%） |
| 注意 | 不会根据内容自动调整各列宽度，所有列仍然均分 |

---

## 二、图片上传（备选方案）

> 推荐优先使用 CLI `--upload-images`（更简单可靠）。仅当 CLI 不可用时才考虑 Playwright。

### 代码模板

```javascript
async (page) => {
  // 1. 进入编辑模式
  await page.click('button:has-text("编辑")');
  await page.waitForTimeout(2000);
  
  // 2. 定位到要插入图片的位置（点击段落）
  await page.click('.ne-editor-body p');
  
  // 3. 点击工具栏图片按钮
  await page.click('[data-testid="toolbar-image"]');
  
  // 4. 等待 file chooser 并上传
  const [fileChooser] = await Promise.all([
    page.waitForEvent('filechooser'),
    page.click('上传图片按钮的选择器')
  ]);
  await fileChooser.setFiles('/path/to/image.png');
};
```

> 复杂度较高，图片按钮选择器可能因语雀版本变化。

---

## 三、已验证不可用的方案（防止重复踩坑）

| 方案 | 结果 | 原因 |
|------|------|------|
| `<td width="N">` | ❌ | 语雀报错 Unsupported attribute |
| `<td style="width:N">` | ❌ | 语雀报错 Unsupported attribute |
| `<colgroup><col>` | ❌ | 报错 table only allows tr children |
| 不同长度内容自适应 | ❌ | 列宽始终均分 |
| JavaScript 修改 DOM 列宽 | ❌ | 临时生效但保存时被编辑器重置 |
| MCP API 设置列宽 | ❌ | 无相关 API |
