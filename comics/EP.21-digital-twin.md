# EP.21 把一座园区装进浏览器

> 对应正文：[21.1 基础入门：大屏 3D 架构与相机漫游](../guide/21-digital-twin/basics.md)
> 延伸阅读：[进阶篇](../guide/21-digital-twin/advanced.md) · [实战篇](../guide/21-digital-twin/practice.md)
> 动手演示：[camera-tour 相机漫游与视角书签](../public/demos/digital-twin/camera-tour.html) · [labels-scale 千级标注方案对比](../public/demos/digital-twin/labels-scale.html) · [campus-twin 园区数字孪生](../public/demos/digital-twin/campus-twin.html)

## 剧情回顾

| 格 | 场景 |
| --- | --- |
| ① 困境 | 老板晨会一句话："咱们也做个数字孪生大屏"。像素酱眼前一黑：整座园区建模，这不得三个月？ |
| ② 揭秘 | 码叔在白板上画出四层架构：底图/标注/特效/UI 各司其职——⑯ 的实例化城市、⑫ 的扫描线、⑬ 的泛光、HTML 面板，全是现成零件。 |
| ③ 实证 | 像素酱一天拼出园区 Demo：定时器模拟告警推送，楼群变红闪烁，点击告警卡片相机直飞事发楼栋，老板当场鼓掌。 |
| ④ 结案 | 坑点清单：pointer-events 忘关 3D 拖不动、CSS2D 标上千个卡成 PPT、飞行不响应拖拽像死机。码叔总结：大屏 = 分层架构 + 零件复用。 |

## 知识点拆解

### ① 大屏不是"建"出来的，是"拼"出来的

面对"数字孪生大屏"这种需求，第一反应容易是"建模地狱"。其实拆开看，园区大屏的每个部分都对应前面学过的成品件：

| 大屏的层 | 职责 | 现成零件 |
| --- | --- | --- |
| 底图层 | 园区三维场景 | ⑯ InstancedMesh + 合并静态几何 |
| 标注层 | 楼名、设备牌 | 21.2 CSS2D / Sprite / 实例化标牌 |
| 特效层 | 扫描环、泛光 | ⑫ ShaderMaterial + ⑬ EffectComposer |
| UI 层 | 标题栏、数据面板、告警列表 | 纯 HTML + CSS，absolute 盖在画布上 |

### ② 四层架构的关键纪律

- **UI 层用 HTML，不用 3D 做**：文字清晰度、排版效率、开发速度全面碾压；
- **事件纪律**：UI 罩子 `pointer-events: none`，可点元素单独 `auto`；
- **通信纪律**：UI 层只读数据层台账，动作走 `flyToBuilding(b)` 这样的语义函数，不直接摸 3D 对象。

### ③ 相机漫游 = 两组向量的插值

```js
// 书签 = 相机位置 + 注视点；飞行 = lerp 插值 + smoothstep 缓动
flight.t += dt / flight.duration
const k = ease(Math.min(flight.t, 1))
camera.position.lerpVectors(flight.fromPos, flight.toPos, k)
controls.target.lerpVectors(flight.fromTarget, flight.toTarget, k)
// 用户拖拽 → controls 'start' 事件 → flight.active = false
```

### ④ 实时数据：入口收敛，mock 可换真

所有告警都进同一个 `pushAlarm()`：现在由 `setInterval` 模拟触发，以后换 WebSocket 只改触发方式。告警一进来分两头——场景层 `setColorAt` 染红闪烁，UI 层列表 prepend 卡片；点击卡片拿台账坐标 `flyTo`。**一份数据，三处呈现**。

## 坑点清单（FAQ）

- **Q：HTML 面板盖上后 3D 拖不动？** UI 罩子容器忘了 `pointer-events: none`，事件全被 HTML 截走；可交互元素再单独开 `auto`。
- **Q：CSS2D 标注上千个就卡？** DOM 节点每帧投影 + 排版是 CPU 黑洞；千级标注换实例化标牌（1 次 draw call），文字用悬停 HUD 补位。
- **Q：飞行途中用户拖拽像"抢方向盘"？** 不响应才是 bug——监听 `controls` 的 `start` 事件立刻 `flight.active = false`，控制权交还用户。
- **Q：经纬度直接当世界坐标，物体狂抖？** float32 精度不够；先减园区原点再换算，坐标保持在千米级小数。

## 自测（Quiz）

1. 大屏四周的标题栏和数据面板，为什么用 HTML 而不是 3D 文字实现？
2. 千级标注为什么首选实例化标牌？CSS2D 和 Sprite 各自的瓶颈是什么？
3. 点击告警列表让相机飞到事发楼栋，这条链路里数据层、UI 层、场景层各做了什么？

<details>
<summary>答案</summary>

1. HTML 的文字清晰度、排版能力（流式布局、表格、滚动列表）与开发效率都远超 3D 文字；3D canvas 只负责中间"会动的地图"。UI 层用 absolute 定位盖在画布上，靠 `pointer-events` 划分事件归属。
2. 实例化标牌一次 draw call 画全部标注，`instanceId` 拾取与 `setColorAt` 高亮都是现成机制。CSS2D 的瓶颈是 DOM 数量（每帧逐节点投影与排版），Sprite 的瓶颈是一张标签一次 draw call、且文字放大易糊。
3. 数据层台账提供楼栋名称与坐标；UI 层渲染告警卡片并把点击事件接到 `flyToBuilding(b)`；场景层的飞行系统读取台账坐标，用 lerp 插值驱动相机 position 与 controls.target 飞往事发楼栋。
</details>
