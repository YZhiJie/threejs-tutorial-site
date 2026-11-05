# EP.06 材质与 PBR 光影

> 对应正文：[6.1 基础入门：四种常用材质对比](../guide/06-material/basics.md)
> 延伸阅读：[进阶篇](../guide/06-material/advanced.md) · [实战篇](../guide/06-material/practice.md)
> 动手演示：[material-compare 四材质同光对比](../public/demos/material/material-compare.html) · [physical-env 环境反射实验](../public/demos/material/physical-env.html) · [material-lab 材质实验室](../public/demos/material/material-lab.html)

## 剧情回顾

| 格 | 场景 |
| --- | --- |
| ① 困境 | 标签君把 `metalness` 拉到 1 想做不锈钢球，结果球黑得像个黑洞，加灯也救不回来。 |
| ② 揭秘 | 码叔摆出四种材质：Basic 不吃光、Lambert 只有漫反射、Phong 有锐利高光、Standard 走 PBR；金属的颜色几乎全部来自**反射环境**。 |
| ③ 实证 | 像素酱用 `PMREMGenerator` 烘焙一张 `RoomEnvironment` 环境贴图赋给 `scene.environment`，金属球瞬间映出整个房间，粗糙度旋钮让反射从镜面到磨砂连续变化。 |
| ④ 结案 | 码叔总结：PBR 两个旋钮 roughness/metalness 描述物理，但反射需要"可反射的内容"。 |

## 知识点拆解

### ① 四种材质速查

| 材质 | 受灯光 | 特点 | 典型用途 |
| --- | --- | --- | --- |
| MeshBasicMaterial | ❌ | 纯色/贴图，无明暗，最便宜 | 天空盒、UI、发光条、调试 |
| MeshLambertMaterial | ✅ | 只有漫反射，无高光 | 低多边形、墙面、远景 |
| MeshPhongMaterial | ✅ | 漫反射 + shininess 锐利高光 | 老式光泽塑料 |
| MeshStandardMaterial | ✅ | PBR：roughness/metalness | **现代项目默认** |

Basic 不是"最差材质"：凡是本来就不该受光的东西（星空、灯条、包围盒），它是最正确最便宜的选择。

### ② PBR 两个核心旋钮

- **roughness 粗糙度**：0 = 镜面反射，1 = 完全粗糙的漫反射；
- **metalness 金属度**：0 = 非金属（塑料/木材/漆面），1 = 纯金属。

金属的特殊性：金属表面几乎不返回漫反射颜色，看到的颜色主要是对环境的反射。场景里**没有环境贴图时，金属球没有任何东西可反射，于是输出黑色**——这不是 bug。

### ③ RoomEnvironment：给 PBR 一面"镜子"

```js
import { RoomEnvironment } from 'three/addons/environments/RoomEnvironment.js'

const pmrem = new THREE.PMREMGenerator(renderer)
scene.environment = pmrem.fromScene(new RoomEnvironment(), 0.04).texture
```

- `scene.environment` 为所有 PBR 材质提供**基于图像的照明（IBL）**，不产生可见背景；
- PMREM 预过滤环境贴图，让不同 roughness 都能得到正确模糊度的反射；
- 想让环境图同时成为背景，再设 `scene.background = envTexture`。

### ④ 能量守恒与直觉调试

roughness 越小，高光/反射越集中清晰；越大越弥散。调材质顺序建议：先给 environment，再定 metalness（金属/非金属是离散选择），最后磨 roughness。拿不准时用同色同光对比四个材质，差异只来自光照模型。

## 坑点清单（FAQ）

- **Q：Standard 材质总是灰蒙蒙？** 高粗糙度表面缺少环境反射内容，补 `scene.environment` 立刻改善。
- **Q：Phong 的 shininess 和 roughness 是一回事吗？** 方向相反且模型不同：shininess 越大高光越锐利，但 Phong 非物理模型；新项目统一用 Standard 的 roughness。
- **Q：MeshPhysicalMaterial 与 Standard 区别？** Physical 扩展 clearcoat（清漆）、transmission（透射/玻璃）、sheen（布料绒光）等，成本更高，按需使用。
- **Q：环境贴图需要外部 HDR 图片吗？** 不需要，`RoomEnvironment` 是纯程序化的室内环境；写实项目再换 HDRLoader 加载 .hdr/.exr。

## 自测（Quiz）

1. metalness=1 的球体在纯黑场景中为什么是黑的？最小修复是什么？
2. 做一块"完全粗糙的红砖墙"和"一面梳妆镜"，roughness 分别接近几？
3. `scene.environment` 与 `scene.background` 的作用有何不同？

<details>
<summary>答案</summary>

1. 金属的可见颜色主要来自**对环境的镜面反射**，场景没有任何环境光/环境贴图时无内容可反射，因此为黑。最小修复：用 **PMREMGenerator + RoomEnvironment** 设置 `scene.environment`。
2. 砖墙 roughness 接近 **1**（漫反射、无清晰反光）；镜子接近 **0**（镜面全反射）。
3. `scene.environment` 只提供 **IBL 照明与反射来源**，画面背景不变；`scene.background` 决定**画布背景**显示什么（纯色、贴图或天空盒）。两者可以分别设置。
</details>
