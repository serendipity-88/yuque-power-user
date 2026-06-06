# Playwright 补充方案

用于解决语雀 MCP/CLI 无法完成的操作。

**适用场景**：
1. 表格列宽调整（拖拽 + 全宽展示）
2. 图片上传的备选方案（当 CLI 不可用时）

---

## 一、表格列宽调整

> ✅ **2026-06-05 实测验证：Playwright 拖拽可设置列宽，全宽展示可扩展表格，均已验证持久化**

### 核心原理

- 列宽信息存储在语雀内部格式，**与 Markdown body 分离**
- 通过 `skylark_doc_detail` 读取的 markdown 表格仍为 `| --- | --- |`，**不会反映**手动设置的列宽
- 拖拽设置的列宽在文档刷新后保持

---

### 列宽分配规则（量化决策）

拖拽前先**分析每列内容类型**，按权重计算目标宽度：

#### 内容类型权重表

| 内容类型 | 权重 | 典型内容 | 说明 |
|---------|:---:|---------|------|
| 标签/分类 | 1 | 策略名、状态、人名、日期 | 短文本，1-5 个字 |
| 简短描述 | 2 | 一句话说明、指标值 | 1-2 行文字 |
| 列表/详情 | 3 | 进展列表（3+ bullet）、多段文字 | 块级内容，需要展开空间 |
| 富媒体 | 2 | 截图、页面引用卡片、链接 | 图片/卡片有固定最小宽度 |

#### 计算方法

```
1. 给每列标注内容类型和权重
2. 目标宽度 = 表格总宽 × (该列权重 / 所有列权重之和)
3. 拖拽距离 = 目标宽度 - 当前宽度（均分值）
```

#### 实际举例

以周报常见的 4 列表格为例（总宽 750px，均分=187.5px/列）：

| 列 | 内容类型 | 权重 | 目标宽度 | vs均分 | 操作 |
|----|---------|:---:|:-------:|:-----:|------|
| 策略 | 标签 | 1 | 83px | -104px | 需缩窄 |
| 进展 | 列表 | 3 | 250px | +63px | 需拉宽 |
| 计划 | 列表 | 3 | 250px | +63px | 需拉宽 |
| 截图&参考 | 富媒体 | 2 | 167px | -21px | 略缩 |

> **简化原则**：标签列缩到刚好放下文字即可（~80-100px），省出的空间全给列表列。不需要精确到像素——拖拽后视觉上"内容不折行"就够了。

#### 拖拽顺序

**从左到右依次拖**（因为拖右边界只影响当前列和右邻列）：
1. 先拖第1列右边界：向左缩窄标签列
2. 再拖第2列右边界：调整列表列和后续列的分界
3. 最后一列宽度自动由剩余空间决定

---

### 方式一：拖拽调整指定列宽

**适用场景**：用户说"把效果列拉宽"、"让第三列更宽"

```javascript
async (page) => {
  // 1. 进入编辑模式
  await page.locator('button:has-text("编辑")').click();
  await page.waitForTimeout(2000);

  // 2. 先测量当前列宽
  const colWidths = await page.evaluate(() => {
    const row = document.querySelector('table.ne-table tr');
    return Array.from(row.querySelectorAll('td')).map(td => 
      Math.round(td.getBoundingClientRect().width)
    );
  });
  const totalWidth = colWidths.reduce((a, b) => a + b, 0);

  // 3. 按内容类型分配权重（根据实际表格修改）
  //    标签=1, 描述=2, 列表=3, 富媒体=2
  const weights = [1, 3, 3, 2]; // 策略/进展/计划/截图
  const totalWeight = weights.reduce((a, b) => a + b, 0);
  const targetWidths = weights.map(w => Math.round(totalWidth * w / totalWeight));

  // 4. 从左到右依次拖拽（跳过最后一列，它自动填充）
  for (let col = 0; col < colWidths.length - 1; col++) {
    const delta = targetWidths[col] - colWidths[col];
    if (Math.abs(delta) < 10) continue; // 差距太小不拖

    // Hover 到该列单元格右边界
    const cell = await page.$(`table.ne-table tr td:nth-child(${col + 1})`);
    const cellBox = await cell.boundingBox();
    await page.mouse.move(cellBox.x + cellBox.width - 2, cellBox.y + cellBox.height / 2);
    await page.waitForTimeout(500);

    const handle = await page.$('.ne-ui-table-resize-right');
    if (!handle) continue;
    const box = await handle.boundingBox();
    const startX = box.x + box.width / 2;
    const startY = box.y + box.height / 2;

    await page.mouse.move(startX, startY);
    await page.mouse.down();
    const steps = Math.max(10, Math.abs(Math.round(delta / 10)));
    for (let i = 1; i <= steps; i++) {
      await page.mouse.move(startX + (delta * i / steps), startY);
      await page.waitForTimeout(15);
    }
    await page.mouse.up();
    await page.waitForTimeout(300);
  }

  // 5. 保存
  await page.locator('button:has-text("更新")').click();
  await page.waitForTimeout(2000);
};
```

---

### 方式二：全宽展示（表格撑满页面宽度）

**适用场景**：用户说"表格太窄了"、"让表格全宽展示"

```javascript
async (page) => {
  // 1. 进入编辑模式
  await page.locator('button:has-text("编辑")').click();
  await page.waitForTimeout(2000);

  // 2. 点击表格激活工具栏
  await page.locator('table.ne-table td').first().click();
  await page.waitForTimeout(800);

  // 3. 确保「自适应宽度」开启（API 创建的表格默认 OFF）
  const needEnable = await page.evaluate(() => {
    const btn = document.querySelector('[data-testid="ne-card-toolbar-item-columnAdaptation"]');
    return btn && !btn.classList.contains('selected');
  });
  if (needEnable) {
    await page.locator('[data-testid="ne-card-toolbar-item-columnAdaptation"]').click();
    await page.waitForTimeout(500);
  }

  // 4. 点击"全宽展示"按钮（自适应 ON 后才可见）
  await page.locator('[data-testid="ne-card-toolbar-item-widthMode"]').click();
  await page.waitForTimeout(1000);

  // 5. 保存
  await page.locator('button:has-text("更新")').click();
};
```

---

### 方式三：按权重调列宽 + 全宽展示（推荐）

**适用场景**：周报表格完整优化——先按内容类型分配列宽比例，再全宽展示

**完整流程**：
1. 进入编辑模式 → 点击表格激活工具栏
2. 确保「自适应宽度」ON（API 创建的表格默认 OFF）
3. 测量当前列宽 → 按权重算目标值 → 从左到右拖拽
4. 点击「全宽展示」→ 保存

```javascript
async (page) => {
  // 1. 进入编辑模式
  await page.locator('button:has-text("编辑")').click();
  await page.waitForTimeout(2000);

  // 2. 点击表格激活工具栏
  await page.locator('table.ne-table td').first().click();
  await page.waitForTimeout(800);

  // 3. 确保「自适应宽度」开启（API 创建的表格默认 OFF！）
  const needEnable = await page.evaluate(() => {
    const btn = document.querySelector('[data-testid="ne-card-toolbar-item-columnAdaptation"]');
    return btn && !btn.classList.contains('selected');
  });
  if (needEnable) {
    await page.locator('[data-testid="ne-card-toolbar-item-columnAdaptation"]').click();
    await page.waitForTimeout(500);
  }

  // 4. 测量当前列宽
  const colWidths = await page.evaluate(() => {
    const row = document.querySelector('table.ne-table tr');
    return Array.from(row.querySelectorAll('td')).map(td =>
      Math.round(td.getBoundingClientRect().width)
    );
  });
  const totalWidth = colWidths.reduce((a, b) => a + b, 0);

  // 5. 按内容类型分配权重（根据实际列内容调整）
  //    标签=1, 描述=2, 列表=3, 富媒体=2
  const weights = [1, 2, 3, 2]; // 例：策略/说明/进展/效果
  const totalWeight = weights.reduce((a, b) => a + b, 0);
  const targetWidths = weights.map(w => Math.round(totalWidth * w / totalWeight));

  // 6. 从左到右依次拖拽（跳过最后一列，它自动填充）
  for (let col = 0; col < colWidths.length - 1; col++) {
    const delta = targetWidths[col] - colWidths[col];
    if (Math.abs(delta) < 10) continue;

    const cell = await page.$(`table.ne-table tr td:nth-child(${col + 1})`);
    const cellBox = await cell.boundingBox();
    await page.mouse.move(cellBox.x + cellBox.width - 2, cellBox.y + cellBox.height / 2);
    await page.waitForTimeout(500);

    const handle = await page.$('.ne-ui-table-resize-right');
    if (!handle) continue;
    const box = await handle.boundingBox();
    const startX = box.x + box.width / 2;
    const startY = box.y + box.height / 2;

    await page.mouse.move(startX, startY);
    await page.mouse.down();
    const steps = Math.max(10, Math.abs(Math.round(delta / 10)));
    for (let i = 1; i <= steps; i++) {
      await page.mouse.move(startX + (delta * i / steps), startY);
      await page.waitForTimeout(15);
    }
    await page.mouse.up();
    await page.waitForTimeout(300);
  }

  // 7. 重新激活工具栏，点击全宽展示
  await page.locator('table.ne-table td').first().click();
  await page.waitForTimeout(500);
  await page.locator('[data-testid="ne-card-toolbar-item-widthMode"]').click();
  await page.waitForTimeout(500);

  // 8. 保存
  await page.locator('button:has-text("更新")').click();
};
```

---

### 关键 DOM 元素

| 元素 | 选择器 | 说明 |
|------|--------|------|
| 列宽拖拽手柄 | `.ne-ui-table-resize-right` | 列右侧边界，hover 表头后可见 |
| 全宽展示按钮 | `[data-testid="ne-card-toolbar-item-widthMode"]` | 双态切换：全宽↔标宽，仅自适应ON时可见 |
| 自适应宽度按钮 | `[data-testid="ne-card-toolbar-item-columnAdaptation"]` | 模式开关，控制全宽展示按钮可见性 |
| 全屏编辑按钮 | `[data-testid="ne-card-toolbar-item-maximize"]` | 临时全屏编辑≠全宽展示 |
| 列等宽按钮 | `[data-testid="ne-card-toolbar-item-equallyColumn"]` | 一次性均分列宽，列宽不等时才出现 |
| 更新/保存按钮 | `button:has-text("更新")` | 编辑模式保存 |
| 表格单元格 | `table.ne-table td` | 表格单元格定位 |
| 表头单元格 | `ne-text:text("列标题名")` | 用标题文字定位列 |

### 表格工具栏 5 个概念详解（2026-06-05 实测验证）

点击表格后，显示 `ne-card-toolbar` 工具栏。以下 5 个概念是**完整的表格宽度控制体系**：

#### 1. 自适应宽度（columnAdaptation）— 模式开关

| 属性 | 值 |
|------|-----|
| testId | `ne-card-toolbar-item-columnAdaptation` |
| 类型 | **Toggle 开关**（selected/unselected） |
| 默认 | 编辑器手动创建的表格默认 ON；**MCP API 创建的表格默认 OFF** |
| 作用 | 启用后，表格在「自适应」模式下运行——列宽可按内容比例分配，全宽展示按钮可用 |
| 关闭后 | 表格进入「固定宽度」模式——列宽锁定为当前像素值，**全宽展示按钮消失** |
| tooltip | 选中时显示"取消自适应"，未选中时显示"自适应宽度" |

> **关键发现**：开启自适应宽度**不会**立即重新分配手动拖拽过的列宽。它控制的是「模式」，不是一次性动作。

#### 2. 列等宽（equallyColumn）— 一次性动作

| 属性 | 值 |
|------|-----|
| testId | `ne-card-toolbar-item-equallyColumn` |
| 类型 | **一次性动作**（点击后按钮消失） |
| 可见条件 | 列宽不相等时才出现；列宽已等分时隐藏 |
| 作用 | 将总表格宽度**均分**给所有列（如 750px/4 = 187.5px） |
| 不影响 | 表格总宽度、自适应模式状态 |

#### 3. 全宽展示 / 标宽展示（widthMode）— 双态切换

| 属性 | 值 |
|------|-----|
| testId | `ne-card-toolbar-item-widthMode` |
| 类型 | **双态 Toggle**（同一个按钮切换两种状态） |
| 前置条件 | **仅当「自适应宽度」开启时可见** |

| 状态 | tooltip | 图标 class | 点击效果 |
|------|---------|-----------|---------|
| 标宽（默认） | "全宽展示" | `ne-icon-c-tb-width-mode-unfold` | 表格从标准宽(750px)扩展到页面宽(~1016px) |
| 全宽（已展开） | "标宽展示" | `ne-icon-c-tb-width-mode-fold` | 表格从页面宽缩回标准宽(750px) |

> **"标宽展示"不是独立按钮**——它是「全宽展示」按钮在全宽状态下的 tooltip 文字，表示"点击可回到标准宽度"。

#### 4. 全屏（maximize）— 临时编辑模式

| 属性 | 值 |
|------|-----|
| testId | `ne-card-toolbar-item-maximize` |
| 类型 | **临时模式**（Escape 退出） |
| 作用 | 打开全屏覆盖层，表格占满整个视口（~1399px），方便编辑大表格 |
| 退出 | 按 Escape 键 |
| 持久化 | ❌ 不持久——退出后表格恢复原始宽度 |

> **全屏 ≠ 全宽展示**：全屏是临时编辑模式，不影响文档保存后的表格宽度；全宽展示是持久化的布局设置。

#### 5. 概念关系总结

```
自适应宽度 (模式开关)
├── ON → 列宽按比例分配，以下子功能可用：
│   ├── 全宽展示/标宽展示 (双态切换) — 控制表格总宽度
│   └── 列等宽 (一次性动作) — 均分列宽（不等宽时才出现）
└── OFF → 列宽固定像素值，全宽展示按钮隐藏

全屏 — 独立功能，临时编辑模式，不影响持久化布局
拖拽手柄 — 手动精调单列宽度，任何模式下都可用
```

| 操作 | 改变表格总宽 | 改变列宽比例 | 持久化 | 模式依赖 |
|------|:---:|:---:|:---:|------|
| 自适应宽度 ON/OFF | ❌ | ❌ | ✅ | 无 |
| 列等宽 | ❌ | ✅ 均分 | ✅ | 需列宽不等 |
| 全宽展示 | ✅ 扩至页面宽 | ❌ 保持比例 | ✅ | 需自适应 ON |
| 标宽展示 | ✅ 缩回标准宽 | ❌ 保持比例 | ✅ | 需自适应 ON |
| 全屏 | ✅ 临时占满视口 | ❌ | ❌ | 无 |
| 拖拽手柄 | ❌ | ✅ 精调单列 | ✅ | 无 |

---

## 二、图片上传（备选方案）

> 推荐优先使用 CLI `--upload-images`（更简单可靠）。仅当 CLI 不可用时才考虑 Playwright。

```javascript
async (page) => {
  await page.locator('button:has-text("编辑")').click();
  await page.waitForTimeout(2000);
  await page.click('.ne-editor-body p');
  await page.click('[data-testid="toolbar-image"]');
  const [fileChooser] = await Promise.all([
    page.waitForEvent('filechooser'),
    page.click('上传图片按钮的选择器')
  ]);
  await fileChooser.setFiles('/path/to/image.png');
};
```

---

## 三、已验证不可用的方案

| 方案 | 结果 | 原因 |
|------|------|------|
| `<td width="N">` | ❌ | 语雀报错 Unsupported attribute |
| `<td style="width:N">` | ❌ | 语雀报错 Unsupported attribute |
| `<colgroup><col>` | ❌ | 报错 table only allows tr children |
| JS 修改 DOM/colgroup 列宽 | ❌ | 临时生效但保存/刷新后被编辑器重置 |
| MCP API 设置列宽 | ❌ | 无相关 API |
| `<cardlink>` 标签 | ❌ | 有毒标签，破坏后续内容 |
| `<mention>` 标签 | ❌ | 有毒标签，破坏后续内容 |
| `<todo>` 标签 | ❌ | 有毒标签，被静默消除 |

---

## 四、重要约束

- **API 创建的表格「自适应宽度」默认 OFF**——必须先开启才能点全宽展示，否则 widthMode 按钮不存在
- **必须 hover 到单元格/表头**才能激活 `.ne-ui-table-resize-right` 拖拽手柄
- 手柄需要在视口内可见，需调用 `scrollIntoViewIfNeeded`
- 拖拽需要用**真实的鼠标事件**（`page.mouse.move/down/up`），`dispatchEvent` 无效
- 移动要平滑（分步），一次性跳转会失败
- `widthMode` 按钮**仅在自适应宽度开启时可见**——关闭自适应宽度会隐藏全宽展示按钮
- 拖拽手柄调整列宽后，「列等宽」按钮会重新出现
