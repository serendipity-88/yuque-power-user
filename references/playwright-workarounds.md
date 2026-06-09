# Playwright 补充方案

用于解决语雀 MCP/CLI 无法完成的操作。

**适用场景**：
1. 表格全宽展示（标宽→全宽，解决容器裁切）
2. 表格列宽拖拽精调
3. 图片上传的备选方案（当 CLI 不可用时）

---

## 列宽控制方案选择

```
创建表格时就知道列宽比例？
├── 是 → API 写 <colgroup>（比例写入 body，建议总和 ≤750px）
│       └── 列多空间紧或总和 >750px？→ 追加 Playwright 开全宽展示
└── 否 → 创建后 Playwright 拖拽 + 全宽展示

两种方案可组合：colgroup 设初始比例 → 标宽下拖拽精调 → 全宽展示扩展总宽
colgroup 详见 html-table-advanced.md
```

**关键关系**：
- colgroup 控制列宽**比例**（API 可写，建议总和 ≤750px；超 750px 不会被剥离，但标准宽度下容器裁切溢出部分）
- 全宽展示控制表格**总宽**（让容器不再裁切，表格按 colgroup 比例自然展开；若同时开自适应宽度则表格撑满页面宽度）
- 全宽展示与自适应宽度**独立**，不是前置关系；全宽展示可直接点击工具栏第一个无名 icon
- 拖拽精调在标宽下操作更可控，建议**先拖后全宽**

---

## 一、表格全宽展示

> ✅ **2026-06-05 实测验证：全宽展示可持久化**

### 核心原理

- 全宽展示是语雀编辑器的 UI toggle，让容器不再裁切表格，表格按 colgroup 比例自然展开
- 全宽展示与「自适应宽度」**独立**，不是前置关系。全宽展示可直接点击工具栏第一个无名 icon（`c-tb-width-mode-unfold/fold`）
- 自适应宽度的作用：让表格列宽自动适应内容，开启后全宽展示下表格会撑满页面宽度（而非保持 colgroup 比例）
- 全宽展示状态**不反映在 YMD/MD 读回中**，无法通过 API 判断当前状态

### 操作步骤

```javascript
async (page) => {
  // 1. 进入编辑模式
  await page.locator('button:has-text("编辑")').click();
  await page.waitForTimeout(2000);

  // 2. 点击目标表格激活工具栏（多表格时用 nth(i) 精确定位）
  const table = page.locator('table.ne-table').nth(0); // 第1个表格
  await table.locator('td').first().click();
  await page.waitForTimeout(800);

  // 3. 点击全宽展示（工具栏第一个无名 icon，c-tb-width-mode-unfold/fold）
  const widthMode = page.locator('[data-testid="ne-card-toolbar-item-widthMode"]');
  await widthMode.waitFor({ state: 'visible', timeout: 3000 });
  const isFullWidth = await widthMode.evaluate(el => {
    const icon = el.querySelector('.ne-icon-c-tb-width-mode-fold');
    return !!icon; // fold = 已全宽，unfold = 标宽
  });
  if (!isFullWidth) {
    await widthMode.click();
    await page.waitForTimeout(1000);
  }

  // 4. 保存
  await page.locator('button:has-text("更新")').click();
  await page.waitForTimeout(2000);
};
```

### 多表格批量操作

```javascript
async (page) => {
  await page.locator('button:has-text("编辑")').click();
  await page.waitForTimeout(2000);

  const tables = await page.locator('table.ne-table').all();
  for (let i = 0; i < tables.length; i++) {
    const table = tables[i];
    await table.locator('td').first().click();
    await page.waitForTimeout(800);

    // 开全宽（检查状态避免误关）
    const widthMode = page.locator('[data-testid="ne-card-toolbar-item-widthMode"]');
    await widthMode.waitFor({ state: 'visible', timeout: 3000 });
    const isFullWidth = await widthMode.evaluate(el => !!el.querySelector('.ne-icon-c-tb-width-mode-fold'));
    if (!isFullWidth) {
      await widthMode.click();
      await page.waitForTimeout(1000);
    }
  }

  await page.locator('button:has-text("更新")').click();
  await page.waitForTimeout(2000);
};
```

---

## 二、表格列宽拖拽精调

> ✅ **2026-06-05 实测验证：拖拽可持久化**
> ⚠️ **建议在标宽（750px）下拖拽，再开全宽。全宽模式下拖拽可能失效。**

### 方向指引

根据每列内容类型判断目标宽度，标签列窄、内容列宽。参考示例：
- 3列 [策略, 进展, 效果]：~100/350/300
- 4列 [项目, 方案, 进展, 效果]：~80/200/270/200

### 拖拽代码

```javascript
async (page) => {
  // 1. 进入编辑模式
  await page.locator('button:has-text("编辑")').click();
  await page.waitForTimeout(2000);

  // 2. 点击目标表格
  const table = page.locator('table.ne-table').nth(0);
  await table.locator('td').first().click();
  await page.waitForTimeout(800);

  // 3. 开启自适应宽度（让列宽按比例分配，拖拽更可控；非全宽展示前置条件）
  const colAdapt = page.locator('[data-testid="ne-card-toolbar-item-columnAdaptation"]');
  await colAdapt.waitFor({ state: 'visible', timeout: 5000 });
  const isOn = await colAdapt.evaluate(el => el.classList.contains('selected'));
  if (!isOn) {
    await colAdapt.click();
    await page.waitForTimeout(500);
  }

  // 4. 测量当前列宽
  const colWidths = await table.evaluate(t => {
    const row = t.querySelector('tr');
    return Array.from(row.querySelectorAll('td')).map(td => Math.round(td.getBoundingClientRect().width));
  });

  // 5. 根据内容判断目标宽度（AI 灵活调整，不套公式）
  // 示例：标签列缩窄，内容列拉宽
  const targetWidths = [100, 350, 300]; // 根据实际列数和内容设定

  // 6. 从左到右依次拖拽（跳过最后一列，自动填充）
  for (let col = 0; col < colWidths.length - 1; col++) {
    const delta = targetWidths[col] - colWidths[col];
    if (Math.abs(delta) < 10) continue;

    const cell = await table.locator('tr td').nth(col);
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
    for (let s = 1; s <= steps; s++) {
      await page.mouse.move(startX + (delta * s / steps), startY);
      await page.waitForTimeout(15);
    }
    await page.mouse.up();
    await page.waitForTimeout(300);
  }

  // 7. 保存
  await page.locator('button:has-text("更新")').click();
  await page.waitForTimeout(2000);
};
```

---

## 三、图片上传（备选方案）

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

## ✅ 已验证可用的方案（2026-06-09 更新）

| 方案 | 用途 | 限制 |
|------|------|------|
| `<colgroup><col width="N" />` | API 设置列宽比例 | 建议 ≤750px；超 750px 不剥离但标准宽度下容器裁切；format=md 读回丢失 |
| Playwright 全宽展示 | 让容器不再裁切表格，按 colgroup 比例自然展开 | 工具栏第一个无名 icon（`c-tb-width-mode-unfold/fold`） |
| Playwright 自适应宽度 toggle | 让列宽自动适应内容，全宽时表格撑满页面 | 独立于全宽展示，非前置关系 |
| Playwright 列宽拖拽 | 单列精调 | 建议标宽下操作，全宽后拖拽可能失效 |

## ❌ 已验证不可用的方案（2026-06-09 更新）

| 方案 | 结果 | 原因 |
|------|------|------|
| `<td width="N">` | ❌ | 语雀报错 Unsupported attribute |
| `<td style="width:N">` | ❌ | 语雀报错 Unsupported attribute |
| JS 修改 DOM/colgroup 列宽 | ❌ | 临时生效但保存/刷新后被编辑器重置 |
| MCP API 设置列宽 | ❌ | 无相关 API |
| `<cardlink>` 标签 | ❌ | 有毒标签，破坏后续内容 |
| `<mention>` 标签 | ❌ | 有毒标签，破坏后续内容 |
| `<todo>` 标签 | ❌ | 被静默消除 |

---

## 四、重要约束

- **全宽展示与自适应宽度独立**：全宽展示可直接点击，不需要先开自适应宽度。自适应宽度让列宽适应内容（全宽时表格撑满页面），全宽展示让容器不再裁切
- **全宽展示只有两档**：标宽（750px 容器）和全宽（容器随页面宽度展开），没有中间宽度
- **全宽展示状态不可读**：YMD/MD 读回不反映全宽状态，Playwright 操作前必须检查当前状态避免误关
- **必须 hover 到单元格/表头**才能激活 `.ne-ui-table-resize-right` 拖拽手柄
- 拖拽需要用**真实的鼠标事件**（`page.mouse.move/down/up`），`dispatchEvent` 无效
- 移动要平滑（分步），一次性跳转会失败
- **colgroup 与全宽展示独立**：colgroup 控制比例，全宽展示控制总宽，两者可自由组合
- **建议操作顺序**：colgroup 写比例 → 标宽下拖拽精调 → 开全宽展示

### 表格工具栏 5 个概念详解（2026-06-10 实测更新）

#### 1. 自适应宽度（columnAdaptation）— 模式开关

| 属性 | 值 |
|------|-----|
| testId | `ne-card-toolbar-item-columnAdaptation` |
| 类型 | **Toggle 开关**（selected/unselected） |
| 默认 | 编辑器手动创建的表格默认 ON；**MCP API 创建的表格默认 OFF** |
| 作用 | 启用后，列宽可按比例分配；全宽展示时表格会撑满页面宽度 |
| 关闭后 | 表格列宽固定像素值；全宽展示时表格保持 colgroup 自然宽度不撑满 |

#### 2. 列等宽（equallyColumn）— 一次性动作

| 属性 | 值 |
|------|-----|
| testId | `ne-card-toolbar-item-equallyColumn` |
| 类型 | **一次性动作**（点击后按钮消失） |
| 可见条件 | 列宽不相等时才出现；列宽已等分时隐藏 |

#### 3. 全宽展示 / 标宽展示（widthMode）— 双态切换

| 属性 | 值 |
|------|-----|
| 位置 | 工具栏第一个无名 icon（`c-tb-width-mode-unfold/fold`） |
| 类型 | **双态 Toggle** |
| 前置条件 | **无**——全宽展示与自适应宽度独立，可直接点击 |
| 标宽→全宽 | 图标 `ne-icon-c-tb-width-mode-unfold`，点击后容器不再裁切表格 |
| 全宽→标宽 | 图标 `ne-icon-c-tb-width-mode-fold`，点击后容器恢复 750px 裁切 |

> **全宽展示的效果取决于自适应宽度状态**：
> - 自适应宽度 OFF + 全宽展示 ON → 表格保持 colgroup 自然宽度（如 800px），容器不裁切
> - 自适应宽度 ON + 全宽展示 ON → 表格撑满页面宽度（取决于视口宽度），列宽按比例自适应

#### 4. 全屏（maximize）— 临时编辑模式

| 属性 | 值 |
|------|-----|
| testId | `ne-card-toolbar-item-maximize` |
| 类型 | **临时模式**（Escape 退出） |
| 持久化 | ❌ 不持久——退出后表格恢复原始宽度 |

> **全屏 ≠ 全宽展示**：全屏是临时编辑模式，不影响保存后的表格宽度。

#### 5. 概念关系总结

```
全宽展示 / 标宽展示 (独立切换)
├── 控制容器是否裁切表格
├── 与自适应宽度独立，可直接操作
└── 效果受自适应宽度影响：
    ├── 自适应 OFF + 全宽 → 表格保持 colgroup 自然宽度
    └── 自适应 ON + 全宽 → 表格撑满页面宽度

自适应宽度 (独立切换)
├── 控制列宽是否按内容自适应
└── 影响全宽展示下表格的最终宽度

列等宽 — 一次性动作，不等宽时出现
全屏 — 独立功能，临时编辑模式，不影响持久化布局
拖拽手柄 — 手动精调单列宽度，建议标宽下操作
colgroup — API 层控制列宽比例，与工具栏功能独立
```

| 操作 | 改变表格总宽 | 改变列宽比例 | 持久化 | 模式依赖 |
|------|:---:|:---:|:---:|------|
| colgroup (API) | ❌ 只变比例不改总宽 | ✅ 自定义 | ✅ YMD 读写 | 无 |
| 自适应宽度 ON/OFF | ❌ | ❌ | ✅ | 无 |
| 列等宽 | ❌ | ✅ 均分 | ✅ | 需列宽不等 |
| 全宽展示 | ✅ 容器不裁切 | ❌ 保持比例 | ✅ | 无 |
| 标宽展示 | ✅ 容器恢复裁切 | ❌ 保持比例 | ✅ | 无 |
| 全屏 | ✅ 临时占满视口 | ❌ | ❌ | 无 |
| 拖拽手柄 | ❌ | ✅ 精调单列 | ✅ | 无（建议标宽下操作） |