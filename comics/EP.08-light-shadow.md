# EP.08 灯光与阴影映射

> 对应正文：[8.1 基础入门：灯光与阴影](../guide/08-light/basics.md)
> 延伸阅读：[进阶篇](../guide/08-light/advanced.md) · [实战篇](../guide/08-light/practice.md)
> 动手演示：[lights 五种光源](../public/demos/light/lights.html) · [shadow-basics 阴影三杠杆](../public/demos/light/shadow-basics.html) · [three-point 三点布光](../public/demos/light/three-point.html)

## 剧情回顾

| 格 | 场景 |
| --- | --- |
| ① 困境 | 标签君按老教程写 `new PointLight(0xffffff, 2)`，两米外的物体还是黑的——灯明明"开了"。 |
| ② 揭秘 | 码叔画出两遍渲染：阴影不是黑贴图，而是从**光源视角**先渲染一张深度图，再从相机视角逐像素比较深度；只有三种灯能投影。 |
| ③ 实证 | 像素酱调清三个杠杆（mapSize、收紧阴影相机、bias/normalBias），再摆出 Key/Fill/Rim 三点布光，产品质感直接拉满。 |
| ④ 结案 | 码叔总结：点光源 r155+ 走物理衰减，强度要几十到几百；阴影是昂贵的两遍渲染，预算省着花。 |

## 知识点拆解

### ① 物理光照：intensity=2 为什么看不见

r155 起 WebGLRenderer 默认使用物理光照单位，`PointLight` / `SpotLight` 默认 `decay = 2`（平方反比衰减）：

```
表面到达亮度 ≈ intensity / distance²
```

距离 2 米时 intensity=2 只剩 0.5，距离 5 米只剩 0.08。使用点光/聚光时强度通常要给**几十到几百**（并配合 `distance` 衰减半径）。Ambient/Hemisphere/Directional 不做距离衰减，仍是 0.x ~ 3 的小数值。

### ② 五种光源与投影能力

| 光源 | 类比 | 投影 |
| --- | --- | --- |
| AmbientLight | 均匀环境底光 | ❌ |
| HemisphereLight | 天空/地面两色半球光 | ❌ |
| DirectionalLight | 太阳（平行光） | ✅ |
| PointLight | 灯泡（全方向） | ✅（6 面深度图，成本最高） |
| SpotLight | 聚光灯（锥形光束） | ✅ |

阴影三件套缺一不可：`renderer.shadowMap.enabled = true`、`light.castShadow = true`、`mesh.castShadow = true`（接收方还要 `receiveShadow = true`）。注意 `SpotLight.target` 是个 Object3D，**必须 add 进场景**，改变它的位置才生效。

### ③ 阴影是两遍渲染

1. **深度 Pass**：从光源位置渲染一遍场景，只记录每个像素最近表面的深度（阴影图）；
2. **相机 Pass**：正常渲染时，把每个像素投影回光源空间与深度图比较，被其他表面挡住的像素就在阴影里。

默认 `PCFShadowMap` 提供柔和边缘，`VSMShadowMap` 更软但可能漏光。

### ④ 清晰阴影三杠杆

1. `light.shadow.mapSize.set(2048, 2048)`：深度图分辨率，越高越锐利，成本越高；
2. **收紧阴影相机**：DirectionalLight 的阴影是正交相机，把 left/right/top/bottom/far 收到刚好包住场景，等效分辨率大幅提升；
3. **bias / normalBias**：消除阴影痤疮（见坑点）。

### ⑤ 三点布光

- **Key 主光**：暖色、强度最高（约 2.6），唯一开启投影，负责造型；
- **Fill 补光**：冷色、较弱（约 0.9），**不投影**，抬起暗部；
- **Rim 轮廓光**：放在主体后上方，勾出一圈亮边，把主体从背景分离。

## 坑点清单（FAQ）

- **Q：阴影表面全是条纹/麻点（shadow acne）？** 平面为主调负 `shadow.bias`（如 -0.0005）；曲面为主调 `shadow.normalBias`（如 0.02），沿法线偏移深度采样。
- **Q：物体看起来悬空、影子和脚分开（peter-panning）？** bias/normalBias 调过头了，往回收。
- **Q：多盏灯都开投影，影子互相矛盾且掉帧？** 投影是昂贵的逐光源 Pass，通常只让一盏 Key 光投影，Fill/Rim 只负责照明。
- **Q：SpotLight 照的方向不对？** 忘了把 `spot.target` 加入场景树；它的位置更新后还需 `target.updateMatrixWorld()`（add 进树会自动）。

## 自测（Quiz）

1. r155+ 物理光照下，PointLight intensity=2 在 4 米处到达亮度约为多少？为什么老教程数值不亮了？
2. 哪三种灯可以投射阴影？其中哪种成本最高，为什么？
3. 阴影痤疮在平面和曲面上分别优先调哪个参数？调过头会出现什么现象？

<details>
<summary>答案</summary>

1. 约 **2 / 4² = 0.125**。默认 decay=2 平方反比衰减使老教程里"1~2 强度"的点光几乎无照明，新单位下点光/聚光强度需要**几十到几百**。
2. **DirectionalLight、SpotLight、PointLight**；**PointLight 成本最高**——它需要渲染立方体贴图的 **6 个面**深度图。
3. 平面优先 **`bias`**（小幅负值，如 -0.0005），曲面优先 **`normalBias`**（如 0.02）。调过头会导致 **peter-panning：影子与物体分离、物体像悬空**。
</details>
