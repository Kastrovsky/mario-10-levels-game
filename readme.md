# 二维平面超级马里奥 · 十关挑战 —— 技术摘要

## 1. 项目概述 

- **交付文件**：`mario_10levels.html`（单文件，内联全部 CSS/JS，约 37 KB）
- **运行方式**：本地双击用 Chrome/Edge 打开，完全离线运行，零外部资源（无 CDN / 外链字体 / 图片）
- **渲染**：Canvas 2D 原生 API（`fillRect` / `arc` / `fillText` / `createLinearGradient`），逻辑分辨率固定 `800x450`
- **玩法**：WASD 操控的横版动作闯关，10 个难度递增关卡，选关解锁、共享生命、计分、Boss 战

---

## 2. 架构

### 2.1 代码结构（单 HTML 内组织顺序）

1. **全局常量**：物理常量、玩家尺寸、场景/敌人常量（`GROUND_Y=400`、`WIN_X=780`、`MAX_LIVES=5` 等）
2. **关卡数据**：`const LEVELS = [...]`，10 个关卡对象
3. **全局状态**：`state / levelIndex / lives / score / unlocked / player / platforms / enemies / spikes` 等
4. **核心类**：
   - `class Platform` —— 静态砖块 + 移动平台（线性往复 / 正弦不规则轨迹）
   - `class Enemy` —— 地面巡逻蘑菇（walker）、飞行敌人（flyer）、Boss（多段血量）
   - `class Player` —— 玩家（移动、重力、下蹲、碰撞、死亡重生）
5. **游戏流程函数**：`loadLevel / startRun / update / render`
6. **主循环**：`gameLoop`（60FPS 节流 + 固定步长物理）
7. **输入**：`keydown / keyup`（仅 WASD + R）、`canvas click`（选关/结算按钮）

### 2.2 状态机

| 状态 | 触发条件 | 行为 |
| --- | --- | --- |
| `menu` | 启动 / 结算后返回 | 渲染选关界面，点击已解锁关卡开始 |
| `playing` | 点击关卡 / 过关动画结束 | 正常物理与渲染 |
| `levelClear` | `player.x > 780` | 冻结物理，显示「第 X 关通过！」1.5 秒后自动进入下一关 |
| `allClear` | 第 10 关通过动画结束 | 显示总分与「返回选关」 |
| `gameOver` | 生命归零 | 显示结算信息与「返回选关」 |

### 2.3 主循环与帧率控制

```js
function gameLoop(ts) {
  requestAnimationFrame(gameLoop);
  const elapsed = (ts - lastTime) / 1000;
  if (elapsed < STEP - 0.0005) return;      // 冗余帧直接丢弃（60FPS 上限）
  lastTime = ts;
  acc += Math.min(elapsed, MAX_FRAME_DELTA); // 防“时间爆炸”
  while (acc >= STEP) update(STEP);          // 固定步长 1/60 秒
  render();
}
```

关键点：
- `STEP = 1/60` 固定物理步长，保证不同刷新率下移动速度一致；
- 间隔不足 `STEP` 的 rAF 帧不更新也不重绘，降低独显功耗；
- 单帧最多累计 `MAX_FRAME_DELTA = 0.1s` 的物理，防止切后台后补算过多步。

### 2.4 输入（严格锁定）

- `W`：跳跃（仅地面状态、且为“新按下”时触发，长按不会二段跳）
- `A / D`：左右移动（按住持续加速，松开后地面摩擦减速）
- `S`：下蹲（碰撞箱高度 ×0.6、速度 ×0.5，隧道内不可站起）
- `R`：重开关卡（保留分数与生命）
- 鼠标点击：选关界面 / 结算按钮
- 全部处理键均 `preventDefault()`，其余键一律忽略；窗口失焦自动清空按键

---

## 3. 关卡数据（`LEVELS`）

每个关卡对象字段：

| 字段 | 说明 |
| --- | --- |
| `name / stars` | 关卡名与星级（选关界面显示） |
| `gravity / jump` | 逐关重力与跳跃初速度（px/帧² / px/帧） |
| `bg` | 天空渐变两色 |
| `ground` | 地面段 `{x, w}`，缺口即深坑（无地面段 = 全靠平台） |
| `platforms` | 固定单向砖块 `{x, y, w, h}`（可从下方穿过，上方可站立） |
| `moving` | 移动平台：线性 `{x,y,w,h,min,max,speed}`；正弦 `{mode:'sine', mid, range, vAmp, speed, phase}` |
| `pipes` | 红色实心管道 `{x, y, w, h}`（四面碰撞） |
| `ceilings` | 头顶实心障碍 `{x, y, w, h}`（强制 S 下蹲通过） |
| `spikes` | 尖刺 `{x, surfaceY, w, h}`（触碰即死，无视无敌） |
| `enemies` | 敌人：walker `{type, x, min, max, groundY, speed}`；flyer `{type, x, min, max, baseY, amp, speed}` |
| `boss` | 巨型 Boss `{x, y, w, h, min, max, speed, groundY, hp}`（hp=3，踩 3 次） |
| `spawnX` | 关卡起点 x |

**坐标/数值语义（务必遵守）**：
- 敌人与移动平台的 `min` = 左边缘下界，`max` = 右边缘上界（`x + w >= max` 时掉头/钳制）；
- flyer 在水平巡逻基础上做正弦上下浮动：`y = baseY + sin(t·0.08 + phase)·amp`；
- 尖刺 `y = surfaceY - h` 自动推出；
- 移动平台携带玩家：平台先更新并记录 `prevX/prevY`，玩家站立其上时叠加位移差。

### 十关难度配置一览

| 关 | 名称 | 重力/跳跃 | 地面 | 平台 | 移动平台 | 管道 | 天花板 | 尖刺 | 敌人 + Boss |
| --- | --- | --- | --- | --- | --- | --- | --- | --- | --- |
| 1 | 新手草原 | 0.55 / -12 | 完整 | 2 固定 | - | 1 | - | - | 1 walker |
| 2 | 轻松河畔 | 0.55 / -12 | 20px 提示坑 | 2 固定 | - | 1 | - | - | 1 walker |
| 3 | 双坑挑战 | 0.58 / -12.5 | 2×50px 坑 | 3 固定（坑中辅助） | - | 1 | - | - | 2 walker |
| 4 | 敌人谷 | 0.6 / -13 | 1 大坑 | 2 固定（偏高） | - | 2（窄通道） | - | - | 3 walker |
| 5 | 三连坑 | 0.6 / -13 | 3×60px 坑 | 3 固定 | - | 1 | - | - | walker + **flyer** |
| 6 | 垂直挑战 | 0.58 / -13.5 | 仅两端 | 5 固定（高低错落） | - | 2 | - | - | walker + flyer |
| 7 | 移动平台 | 0.5 / -14 | 中间一段 | 4 固定 + **2 线性移动** | 2 | 1 | - | - | walker + flyer |
| 8 | 精密操控 | 0.48 / -14.5 | 无（全靠平台） | 6 固定 + 2 移动 | 2 | 2 | **1（强制下蹲）** | - | 3 flyer |
| 9 | 综合试炼 | 0.5 / -14 | 起点+终点 | 5 固定 + **3 正弦移动** | 3 | 2 | 2 | 2 | 3 flyer（1 高速） |
| 10 | 最终城堡 | 0.45 / -15 | 仅起点 | 8 固定 + **4 高速移动** | 4 | 3 | - | 3 | walker + 3 flyer + **Boss(hp3)** |

难度递增覆盖：地面完整性（完整 → 深坑 → 无地面）、平台复杂度（固定 → 移动 → 不规则轨迹）、敌人密度与类型（walker → flyer → 高速 flyer → Boss）、跳跃精度（重力递减/跳跃初速递增）、障碍密度（管道/天花板/尖刺）。

---

## 4. 关键函数

### 4.1 关卡构建

- **`loadLevel(idx)`**：按关卡数据重建 `platforms / movingPlatforms / enemies / spikes`，设置 `G / JUMP`，创建玩家并定位出生点。
- **`startRun(idx)`**：`lives = 5`、`score = 0` 后加载关卡并进入 `playing`（选关开始新一轮）。
- **`spawnYFor(level, x)`**：取包含出生点的地面段或平台顶计算站立高度，否则回退 `y=300`。
- **`saveUnlock()`**：通关解锁写入 `localStorage`（键 `mario10_unlocked_v1`，上限 10）。

### 4.2 帧更新 `update(dt)`

```js
if (state === 'levelClear') { clearTimer -= dt; ... 自动进下一关/全通关 }
if (state !== 'playing') return;
for (const mp of movingPlatforms) mp.update();  // 移动平台先动
player.update(dt, enemies, spikes);             // 玩家物理 + 碰撞
for (const enemy of enemies) enemy.update();    // 敌人巡逻
if (player.x > WIN_X) { score += 500; unlock; state = 'levelClear'; }
```

### 4.3 玩家物理 `Player.update`

1. 无敌倒计时、下蹲状态同步（`setCrouching`）；
2. 水平输入：目标速度（下蹲 ×0.5）→ 每帧 `ACCEL` 加速 / 地面 `FRICTION` 减速；
3. 重力：`this.vy += G`（逐关数值），软性终端速度 `MAX_FALL_SPEED = 18` 防穿透；
4. 分轴移动：先 `checkCollision(true)`（X 轴），再 `checkCollision(false)`（Y 轴）；
5. 被移动平台携带（叠加平台帧位移差）；
6. 敌人碰撞（踩踏/侧面）、尖刺、坠落（`y > 480`）判定。

### 4.4 碰撞响应 `Player.checkCollision(isHorizontal)`

- **X 轴（横向）**：遍历全部实心障碍，取**位移最小的弹回点**（解决管道与天花板并排时的来回推挤），弹回边缘并 `vx = 0`；
- **Y 轴（竖向）**：
  - 单向砖块：仅在下落且上一帧脚底未越过砖顶时托住（`prevBottom <= p.y`）—— 可穿下不可穿上；
  - 实心平台：落地托住（`y = p.y - 玩家高`）或上升顶头压回；
  - 落点坐标均做**修正**，而非只置 `onGround`。

### 4.5 敌人伤害判定（严格规则）

```js
inStompWindow = bottom >= centerY && bottom <= centerY + 10; // 脚底在中心点下方 10px 内
comingFromAbove = prevBottom <= enemy.centerY;               // 正从上方落下
if (vy > 0 && inStompWindow) { 踩踏：+100；Boss 扣血并弹起（0.45s 无敌防上升误杀）}
else if (invincibleTimer <= 0 && !(vy > 0 && comingFromAbove)) { 侧面碰撞：死亡 }
```

`comingFromAbove` 保护了「脚底穿过敌人顶边但尚未进入踩踏窗口」的帧，避免下落踩踏被误判为侧面碰撞。

### 4.6 下蹲与天花板

- `setCrouching(true)`：高度 32→19，脚底锚定；
- `setCrouching(false)`：先检测恢复后的碰撞箱是否与实心平台重叠，重叠则**保持下蹲**（隧道内不可站起），腾出空间后自动恢复；
- 天花板（`ceilings`，实心）同时参与横向阻挡：站立时被 x-pass 挡在入口，下蹲后通过。

### 4.7 渲染 `render`

- `renderMenu()`：标题 + 10 个关卡按钮（已解锁橙色 / 未解锁灰色锁），命中区域写入 `menuButtons` 供点击测试；
- `renderGame()`：天空渐变 → 平台（地面/管道/天花板/砖块/移动平台）→ 尖刺 → 敌人 → 玩家 → HUD（左上「第 X/10 关」+ 调试面板 FPS/坐标/velocityY，右上得分/生命）→ 过关遮罩；
- `drawCenterPanel()`：allClear / gameOver 共用结算面板与「返回选关」按钮（区域 `(300,250,200,46)`）。

---

## 5. 关键常量速查

| 常量 | 值 | 说明 |
| --- | --- | --- |
| `STEP` | 1/60 | 固定物理步长 |
| `MOVE_SPEED` | 4.5 | 水平最大速度 px/帧 |
| `FRICTION` | 0.85 | 地面摩擦系数 |
| `ACCEL` | 0.9 | 水平加速度 |
| `PLAYER_W/H` | 24 / 32 | 玩家碰撞箱 |
| `CROUCH_H_FACTOR` | 0.6 | 下蹲高度比例 |
| `GROUND_Y` | 400 | 地面顶边 |
| `WIN_X` | 780 | 通关判定线 |
| `MAX_LIVES` | 5 | 全关卡共享生命 |
| `INVINCIBLE_TIME` | 1.5s | 重生无敌时长 |
| `STOMP_SCORE` / `CLEAR_SCORE` | 100 / 500 | 踩敌与通关得分 |
| `MAX_FALL_SPEED` | 18 | 终端下落速度 |

---

## 6. 验证情况

开发期通过 Node + vm 桩环境运行 44 项自动化断言，全部通过：

- 10 个关卡的所有**必经跳跃可达性**（多速度 × 多时机暴力验证，含移动/正弦平台时机）；
- 关键机制：Boss 三踩与弹起、下蹲隧道进出、尖刺即死、飞行敌人站位安全（不重叠可站立平台）、死亡重生、过关动画 1.5s 自动推进、解锁持久化、游戏结束、选关锁定、R 重开；
- JS 语法检查通过；外部网络引用 0 处。

开发中修复的典型问题：下落踩踏被“窗口前重叠”误判致死（`comingFromAbove` 保护）、Boss 巡逻边界语义导致第一帧弹飞、移动平台载人撞尖刺、多障碍并排横向推挤抖动（最小位移弹回）。
