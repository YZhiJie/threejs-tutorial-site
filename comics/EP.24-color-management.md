# EP.24 同一抹橙色的两种身份

> 对应正文：[24.1 基础入门：色彩空间与线性工作流](../guide/24-color-graphics/basics.md)
> 延伸阅读：[24.2 进阶应用：精度、HDR 与图形学基础原理](../guide/24-color-graphics/advanced.md) · [24.3 实战案例：跨设备一致的色彩校准工作台](../guide/24-color-graphics/practice.md)
> 动手演示：[color-space-lab 色彩空间对比台](../public/demos/color-graphics/color-space-lab.html) · [banding-precision-lab 色带与精度实验室](../public/demos/color-graphics/banding-precision-lab.html) · [color-checker 跨设备色彩校准台](../public/demos/color-graphics/color-checker.html)

## 剧情回顾

| 格 | 场景 |
| --- | --- |
| ① 困境 | 像素酱发现设计稿的橙色 `#D67E2C` 到了线上变成 `#C9772F`，标签君在另一台机器上看着更黄，两人当场为"谁的颜色对"吵起来。 |
| ② 揭秘 | 码叔画出四步色彩链路：贴图解码 → 线性计算 → 色调映射 → sRGB 编码；颜色不是感觉，是一条可以逐段检查的流水线。 |
| ③ 实证 | 摆出 24 色卡与九级灰阶，逐格读取屏幕十六进制：错误链路整块发灰，正确链路与设计值对齐。 |
| ④ 结案 | 三条纪律写成色卡：颜色贴图设 sRGB、数据贴图不解码、输出色彩空间固定；参数定版后，颜色才有据可查。 |

## 知识点拆解

### ① 同一色号，为什么会有两张脸

设计稿的 `#D67E2C` 是 sRGB 编码值，进入渲染管线后要先解码成线性值参与光照，最后再编码回 sRGB。任何一段漏掉或重复，屏幕上的十六进制就会变。**同一份贴图、同一组灯光，只要 `texture.colorSpace` 或 `renderer.outputColorSpace` 有一个错位，整块颜色就会系统性偏移**——这不是显示器坏了，是链路断了。

### ② 四步链路：进门前脱鞋，出门前穿鞋

| 步骤 | 做什么 | 对应开关 |
| --- | --- | --- |
| ① 解码 | 颜色贴图 sRGB → 线性 | `texture.colorSpace = SRGBColorSpace` |
| ② 计算 | 光照、混合、后处理全在线性空间 | 工作色彩空间（默认 Linear-sRGB） |
| ③ 色调映射 | 把 HDR 亮度压回 0~1 | `renderer.toneMapping` / `toneMappingExposure` |
| ④ 编码 | 线性 → sRGB 写给屏幕 | `renderer.outputColorSpace` |

口诀是"**进门前脱鞋，出门前穿鞋**"：贴图进门时脱掉 sRGB，屏幕出门前再穿上。中间所有房间都只认线性值。

### ③ 色卡与取色：把"我觉得"变成数字

24 色卡覆盖肤色、天空、植物、高饱和色与中性色，九级灰阶负责暴露对比度与黑位问题。排查时不看感觉，按顺序做：同一环境截图 → 取色记录十六进制 → 逐环节核对贴图、输出、色调映射、后处理 → 把确认过的参数写成一份 `COLOR_CONFIG` 定版。**能复现的差异才是可修复的差异**。

### ④ 三条纪律与一份配置

- 颜色贴图必须显式 `SRGBColorSpace`，数据贴图（法线 / 粗糙度 / AO）必须 `NoColorSpace`；
- `renderer.outputColorSpace` 固定 sRGB，除调试外不许改成 Linear；
- 色调映射与曝光从同一份配置读取，后处理链最多做一次 tone mapping。

把 `outputColorSpace`、`toneMapping`、`toneMappingExposure`、贴图色彩空间、HDR 缓冲类型收进一个带版本号的常量对象，代码评审时颜色变化就有据可查。

## 坑点清单（FAQ）

- **Q：颜色整体发灰、发淡？** 颜色贴图被当成线性数据了。设 `SRGBColorSpace`，并记得 `needsUpdate = true`。
- **Q：画面整体偏暗、饱和色发闷？** 输出漏了 sRGB 编码。`outputColorSpace` 改回 `SRGBColorSpace`，不要用曝光硬补。
- **Q：法线贴图让光照发脏？** 数据贴图被 sRGB 解码了。法线、粗糙度、AO、ORM 一律 `NoColorSpace`。
- **Q：暗部渐变出现色带？** 8 位缓冲精度不够。先加抖动，再考虑 HalfFloat RenderTarget；抖动必须在量化之前。
- **Q：用了后处理颜色变淡？** 可能做了两次色调映射。让 `OutputPass` 统一负责压缩与编码。

## 自测（Quiz）

1. 颜色贴图与法线贴图分别应该设置什么 `colorSpace`？为什么不能一视同仁？
2. 完整的四步色彩链路是什么？`toneMapping` 和 `outputColorSpace` 各自位于哪一步？
3. 团队里出现"设计稿与线上颜色不一致"，你会按什么顺序排查？最终如何避免再次发生？

<details>
<summary>答案</summary>

1. 颜色贴图设 `THREE.SRGBColorSpace`，因为它存的是给人看的 sRGB 颜色，采样时要解码成线性；法线贴图设 `THREE.NoColorSpace`（或 `LinearSRGBColorSpace`），因为它存的是方向分量，不是颜色，被解码会扭弯法线、导致光照发脏。口诀：能当颜色画出来的图用 sRGB，只能当数字读的图用 NoColorSpace。
2. 四步是：① 贴图解码（sRGB → 线性）；② 线性空间计算（光照、混合、后处理）；③ 色调映射（在仍为线性的 HDR 值上压缩）；④ 输出编码（线性 → sRGB）。`toneMapping` 属于第 ③ 步，`outputColorSpace` 属于第 ④ 步。
3. 排查顺序：统一截图环境 → 取色记录十六进制 → 逐环节核对贴图 colorSpace、输出 colorSpace、toneMapping / exposure、后处理是否重复映射 → 把确认过的参数写成 `COLOR_CONFIG` 定版 → 每次发布保留色卡基线截图做回归。避免再次发生靠的是"配置收敛 + 视觉回归"，而不是反复口头对色。
</details>
