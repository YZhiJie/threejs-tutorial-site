# EP.17 把 for 循环搬进显卡

> 对应正文：[17.1 基础入门：Points 与粒子贴图](../guide/17-particles-gpgpu/basics.md)
> 延伸阅读：[进阶篇](../guide/17-particles-gpgpu/advanced.md) · [实战篇](../guide/17-particles-gpgpu/practice.md)
> 动手演示：[points-basic Points 粒子基础](../public/demos/particles/points-basic.html) · [gpgpu-particles GPGPU 粒子](../public/demos/particles/gpgpu-particles.html) · [data-flow-particles 万级粒子数据洪流](../public/demos/particles/data-flow-particles.html)

## 剧情回顾

| 格 | 场景 |
| --- | --- |
| ① 困境 | 像素酱用 JS for 循环逐帧更新 5 万颗粒子，每帧 5 万次运算外加整段数组重传显存，页面直接卡成 12 帧的幻灯片。 |
| ② 揭秘 | 码叔点破：别让数据在 CPU 和 GPU 之间来回跑——把粒子位置存进 float 纹理，让片元着色器当计算器，两张纹理 ping-pong 互换，数据永不出显存。 |
| ③ 实证 | 同样的物理公式搬进 GPU：10 万颗粒子零上传，稳定 60 帧，而 CPU 方案 5 万颗就只剩 12 帧。 |
| ④ 结案 | 码叔给出 GPGPU 避坑指南：先探 float 纹理扩展、uDelta 必须 clamp、粒子数就是纹素数先算显存账。 |

## 知识点拆解

### ① CPU 更新粒子的两道关卡

```js
for (let i = 0; i < 50000; i++) {
  pos[i] += vel[i] * dt // 5 万次串行运算
}
geometry.attributes.position.needsUpdate = true // 整段数组每帧重传显存
```

- **关卡一：JS 单线程串行**。5 万次循环在 16.6ms 帧预算里已经奢侈，物理稍复杂（比如两两距离 O(n²)）直接爆炸；
- **关卡二：总线搬运**。`needsUpdate` 会把整个 Float32Array 从内存复制到显存，5 万粒子 × 3 分量 × 4 字节 ≈ 600KB，每帧一趟，带宽成为第二瓶颈。

### ② 位置存纹理，片元着色器当计算器

GPGPU 的核心思想：**粒子状态留在显存，让 GPU 自己算下一帧**。

- 一张 RGBA float 纹理 = 一个巨大的 vec4 数组，每个纹素存一粒粒子的 xyz（w 存种子/寿命）；
- 往纹理上渲染一个全屏四边形，片元着色器逐纹素"读旧值 → 算新值 → 写回"——每个片元就是一粒粒子的专属计算器；
- 着色器不能读写同一张纹理，所以准备 A/B 两张**乒乓互换**：本帧读 A 写 B，下帧读 B 写 A；
- 这一切用 `GPUComputationRenderer` 三行骨架就能跑：`addVariable` 注册状态纹理与更新着色器、`setVariableDependencies` 声明互相读取关系、每帧 `gpu.compute()`。

### ③ 实证数据：省的是搬运费与串行费

| 指标 | CPU for 循环 ×5 万 | GPU 纹理计算 ×10 万 |
| --- | --- | --- |
| 每帧 JS 工作 | 5 万次运算 + 600KB 上传 | 只喂几个 uniform |
| FPS | 12（幻灯片） | 60（流畅） |
| 粒子数翻倍 | 线性变慢 | 几乎无感（并行核心分摊） |

粒子多一倍反而帧率纹丝不动——GPU 有几千个并行核心，"互相独立的粒子更新"正是它的天职。

### ④ 收尾：三条 GPGPU 规矩

1. **先探能力**：渲染到 float 纹理需要 `EXT_color_buffer_float`，老设备降级 `gpu.setDataType(THREE.HalfFloatType)`；
2. **clamp 帧间隔**：`Math.min(delta, 0.033)`——切后台回来 delta 巨大，不截断粒子会被一脚踹飞；
3. **粒子数 = 纹素数**：WIDTH×WIDTH 就是粒子上限，float 纹理每张 64×64 占 256KB，百万级之前先算显存账。

## 坑点清单（FAQ）

- **Q：粒子编译通过却纹丝不动？** 先查 `setVariableDependencies`——要读旧值的变量必须把自己写进依赖列表，漏写则采样全为 0。
- **Q：怎么确认设备支持 GPGPU？** `gpu.init()` 返回非 null 即失败；另查 `renderer.extensions.has('EXT_color_buffer_float')`，不支持就降 HalfFloatType。
- **Q：切后台再回来粒子全飞了？** 帧间隔没 clamp，巨大的 dt 让欧拉积分一步跨出场景。`uDelta = Math.min(delta, 0.033)`。
- **Q：粒子跑到远处开始抖动？** float 纹理是 32 位精度，坐标绝对值越大越粗糙；模拟空间尽量以原点为中心，别存超大世界坐标。

## 自测（Quiz）

1. CPU 方案更新 5 万粒子，每帧的两笔主要开销分别是什么？
2. 为什么 GPGPU 需要两张纹理 ping-pong，而不是一张纹理就地读写？
3. `setVariableDependencies(posVar, [velVar])` 少了 posVar 自己，会发生什么？

<details>
<summary>答案</summary>

1. ① JS 单线程串行执行 5 万次（或更多）数学运算，吃掉帧预算；② `needsUpdate` 触发整段数组从内存到显存的全量上传，吃掉总线带宽。两头烧，所以 12 帧。
2. 片元着色器**读写同一张纹理是未定义行为**——同帧内你读到的可能是旧值也可能是刚写的新值。两张纹理明确分工"本帧读 A 写 B、下帧互换"，时序干净、结果确定。
3. posVar 的着色器里采样不到自己的旧位置（纹理不会被注入为 uniform），`texture2D(texturePosition, uv)` 采到全 0 或编译失败，粒子位置无法累积更新——表现为全部瘫在原点或不动。
</details>
