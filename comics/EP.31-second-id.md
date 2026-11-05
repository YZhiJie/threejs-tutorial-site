# EP.31 GPU 的第二张身份证

> 对应正文：[31.1 基础入门：WebGPU 是什么与能力探测](../guide/31-webgpu/basics.md)
> 延伸阅读：[31.2 进阶应用：WebGPURenderer 与 WGSL 管线](../guide/31-webgpu/advanced.md) · [31.3 实战案例：可热切换的渲染后端](../guide/31-webgpu/practice.md)
> 动手演示：[capability-lab 能力探测台](../public/demos/webgpu/capability-lab.html) · [webgpu-renderer-lab 程序化 IBL](../public/demos/webgpu/webgpu-renderer-lab.html) · [raw-wgsl-lab 原生 WGSL 三角形](../public/demos/webgpu/raw-wgsl-lab.html)

## 剧情回顾

| 格 | 场景 |
| --- | --- |
| ① 困境 | 标签君听说"WebGPU 要来了"，担心整个项目要推倒重写；像素酱拦下他：先探测设备，再决定要不要动。 |
| ② 揭秘 | 码叔把 WebGL 与 WebGPU 的差异画成四组对照：全局状态换成管线对象、隐式绑定换成 bind group、GLSL 换成 WGSL、同步创建换成异步初始化。同一套 Scene 代码，只是换了一套底盘。 |
| ③ 实证 | 像素酱打开能力探测台：adapter 信息、关键 limits、features 列表一目了然，结论是"建议 WebGPU"；切到不支持的机器，页面继续用 WebGL2，画面照样能用。 |
| ④ 结案 | 码叔写下迁移三原则：新项目试水、旧项目保留回退、自定义 GLSL 单独评估。结论只有一句：了解即可，不纳入考核，迁移成本可控。 |

## 知识点拆解

### ① WebGPU 不是"更快的 WebGL"，是另一套接口

WebGL 的核心是一台**全局状态机**：`useProgram`、`bindTexture`、`enable` 一个个开关地设，驱动在背后替你推断资源绑定。WebGPU 把这一切显式化：管线是不可变对象、资源要打包成 bind group、初始化是异步的、计算着色器是一等公民。

这不是"版本升级"，而是**换了一套底盘**。好处是 CPU 侧的驱动开销更低、资源可见性可预测、compute 能直接上车；代价是代码更长、概念更多。所以 Three.js 的 `WebGPURenderer` 不是可有可无的封装，而是让业务代码继续写 `Scene`、`Mesh`、`Material` 的关键一层。

### ② 能力探测的三段式与降级链

支持不支持，不看 UA，只看运行时：

| 步骤 | 检查 | 失败时 |
| --- | --- | --- |
| ① | `navigator.gpu` 是否存在 | 继续 WebGL2 |
| ② | `requestAdapter()` 是否返回 adapter | 继续 WebGL2 |
| ③ | `requestDevice()` 是否成功 | 继续 WebGL2 |

拿到 adapter 后，再看 `adapter.info`、`adapter.limits`、`adapter.features`。`limits` 决定"能不能跑"，`features` 决定"能不能开高画质"，`info` 只用来分维度上报，不要当精确设备指纹。探测要带超时，不能让首屏等 GPU；**先出画面，再升级后端**，才是对用户友好的顺序。

### ③ `WebGPURenderer` 的启动时序与自动回退

`WebGPURenderer` 的启动不是 `new` 完就能画：

```
new WebGPURenderer → setSize → await init() → 挂 canvas → setAnimationLoop
```

`await init()` 之前不要渲染，因为它要申请 adapter 与 device；`init()` 可能失败，所以必须捕获 Promise 拒绝并准备回退。Three.js 内置了自动回退：没有 WebGPU 时自动使用 WebGL2 后端。想主动对照，就传 `forceWebGL: true`——注意它是**构造参数**，运行时改不了，想切换只能销毁重建。

判断实际后端不要靠猜：

```js
const isWebGPU = renderer.backend.isWebGPUBackend
```

这个字段应该同时用于 UI 展示与埋点，它是"后端选择的唯一真相"。

### ④ 迁移三原则：把风险关在笼子里

- **新项目试水**：新项目可以直接用 `WebGPURenderer`，保留 WebGL2 回退；旧项目没有明确收益就先不动。
- **旧项目保留回退**：后端选择要收敛到 `createRenderer()`，不要让 `if (navigator.gpu)` 散落在业务代码里。
- **自定义 GLSL 单独评估**：GLSL 不会自动变成 WGSL，`ShaderMaterial`、`onBeforeCompile` 注入、`EffectComposer` 链都是硬成本，必须单独排期。

标准材质会被自动转换成 NodeMaterial，业务代码改动很小；真正的大头是自定义着色器与后处理。把这条边界说清楚，"迁移成本可控"才不是一句空话。

## 坑点清单（FAQ）

- **Q：浏览器不支持 WebGPU，页面会白屏吗？** 不会，只要你按"探测 → 回退"的顺序写。`WebGPURenderer` 本身也会自动回退到 WebGL2；页面层面再补一句提示文案即可。
- **Q：WebGPU 一定更快吗？** 不一定。它主要降低 CPU 驱动开销、提供 compute 能力；如果瓶颈在像素填充或带宽，换后端不会有肉眼可见的帧率提升。先 profiler，再谈迁移。
- **Q：自定义 GLSL 材质怎么办？** 不能直接复用。按成本从低到高：标准材质 + 参数调节 → NodeMaterial / TSL 重写 → 原生 WGSL 手写。不要指望自动翻译。
- **Q：移动端能用吗？** 部分机型可以，但差异很大。统一用运行时探测 + WebGL2 回退，不要按 UA 写死；同时关注发热与耗电。
- **Q：`forceWebGL` 和自动回退有什么区别？** 一个是主动选择，一个是被动兜底。埋点时必须区分，否则回退率会被高估，"用户手动降级"会被误判成"兼容性故障"。
- **Q：热切换后画面变黑，先查什么？** 查四件事：旧循环有没有停、后处理与 RenderTarget 有没有重建、`ShaderMaterial` 有没有被带过去、`init()` 有没有完成。

## 自测（Quiz）

1. WebGPU 与 WebGL 在状态管理、资源绑定、着色器语言上的核心差异分别是什么？
2. 写出能力探测的三段式，并说明每一步失败时应该落到哪条后端。
3. 如果团队决定"部分模块试用 WebGPU"，你会怎样设计回退与埋点，保证每一步都能回滚？

<details>
<summary>答案</summary>

1. 状态管理：WebGL 是全局状态机，逐次设置开关；WebGPU 是不可变管线对象，创建时一次性烘焙状态。资源绑定：WebGL 隐式绑定、驱动自动推断；WebGPU 显式 bind group + 管线布局。着色器语言：WebGL 用 GLSL / ESSL；WebGPU 用 WGSL。此外 WebGPU 原生支持 compute，初始化是异步的。
2. ① 判断 `navigator.gpu`，不存在就继续 WebGL2；② `requestAdapter()`，返回 null 就继续 WebGL2；③ `requestDevice()`，失败就继续 WebGL2。三步都成功才使用 WebGPU；探测要带超时，不能阻塞首屏。实际后端用 `renderer.backend.isWebGPUBackend` 确认。
3. 先做能力探测与埋点、不改渲染路径；把后端收敛到 `createRenderer()` 返回的统一接口；用功能开关或 URL 参数只对内部用户开放；逐模块迁移，每个模块保留独立回滚；埋点记录 `requested` 与 `actual`、失败原因、探测耗时与切换耗时；后处理与自定义着色器单独排期，不跟着基础渲染一起切。
</details>
