# EP.07 纹理 UV 与流动

> 对应正文：[7.1 基础入门：纹理贴图与 CanvasTexture](../guide/07-texture/basics.md)
> 延伸阅读：[进阶篇](../guide/07-texture/advanced.md) · [实战篇](../guide/07-texture/practice.md)
> 动手演示：[canvas-texture 程序化贴图](../public/demos/texture/canvas-texture.html) · [wrapping 环绕与 repeat](../public/demos/texture/wrapping.html) · [flowing-texture 赛博流动地板](../public/demos/texture/flowing-texture.html)

## 剧情回顾

| 格 | 场景 |
| --- | --- |
| ① 困境 | 标签君贴的网格图发暗发灰；设了 `repeat.set(4,4)` 瓷砖却毫无反应，只有一张被拉伸的大图。 |
| ② 揭秘 | 码叔摊开 UV：每个顶点都有 0~1 的纹理坐标，GPU 沿 UV 采样贴图；发暗是色彩空间没标 sRGB，repeat 无效是环绕模式仍是默认 Clamp。 |
| ③ 实证 | 像素酱补上 `colorSpace = SRGBColorSpace` 与 `wrapS/wrapT = RepeatWrapping`，再逐帧推 `texture.offset.x`，赛博地板的网格线开始无限流动。 |
| ④ 结案 | 码叔总结：颜色贴图标 sRGB、数据贴图保线性；斜看模糊加 anisotropy，Canvas 重绘后 needsUpdate。 |

## 知识点拆解

### ① UV：顶点到图片的映射表

几何体每个顶点带一组 `(u, v)`，范围通常 0~1，原点在图片**左下角**。GPU 在三角形内部对 UV 插值，逐像素从贴图读取颜色（采样），把 2D 图片"裹"到 3D 表面。贴图朝向不对时，先怀疑 UV 方向与 `texture.flipY`。

### ② 发暗：色彩空间必须标对

r152 起色彩管理默认开启：

```js
// 颜色类贴图：必须标 sRGB，否则经线性转换后发暗发灰
texture.colorSpace = THREE.SRGBColorSpace
```

- 颜色类（`map`、`emissiveMap`）：设 **SRGBColorSpace**；
- 数据类（`normalMap`、`roughnessMap`、`metalnessMap`、`aoMap`）：保持**线性**，设了反而出错。

### ③ repeat 无效：先改环绕模式

Texture 默认是 ClampToEdgeWrapping（超出 0~1 的 UV 反复采样边缘像素）。想要瓷砖效果：

```js
texture.wrapS = THREE.RepeatWrapping
texture.wrapT = THREE.RepeatWrapping
texture.repeat.set(4, 4)
```

其它模式：MirroredRepeatWrapping（镜像平铺，接缝更隐蔽）；ClampToEdgeWrapping（默认，适合 UI/光照贴图）。

### ④ 流动贴图与清晰度

- 流动：每帧 `texture.offset.x += speed * dt`，配合 Repeat 实现无限滚动（传送带、赛博地板、流水）；
- 斜视摩尔纹/糊：提高 `texture.anisotropy`（上限 `renderer.capabilities.getMaxAnisotropy()`）；
- 程序化纹理零资源方案：`<canvas>` 画图 → `new THREE.CanvasTexture(canvas)`，重绘 canvas 后必须 `texture.needsUpdate = true` 才会重新上传。

## 坑点清单（FAQ）

- **Q：法线贴图也设了 sRGB，凹凸感怪怪的？** normalMap 是数据不是颜色，必须保持线性空间。
- **Q：repeat 设了 2×2 还是只显示一次？** 确认 wrapS/wrapT 都改成 RepeatWrapping；Clamp 模式下 repeat 大于 1 没有平铺效果。
- **Q：CanvasTexture 动态重绘后画面不更新？** 触发 `texture.needsUpdate = true`；频繁整图重绘有开销，只在内容真正变化时设置。
- **Q：贴图上下颠倒？** Three.js 默认 `flipY = true` 以匹配图片左上角原点的约定；与自定义 UV/数据纹理不匹配时手动切换。
- **Q：地面远处网格闪烁？** 共面 z-fighting 时用 polygonOffset 拉开层级，并参考 EP.03 收紧 near/far。

## 自测（Quiz）

1. 颜色贴图发暗，应该设置哪个属性？法线贴图也要设吗？
2. `texture.repeat.set(4,4)` 不生效，还缺哪两个设置？
3. 想让地面网格每帧向前流动，逐帧修改纹理的哪个属性？为什么不用移动几何体？

<details>
<summary>答案</summary>

1. 设置 **`texture.colorSpace = THREE.SRGBColorSpace`**；法线贴图**不要设**，它是线性数据类贴图，设 sRGB 会导致凹凸计算错误。
2. **`wrapS = THREE.RepeatWrapping`** 与 **`wrapT = THREE.RepeatWrapping`**；默认的 ClampToEdge 会忽略 0~1 之外的重复采样。
3. 修改 **`texture.offset`**（如 `offset.y += speed * dt`）。偏移只改采样坐标、成本极低；移动几何体则要持续重建/平移顶点或网格，开销大且不适合无限平铺。
</details>
