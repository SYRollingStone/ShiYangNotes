# 一. Unity Editor 部分
1. Unity中是用官方Spline插件，编辑节点与曲线。类似过山车这种单线轨道游戏推荐Catmull曲线，其他类型如公路、滑梯等可以尝试选择三次Bezier曲线。
2. 利用Agent编写导出节点Json功能。
3. Unity与Three.js坐标系规则不同，注意导出json处理Transform关系。
具体看同目录下：Unity2Three坐标系转换 Skill.md

我在走过 使用SPline插件 - 自己设计Catmull曲线 - 自己尝试用三次贝塞尔实现 ，最终确认用Spline即可它都考虑到了，因为最佳实践推荐使用Spline插件所以Unity部分足够的简单了。

# 二. Three.js 部分
1. Three.js自带Catmull曲线和Bezier曲线的实现，可以无缝衔接。
2. 重点与核心：投影法做普通轨道，平行传输法+打特殊点确定特别段落 解决 投影法中投影与世界Y重叠导致的轨道翻折问题。有兴趣看第五部分。
![翻折问题](https://me.majigame.com/_astro/%E6%89%AD%E6%9B%B2%E7%9A%84%E8%BD%A8%E9%81%93.CAj3OpgN_1UJjdt.webp)

# 三. 工程内容
提供上线版本game_content在同目录。

到此为止即可，下面有兴趣可以看一下。

# 四. 涉及的所有基础概念
## Lerp插值
### 1. 一维lerp
从A到B，走到百分之t, 0<=t<=1
```
lerp(a, b, t)  = A * (1-t) + B * t
```

比如一条直线中A=10,B=20,t=0.2
```
lerp(A,B,t) = 10*(1-0.2)+20*0.2 = 8+4=12
```
### 2. 二维Lerp
A是点(ax,ay),B是(bx,by),t依然是百分比
点A到点B的方向向量v=B-A = (xb-xa,yb-ya)
lerp(A,B,t)的意思是从点A出发沿着v的方向走到B的百分比t的那个点的坐标是多少。
```
lerp(A,B,t) = (
xa + t * (xb - xa),
ya + t * (yb-ya)
)
```
三维同理。

## 三次贝塞尔曲线
### 直观看
https://observablehq.com/@mbostock/de-casteljaus-algorithm?utm_source=chatgpt.com#cell-0

![三次贝塞尔曲线](https://me.majigame.com/_astro/%E4%B8%89%E6%AC%A1%E8%B4%9D%E5%A1%9E%E5%B0%94.C0r3n4jA_Z28TWPP.webp)

利用4个蓝色的点获得一条黑色点形成的路径曲线

4个点：P0,P1,P2,P3 ,图中为蓝色。

先在P0到P1,P1到P2,P2到P3插值，得到三个橘黄色点O1,O2,O3，

再在O1到O2，O2到O3差值，得到黑色点B1.

B1的所有差值点连成一起就是黑色曲线。

除了起点和终点，剩下的两个蓝色点就如Photoshop钢笔工具中的“手柄”。

## Hermite曲线
https://spline.sudohub.dev/index.html
我知道起点来的方向，我知道终点走的方向，然后得到这个曲线。

## CatmullRom曲线
简称CR曲线

https://www.desmos.com/calculator/9kazaxavsf?utm_source=chatgpt.com&lang=zh-CN

有四个点P0,P1,P2,P3.
目的是我们要得到P1到P2这一段的曲线，
用P0和P2得到P1附近的方向，用P1和P3得到P2附近的方向。
用前后相邻点，自动估计当前点的切线方向。  

SR和三次Bezier如何选择：SR不会有突然的弯折突变，三次Bezier能精准控制曲线但是可能存在节点处的突变。

## 完全看不懂数学公式，嘻嘻

# 五、轨道投影法的问题
## 问题
我们通过上面的内容可以构造出任意的线。那么可以尝试生成轨道，首先利用"投影法"但会出现下面的问题：

![翻折问题](https://me.majigame.com/_astro/%E6%89%AD%E6%9B%B2%E7%9A%84%E8%BD%A8%E9%81%93.CAj3OpgN_1UJjdt.webp)

投影法：世界 Y+ 向量投影到切线的法平面上。

一般世界Y都是向上的，切线就是当前点的前进反向，所以在图片中的位置切线和Y反向几乎相同。

当切线趋近竖直（≈ 指向 ±Y）时，|dotTUp| → 1，法平面内几乎没有 Y+ 分量可提取，投影结果接近零向量，法线方向变得数值不稳定——只要轨道稍有左右偏，结果可能朝完全相反的方向，产生 180° 翻转。

竖向回环（Vertical Loop）在顶点处切线完全竖直，这就是问题的发生点。

## 解决 ：平行传输（Parallel Transport）

平行传输不依赖世界坐标系，而是用帧间最小旋转把前一帧的法线"运输"到当前帧
```
// 每一帧：
const axis  = cross(prevTangent, currentTangent);   // 切线转动轴
const angle = acos(dot(prevTangent, currentTangent)); // 转动角
ptNormal = prevNormal.applyAxisAngle(axis, angle);   // 旋转传递
```

这样法线的传递完全由曲线几何自身决定，不涉及世界 Y+ 方向，切线垂直时也不会退化。
