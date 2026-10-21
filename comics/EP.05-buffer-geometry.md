# EP.05 几何体与顶点流

> 对应正文：[5.1 基础入门：内置几何体与 BufferGeometry](../guide/05-geometry/basics.md)
> 延伸阅读：[进阶篇](../guide/05-geometry/advanced.md) · [实战篇](../guide/05-geometry/practice.md)
> 动手演示：[primitives 12 种内置几何体](../public/demos/geometry/primitives.html) · [custom-geometry 自定义几何体](../public/demos/geometry/custom-geometry.html) · [vertex-morph 顶点波浪形变](../public/demos/geometry/vertex-morph.html)

## 剧情回顾

| 格 | 场景 |
| --- | --- |
| ① 困境 | 标签君每帧改顶点位置想做波浪地面，地面纹丝不动；后来波浪动了，光照却像塑料贴片一样错误。 |
| ② 揭秘 | 码叔摊开顶点属性表：几何体就是几排 float——`position` 位置、`normal` 法线、`uv` 纹理坐标，GPU 每三个顶点拼一个三角形。 |
| ③ 实证 | 像素酱每帧写 `position` 数组后标记 `needsUpdate` 重新上传 GPU，再调用 `computeVertexNormals()` 重算法线，波浪的明暗立刻跟着起伏正确变化。 |
| ④ 结案 | 码叔总结：改 CPU 端数据只是改内存，`needsUpdate` 才是"发车信号"；动了形状别忘了法线。 |

## 知识点拆解

### ① 几何体的本质：带属性的顶点数组

Three.js 不用公式描述形状，一切几何体都是 `BufferGeometry` + 若干 `BufferAttribute`：

| 属性 | 含义 | 每顶点 |
| --- | --- | --- |
| `position` | 顶点坐标 xyz | 3 float |
| `normal` | 法线方向（决定受光） | 3 float |
| `uv` | 纹理坐标 | 2 float |
| `color` | 顶点色 | 3 float（可选） |

几何体只存形状数据，可以被多个 Mesh 共享；Mesh 负责把它和材质组合起来挂进场景。

### ② 修改顶点的完整四步

```js
const pos = geo.attributes.position
for (let i = 0; i < pos.count; i++) {
  const x = pos.getX(i), z = pos.getZ(i)
  pos.setY(i, Math.sin(x * 2 + t) * Math.cos(z * 2 + t) * 0.3)
}
pos.needsUpdate = true          // ③ 通知 GPU 重新上传 position
geo.computeVertexNormals()      // ④ 形状变了，法线必须重算
```

没有 `needsUpdate = true`，CPU 数组改得再热闹，GPU 仍用旧数据渲染；不重算法线，光照方向停留在初始平面，波浪像一张发亮的贴纸。

### ③ 索引几何体与细分预算

`BoxGeometry` 这类内置几何体使用 **index 索引**：顶点只存一份，三角形用索引复用，减少内存。三角形数计算方式：

```js
const tri = geo.index ? geo.index.count / 3 : geo.attributes.position.count / 3
```

球体 `SphereGeometry(r, widthSegments, heightSegments)` 分段越高越平滑，但顶点与三角形同步增长。展示场景 32×24 通常足够，移动端可降到 16×12。

### ④ 平滑着色 vs flatShading

平滑着色在顶点间插值法线，曲面圆润；`flatShading: true` 让每个三角形使用自己的面法线，呈现棱角分明的 low-poly 质感——正多面体、水晶、低多边形地形常用。

## 坑点清单（FAQ）

- **Q：逐帧 needsUpdate 上传大网格卡顿？** 顶点流上传有带宽成本。缩小网格规模、只在需要时更新，或把形变搬进顶点着色器（GPU 计算）。
- **Q：getX/getZ 和直接读 array 有什么区别？** `getX(i)` 等价于 `array[i*3]`，语义更清晰；写入后仍需 `needsUpdate`。
- **Q：computeVertexNormals 后接缝处发黑？** 硬边/接缝需要拆分顶点（不共享索引顶点），或在导出建模时开启平滑组/法线导出。
- **Q：PlaneGeometry 默认朝向哪？** 在 XY 平面、法线朝 +Z；当地面用要 `rotation.x = -Math.PI/2`，必要时材质开 `DoubleSide`。

## 自测（Quiz）

1. 每帧修改了 position 数组但画面不变，缺的两个关键操作是什么？
2. 顶点属性 position / normal / uv 每个顶点各占几个 float？uv 为哪个阶段服务？
3. 想要棱角分明的低多边形水晶效果，材质上应该开启哪个选项？

<details>
<summary>答案</summary>

1. **`position.needsUpdate = true`**（把 CPU 新数据重新上传 GPU）与 **`geometry.computeVertexNormals()`**（形状改变后重算法线，光照才正确）。
2. position **3** 个、normal **3** 个、uv **2** 个；UV 服务于**纹理采样阶段**，告诉 GPU 每个顶点对应贴图上的哪个位置。
3. **`flatShading: true`**：每个三角形使用独立面法线，呈现明显棱面而不是平滑插值。
</details>
