# Unity ↔ Three.js 坐标系转换 Skill

> 本 Skill 适用于本项目中 Unity 编辑器端与 Three.js 渲染端之间的所有 Transform 数据交换。
> 凡涉及 `position` / `rotation` / `roll` 的操作，必须遵循本文约定的转换规则。
> **每次会话开始必须加载本文件。**

---

## 1. 核心差异

| 属性 | Unity | Three.js |
|------|-------|----------|
| 坐标系手性 | **左手系** (Left-Handed) | **右手系** (Right-Handed) |
| 默认上轴 | Y ↑ (up) | Y ↑ (up) |
| 默认前方 | Z+ (蓝轴 forward) | **本项目约定 X+ (红轴 forward)** |
| 旋转正方向 | 顺时针（看向轴正方向时） | 逆时针 / 右手定则 |
| 角度单位 | 度 (°) | **弧度 (rad)** ✅ 本项目统一用弧度 |

> **重点**：本项目的 Three.js 代码中，轨道行进方向是 **X+** 轴，与 Unity 默认 Z+ 不同。

---

## 2. Position 位置转换（本项目采用简化版）

### ⚠️ 重要：本项目不做左右镜像翻转

本项目采用**简化映射**，不翻转 X 轴符号。这样 Unity 中的左转/右转在 Three.js 中视觉方向一致，避免设计时的认知混乱。

### 2.1 本项目采用的映射公式

```
Unity (X, Y, Z)  →  Three.js (Z, Y, X)
   │  │  │                  │  │  │
  右 上 前                 前 上  右
```

等价公式：

```js
// Unity position → Three.js position
threePos.x =  unityPos.z;   // Unity 前方 → Three.js X+
threePos.y =  unityPos.y;   // Y 轴不变
threePos.z =  unityPos.x;   // Unity 右方 → Three.js Z+（不翻转）
```

### 2.2 C# 导出代码

```csharp
// Unity → Three.js
float tx =  uPos.z;    // 前方映射
float ty =  uPos.y;    // 高度不变
float tz =  uPos.x;    // 左右不翻转
```

### 2.3 C# 导入代码（反向转换）

```csharp
// Three.js → Unity
float ux =  threePos.z;   // Three.js Z → Unity X
float uy =  threePos.y;   // 高度不变
float uz =  threePos.x;   // Three.js X → Unity Z
```

### 2.4 验证清单

- [ ] Unity 中站在原点看向 +Z → Three.js 中站在原点看向 +X
- [ ] Unity 中向右平移 (+X) → Three.js 中向 +Z 平移（同侧）
- [ ] Unity 中向上平移 (+Y) → Three.js 中向上平移 (+Y)
- [ ] Unity 中左转弯 → Three.js 中也是左转弯

---

## 3. Roll 扭转值转换

Roll 定义：**绕轨道前进方向的旋转角度**。

| Unity | Three.js |
|-------|----------|
| 绕 +Z 轴旋转（看向 +Z 方向，顺时针为正） | 绕 +X 轴旋转（右手定则，看向 +X 时 CCW 为正） |
| 单位：度 (°) | **单位：弧度 (rad)** |

### 转换规则

**Roll 数值取反 + 单位转换**。

```
threeRoll(rad) = -unityRoll(°) × (π / 180)
```

反向:
```
unityRoll(°) = -threeRoll(rad) × (180 / π)
```

验证：
- Unity 中顺时针拧 90° → Three.js 中 `roll = -1.5708 rad`（-π/2），视觉效果一致

---

## 4. Scale 缩放转换

缩放直接传递，**不需转换**：

```
threeScale = unityScale
```

---

## 5. 完整导出 JSON 转换函数

在 Unity C# 导出脚本 (`TrackExporter.cs`) 中使用：

```csharp
// C# — Unity 端导出
Vector3 uPos = node.transform.position;   // Unity position
float uRoll = node.rollDegrees;           // 用户在 Inspector 中设的度数

// 转换为 Three.js 坐标（简化版，不翻转左右）
float tx = uPos.z;
float ty = uPos.y;
float tz = uPos.x;      // 注意：不取反
float tRoll = -uRoll * Mathf.Deg2Rad;      // 度 → 弧度, 取反

// 写入 JSON
jsonNode.position = new float[] { tx, ty, tz };
jsonNode.roll = tRoll;
```

对应的反向导入：

```csharp
// C# — Unity 端导入
float ux = nd.position[2];    // Three.js Z → Unity X（注意：不取反）
float uy = nd.position[1];    // Y 不变
float uz = nd.position[0];    // Three.js X → Unity Z
float rollDeg = -nd.roll * Mathf.Rad2Deg;    // 弧度 → 度, 取反
```

对应的 Three.js 加载函数：

```js
// JavaScript — HTML 端加载
trackBuilder = new TrackBuilder(jsonTrack.nodes.map(n => ({
  position: n.position,   // 直接使用 [tx, ty, tz]
  roll: n.roll             // 直接使用弧度值
})));
```

---

## 6. 常见陷阱

| 陷阱 | 说明 | 解决 |
|------|------|------|
| 度 vs 弧度 | Unity 用度，Three.js 用弧度 | `Mathf.Deg2Rad` × π/180 |
| 左右方向 | 本项目不翻转 X，Unity 左转 = Three.js 左转 | 导出时 `tz = uPos.x`（无负号） |
| Roll 方向颠倒 | 如果轨道 Roll 方向反了 | 检查 `roll` 的正负号，尝试取反 |
| 前进方向 | Unity +Z 前方 → Three.js +X 前方 | `tx = uPos.z` |

---

## 7. 坐标系速查表

```
            Unity                    Three.js (本项目)
            ────                    ────────
            +Y 上                   +Y 上
             │                       │
             │                       │
             └──── +Z 前             └──── +X 前
            /                       /
           /                       /
         +X 右                   +Z 右

POSITION:  (x, y, z)  →  (z, y, x)    ← 不翻转左右
ROLL:      deg        →  -deg × π/180  (取反+弧度)
SCALE:     1:1 直传
```

---

## 8. 历史变更

- **2026-05-12**: 位置转换从 `(z, y, -x)` 改为 `(z, y, x)`（简化版）。
  原因：`-x` 导致左手系→右手系的镜像翻转，Unity 中设 Left 的弯道在 Three.js 中变成右转，
  造成设计时的认知混乱。去掉负号后 Unity 和 Three.js 的左右方向一致。
