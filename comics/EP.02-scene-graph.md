# EP.02 场景图与层级矩阵

> 对应正文：[2.1 基础入门：场景图与 3D 坐标系统](../guide/02-scene/basics.md)
> 延伸阅读：[进阶篇](../guide/02-scene/advanced.md) · [实战篇](../guide/02-scene/practice.md)
> 动手演示：[scene-graph 场景图与坐标](../public/demos/scene/scene-graph.html) · [solar-system 三层 Group 太阳系](../public/demos/scene/solar-system.html) · [hierarchy-fog 层级与雾](../public/demos/scene/hierarchy-fog.html)

## 剧情回顾

| 格 | 场景 |
| --- | --- |
| ① 困境 | 标签君让月亮绕着地球转，地球又绕着太阳转，手写三角函数算坐标算到崩溃，月亮还经常"私奔"。 |
| ② 揭秘 | 码叔画出场景图：Scene 是一棵树，每个节点都有自己的局部坐标，渲染时沿父链把变换**累乘成世界矩阵**。 |
| ③ 实证 | 像素酱搭出三层 Group：太阳组 → 地球轨道组 → 月球轨道组，每层只负责自己的自转/公转，月亮再也不会跑丢。 |
| ④ 结案 | 码叔总结：position 只是"相对父节点"的话；世界坐标用 `getWorldPosition` 读。 |

## 知识点拆解

### ① Scene 不是画布，是一棵树

`scene` 是根节点，Mesh、Group、Light、Camera 都是树上的节点。Three.js 每帧从根出发 `traverse` 整棵树，把每个节点的局部变换（平移/旋转/缩放）与所有祖先的变换相乘，得到该节点的**世界矩阵**，再送进 GPU。

推论：**父节点的一切变换都会传递给子孙**。父 Group 移动 5 米，整组子物体一起平移；父节点缩放 2 倍，子节点的世界速度也被放大。

### ② Object3D：所有节点的共同基类

Scene、Mesh、Group、Light、Camera 都继承自 `Object3D`，所以都会：

- 五个变换属性：`position`、`rotation`（欧拉角）、`scale`、`quaternion`、`visible`；
- 三个树操作：`add()`、`remove()`、`traverse()`；
- 世界值读取：`getWorldPosition(v)`、`getWorldQuaternion(q)`。

### ③ 局部坐标 vs 世界坐标

`mesh.position` 存的永远是**相对父节点**的局部坐标。父 Group 在 (5,0,0)，子 mesh 局部坐标 (1,0,0)，它的世界坐标就是 (6,0,0)：

```js
const worldPos = new THREE.Vector3()
mesh.getWorldPosition(worldPos) // 沿父链累乘，拿到真实世界位置
```

### ④ 用 Group 搭太阳系：每层只管一件事

```js
const solar = new THREE.Group()      // 太阳组：可整体平移
const earthOrbit = new THREE.Group() // 地球轨道组：自转=公转
const moonOrbit = new THREE.Group()  // 月球轨道组
solar.add(sun)
solar.add(earthOrbit)
earthOrbit.add(earth)
earthOrbit.add(moonOrbit)
moonOrbit.add(moon)
earth.position.set(6, 0, 0)
moon.position.set(1.5, 0, 0)
// 循环里：earthOrbit.rotation.y += 0.01（月球自动跟着走）
```

层级旋转天然复合出"月球绕地球、地球绕太阳"的复合运动，无需手写任何嵌套正弦函数。

## 坑点清单（FAQ）

- **Q：position 设了为什么物体不动？** 依次检查：物体是否已 `add` 进树；是否每帧又被动画代码覆盖；父节点是否本身有旋转/缩放，把局部坐标变到了意料之外的位置。
- **Q：traverse 时给所有节点开阴影，灯也被改了？** `traverse` 会遍历到 Light/Camera/Group。处理前用 `if (obj.isMesh)` 过滤。
- **Q：子节点需要跟随父节点旋转，但不要继承父节点的缩放？** 场景图的变换是整体累乘的，无法只继承一部分；常见解法是调整层级结构，或在每帧用世界坐标反算补偿。
- **Q：旋转出现万向节死锁？** `rotation` 欧拉角在特定角度组合会丢失一个自由度，复杂旋转改用 `quaternion` 或 `lookAt`。

## 自测（Quiz）

1. 父 Group 位于 (10, 0, 0) 且 scale 为 2，子 mesh 局部坐标为 (1, 0, 0)，它的世界坐标是？
2. 想批量给场景中所有 Mesh 开启 `castShadow`，用哪个 API？遍历时要注意什么？
3. 月球公转的"零三角函数"实现中，月球的位置为什么不需要每帧重新计算？

<details>
<summary>答案</summary>

1. **(12, 0, 0)**：世界位置 = 父位置 10 + 局部 1 × 父缩放 2。
2. `scene.traverse(obj => { if (obj.isMesh) obj.castShadow = true })`；注意 traverse 也会访问 Light/Camera/Group，必须用 **isMesh** 过滤。
3. 月球是 `moonOrbit` 组的子节点，只设置一次局部 position；每帧旋转的是它的祖先 `earthOrbit` / `moonOrbit`，世界矩阵由场景图自动累乘，子物体跟随父节点做圆周运动。
</details>
