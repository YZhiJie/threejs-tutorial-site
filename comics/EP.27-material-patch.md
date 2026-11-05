# EP.27 在别人的着色器里插一句嘴

> 对应正文：[27.1 基础入门：onBeforeCompile 与材质补丁](../guide/27-material-shader-advanced/basics.md)
> 延伸阅读：[27.2 进阶应用：自定义 PBR 与材质扩展架构](../guide/27-material-shader-advanced/advanced.md) · [27.3 实战案例：材质预设库与运行时切换](../guide/27-material-shader-advanced/practice.md)
> 动手演示：[onbeforecompile-lab 材质补丁台](../public/demos/material-advanced/onbeforecompile-lab.html) · [custom-pbr-lab 自定义 PBR 对比台](../public/demos/material-advanced/custom-pbr-lab.html) · [material-preset-lab 材质预设库](../public/demos/material-advanced/material-preset-lab.html)

## 剧情回顾

| 格 | 场景 |
| --- | --- |
| ① 困境 | 像素酱想给产品球加一层会波动的霓虹效果，顺手写了 `ShaderMaterial`；效果是有了，可金属不再反射环境、也不吃阴影，整颗球像贴纸。标签君指出：这是"另起炉灶"，不是扩展。 |
| ② 揭秘 | 码叔摊开内置材质的拼装图：`#include <begin_vertex>`、`<color_fragment>`、`<lights_fragment_end>` 全是可以插话的窗口，`onBeforeCompile` 就在这些窗口前排队。 |
| ③ 实证 | 三个开关分别注入顶点波动、片元染色与菲涅尔边缘光，draw call 与程序数纹丝不动——补丁没有新建材质，只是让原着色器多说了一句话。 |
| ④ 结案 | 四条纪律写进规范：差异做成 uniform、缓存键写版本号、补丁贴着 include、影子另配深度材质。 |

## 知识点拆解

### ① 另起炉灶的代价：PBR 不是白送的

`ShaderMaterial` 只拿到你写的那几行 GLSL。内置材质的 IBL 反射、灯光循环、阴影采样、雾效、色调映射、输出编码，全都在你没写的地方。所以"全息效果做出来了，但球不像产品了"，不是效果不够好，而是**整条管线被绕过了**。要保留 PBR，就得在它的流程里做加法，而不是重写它。

### ② 内置 shader 是拼装出来的

`MeshStandardMaterial` / `MeshPhysicalMaterial` 的源码由几十个 chunk 拼成。`onBeforeCompile(shader, renderer)` 在 include 展开**之前**被调用，所以你能拿到带占位符的完整字符串，再用 `replace` 把代码插进指定窗口：

| 注入点 | 阶段 | 能做什么 |
| --- | --- | --- |
| `begin_vertex` | 顶点 | 顶点波动、风摆、形变 |
| `color_fragment` | 片元·基础色 | 染色、图案、贴图混合 |
| `lights_fragment_end` | 片元·光照后 | 边缘光、sheen、自定义高光项 |
| `dithering_fragment` | 片元·输出前 | 屏幕空间噪点、暗角 |

注入点越靠后，越像"在成片上加滤镜"；越靠前，越容易被后续步骤覆盖。选点比写公式更重要。

### ③ 补丁不产生新的绘制批次

拨动三个开关时，`draw calls` 始终是"两颗球 + 地面"的 3，`program` 数也不增加。因为补丁只改了同一份 shader 的文本，没有新建材质、没有新建物体。**补丁的价值就在于此：用一份材质的能力，做三份材质的活。**

### ④ 缓存键与 uniform：补丁系统的两条命门

- 默认缓存键取 `onBeforeCompile.toString()`，闭包里的数值不在其中；两个"源码相同、参数不同"的补丁会撞键，表现为"改一个，全部跟着变"。
- 修法有两种：把差异拼进 `customProgramCacheKey`，或干脆把差异做成 uniform、让 GLSL 文本永远一致（推荐后者）。
- uniform 对象要放在材质外、由钩子挂载。重新编译时钩子会再跑一次，内部 `new` 出来的引用会失效。
- 改 `uniform.value` 不需要 `needsUpdate`；只有改 GLSL 文本、改宏、改影响 shader 结构的属性才需要。

## 坑点清单（FAQ）

- **Q：补丁写了，画面没反应？** 先确认 `onBeforeCompile` 真的被调用，再确认 `replace` 的目标字符串存在——匹配不到时它不会报错。
- **Q：只改一个物体，其他物体也变了？** `customProgramCacheKey` 撞车，复用到了同一份已编译程序。
- **Q：拖滑条就卡一下？** 在回调里调了 `needsUpdate`，触发重新编译。改 uniform 不需要它。
- **Q：球在动，影子不动？** 阴影走 `customDepthMaterial`，补丁要再给深度材质补一份顶点逻辑。
- **Q：升级 Three.js 后补丁失效？** chunk 名称或内部实现变了。补丁要贴着官方 include 写，并在版本升级时跑一次回归。

## 自测（Quiz）

1. `onBeforeCompile` 在 include 展开之前还是之后被调用？这个时序为什么决定了"字符串替换"是可行的注入方式？
2. 默认的 `customProgramCacheKey` 为什么会让两个不同补丁复用同一份程序？你打算怎么修？
3. 顶点补丁让球体动了，但影子没动。缺失的是哪一份材质？怎么补？

<details>
<summary>答案</summary>

1. 在 include 展开之前。此时 `shader.vertexShader` / `shader.fragmentShader` 里仍是 `#include <begin_vertex>` 这样的占位符，所以可以用 `replace` 精确地在占位符位置插入自己的代码；如果等展开之后再拿到字符串，注入点就被淹没在大段实现里了。
2. 默认键取 `onBeforeCompile.toString()`，只包含函数源码，不包含闭包捕获的数值。两个源码相同、只是插入了不同颜色/参数的补丁会得到同一个键。修法：把区分字段拼进 `customProgramCacheKey`，或把差异做成 uniform、让 GLSL 文本保持一致，键只保留版本号。
3. 缺的是 `customDepthMaterial`（点光源场景是 `customDistanceMaterial`）。阴影渲染用的是 `MeshDepthMaterial`，不会执行表面材质里的补丁。补法：新建一个 `MeshDepthMaterial`，用同样的顶点注入逻辑，赋给 `mesh.customDepthMaterial`。
</details>
