# snake.html 代码审查报告

> 审查日期：2026-03-12
> 文件：`snake.html`（共 691 行）

---

## 总体评价

整体结构清晰，游戏核心逻辑完整，渲染效果美观（棋盘背景、苹果绘制、晕头动画均有较高完成度）。但存在一个**运行期 Bug**、数处**代码质量问题**以及若干**改进空间**，详见下文。

---

## 一、严重 Bug

### 1.1 晕头动画期间按键可打断游戏流程

**位置：** `keydown` 事件处理器（第 625 行起）

**问题描述：**
`startDizzy()` 执行时将 `running` 设置为 `false`，但 `dizzy` 为 `true`，动画仍在 `requestAnimationFrame` 中播放。键盘事件仅判断 `!running && !paused`，此时该条件为真，按任意方向键会立即调用 `startGame()`，强制中断晕头动画、直接重置游戏。

**复现步骤：** 让蛇撞墙，碰撞后立刻按方向键。

**修复建议：**
在所有方向键分支中加入 `dizzy` 判断：

```javascript
if (!running && !paused) {
  if (dizzy) break; // 等待晕头动画结束
  startGame();
  break;
}
```

---

## 二、中等问题

### 2.1 `sortBy` 函数为死代码

**位置：** 第 162–170 行

**问题描述：**
定义了一个通用排序工具函数 `sortBy`，但在整个文件中从未被调用。属于无用代码，增加维护负担，应删除。

```javascript
// 通用排序工具：按 keyFn 返回值排序，descending=true 则降序，不修改原数组
function sortBy(arr, keyFn, descending = false) { ... }
```

---

### 2.2 `spawnFood` 存在无限循环风险

**位置：** 第 208–220 行

**问题描述：**
`do...while` 循环不断随机取格子，直到找到一个不被蛇体和食物占用的位置。当棋盘（20×20 = 400 格）接近被填满时，随机命中率趋近于零，循环将卡死页面主线程。

**修复建议：**
在循环前检查可用格子数量，若不足则跳过生成：

```javascript
function spawnFood() {
  const totalCells = COLS * ROWS;
  if (snake.length + foods.length >= totalCells) return; // 无空间则不生成
  let pos;
  do {
    pos = {
      x: Math.floor(Math.random() * COLS),
      y: Math.floor(Math.random() * ROWS),
    };
  } while (
    snake.some((s) => s.x === pos.x && s.y === pos.y) ||
    foods.some((f) => f.x === pos.x && f.y === pos.y)
  );
  foods.push(pos);
}
```

---

### 2.3 `gameOver` 中冗余的 `clearInterval`

**位置：** 第 605–613 行

**问题描述：**
`gameOver()` 由 `dizzyLoop` 在动画结束后调用，而 `startDizzy()` 中已经执行过 `clearInterval(gameLoop)`。`gameOver()` 内再次调用是冗余的，虽不会报错，但容易引发对执行顺序的误解。

---

### 2.4 `tick()` 是无意义的薄包装

**位置：** 第 259–261 行

```javascript
function tick() {
  update();
}
```

`tick` 仅仅转发给 `update`，`setInterval(tick, speed)` 可直接改为 `setInterval(update, speed)`，删除 `tick` 函数。

---

## 三、代码质量问题

### 3.1 鼻孔绘制逻辑四个分支中有两组完全重复

**位置：** 第 448–477 行

`dir.x === 1` 与 `dir.x === -1` 分支的鼻孔绘制代码完全相同；`dir.y === -1` 与 `else` 分支的鼻孔绘制代码也完全相同。可以合并为两个分支：

```javascript
if (dir.x !== 0) {
  // 左右移动时鼻孔上下排列
  ctx.arc(nx, ny - 2, 1.2, 0, Math.PI * 2);
  ctx.fill();
  ctx.arc(nx, ny + 2, 1.2, 0, Math.PI * 2);
  ctx.fill();
} else {
  // 上下移动时鼻孔左右排列
  ctx.arc(nx - 2, ny, 1.2, 0, Math.PI * 2);
  ctx.fill();
  ctx.arc(nx + 2, ny, 1.2, 0, Math.PI * 2);
  ctx.fill();
}
```

---

### 3.2 `drawSnakeBody` 中路径构建代码重复两遍

**位置：** 第 328–374 行

主体颜色描边和暗色边缘描边各自完整地写了一遍"从尾到头遍历并连线"的循环，可抽取为私有函数：

```javascript
function buildSnakePath() {
  ctx.beginPath();
  ctx.moveTo(
    snake[snake.length - 1].x * GRID + GRID / 2,
    snake[snake.length - 1].y * GRID + GRID / 2
  );
  for (let i = snake.length - 2; i >= 0; i--) {
    ctx.lineTo(snake[i].x * GRID + GRID / 2, snake[i].y * GRID + GRID / 2);
  }
}
```

---

### 3.3 变量声明风格不一致

**位置：** 第 172–187 行

顶部 `let` 块声明了 `highScore`，但随后第 187 行又单独写了 `highScore = 0`，而其他变量均通过 `initGame()` 初始化。`highScore` 应保持独立（不在 `initGame` 中重置是正确的），但初始值赋值应与声明合并：

```javascript
let highScore = 0; // 声明时直接初始化，删除第 187 行的单独赋值
```

---

## 四、功能缺失 / 改进建议

| 优先级 | 问题                     | 说明                                                                |
| ------ | ------------------------ | ------------------------------------------------------------------- |
| 中     | 最高分未持久化           | 页面刷新后高分归零，可用 `localStorage` 保存                        |
| 中     | 无移动端触摸支持         | 没有 `touchstart/touchend` 滑动手势，移动设备无法游玩               |
| 低     | 游戏结束界面不显示等级   | 结束时只显示得分，可补充"最高等级"信息                              |
| 低     | Canvas 缺少 `aria-label` | 无障碍性欠缺，建议添加 `aria-label="贪吃蛇游戏"`                    |
| 低     | 分数/等级跳变            | 高等级时每次得分为 `level * 10`，可能在同一局内连续跨级，体感略突兀 |

---

## 五、问题汇总

| 编号 | 类型     | 描述                            | 严重程度 |
| ---- | -------- | ------------------------------- | -------- |
| B-1  | Bug      | 晕头动画期间按键打断游戏        | 严重     |
| B-2  | Bug      | `spawnFood` 无限循环风险        | 中等     |
| Q-1  | 代码质量 | `sortBy` 死代码                 | 中等     |
| Q-2  | 代码质量 | `gameOver` 冗余 `clearInterval` | 轻微     |
| Q-3  | 代码质量 | `tick()` 无意义包装             | 轻微     |
| Q-4  | 代码质量 | 鼻孔绘制四分支两两重复          | 轻微     |
| Q-5  | 代码质量 | 蛇身路径构建逻辑重复            | 轻微     |
| Q-6  | 代码质量 | `highScore` 声明与赋值分离      | 轻微     |
| F-1  | 功能缺失 | 高分无持久化                    | 建议     |
| F-2  | 功能缺失 | 无移动端触摸支持                | 建议     |

---

## 六、必须修复项

优先修复以下两项，其余可视情况处理：

1. **B-1**：晕头动画期间加 `dizzy` 判断，防止按键打断动画
2. **B-2**：`spawnFood` 增加空格数量检查，防止卡死
