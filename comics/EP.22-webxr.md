# EP.22 浏览器里的另一个世界

> 对应正文：[22.1 基础入门：WebXR 会话与最小改造](../guide/22-webxr/basics.md)
> 延伸阅读：[进阶篇](../guide/22-webxr/advanced.md) · [实战篇](../guide/22-webxr/practice.md)
> 动手演示：[vr-showroom 展厅 VR 化](../public/demos/webxr/vr-showroom.html) · [ar-place AR 产品放置](../public/demos/webxr/ar-place.html)

## 剧情回顾

| 格 | 场景 |
| --- | --- |
| ① 困境 | 老板一句"展厅做个 VR 版"，像素酱眼前一黑：学 Unity、买头显、写 C#，这不得半年？ |
| ② 揭秘 | 码叔敲开浏览器：WebXR 是浏览器原生的沉浸式会话——`renderer.xr.enabled` + `VRButton` + `setAnimationLoop`，三行改造就进头显。 |
| ③ 实证 | 同一份展厅代码：桌面 OrbitControls 照看；戴上头显，手柄射线选展品；手机 AR 里瞄准环一扫，椅子落进真实客厅。 |
| ④ 结案 | 坑点清单：rAF 让 VR 画面冻住、非 HTTPS 按钮不出现、hit-test 要找有纹理的亮平面。码叔总结：一份场景，多端呈现。 |

## 知识点拆解

### ① WebXR 不是新引擎，是浏览器的新窗口

面对"做个 VR"的需求，直觉是去学 Unity/Unreal。其实 WebXR 让浏览器本身成为 XR 运行时：

| 顾虑 | 真相 |
| --- | --- |
| 要学新引擎？ | 还是 three.js，场景代码一行不删 |
| 要装运行时？ | Chrome / Edge / Quest Browser 原生支持 |
| 要买头显才能开发？ | WebXR API Emulator 扩展即可模拟 |

### ② 最小改造三行 + 一条纪律

```js
renderer.xr.enabled = true                                 // 打开 XR 通道
document.body.appendChild(VRButton.createButton(renderer)) // 入口按钮
renderer.setAnimationLoop(loop)                            // 纪律：循环必须走它
```

会话期间相机位姿由头显覆写，要移动玩家就移动相机的父级（dolly）；默认参考系 `local-floor`，场景单位就是真实世界的米。

### ③ VR 手柄 = 空间版射线拾取，AR 靠 hit-test

10.1 的 Raycaster 思路原样搬进三维空间：起点换成手柄位置、方向换成手柄 -Z；`select` 事件对应 click；激光线命中缩短给出反馈——拾取、高亮、`userData` 挂数据，全部复用。AR 侧的核心是 hit-test：从视线向真实环境投射，命中平面就驱动 reticle 瞄准环，点屏把商品放到环上。

### ④ 回退是产品底线

桌面/手机浏览器打开 → OrbitControls 预览一个功能不丢；进入会话 → 相机交还头显、AR 隐藏虚拟地板、dom-overlay 显示工具条；退出 → 全部还原。iOS 不走 WebXR，Quick Look（USDZ + `rel="ar"`）是常用兜底双轨。

## 坑点清单（FAQ）

- **Q：进 VR 后画面冻住/黑屏？** 渲染循环还在用 `requestAnimationFrame`——沉浸式会话里帧调度归 XR 系统，必须换成 `renderer.setAnimationLoop`。
- **Q：VR 按钮根本不出现？** WebXR 只在安全上下文可用：HTTPS 或 localhost；`http://` 局域网 IP 会被判定为不安全。
- **Q：AR 里一直找不到平面？** 平面检测需要光线充足 + 平面有纹理 + 缓慢移动手机建图；纯色桌面是最难识别的场景。
- **Q：XR 里拖不动相机了？** 会话中相机由头显姿态覆写，OrbitControls 要在 `sessionstart` 里禁用，退出再启用。

## 自测（Quiz）

1. 为什么 WebXR 应用里渲染循环必须用 `renderer.setAnimationLoop`，而不能用 `window.requestAnimationFrame`？
2. 桌面鼠标拾取和 VR 手柄拾取，在射线构造上有哪些异同？
3. 一个 AR 电商页要同时服务 Android 与 iOS 用户，技术路线怎么选？

<details>
<summary>答案</summary>

1. 进入沉浸式会话后，帧调度权从 window 移交给 XR 系统（头显刷新率 72/90/120Hz），window 的 rAF 不再代表头显刷新。`renderer.setAnimationLoop` 会在会话中自动切换为 `session.requestAnimationFrame`、退出后切回 window rAF，桌面与 XR 一套代码通吃。
2. 相同点：都用同一个 Raycaster 求交、同一套 `userData`/高亮逻辑。不同点：桌面以 pointer 的 NDC + 相机用 `setFromCamera` 构造射线；手柄则以 `controller.matrixWorld` 取位置做起点、手柄 -Z 方向经矩阵变换做方向；触发从 click 换成 `select` 事件。
3. Android 走 WebXR（immersive-ar + hit-test）；iOS Safari 不支持 WebXR，用系统级 Quick Look 兜底——准备 USDZ 资产、以 `rel="ar"` 链接调起。商品数据层尽量共用，只分叉呈现层。
</details>
