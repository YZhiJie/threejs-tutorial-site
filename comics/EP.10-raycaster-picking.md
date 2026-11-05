# EP.10 射线拾取与拖拽

> 对应正文：[10.1 基础入门：Raycaster 与交互](../guide/10-interaction/basics.md)
> 延伸阅读：[进阶篇](../guide/10-interaction/advanced.md) · [实战篇](../guide/10-interaction/practice.md)
> 动手演示：[raycaster 点击拾取](../public/demos/interaction/raycaster.html) · [drag-controls 拖拽协作](../public/demos/interaction/drag-controls.html) · [showroom 交互展厅](../public/demos/interaction/showroom.html)

## 剧情回顾

| 格 | 场景 |
| --- | --- |
| ① 困境 | 标签君把 `e.clientX/clientY` 直接喂给 `setFromCamera`，明明点的是盒子，选中的却是地板，点屏幕角落更是完全失灵。 |
| ② 揭秘 | 码叔画出坐标转换：DOM 像素必须先换成 **NDC**（原点在画布中心、范围 -1~1、y 轴向上），相机再从眼睛穿过该像素发射一根射线，命中结果按距离排序。 |
| ③ 实证 | 像素酱接入 DragControls，发现一拖物体镜头也跟着转——在 dragstart 关掉 OrbitControls、dragend 恢复，世界清净；俯视拖拽再用 XZ 平面求交并锁住 y。 |
| ④ 结案 | 码叔总结：屏幕没有深度，射线负责补上；拾取是几何题，不是 DOM 事件题。 |

## 知识点拆解

### ① 像素 → NDC → 射线

```js
const rect = renderer.domElement.getBoundingClientRect()
const ndc = new THREE.Vector2(
  ((e.clientX - rect.left) / rect.width) * 2 - 1,   // x: 0..宽 → -1..1
  -((e.clientY - rect.top) / rect.height) * 2 + 1   // y 翻转：DOM 向下为正
)
raycaster.setFromCamera(ndc, camera)
const hits = raycaster.intersectObjects(pickables, false)
```

- **必须减去 rect 偏移**：页面有边距、滚动、容器不是全屏时，直接用 clientX/Y 会整体偏移；
- **y 必须取反**：DOM 坐标向下增长，NDC 向上增长；
- 命中数组按距离从近到远排序，取 `[0]`；每项含 `object`、`point`（命中点）、`face`、`uv`，实例化网格还有 `instanceId`。

### ② 拖拽：与镜头控制器的休战协议

DragControls 和 OrbitControls 都监听指针事件，会同时响应：

```js
drag.addEventListener('dragstart', () => { controls.enabled = false })
drag.addEventListener('dragend',   () => { controls.enabled = true  })
```

TransformControls（Gizmo 变换工具）使用完全相同的模式让出控制权。

### ③ 俯视/平面拖拽

不想用 DragControls 时，手动与一张数学平面求交：

```js
const plane = new THREE.Plane(new THREE.Vector3(0, 1, 0), 0) // y=0 的 XZ 平面
const p = new THREE.Vector3()
raycaster.ray.intersectPlane(plane, p)
mesh.position.x = p.x
mesh.position.z = p.z   // 显式不写 y，避免俯视时上下飘
```

### ④ click 与拖拽的判定

pointerup 时若相对 pointerdown 位移超过约 5px，视为拖拽而非点击，取消拾取/弹信息面板，避免"转完镜头误开详情"。信息面板（文字、按钮）始终用 **HTML 覆盖层**实现，不要在 3D 里造 UI。

## 坑点清单（FAQ）

- **Q：点击位置整体偏移？** 没用 `getBoundingClientRect()` 减掉画布在页面中的位置；检查是否有滚动/居中容器。
- **Q：点地板能选中，点 UI 也穿透？** pointerdown 落在 HTML 面板上时 `stopPropagation` 或判断 `e.target` 不是 canvas 就直接返回。
- **Q：上万个物体 intersectObjects 卡顿？** 用 InstancedMesh（一次求交，读 `instanceId` 区分）、空间加速结构（Octree / three-mesh-bvh）缩小候选集，而不是逐网格穷举。
- **Q：鼠标悬停高亮频繁创建材质卡顿？** 复用一个高亮材质或只改 `emissive`，移出时恢复原状态。
- **Q：拖拽时物体穿到地面以下？** 与水平平面求交并固定 y，或对结果 y 做 clamp。

## 自测（Quiz）

1. 写出 clientX/clientY 转 NDC 的两条公式，并解释 y 为什么取反。
2. DragControls 与 OrbitControls 同时响应指针时，用哪两个事件、改哪个属性解决？
3. 拾取一万棵树（InstancedMesh）时，如何知道射线具体命中了哪一棵？

<details>
<summary>答案</summary>

1. `ndc.x = (clientX - rect.left) / rect.width * 2 - 1`；`ndc.y = -(clientY - rect.top) / rect.height * 2 + 1`。NDC 的 **y 轴向上为正**，而 DOM 事件坐标向下增长，所以要翻转。
2. 监听 **`dragstart`** 设 `controls.enabled = false`，监听 **`dragend`** 设回 **`true`**；让 OrbitControls 在拖拽物体期间让出指针控制权。
3. `intersectObjects` 返回的命中项里带 **`instanceId`**，它就是被命中实例在 setMatrixAt 时写入的索引，据此高亮/操作对应那一棵树即可，无需一万个独立 Mesh。
</details>
