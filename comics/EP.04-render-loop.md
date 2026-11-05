# EP.04 渲染循环与帧率自由

> 对应正文：[4.1 基础入门：WebGLRenderer 与渲染循环](../guide/04-renderer/basics.md)
> 延伸阅读：[进阶篇](../guide/04-renderer/advanced.md) · [实战篇](../guide/04-renderer/practice.md)
> 动手演示：[render-loop 按帧 vs 按时间](../public/demos/renderer/render-loop.html) · [adaptive-fps 帧率自适应](../public/demos/renderer/adaptive-fps.html) · [quality 画质分级](../public/demos/renderer/quality.html)

## 剧情回顾

| 格 | 场景 |
| --- | --- |
| ① 困境 | 标签君在 144Hz 电竞屏上演示，方块转得像电风扇；同事 30fps 笔记本上却慢如蜗牛——同一行 `+= 0.02`。 |
| ② 揭秘 | 码叔点破：按"帧"写动画，速度就被帧率绑架；正确写法是**速度 × deltaTime（真实秒数）**。 |
| ③ 实证 | 像素酱用 `THREE.Timer` 重写循环（r183 起 Clock 弃用），两个立方体在 60/30/15fps 下速度完全一致；再做帧率自适应，用迟滞区间避免画质来回横跳。 |
| ④ 结案 | 码叔总结：`timer.update()` 每帧先调，运动全部乘 dt，单帧步长钳 0.05 防空穿梭。 |

## 知识点拆解

### ① 按帧 vs 按时间

```js
// ❌ 帧率绑定：60fps 每秒 1.2rad，30fps 只有 0.6rad
cube.rotation.x += 0.02

// ✅ 时间绑定：任何帧率每秒都是 1.2rad
cube.rotation.x += 1.2 * delta
```

浏览器用 `requestAnimationFrame` 在每次屏幕刷新前回调，屏幕 60Hz / 120Hz / 144Hz 都可能出现。只有以真实秒数为单位，动画速度才与硬件解耦——这叫**帧率无关（frame-rate independent）**。

### ② r186 的 THREE.Timer 三步固定用法

```js
const timer = new THREE.Timer()
timer.connect(document)   // 借助页面可见性 API，切后台自动规避超大 delta
renderer.setAnimationLoop(() => {
  timer.update()                  // ① 每帧先更新内部计时
  const delta = timer.getDelta()  // ② 取距上一帧的秒数，做运动
  // const t = timer.getElapsed() // 累计秒数，做 sin(t) 之类的时间曲线
})
```

注意：`THREE.Clock` 自 r183 起弃用；r186 的 Timer 用 `getDelta()` / `getElapsed()`，没有 `elapsedTime` 属性。

### ③ 切后台回来的"瞬移"

rAF 在后台标签页被暂停，回来的第一帧 delta 可能长达数秒，物体会一下飞出屏幕。两道保险：

1. `timer.connect(document)` 借助可见性 API 自动处理；
2. 手动钳制：`const dt = Math.min(timer.getDelta(), 0.05)`。

### ④ 帧率自适应与迟滞

动态画质不能"掉到 55 就降档、回到 60 立刻升档"，否则档位来回抖动。做法是设两个阈值形成迟滞区间：持续低于 50fps 一段时间才降一档，持续高于 58fps 才允许升档。可调节的档位包括：`renderer.setPixelRatio`、阴影 mapSize、几何体细分、是否开启后处理。

## 坑点清单（FAQ）

- **Q：为什么 timer.update() 必须每帧都调，即使这帧提前 return？** getDelta 依赖内部"上次 update 时间"，跳帧时不 update 会让下一帧 delta 累积成超大值。
- **Q：getDelta 和 getElapsed 怎么选？** 前者是相邻帧间隔，驱动运动；后者是开机以来累计秒数，驱动周期曲线。
- **Q：setAnimationLoop 比手写 rAF 好在哪？** 内置页面隐藏暂停、WebGL 上下文丢失自动恢复，r152+ 官方推荐。
- **Q：想做子弹时间（慢动作）怎么办？** 给 delta 乘一个全局时间缩放系数（如 0.3），所有运动统一变慢，无需改各物体速度。

## 自测（Quiz）

1. `rotation += 0.02` 在 144Hz 屏幕上比 60Hz 快多少倍？为什么？
2. r186 中取 delta 的标准对象与三步调用分别是什么？
3. 切后台 10 秒再回来物体飞出画面，给出两种修复方式。

<details>
<summary>答案</summary>

1. **2.4 倍**（144/60）。固定增量每帧执行一次，每秒执行次数等于帧率，帧率越高速度越快。
2. 对象是 **`THREE.Timer`**（Clock 已弃用）：① `timer.connect(document)` 初始化；② 每帧先 `timer.update()`；③ 再 `timer.getDelta()`。
3. ① **`timer.connect(document)`** 利用页面可见性 API 规避；② 手动钳制 **`Math.min(timer.getDelta(), 0.05)`**，把单帧步长限制在 50ms 内。
</details>
