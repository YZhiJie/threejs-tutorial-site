# EP.03 相机与视锥体

> 对应正文：[3.1 基础入门：透视相机与视锥体](../guide/03-camera/basics.md)
> 延伸阅读：[进阶篇](../guide/03-camera/advanced.md) · [实战篇](../guide/03-camera/practice.md)
> 动手演示：[perspective-fov 视场角实验](../public/demos/camera/perspective-fov.html) · [orthographic-switch 透视正交切换](../public/demos/camera/orthographic-switch.html) · [four-viewports 四视口编辑器](../public/demos/camera/four-viewports.html)

## 剧情回顾

| 格 | 场景 |
| --- | --- |
| ① 困境 | 标签君想做一条无限贴地公路，地面和路面贴在一起后满屏斑马纹闪烁，转个角度就互相穿模。 |
| ② 揭秘 | 码叔画出视锥体：相机看得见的空间是个被 near/far 削平的四棱锥；斑马纹是 near 太小导致的深度缓冲精度问题（z-fighting）。 |
| ③ 实证 | 像素酱对比透视与正交相机：一个近大远小有空间感，一个尺寸不变适合工程图；再用 `setViewport` 拼出四视图编辑器。 |
| ④ 结案 | 码叔总结：near 尽量大、far 按需设；改任何投影参数都要 `updateProjectionMatrix()`。 |

## 知识点拆解

### ① PerspectiveCamera 的四个参数

```js
new THREE.PerspectiveCamera(fov, aspect, near, far)
```

- **fov**：垂直视场角（度）。变小是望远镜（透视压缩），变大是广角（边缘变形），人眼约 60°；
- **aspect**：宽高比，应等于画布宽/高，不匹配画面会拉伸；
- **near / far**：近、远裁面距离，只有夹在视锥体内的物体才渲染（视锥剔除）。

fov 改变的是"视锥体张角"，和移动相机都能改变取景，但机制完全不同：一个换镜头，一个换站位。

### ② z-fighting：深度缓冲的精度战争

GPU 用深度缓冲（depth buffer）记录每个像素最近的表面。near 太小、far 太大时，巨大的深度范围被压缩进有限的精度里，两个共面或极近的表面就无法区分前后，表现为闪烁的斑马纹。

经验法则：

- **near 尽量大**：室内/产品场景 `0.1` 起步，别用 0.0001；
- **far 按需要设**：看得见 100 米就别设 10000；
- 必须共面的两个面（如贴花），用 `polygonOffset` 或手动拉开极小距离。

### ③ 透视 vs 正交

| | PerspectiveCamera | OrthographicCamera |
| --- | --- | --- |
| 平行线 | 远处汇聚（近大远小） | 永远平行（尺寸不变） |
| 适用 | 游戏、产品展示、写实场景 | CAD、2.5D、编辑器顶/前/侧视图 |
| 参数 | fov | left/right/top/bottom |

### ④ 一帧多相机：四视口

同一渲染循环里反复"设置视口 → 渲染"即可在一个 canvas 上分屏：

```js
renderer.setScissorTest(true)
renderer.setViewport(x, y, w, h)
renderer.setScissor(x, y, w, h)
renderer.render(scene, camera)
```

四视口演示用一台透视 + 三台正交（顶/前/侧）模拟 3D 建模软件布局。

## 坑点清单（FAQ）

- **Q：改了 camera.fov 画面没反应？** fov/aspect/near/far/zoom 任何一个改动后都必须 `camera.updateProjectionMatrix()`。
- **Q：OrbitControls 下 camera.lookAt 不生效？** 控制器每帧用 `controls.target` 覆盖相机朝向，应该改 target 而不是手动 lookAt。
- **Q：正交相机下滚轮缩放没反应？** 正交相机的"缩放"靠缩放 view 尺寸或调 `camera.zoom`，OrbitControls 已内置支持。
- **Q：resize 后画面变形？** aspect 与容器宽高比失配，重算 aspect 并更新投影矩阵。

## 自测（Quiz）

1. 地面上的斑马线闪烁属于什么现象？最有效的两个缓解手段是什么？
2. 想做一个建筑平面图（不允许近大远小），应该选哪种相机？
3. fov 从 50 调到 20 但相机不动，画面里的物体会怎样？为什么？

<details>
<summary>答案</summary>

1. **z-fighting（深度缓冲精度不足）**；手段：**near 尽量取大**、**far 按实际需要收紧**；贴花类可加 polygonOffset。
2. **OrthographicCamera 正交相机**，平行线保持平行、尺寸不随距离变化。
3. 物体看起来被**放大、视野变窄**（望远镜效果）。fov 变小使视锥体张角收窄，同样屏幕容纳的场景变少，物体占比变大，但透视关系没有改变。
</details>
