# EP.23 让世界有了重力

> 对应正文：[23.1 基础入门：物理引擎与第一个下落小球](../guide/23-physics/basics.md)
> 延伸阅读：[23.2 进阶应用：刚体碰撞与角色控制器](../guide/23-physics/advanced.md) · [23.3 实战案例：3D 闯关小游戏](../guide/23-physics/practice.md)
> 动手演示：[physics-basic 第一个下落小球](../public/demos/physics/physics-basic.html) · [character-controller 角色控制器](../public/demos/physics/character-controller.html) · [mini-game 3D 闯关小游戏](../public/demos/physics/mini-game.html)

## 剧情回顾

| 格 | 场景 |
| --- | --- |
| ① 困境 | 像素酱给游戏主角手写"跳跃动画"，改了二十版缓动曲线还是一眼假，被标签君当场吐槽：像提线木偶。 |
| ② 揭秘 | 码叔在白板上画出两个世界：物理世界只算坐标，渲染世界只管画面——动画是剧本，物理是自然法则。 |
| ③ 实证 | 三十行 cannon-es：松手的小球真实弹跳，`collide` 事件顺手挂了金币音效，调参从"改曲线"变成"填物理量"。 |
| ④ 结案 | 坑点清单：步长太大穿地、渲染忘抄坐标、滥用三角面碰撞体。码叔总结：动画演轨迹，物理算轨迹。 |

## 知识点拆解

### ① 手搓动画的天花板：轨迹是写死的

- `ball.y = Math.abs(Math.sin(t)) * h` 这类式子本质是"导演排练"：每一帧在哪都是剧本写好的；
- 剧本里没有"碰撞"这个概念：球不会被别人撞偏，也不能被玩家推一把；
- 地面换个高度、场景加一阵风，剧本就得重排——物理引擎只需要改一个参数。

### ② 物理世界与渲染世界分离

| | 渲染世界 | 物理世界 |
| --- | --- | --- |
| 库 | Three.js | cannon-es / rapier |
| 组成 | Scene → Mesh | World → Body |
| 职责 | 画面好不好看 | 坐标怎么算 |
| 可见性 | 被相机拍到 | 永远不可见 |

每帧的同步纪律只有两步：先 `world.step(1/60, dt, 3)` 推进物理，再把 `body.position / quaternion` 抄给对应 Mesh。坐标系不用换算——cannon-es 与 Three.js 同为右手系、Y 轴朝上。

### ③ 三十行真实弹跳 + 碰撞事件

```js
world.step(1 / 60, dt, 3)
mesh.position.copy(body.position)
playerBody.addEventListener('collide', (e) => { /* 吃金币、播音效 */ })
```

弹性与摩擦写在 `ContactMaterial` 上，一对物理材质一份约定；把刚体的 `collisionResponse` 设为 `false`，它就变成"只发事件、不弹开"的感应区——金币和终点线都是这么做的。

### ④ 手感是调出来的

物理对了不等于手感好。两个经典技巧：**土狼时间**（走出边缘后 0.12 秒内仍能起跳）与**跳跃缓冲**（落地前 0.15 秒内按跳自动补跳），都用时间戳实现，二十行代码，把"玩具手感"变成"专业手感"。移动平台则用 kinematic 刚体 + velocity 驱动，摩擦才会真的载人。

## 坑点清单（FAQ）

- **Q：小球直接穿过地面？** 步长太大的隧道效应。固定步长 `world.step(1/60, dt, 3)`，并给 dt 加上限截断，切后台回来就不会一步跨进地里。
- **Q：画面一顿一顿，物理和渲染不同步？** `step` 要放在渲染循环内、`render` 之前，每帧一次；要丝滑就抄 `body.interpolatedPosition` 而不是 `position`。
- **Q：角色贴墙卡住或翻倒？** 角色 ↔ 墙单独建一个摩擦为 0 的 ContactMaterial；`fixedRotation = true` 锁死旋转，记得 `updateMassProperties()`。
- **Q：站上移动平台不被带走？** 平台必须是 kinematic 且每帧写 velocity 驱动；每帧瞬移 position 的平台在物理上"没有速度"，摩擦无从产生。

## 自测（Quiz）

1. 物理世界与渲染世界各自负责什么？它们之间每帧的同步顺序是什么？
2. 为什么金币要做成 `collisionResponse = false` 的感应体？收集时必须处理哪三件事？
3. 土狼时间和跳跃缓冲分别解决什么手感问题？为什么用时间戳而不是布尔标志？

<details>
<summary>答案</summary>

1. 物理世界（cannon-es 的 World/Body）按物理规律计算坐标与姿态，永远不可见；渲染世界（Three.js 的 Scene/Mesh）只负责画面。每帧先 `world.step(1/60, dt, 3)` 推进物理，再把 `body.position / quaternion` 抄给对应 Mesh，最后 `renderer.render` 出画面。
2. 感应体只发 `collide` 事件、不产生排斥力，角色穿过金币不会被弹开。收集三件事缺一不可：置 `taken` 标志去重（事件可能连发多帧）、`scene.remove` 下架网格、`world.removeBody` 物理除名（否则原地还踩着看不见的金币）。
3. 土狼时间解决"走出平台边缘按跳无反应"——离地后 0.12 秒内仍允许起跳；跳跃缓冲解决"落地瞬间按键被吞"——落地前 0.15 秒内的跳跃键落地自动补跳。时间戳把"状态"变成"事实记录"，起跳条件是纯计算 `t - lastGrounded < COYOTE`，没有标志清零的时机问题，调手感只需改一个秒数常量。
</details>
