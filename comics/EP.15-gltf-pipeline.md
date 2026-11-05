# EP.15 glTF 资产管线：200MB 模型瘦身记

> 对应正文：[15.1 基础入门：glTF 格式与 GLTFLoader](../guide/15-gltf-pipeline/basics.md)
> 延伸阅读：[进阶篇](../guide/15-gltf-pipeline/advanced.md) · [实战篇](../guide/15-gltf-pipeline/practice.md)
> 动手演示：[first-gltf 加载第一个 glTF 模型](../public/demos/gltf/first-gltf.html) · [draco-ktx2-compare 压缩管线对比](../public/demos/gltf/draco-ktx2-compare.html) · [model-viewer 模型展示页](../public/demos/gltf/model-viewer.html)

## 剧情回顾

| 格 | 场景 |
| --- | --- |
| ① 困境 | 标签君兴冲冲接过美术给的模型——200MB！网页一打开进度条爬了半分钟，浏览器直接卡死。 |
| ② 揭秘 | 码叔摊开 glTF 的结构图：JSON 场景描述 + 几何 + 贴图 + 动画全打包；再祭出两件兵器——DRACO 压几何、KTX2 压贴图。 |
| ③ 实证 | 像素酱一条 `gltf-transform` 命令跑完，200MB 瘦到 5MB，页面秒开，骨骼动画照播不误。 |
| ④ 结案 | 码叔贴出坑点清单并总结：模型不是美术导完就结束了，管线才是资产的最后一公里。 |

## 知识点拆解

### ① glTF：3D 界的 JPEG

glTF 是 Khronos 制定的开放交换标准，一种格式装下整个模型：场景层级（nodes）、网格（meshes）、PBR 材质、骨骼与动画（skins/animations）。两种形态：

| 形态 | 结构 | 适用 |
| --- | --- | --- |
| .gltf | JSON 主文件 + 外部 .bin + 外部贴图 | 调试期，可读可改 |
| .glb | 单文件二进制全打包 | **生产环境首选**，不丢文件 |

### ② 压缩两件兵器

- **DRACO（几何）**：顶点量化 + 熵编码，几何体积减 40%~90%。加载端配 `DRACOLoader` + `setDecoderPath()`，解码器首次遇到压缩模型时才按需下载；
- **KTX2/Basis（贴图）**：转成 GPU 块压缩格式，文件更小的同时**显存里也不解压**，1024² 贴图显存从 5.6MB 降到约 0.7MB。加载端配 `KTX2Loader` + `setTranscoderPath()`，**必须 `detectSupport(renderer)`**。

实测（Soldier.glb）：原始 2.06MB → +DRACO 1.19MB → 再叠 WebP 贴图 0.80MB → 贴图换 KTX2 后约 0.5MB，累计 −75%。

### ③ 动画照播：AnimationMixer

压缩只动几何与贴图的字节排布，`gltf.animations` 里的骨骼动画片段完好无损。`AnimationMixer` + `clipAction(clip).play()` 即可播放；切换动作用 `next.reset().fadeIn(0.3).play()` 配 `current.fadeOut(0.3)` 交叉过渡；千万别忘每帧 `mixer.update(delta)`。

### ④ 管线意识

从"美术导出"到"网页秒开"之间隔着一条资产管线：Blender 导出（.glb、应用变换、勾 Draco）→ gltf-transform/gltfpack 后处理（WebP/KTX2）→ 加载端装配解码器与 LoadingManager。任何一环缺失，用户看到的就是白屏或黑模型。

## 坑点清单（FAQ）

- **Q：模型加载出来全黑？** glTF 全是 PBR 材质，没有 `scene.environment` 就没有可反射的内容；用 PMREMGenerator + RoomEnvironment 补上即可。
- **Q：动画不播？** 先打印 `gltf.animations` 看是不是空（导出没勾动画）；再查渲染循环里有没有每帧 `mixer.update(delta)`。
- **Q：DRACO 报 decoder 404？** `setDecoderPath` 指目录且结尾带 `/`，目录下要有 wrapper.js + decoder.js + decoder.wasm 三件套。
- **Q：KTX2 贴图不显示？** 九成是漏了 `ktx2Loader.detectSupport(renderer)`。
- **Q：贴图 404 但本地好的？** 贴图路径相对于 .gltf 文件，且 Linux 服务器区分大小写；换 .glb 单文件可绝育此问题。

## 自测（Quiz）

1. .gltf 与 .glb 有什么区别？生产环境推荐哪个，为什么？
2. DRACO 与 KTX2 分别压缩什么？KTX2 除了文件小还有什么收益？
3. 切换骨骼动画时为什么要用 fadeIn/fadeOut 而不是直接 stop/play？

<details>
<summary>答案</summary>

1. .gltf 是 JSON 主文件 + 外部 .bin/贴图，多文件可读可改；.glb 把全部内容打包成单文件二进制。生产推荐 **.glb**：一个请求、不会漏传贴图、没有相对路径和大小写问题。
2. DRACO 压**几何**（顶点/法线/UV），KTX2 压**贴图**。KTX2 的额外收益是贴图在**显存中保持块压缩格式**不解压，显存占用降 4~8 倍，移动端收益尤其大。
3. 直接 stop/play 会让姿态瞬间跳变（还可能闪回 T-Pose）；`reset().fadeIn(0.3).play()` 与 `fadeOut(0.3)` 让新旧动作权重在 0.3 秒内平滑交叉过渡，视觉上动作是"渐变"过去的。
</details>
