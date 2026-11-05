# EP.19 给帧率发工资：16.6ms 怎么花

> 对应正文：[19.1 基础入门：性能预算与采集](../guide/19-engineering-performance/basics.md)
> 延伸阅读：[进阶篇](../guide/19-engineering-performance/advanced.md) · [实战篇](../guide/19-engineering-performance/practice.md)
> 动手演示：[dispose-lab 内存泄漏实验室](../public/demos/performance/dispose-lab.html) · [quality-tiers 画质分级适配](../public/demos/performance/quality-tiers.html)

## 剧情回顾

| 格 | 场景 |
| --- | --- |
| ① 困境 | 像素酱的页面在开发机上 60 帧，却被投诉：用户红米老机只有 20 帧、帧时间 47ms——同一套代码，高端机满帧、低端机崩盘，没有预算表，优化全凭感觉。 |
| ② 揭秘 | 码叔点破：每帧 16.6ms 是总预算，JS 脚本领 4ms、CPU 提交领 6ms、GPU 渲染领 6ms——渲染、脚本、合成都从这笔钱里领，谁超支谁拖帧。 |
| ③ 实证 | 画质分三档后自动选档：低端机切低档（dpr 1、关阴影、关 Bloom、实例 6000→1200），帧时间 47ms→17ms，FPS 21→55；预算表贴在工位，每版按指标验收。 |
| ④ 结案 | 码叔总结：优化不是感觉，是指标；附坑点清单——别只看平均 FPS、dispose 别漏共享资源、自动降档要加锁定。 |

## 知识点拆解

### ① 16.6ms：帧时间的"工资单"

60fps 意味着每帧只有 **16.6ms**（1000/60），JS 脚本、CPU 提交、GPU 渲染、浏览器合成全从这一份预算里领钱，任何一项超支整帧迟到。性能预算就是把这份工资单提前写死：

| 指标 | 高端桌面 | 中端手机 | 低端机 |
| --- | --- | --- | --- |
| 帧时间 | ≤16.6ms | ≤16.6ms | ≤33.3ms |
| draw calls | ≤3000 | ≤500 | ≤150 |
| 纹理显存 | ≤2GB | ≤512MB | ≤256MB |

预算按**目标设备中最弱的一档**定，并预留约 30% 余量给系统与 GC。

### ② 采集三板斧

- **Stats.js**：FPS / MS / MB 三页面板，开发期常驻；MS 页比 FPS 更直观；
- **Performance API**：`performance.now()` 掐帧时间、`performance.memory.usedJSHeapSize` 读 JS 堆（仅 Chrome 系）；
- **DevTools Performance 面板**：红色长帧定位掉帧现场，火焰图看钱被谁领走。

关键认知：**峰值帧时间比平均帧率更能反映卡顿**——平均 60fps 也可能偶尔一帧 80ms，人眼一定能察觉。

### ③ 实证：画质三档分级

同一楼群场景三档切换，按"整帧成本"从高到低动刀：

```js
const TIERS = {
  high: { dpr: 2,   shadow: 2048, bloom: true,  count: 6000 },
  mid:  { dpr: 1.5, shadow: 1024, bloom: false, count: 3000 },
  low:  { dpr: 1,   shadow: 0,    bloom: false, count: 1200 }
}
```

启动时设备特征（cores / memory / dpr）粗判 + 试跑 2 秒精调；运行中滚动统计帧时间，**连续 3 个窗口超标才降档**，降后锁定 10 秒防横跳。实测低端机帧时间 47ms→17ms，FPS 回到 55。

### ④ 配套纪律：内存与验收

- **dispose 全家桶**：geometry / material / texture / renderTarget 逐一释放，共享资源引用计数，SPA 路由切换时按序收尸（停循环 → 摘监听 → 释放场景 → renderer.dispose）；
- **性能报告模板**：机型 / 档位 / FPS / 帧时间均值与峰值 / calls / 三角形 / 内存，结论只有"达标 / 可接受 / 不达标"，没有"还行"。

## 坑点清单（FAQ）

- **Q：平均 60fps 为什么还卡？** FPS 是平均值，会掩盖尖峰长帧。盯帧时间的峰值与方差（Stats MS 页、自写 HUD），常见元凶是 GC、纹理首次上传与着色器即时编译。
- **Q：dispose 了内存还涨？** 查三处：共享材质/纹理被漏算或重复释放、RenderTarget 与 PMREM 没 dispose、JS 侧引用没断（数组/闭包/回调里还挂着）。
- **Q：自动降档被投诉"画面突然变糊"？** 连续超标才降、降档弹提示、允许用户手动锁档；降档后加锁定期，避免两档间反复横跳比掉帧更晃眼。
- **Q：performance.memory 非 Chrome 没有怎么办？** `if (performance.memory)` 兜底显示 N/A；显存自己按"宽×高×4×1.33"估算；跨浏览器验收以帧时间为硬指标。

## 自测（Quiz）

1. 为什么性能预算要按"目标设备中最弱的一档"来定，而不是按团队开发机的水平？
2. 统计自动降档时，为什么用"连续 3 个窗口帧时间超标"而不是"一帧超过 20ms 就降"？
3. SPA 路由切走 3D 页面时，只把 `<canvas>` 从 DOM 移除会有什么后果？至少补两步收尸操作。

<details>
<summary>答案</summary>

1. 预算的意义是**保证体验下限**。开发机普遍是高端设备，按它定预算，低端机必然超支掉帧——而低端机用户往往正是大众产品的基本盘。按最弱档定红线、预留 30% 余量，高端机自然轻松达标。
2. 单次掉帧可能只是 GC、纹理上传等瞬时抖动，立刻降档会让画质无谓波动、在两档间反复横跳。连续多个统计窗口超标才说明是**持续性负载过高**，此时降档才是真对症；配合降后锁定期，画质才稳定。
3. canvas 移除了，但渲染循环还在跑、GPU 资源全活着，来回切换十几次低端机直接白屏。收尸顺序：`renderer.setAnimationLoop(null)` 停循环 → 摘事件监听 → `scene.traverse` 全树 dispose → `composer?.dispose()` → `renderer.dispose()` + `forceContextLoss()`。
</details>
