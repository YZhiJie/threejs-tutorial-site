# EP.16 一万栋楼与一次 draw call

> 对应正文：[16.1 基础入门：性能瓶颈认知与读数](../guide/16-large-scene/basics.md)
> 延伸阅读：[进阶篇](../guide/16-large-scene/advanced.md) · [实战篇](../guide/16-large-scene/practice.md)
> 动手演示：[drawcall-benchmark 独立 Mesh vs InstancedMesh](../public/demos/large-scene/drawcall-benchmark.html) · [lod-demo LOD 距离分级](../public/demos/large-scene/lod-demo.html) · [city-10k 万级实例城市](../public/demos/large-scene/city-10k.html)

## 剧情回顾

| 格 | 场景 |
| --- | --- |
| ① 困境 | 像素酱的 3D 城市只有 3000 栋楼，每栋楼才 12 个三角形，页面却卡成 PPT，拖动视角都顿挫。 |
| ② 揭秘 | 码叔点破：draw call 是 CPU 给 GPU"打电话"的次数，3000 栋楼 = 3000 次通话，16.6ms 的帧预算全花在通话费上；`InstancedMesh` 一次通话画全部。 |
| ③ 实证 | 像素酱把城市实例化：一万栋楼 draw calls 从 3000+ 降到 6，FPS 从 20 回到 60，三角形总数一栋没少。 |
| ④ 结案 | 码叔总结：先砍 draw call 再谈别的；附上坑点清单——needsUpdate 别忘、LOD 阈值别太密、巨型 InstancedMesh 要分块。 |

## 知识点拆解

### ① 三大瓶颈来源与帧预算

| 瓶颈 | 瓶颈在哪一端 | 直接读数 |
| --- | --- | --- |
| draw call 次数 | CPU → GPU 通信 | `renderer.info.render.calls` |
| 三角形总数 | GPU 顶点/像素 | `renderer.info.render.triangles` |
| 纹理与显存 | GPU 显存 | `renderer.info.memory.geometries / textures` |

60fps 意味着每帧只有 **16.6ms** 预算，CPU 提交与 GPU 渲染合起来不能超时。3000 栋楼的场景中，几何明明很轻，帧率却崩——瓶颈是 CPU 侧的 3000 次 draw call，不是显卡。

### ② InstancedMesh：一次通话画全部

```js
const city = new THREE.InstancedMesh(boxGeo, mat, 10000)
city.setMatrixAt(i, dummy.matrix) // 每实例位置/旋转/缩放
city.setColorAt(i, color)         // 每实例颜色
```

- **instanceMatrix**：每实例一个 4×4 矩阵，GPU 在顶点着色器里按实例序号取用；
- **instanceColor**：每实例颜色，不需要材质开 vertexColors；
- draw call **不随实例数增长**——一万栋楼也是一次提交。

### ③ 实证数据：省的是通话费不是几何量

| 指标 | 独立 Mesh ×3000 | InstancedMesh ×10000 |
| --- | --- | --- |
| draw calls | ~3000+ | **个位数** |
| FPS | ~20，拖动顿挫 | **稳定 60** |
| triangles | 3.6 万 | 12 万（反而更多） |

三角形更多的实例化版本反而更流畅——因为卡顿的根源从来不是几何量，而是 CPU-GPU 通话次数。

### ④ 配套药方：LOD 与视锥剔除

- **THREE.LOD**：`addLevel(mesh, distance)` 按相机距离切档，远处的楼用 12 个三角形的盒子就足够；
- **视锥剔除**：默认开启，但粒度是物体级——超大 InstancedMesh 只有 1 个包围球，要按街区**分块**才能让视野外城区整帧跳过。

## 坑点清单（FAQ）

- **Q：setMatrixAt 改了矩阵但画面没动？** 写完矩阵必须 `mesh.instanceMatrix.needsUpdate = true`；改颜色同理，`instanceColor.needsUpdate = true`。
- **Q：LOD 切换瞬间模型跳变明显？** 加宽相邻档位的过渡距离、增加中间档，必要时做透明度淡入淡出。
- **Q：draw call 多少算多？** 经验值中端手机 < 500；更可靠的判据是 Stats 的 MS 页——CPU 提交吃掉大半 16.6ms 预算就该动手了。
- **Q：renderer.info 读数什么时候清零？** 每次 `render()` 自动 reset（`info.autoReset` 默认 true）；多 pass 渲染要关 autoReset 并手动 `info.reset()`。

## 自测（Quiz）

1. 3000 个独立 Mesh、每个只有 12 三角形，为什么也会卡？瓶颈在 CPU 还是 GPU？
2. 一万栋楼的 InstancedMesh 只有 1 个包围球，对视锥剔除有什么影响？怎么补救？
3. 实例颜色改完不生效，最少需要补哪行代码？

<details>
<summary>答案</summary>

1. 瓶颈在 **CPU**：每栋楼一次 draw call，3000 次状态准备与提交把 16.6ms 帧预算吃光。GPU 渲染的三角形总量很小，并不忙。解法：InstancedMesh 一次提交。
2. 视锥剔除以**物体**为单位，1 个巨型 InstancedMesh 要么全画要么全不画，剔除基本失效。补救：按空间**分块**成多个 InstancedMesh（如每街区一个），并给每个块设置准确的 `boundingSphere`。
3. `mesh.instanceColor.needsUpdate = true`（首次 `setColorAt` 后 instanceColor 缓冲才创建，改完必须标记上传）。
</details>
