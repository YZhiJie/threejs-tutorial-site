# EP.11 综合实战与性能优化

> 对应正文：[11.1 基础入门：项目整合与性能优化](../guide/11-capstone/basics.md)
> 延伸阅读：[进阶篇](../guide/11-capstone/advanced.md) · [实战篇](../guide/11-capstone/practice.md)
> 动手演示：[bootstrap 分层启动骨架](../public/demos/capstone/bootstrap.html) · [integrated-scene 整合场景](../public/demos/capstone/integrated-scene.html) · [optimization-lab 两千立方体实验台](../public/demos/capstone/optimization-lab.html)

## 剧情回顾

| 格 | 场景 |
| --- | --- |
| ① 困境 | 标签君摆了 2000 个独立立方体，`renderer.info.render.calls` 也是 2000，帧率掉到个位数。 |
| ② 揭秘 | 码叔出示合批三选一：独立 Mesh 2000 calls；InstancedMesh ≈1 call 且逐实例可动可拾取；mergeGeometries ≈1 call 但是静态烘焙。先回答"它们还要不要单独动"。 |
| ③ 实证 | 像素酱把代码拆成 App → Stage → WorldBuilder → Loop 五层，循环里钳制 dt、用 info 面板实时盯 calls/triangles，远处加 LOD、离场物体走 dispose，帧率回升。 |
| ④ 结案 | 码叔总结：性能优化不靠手感，靠 calls 和 triangles 两个指标驱动决策；毕业快乐，从一个黑屏到一座能跑的城。 |

## 知识点拆解

### ① 五层架构：单向依赖

| 层 | 职责 |
| --- | --- |
| **App** | 入口与装配：读取配置、new 出下面四层并串联 |
| **Stage** | renderer / scene / camera / controls 与画布、resize、pixelRatio |
| **WorldBuilder** | 只负责"往场景里放东西"：灯光、地面、物体、合批策略 |
| **Loop** | setAnimationLoop、dt 计算与钳制、更新与渲染顺序 |
| （交互层） | Raycaster / DragControls，只读 Stage 与 WorldBuilder 暴露的对象 |

依赖方向严格单向（App 持有一切，下层不反向 import 上层），加功能时知道该改哪一层，避免"所有逻辑糊在一个 main.js"。

### ② dt 钳制

```js
const dt = Math.min(timer.getDelta(), 0.05) // 切后台/卡帧后最多前进 50ms
```

防止标签页切回时物体穿越、碰撞系统爆炸。

### ③ 合批三选一

| 方案 | calls | 可否单独动 | 适用 |
| --- | --- | --- | --- |
| 2000 个独立 Mesh | 2000 | ✅ 完全独立 | 物体数量少、材质各异 |
| **InstancedMesh** | ≈1 | ✅ 逐实例矩阵/颜色 | 草丛、人群、树、可拾取阵列 |
| **mergeGeometries** | ≈1 | ❌ 烤成一个网格 | 静态建筑、远景、地形装饰 |

```js
// InstancedMesh 写入后必须通知 GPU
instanced.setMatrixAt(i, matrix)
instanced.setColorAt(i, color)
instanced.instanceMatrix.needsUpdate = true
if (instanced.instanceColor) instanced.instanceColor.needsUpdate = true
```

合并的前提是**材质相同**；不同材质的网格无法合为一次绘制。

### ④ LOD 与资源释放

- **LOD（细节层次）**：`lod.addLevel(meshHigh, 0)`、`addLevel(meshMid, 30)`、`addLevel(meshLow, 80)`，相机距离自动切换模型精度；
- **dispose 三件套**：离场对象遍历释放 `geometry.dispose()`、`material.dispose()`，材质中的贴图还要 `texture.dispose()`；释放后看 `renderer.info.memory` 验证；
- **量化决策**：每帧读 `renderer.info.render.calls` 与 `.triangles`，优化前后用数字对比，不靠"感觉流畅了"。

### ⑤ 上线清单

- `setPixelRatio(Math.min(devicePixelRatio, 2))` 封顶，移动端可降到 1.5；
- 阴影按设备分档（mapSize 1024/2048）；
- 移动端真机实测（桌面模拟器不能代表真实 GPU/散热）；
- 提供 WebGL 不可用时的降级提示（检测 WebGLRenderingContext 创建结果）。

## 坑点清单（FAQ）

- **Q：InstancedMesh 颜色不更新？** setColorAt 首次调用才会创建 instanceColor，之后要置 `instanceColor.needsUpdate = true`。
- **Q：merge 之后还想点选其中一把椅子？** 不行——合并后它只是一个大网格的一部分；需要独立拾取就选 InstancedMesh（读 instanceId）。
- **Q：remove(mesh) 之后显存没降？** 从场景树移除只停止渲染，geometry/material/texture 仍占 GPU 资源，必须逐一 dispose。
- **Q：calls 不高但还是卡？** 看 triangles（顶点/片元过重）、阴影 Pass 数量、后处理链与 JS 主线程耗时；calls 只反映提交次数。
- **Q：2000 个不同颜色材质为什么合批失败？** 材质不同无法合批；改用 InstancedMesh + instanceColor，或合并到带顶点色的单一材质。

## 自测（Quiz）

1. 2000 个相同材质的静态装饰柱与 2000 棵需要独立摇曳、可被点选的树，分别选哪种合批方案？
2. 使用 InstancedMesh 设置完每个实例的矩阵后，必须设置哪个属性？为什么？
3. 从场景移除一个物体并释放显存，需要 dispose 哪三类资源？用什么对象验证释放结果？

<details>
<summary>答案</summary>

1. 静态装饰柱用 **mergeGeometries（静态烘焙，≈1 calls）**；需要独立运动与拾取的树用 **InstancedMesh（≈1 calls，逐实例矩阵 + instanceId 拾取）**。独立 Mesh 都会产生约 2000 次 draw call。
2. **`instanced.instanceMatrix.needsUpdate = true`**（改过颜色还要 `instanceColor.needsUpdate = true`）。setMatrixAt 只写 CPU 端缓冲数组，需要标记后才会重新上传 GPU。
3. 逐一释放 **geometry（几何体）、material（材质）、texture（材质引用的贴图）** 三类资源的 `dispose()`；之后查看 **`renderer.info.memory`（配合 render.calls/triangles）** 确认资源真正回收。
</details>

---

## 系列完结

至此 11 话走完：**场景 → 场景图 → 相机 → 渲染循环 → 几何体 → 材质 → 纹理 → 灯光阴影 → 动画 → 交互 → 整合优化**。回看 EP.01 的黑屏四件套，你现在已经能独立搭出一座有光照、有动画、能交互、跑得动的三维城市。祝渲染愉快。
