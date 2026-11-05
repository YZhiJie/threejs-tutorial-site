# EP.13 后处理管线与帧率保卫战

> 对应正文：[13.1 基础入门：渲染目标与第一个 Bloom](../guide/13-postprocessing/basics.md)
> 延伸阅读：[进阶篇](../guide/13-postprocessing/advanced.md) · [实战篇](../guide/13-postprocessing/practice.md)
> 动手演示：[first-bloom 第一个泛光](../public/demos/postprocessing/first-bloom.html) · [pass-order-lab Pass 顺序实验室](../public/demos/postprocessing/pass-order-lab.html) · [product-showcase 产品级画面合成](../public/demos/postprocessing/product-showcase.html)

## 剧情回顾

| 格 | 场景 |
| --- | --- |
| ① 困境 | 像素酱给场景一口气叠了 SSAO、Bloom、Film、Glitch、Bokeh 五个 Pass，帧率从 60 掉到 20，画面是华丽了，页面却卡成幻灯片。 |
| ② 揭秘 | 码叔画出 Pass 流水线：链上每一环都是"读上一环的图 → 全屏加工 → 写给下一环"，5 个效果 Pass 就是每帧 5 次以上的全屏绘制；省钱靠半分辨率与合并。 |
| ③ 实证 | 像素酱把 SSAO 换成烘焙 AO 贴图，把 Film 与暗角合并进一个自定义 ShaderPass，链从 5 环瘦到 2 环，帧率回到 60。 |
| ④ 结案 | 坑点清单：threshold 太低满屏泛光、resize 漏了 composer.setSize、OutputPass 没放链尾。码叔总结：先问要不要，再问放哪环——帧率是预算。 |

## 知识点拆解

### ① 每个 Pass ≈ 至少一次全屏绘制

后处理链不是"声明一堆效果"，而是一条每帧执行的全屏流水线。帧预算可以粗算：

```
帧开销 ≈ 场景绘制 × 1 + Σ(每个 Pass 的全屏绘制 × 其内部趟数)
```

UnrealBloomPass 内部是半分辨率 5 级模糊（10+ 趟，但分辨率递减）；SSAOPass 还要额外生成法线/深度缓冲，是最贵的一档。叠 5 个效果 Pass，中端设备很容易跌破 30fps。

### ② 省钱两招：半分辨率与合并

- **半分辨率**：模糊类效果（Bloom、雾、辉光）对分辨率不敏感，在半分辨率甚至四分之一分辨率工作，肉眼几乎无差；
- **合并**：暗角、噪点、色调分离、饱和度这类"单次采样、逐像素变换"的小效果，可以合进**同一个自定义 ShaderPass**，一次全屏绘制做 N 件事。

### ③ 砍并实战

| 原方案 | 优化后 | 省下的开销 |
| --- | --- | --- |
| SSAOPass | 烘焙 AO 贴图到模型 uv2 | 最大的一档直接归零 |
| FilmPass + 暗角 ShaderPass | 合并为一个 ShaderPass | 省一次全屏绘制 |
| GlitchPass 常驻 | 事件触发时才 enabled | 常驻开销归零 |

优化后链路：`RenderPass → UnrealBloomPass → StylePass(噪点+暗角) → OutputPass`，画面观感几乎不变，帧率 20 → 60。

### ④ 顺序与收尾的纪律

- `RenderPass` 永远第一，`OutputPass` 永远最后且只加一个；
- 效果之间的顺序用一句话判断：**"这个效果应不应该看到那个效果的结果？"**（Bloom 在 Afterimage 前，拖尾就带辉光）；
- 临时关效果用 `pass.enabled = false`，不要反复增删 Pass。

## 坑点清单（FAQ）

- **Q：threshold 太低会怎样？** 画面中大部分区域都超过阈值参与泛光，结果满屏糊成一片、亮度爆炸。抬高阈值，只让真正的发光体起辉。
- **Q：resize 后辉光/暗角错位？** 只调了 `renderer.setSize`，忘了 `composer.setSize`——渲染目标还是旧尺寸。两个都要调，相机也不要漏。
- **Q：接上 composer 后颜色发灰/发暗？** 九成是 OutputPass 缺失或没放链尾：色调映射与 sRGB 转换没人做了。
- **Q：走 composer 后抗锯齿失效？** `antialias:true` 只对直接上屏有效；渲染目标需要在 WebGL2 下用 `{ samples: 4 }` 的 WebGLRenderTarget 构造 composer，或链尾加 SMAAPass。

## 自测（Quiz）

1. 为什么说"每个 Pass 至少一次全屏绘制"？它对帧预算意味着什么？
2. Bloom 放在 Afterimage 之前与之后，视觉效果有什么差别？
3. 一条链是 `RenderPass → Bloom → OutputPass`，你想加噪点和暗角，性能最优的做法是什么？

<details>
<summary>答案</summary>

1. 每个 Pass 都要把上一环输出的整张纹理逐像素读入、加工、写出，这是一次覆盖全屏的片元计算。意味着 Pass 数量直接线性吃掉帧时间，移动端每环都要掂量。
2. Bloom 在前：Afterimage 混合的是"已泛光"的画面，拖尾带辉光，更梦幻；Afterimage 在前：Bloom 只对当前帧最亮处起辉，拖尾不发光，更干净可控。
3. 把噪点和暗角合并进**一个自定义 ShaderPass**（一次全屏绘制做两件事），放在 Bloom 之后、OutputPass 之前；比挂两个独立 Pass 省一次全屏绘制。
</details>
