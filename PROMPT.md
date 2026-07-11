# F1 Grand Prix — F1 赛车游戏全栈实现

## 项目概述

一个基于 Canvas 的 F1 赛车游戏，支持双人对战、AI 对战、6 级难度、15 条真实 F1 赛道、DRS 加速系统、完全圈数统计等真实赛车要素。全部代码包含在单个 `index.html` 文件中。

- **技术栈**: 纯前端 HTML5 Canvas + JavaScript（无外部依赖）
- **核心玩法**: 方向盘控制、速度/档位模拟、圈数计时、DRS 加速、赛道边缘检测与重设
- **AI 系统**: Pure Pursuit + 曲率速度剖面 + 比例速度控制
- **赛道数据**: 15 条无交叉的真实 F1 赛道，基于 meijersa/f1-circuits GeoJSON 数据集生成

---

## 目录

1. [配置系统](#1-配置系统)
2. [赛道系统](#2-赛道系统)
3. [赛车物理](#3-赛车物理)
4. [玩家控制与 DRS](#4-玩家控制与-drs)
5. [AI 驾驶系统](#5-ai-驾驶系统)
6. [速度剖面预计算](#6-速度剖面预计算)
7. [发车与比赛流程](#7-发车与比赛流程)
8. [碰撞系统](#8-碰撞系统)
9. [尾流/滑流系统](#9-尾流滑流系统)
10. [DRS 减阻系统](#10-drs-减阻系统)
11. [渲染系统](#11-渲染系统)
12. [HUD 与 UI](#12-hud-与-ui)
13. [开发流程与演进历史](#13-开发流程与演进历史)

---

## 1. 配置系统

### 物理常量 (`CFG`)

```javascript
const CFG = {
  trackWidth: 220,    // 赛道宽度（像素）
  maxSpeed: 450,      // 最高速度
  accel: 220,         // 加速度
  brake: 380,         // 刹车减速度
  drag: 40,           // 空气阻力
  offTrackDrag: 500,  // 冲出赛道后的额外阻力
  turnLow: 3.0,       // 低速转向灵敏度
  turnHigh: 0.85,     // 高速转向灵敏度
};
```

### DRS 常量

```javascript
const DRS_BOOST = 45;              // DRS 速度加成
const DRS_DURATION = 3.0;          // DRS 持续时长（秒）
const DRS_COOLDOWN = 10.0;         // 正常冷却（秒）
const DRS_COOLDOWN_CLEAN = 5.0;    // 干净圈冷却（秒）
const DRS_STRAIGHT_THRESHOLD = 600; // 直道判定阈值（速度剖面值）
```

### 难度等级 (`DIFFICULTY`)

6 级难度等距梯度，通过 5 个参数调节 AI 性能：

| 等级 | speedMult | cornerSpeed | accelFactor | brakeFactor | errorScale |
|------|-----------|-------------|-------------|-------------|------------|
| 简单 | 0.90 | 0.62 (80%) | 0.80 | 0.6 | 1.5 |
| 中等 | 1.00 | 0.66 (85%) | 0.91 | 0.7 | 0.5 |
| 困难 | 1.10 | 0.69 (88%) | 1.02 | 0.8 | 0.2 |
| 超级困难 | 1.20 | 0.72 (92%) | 1.13 | 0.9 | 0.05 |
| 地狱级 | 1.30 | 0.75 (96%) | 1.24 | 1.0 | 0.01 |
| 首领 | 1.40 | 0.78 (100%) | 1.35 | 1.0 | 0.001 |

- `speedMult`: 最高速度倍率，AI 极速上限 `min(450 * speedMult, 600+boost)`
- `cornerSpeed`: 过弯速度系数（`cornerSpeed / 0.78` 为速度剖面缩放比，即过弯百分比）
- `accelFactor`: 加速效率倍率
- `brakeFactor` / `errorScale`: **当前版本未实装**（死代码，保留备用）

**注意**：当前版本 `errorScale`、`brakeFactor`、`errDrift` 三个参数已定义但未在 AI 代码中引用。AI 走线精度和刹车力度不受难度影响。

### 圈数选项

```javascript
const LAP_OPTIONS = [3, 5, 8]; // 可选圈数
```

### 游戏设置

```javascript
const gameSettings = {
  mode: 'single',   // 'single' | 'dual' — 单人/双人
  aiCount: 1,       // 0 | 1 | 2 — AI 对手数量
};
```

---

## 2. 赛道系统

### 赛道数据来源

15 条无自交（WP-intersect=0）的真实 F1 赛道，来自 [meijersa/f1-circuits](https://github.com/meijersa/f1-circuits) GeoJSON 数据集：

| 序号 | 赛道 | ID | 路径点数 |
|:---:|:-----|:---|:--------:|
| 1 | 巴林赛道 (Bahrain) | `bahrain` | 63 |
| 2 | 澳大利亚赛道 (Melbourne) | `melbourne` | 58 |
| 3 | 上海赛道 (Shanghai) | `shanghai` | 70 |
| 4 | 西班牙赛道 (Barcelona) | `barcelona` | 68 |
| 5 | 摩纳哥赛道 (Monaco) | `monaco` | 61 |
| 6 | 加拿大赛道 (Montreal) | `montreal` | 44 |
| 7 | 奥地利赛道 (Spielberg) | `spielberg` | 56 |
| 8 | 英国赛道 (Silverstone) | `silverstone` | 62 |
| 9 | 匈牙利赛道 (Hungaroring) | `hungaroring` | 63 |
| 10 | 蒙扎赛道 (Monza) | `monza` | 56 |
| 11 | 新加坡赛道 (Marina Bay) | `marina` | 57 |
| 12 | 墨西哥赛道 (Mexico City) | `mexico` | 56 |
| 13 | 阿布扎比赛道 (Yas Marina) | `yas_marina` | 51 |
| 14 | 伊莫拉赛道 (Imola) | `imola` | 48 |
| 15 | 荷兰赛道 (Zandvoort) | `zandvoort` | 70 |

**生成流水线**（`dev/gen_all_tracks.py`）：
1. 从 GeoJSON 提取 GPS 坐标
2. `gps_to_local()`: 局部笛卡尔坐标系转换（横轴墨卡托）
3. `scale_to_target()`: 统一缩放到目标跨度 6500px
4. `smooth_loop()`: 三重高斯平滑（sigma=[2,3,3]）
5. `resample_uniform()`: 均匀重采样（间距 280px）
6. 二次精细平滑 + 重采样

**筛选标准**：所有赛道 WP-intersect=0（路径点无自交）。被排除的赛道：铃鹿(figure-8)、斯帕、COTA、英特拉格斯（有交叉路径），以及吉达、巴库、迈阿密、拉斯维加斯、卢塞尔（OSM 来源或数据质量问题）。

### 路径点与样条插值

使用 Catmull-Rom 样条插值（`catmullRom`），每段 12 个细分点，生成完整赛道点数组。

```javascript
function catmullRom(t, p0, p1, p2, p3) {
  const t2 = t * t, t3 = t2 * t;
  return 0.5 * ((2 * p1) + (-p0 + p2) * t + (2 * p0 - 5 * p1 + 4 * p2 - p3) * t2 + (-p0 + 3 * p1 - 3 * p2 + p3) * t3);
}
```

### 赛道点属性

`track` 数组中每个点包含：

| 属性 | 含义 |
|------|------|
| `.x` / `.y` | 世界坐标 |
| `.tx` / `.ty` | 切线方向（单位向量） |
| `.nx` / `.ny` | 法线方向（指向赛道外侧） |
| `.dist` | 从起点到该点的累计路径长度 |
| `.totalLen` | 赛道总周长 |

### 赛道选择器

覆盖全屏的赛道选择器，每页显示 4 条赛道预览卡片，支持翻页（◀ ▶ 箭头），共 4 页（4+4+4+3）。

### 最近赛道点查询

两步搜索策略——粗搜索（步长 3）→ 精搜索（±4 邻域）：

```javascript
function nearestTrackPoint(x, y) {
  let best = 0, bd = Infinity;
  for (let i = 0; i < track.length; i += 3) { ... }
  const s = Math.max(0, best - 4), e = Math.min(track.length, best + 5);
  for (let i = s; i < e; i++) { ... }
  return { idx: best, dist: Math.sqrt(bd), point: track[best] };
}
```

返回：索引、距离、赛道点引用。

---

## 3. 赛车物理

### 赛车属性

每辆赛车是一个对象，完整关键属性：

```javascript
{
  x, y,             // 位置
  speed,            // 当前速度
  heading,          // 朝向（弧度，0=向上）
  lapCount,         // 已完成圈数
  lastSector,       // 最后通过区域
  onTrack,          // 是否在赛道上
  blocked,          // 是否被锁定（发车格）
  braking,          // 刹车灯状态
  color, accent,    // 外观（车身色 + 翼/条纹强调色）
  num,              // 车号
  // 配色方案 (F1 车队灵感):
  //   p1 #16: 法拉利红 #dc0000 + 金色 #ffd700
  //   p2 #01: 梅赛德斯青 #00d2be + 银 #c0c0c0
  //   a1 #44: 红牛蓝 #1e3a5f + 红 #ff1e00
  //   a2 #77: 迈凯伦木瓜橙 #ff7300 + 深蓝 #002147
  lapCooldown,      // 圈计数冷却
  lapTime,          // 当前单圈时间
  lapStartTime,     // 当前圈开始时间
  offTrackTimer,    // 冲出赛道计时（>1秒触发重设）
  _prevIdx,         // 上一帧最近赛道索引（圈检测）
  _hasWrapped,      // 已绕过赛道末端（圈检测）
  // 尾流属性
  slipstreamTimer,  // 尾流充电计时器
  slipStreamBoost,  // 尾流速度加成量（0 或 45）
  // DRS 属性
  drsActive,        // DRS 是否激活
  drsTimer,         // DRS 剩余时间
  drsCooldown,      // DRS 冷却剩余时间
  // 干净圈属性
  hadOfftrack,      // 本圈是否冲出去过
  hadCollision,     // 本圈是否碰撞过
  lastLapClean,     // 上一圈是否干净（无冲出+无碰撞）
}
```

AI 额外属性：`trackIdx`, `prevSpeed`, `lineBias`, `errDrift`, `errTimer`

### 移动公式

每帧更新：

```javascript
car.x += Math.sin(car.heading) * car.speed * dt;
car.y -= Math.cos(car.heading) * car.speed * dt;
```

坐标系：`heading=0` 时车头朝上，x 正方向为右，y 正方向为下。

### 速度控制

- **加速**: `speed = min(speed + accel * dt, 400 + slipStreamBoost + drsBoost)`
  - 基础极速上限 400
  - 尾流激活时 +45（上限 445）
  - DRS 激活时 +45（与尾流叠加达 490）
- **刹车**: `speed = max(speed - brake * dt, 0)`
- **空气阻力**: `speed -= drag * (speed/maxSpeed) * dt`（与速度比成比例）
- **冲出赛道阻力**: `speed -= offTrackDrag * dt`（强力减速）

### 转向

转向灵敏度随速度变化：

```javascript
const turn = turnLow + (turnHigh - turnLow) * Math.min(speed / maxSpeed, 1);
// 低速时转向灵敏 (3.0)，高速时转向迟钝 (0.85)
```

### 冲出赛道与重设

当赛车中心到赛道中心线距离 `> trackWidth * 0.55`（即 121px）时视为 off-track：

1. 立即施加 `offTrackDrag`（500 px/s²）减速
2. 启动 `offTrackTimer` 计时
3. 超过 1 秒后自动重设到最近赛道点（位置 + 朝向 + 重置速度为 50）

玩家和 AI 完全共享此逻辑。

---

## 4. 玩家控制与 DRS

### 按键映射

| 操作 | 玩家 1（箭头键） | 玩家 2（WASD） |
|:----:|:----------------:|:--------------:|
| 加速 | ↑ | W |
| 刹车 | ↓ | S / **空格** |
| 左转 | ← | A |
| 右转 | → | D |
| **DRS** | **右 Option (Alt)** | **空格** |

**注意**：P2 的空格键同时作为刹车和 DRS 激活键。

### DRS 激活

```javascript
if (keyState[km.drs]) {
  const n = nearestTrackPoint(car.x, car.y);
  if (speedProfile[n.idx] > DRS_STRAIGHT_THRESHOLD && car.drsCooldown <= 0 && !car.drsActive) {
    car.drsActive = true;
    car.drsTimer = DRS_DURATION;
    car.drsCooldown = car.lastLapClean ? DRS_COOLDOWN_CLEAN : DRS_COOLDOWN;
  }
}
```

- 仅直道可用（speedProfile > 600）
- 激活后持续 3 秒
- 冷却 10 秒（完成干净圈后降为 5 秒）
- DRS 与尾流可叠加（总速度上限 400+45+45=490）

---

## 5. AI 驾驶系统

### 总体架构

简洁的三层架构，不使用机器学习：

```
1. 赛道位置追踪 → trackIdx 维护
2. Pure Pursuit 中心线追踪 + 避让
3. 曲率速度剖面 + 比例控制器 速度控制
```

### 5.1 赛道位置追踪

跟踪 AI 在赛道上的当前位置索引，带环绕修正：

```javascript
let idxDiff = n.idx - car.trackIdx;
if (idxDiff < -track.length / 2) idxDiff += track.length;
if (idxDiff > track.length / 2) idxDiff -= track.length;
if (Math.abs(idxDiff) < track.length / 3) car.trackIdx = n.idx;
```

允许小幅后退修正，防止急弯中索引跳跃。

### 5.2 Pure Pursuit 中心线追踪

核心转向逻辑：追踪前方一定距离的赛道中心线目标点。

**路径选择**：AI 严格沿赛道中心线行驶，走线稳定可预测。

**动态前视距离**：`lookaheadDist = max(speed * 2.0, 100)`，范围 [100, 500]。高速看更远（平滑），低速看更近（精确过弯）。

**转向增益**：`2.5`，最大转向速率 `4.0 rad/s`。

```javascript
const steerCmd = Math.sign(headingErr) * Math.min(Math.abs(headingErr) * steerGain, maxSteerRate);
car.heading += steerCmd * dt;
```

### 5.3 AI 避让

近距离（< 65px）避让其他车辆，推离强度与距离成反比。AI-AI 之间没有弹性碰撞，仅靠此推挤保持间距。

### 5.4 速度控制

基于 Menger 曲率速度剖面 + 比例控制器：

1. **扫描前方**: 向前扫描速度剖面，查找弯道最低安全速度
2. **难度缩放**: `cornerLimit = minProfileSpeed * (cornerSpeed / 0.78)`
3. **目标速度**: `targetSpeed = min(diffMax, cornerLimit)` — 取直道极速和弯道限速的较低值
4. **比例速度控制**: 用 `spdErr = targetSpeed - currentSpeed` 做比例加减速
   - 误差大 → 强减速（`-spdErr * 2.0`，上限 `CFG.brake`）
   - 误差小 → 轻微减速
   - 接近目标 → 自然平滑趋近
   - 死区 ±3 防止微观振荡
5. **尾流/DRS 加成**: 激活时 diffMax 相应增加

### 5.5 AI DRS

AI 在直道上且速度 > 300 时自动激活 DRS：

```javascript
if (speedProfile[car.trackIdx] > DRS_STRAIGHT_THRESHOLD && car.drsCooldown <= 0 && !car.drsActive && car.speed > 300) {
  car.drsActive = true;
  car.drsTimer = DRS_DURATION;
  car.drsCooldown = car.lastLapClean ? DRS_COOLDOWN_CLEAN : DRS_COOLDOWN;
}
```

---

## 6. 速度剖面预计算

在 `startCountdown()` 中调用 `computeRacingLineAndProfile()` 完成预计算。

### 最优路径（Racing Line）计算

1. **Menger 曲率计算**: 在赛道中心线上用 5 点窗口计算曲率
2. **向内偏移**: `off = -sign(curv) * min(|curv| * 8000, maxOff)`，`maxOff = trackWidth * 0.40`
3. **Gaussian 平滑**: 2 次 5 点高斯平滑，附带边界钳制（`clampDist = trackWidth * 0.42`）保证不超出赛道
4. **切线/法线**: 从平滑后的路径点重新计算

### 中心线速度剖面（Forward-Backward Min-Time Pass）

1. **曲率限制速度**: `vCurv[i] = sqrt(latAccel / |curv[i]|)`，`latAccel = 500`，上限 660
2. **前进扫描**（加速限制）: 从起点向后，逐点限制加速能力
3. **后退扫描**（刹车限制）: 从终点向前，逐点限制刹车能力
4. **平滑处理**: 3 次滑动窗口平均

结果：`speedProfile[i]` 表示第 i 个赛道点的理论最高安全速度。

---

## 7. 发车与比赛流程

### 发车格

赛车排布在主直道上，位置由 `startWp` 决定。

### 倒计时

5 个红灯逐一亮起（间隔 0.6 秒），全部亮起后等待 1.2 秒，绿灯亮起比赛开始。

**倒计时开始时**：调用 `computeRacingLineAndProfile()` 计算最优路径和速度剖面。

### 圈数检测

基于赛道环绕检测，关键状态 `_prevIdx` 和 `_hasWrapped`：

1. **Wrap 检测**: 赛车从赛道末端（>70%）穿越到起点（<30%）→ `_hasWrapped = true`
2. **终点线检测**: `_hasWrapped` 为 true 且从 `finishIdx` 之前穿越到之后 → 计圈
3. **干净圈评估**: 计圈时检查 `hadOfftrack` 和 `hadCollision` 标志，更新 `lastLapClean`
4. 冷却时间 1.5 秒防止重复计圈

### 完赛条件

任意赛车完成设定圈数（3/5/8 圈可选）后，比赛结束并显示排名。

---

## 8. 碰撞系统

### 弹性碰撞

检测两车距离 < 38px：
- 沿法线方向推开（重叠量 × 0.6）
- 速度交换带阻尼（动量 × 0.12 / × 0.6）
- 碰撞位置产生火花效果
- 设置 `hadCollision = true`（用于干净圈判定）
- AI-AI 碰撞跳过

---

## 9. 尾流/滑流系统

### 工作原理

后车跟随前车时获得的极速加成：

- **检测距离**: 250px（约 3 个车身长度）
- **方向一致性**: 航向偏差 < 0.5 rad
- **充电时间**: 0.5 秒持续跟随 → 激活尾流
- **加速效果**: +45 极速加成
- **衰减**: 脱离尾流后快速衰减（-2x/秒）

### 视觉反馈

- 车速 > 200 时车尾绘制淡蓝色渐变气流线条
- 尾流激活时 HUD 显示 `[尾流]` 标签

---

## 10. DRS 减阻系统

### 原理

手工触发的直道加速系统，模拟真实 F1 的 Drag Reduction System。

### 特性

| 参数 | 值 |
|:----:|:---:|
| 速度加成 | +45（与尾流叠加达 +90） |
| 持续时间 | 3 秒 |
| 正常冷却 | 10 秒 |
| 干净圈冷却 | 5 秒 |
| 激活条件 | 直道（speedProfile > 600）+ 冷却完毕 |
| 操作方式 | P1: 右 Option / P2: 空格 |

### 干净圈判定

每圈结束时自动评估：

```javascript
c.lastLapClean = !c.hadOfftrack && !c.hadCollision;
```

- `hadOfftrack`：冲出赛道时设置（即使在 1 秒重生前就返回赛道）
- `hadCollision`：碰撞时设置（两车均标记）
- `lastLapClean` 影响 DRS 冷却时间：10 秒 → 5 秒

### 视觉反馈

- DRS 激活时车尾喷出绿色渐变尾焰 + 粒子
- HUD 显示：
  - `[DRS]` 青色/黄色 — 直道上可激活
  - `[DRS 2.3s]` 绿色 — 激活中倒计时
  - `[DRS CD 8.1s]` 红色 — 冷却中
- HUD 显示 `[干净圈]` 绿色徽章

---

## 11. 渲染系统

### 赛道渲染

- **路面**: 深灰色填充，白色虚线标边缘
- **路肩**: 红白交替条纹（曲率变化点）
- **发车线/终点线**: 黑白棋盘格
- **计时器旗杆**: 红色三角旗

### 赛车渲染

- 完整 F1 车体绘制（端板、前翼、尾翼、Halo、中线、号码）
- 车轮带纹理
- 刹车灯：尾部红色发光效果

### 气动尾流（`drawWake`）

速度 > 300 时，车尾绘制渐变半透明气流线条。

### DRS 尾焰（`drawDRS`）

DRS 激活时车尾绘制绿色渐变三角形 + 绿色粒子。

### 速度线（`drawSpeedLines`）

速度 > 120 时，车侧随机线条模拟高速感。

### 草地、轮胎痕迹、火花

- 草地：4000 个随机草叶 + 径向渐变
- 轮胎痕迹：最多 2500 条，逐渐透明
- **刹车痕迹**：所有赛车刹车时在赛道留下黑色印记（最多 3000 条），保留 5 秒
- 火花：碰撞时产生，生命周期 0.2-0.6 秒

### 相机系统

- **全部聚焦**: 所有赛车包围盒中心 + 平滑跟随（阻尼系数 0.09）
- **单目标聚焦**: 平滑跟随指定车辆
- **动态缩放**: 缩放值在 0.85（单目标）到 0.15（最大拉远）之间平滑过渡
  - 双极低通滤波：先对 `targetZoom` 做一级平滑（系数 0.05），再对 `currentZoom` 做二级平滑（系数 0.10）
- **双相机系统（双人分屏）**: `cam` 和 `cam2` 独立追踪各自目标
  - 使用 `renderView(ox, oy, vw, vh, camObj, zoomVal, camPlayer)` 渲染，`ctx.save/clip/restore` 实现视口隔离

### 小地图（Minimap）

- 右下角，140×120px 半透明背景
- 绘制赛道区域 + 赛车位置点
- 当前玩家高亮带白色边框

---

## 12. HUD 与 UI

- **左侧面板**: 排名、比赛时间、单圈时间、速度、档位、圈数
- **右侧面板**: 其他车辆实时数据
- **底部**: 难度、焦点、操作提示
- **DRS 状态**: DRS 可用/激活/冷却 + 干净圈徽章
- **结束面板**: 获胜者、排名、总用时

### 赛道选择器

- 点击 ⚇ 赛道 按钮打开
- 每页 4 条赛道预览卡片（260×270px）
- ◀ ▶ 箭头翻页
- 绿色边框标记当前选中的赛道
- 底部显示页码（如 "第 2/4 页"）

---

## 13. 开发流程与演进历史

### v0 — 原型
- 基础赛道生成（Catmull-Rom 样条）
- 玩家操控 + 基础渲染

### v1 — v3.6
- AI、双人模式、碰撞系统、尾流、分屏等逐步迭代

### v4 — 真实 GPS 赛道 + DRS
- 引入真实 F1 赛道数据（meijersa/f1-circuits GeoJSON）
- 24 条赛道 → 筛选为 15 条无自交赛道
- 赛道生成流水线：GPS → 平滑 → 重采样 → 赛道点
- 添加 DRS 加速系统（P1 Alt / P2 空格）
- 干净圈跟踪机制（hadOfftrack + hadCollision）
- DRS 冷却：正常 10 秒 / 干净圈 5 秒
- 等距 AI 难度梯度（0.90×~1.40× / 80%~100% 过弯）
- 赛道选择器分页（4 条/页，共 4 页）
- 赛事里程碑版本管理（versions/ 目录）

---

## 关键技术决策

1. **为什么用真实 GPS 赛道而非手绘？**
   真实 F1 赛道布局能提供更好的游戏体验。通过 meijersa/f1-circuits GeoJSON 数据集获取 21 条赛道，加上 OpenStreetMap 补充 3 条，经平滑和缩放处理后统一到游戏坐标系。

2. **为什么只保留 15 条赛道？**
   从 24 条赛道中过滤掉自交（WP-intersect>0）的赛道，确保赛车在赛道上的行驶逻辑正确。铃鹿的 figure-8、斯帕的交叉段等会导致圈数检测异常。

3. **为什么 DRS 与尾流不互斥？**
   DRS 和尾流在真实 F1 中可同时使用（在 DRS 检测区内且与前车差距 < 1 秒）。游戏中叠加提供更多策略深度。

4. **为什么干净圈只需无冲出+无碰撞？**
   这是干净圈的最小合理定义。真实 F1 的赛道限制更复杂（切弯、超出赛道线等），但游戏中的 off-track 检测已覆盖主要违规行为。

5. **为什么圈数检测状态需要保护？**
   冲出赛道重设时如果重置 `_hasWrapped`，已绕圈的进度丢失，会导致圈数少计。将 `_prevIdx` 移入 on-track 条件内防止 off-track 帧污染，联合重生点同步策略，确保冲出赛道不影响圈数统计。

6. **为什么 errorScale/brakeFactor 未实装？**
   当前 AI 的走线精度和刹车力度在所有难度下相同，差异仅体现在速度上。这些参数保留用于未来需要更精细难度调校时的扩展。

---

## 项目结构

```
F1/
├── index.html              ← 成品（15 条赛道 DRS 主文件）
├── dev/                    ← 开发文件
│   ├── gen_all_tracks.py   ← 15 条赛道生成脚本
│   ├── gen_tracks_final.py ← 旧版生成脚本（单条赛道）
│   ├── parse_circuits.py   ← GPS 数据解析脚本
│   ├── design_tracks*.py   ← 赛道设计脚本（历史）
│   ├── PROMPT.md           ← 本文档
│   ├── data/
│   │   └── f1-circuits.geojson  ← F1 赛道 GPS 数据源
│   ├── track_viz/          ← SVG/PNG 可视化文件
│   └── 赛道地图*.jpg/png/webp ← 参考图片
└── versions/               ← 历史里程碑版本
    ├── F1-v2.html ~ v4.html    ← 迭代版本
    ├── v4-milestone.html       ← v4 里程碑
    ├── f1-milestone-v2-multi-circuit.html  ← 11 赛道里程碑
    └── f1-milestone-v2-15-circuits.html    ← 15 赛道最终版
```

## 启动方式

```bash
# 直接浏览器打开 index.html 即可
open index.html
```
