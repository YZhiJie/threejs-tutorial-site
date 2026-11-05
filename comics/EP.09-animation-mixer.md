# EP.09 缓动与动画混合器

> 对应正文：[9.1 基础入门：动画系统](../guide/09-animation/basics.md)
> 延伸阅读：[进阶篇](../guide/09-animation/advanced.md) · [实战篇](../guide/09-animation/practice.md)
> 动手演示：[easing 缓动曲线对比](../public/demos/animation/easing.html) · [keyframe-mixer 关键帧混合器](../public/demos/animation/keyframe-mixer.html) · [timeline-player 时间轴播放器](../public/demos/animation/timeline-player.html)

## 剧情回顾

| 格 | 场景 |
| --- | --- |
| ① 困境 | 标签君让小球落地，`y = speed * t` 匀速直线，球像贴地飞行，没有下坠加速也没有触底回弹。 |
| ② 揭秘 | 码叔画出缓动曲线：缓动函数吃进去 0~1、吐出来还是 0~1，但线性是直线，easeOut 先快后慢，Back/Elastic 还会越过终点再回弹。 |
| ③ 实证 | 像素酱用 KeyframeTrack + AnimationClip + AnimationMixer 驱动机器人，还做了时间轴拖拽：设 `action.time` 后 `mixer.update(0)` 强制刷新，任意帧即时预览。 |
| ④ 结案 | 码叔总结：play() 只是登记，真正推进时间的是每帧 mixer.update(dt)——动画的本质是关于时间的函数。 |

## 知识点拆解

### ① 手动缓动：p = f(t)

```js
const p = elapsed / duration              // 线性进度 0~1
const eased = 1 - Math.pow(1 - p, 3)       // easeOutCubic：先快后慢
box.position.y = THREE.MathUtils.lerp(startY, endY, eased)
```

- `t` 用**真实秒数**计算，天然帧率无关；
- easeIn = 起步慢（加速），easeOut = 起步快（减速），InOut = 两头慢；
- `Back` 系列越过终点再回来（过冲），`Elastic` 指数衰减摆动（果冻感）；
- ping-pong 可直接用三角函数：`p = 0.5 - 0.5 * Math.cos(angle)`。

### ② 动画系统三件套

```js
const track = new THREE.VectorKeyframeTrack(
  '.position', [0, 1, 2], [0,0,0,  2,0,0,  0,2,0]
)
const clip = new THREE.AnimationClip('move', 2, [track])
const mixer = new THREE.AnimationMixer(robot)
const action = mixer.clipAction(clip)
action.play() // 只是登记"开始播放"，不推进时间
```

- **KeyframeTrack**：一条属性轨道（position/quaternion/scale/材质数值…），记录关键时间点上的关键值；
- **AnimationClip**：若干轨道打包成一段完整动画；
- **AnimationMixer**：播放器，按当前时间在关键帧之间插值并写回对象；
- 多个角色共享同一个 mixer，动作才能严格对齐。

### ③ 每帧必须喂时间

```js
renderer.setAnimationLoop(() => {
  mixer.update(timer.getDelta()) // 漏掉这行，play 后永远静止
  renderer.render(scene, camera)
})
```

### ④ 时间轴拖拽（scrubbing）

```js
action.paused = true
action.time = slider.value
mixer.update(0) // delta 传 0 也会按当前 time 重新计算并写入姿态
```

拖动期间设 paused，只写 `action.time` 再补一次 `update(0)`，松手不需要等下一帧。播放模式：`LoopRepeat`（默认）、`LoopOnce`（播一次）、`LoopPingPong`（往返）；`action.timeScale = -1` 即倒放。

## 坑点清单（FAQ）

- **Q：clipAction().play() 之后模型纹丝不动？** 99% 是渲染循环里漏了 `mixer.update(dt)`；play 只负责登记状态。
- **Q：拖进度条画面不跟手，松手才刷新？** 设完 `action.time` 立即调用 `mixer.update(0)` 强制求值一次。
- **Q：倒放/慢放怎么实现？** 调 `action.timeScale`：1 正常、0.5 半速、-1 倒放。
- **Q：加载的 GLB 动画名怎么找？** 模型里动画在 `gltf.animations`（AnimationClip 数组），直接 `mixer.clipAction(gltf.animations[0])`，可按 clip.name 挑选。
- **Q：两段动画切换生硬？** 用 `action.crossFadeTo(otherAction, 0.3)` 做 0.3 秒混合过渡。

## 自测（Quiz）

1. `play()` 已调用却不动，最可能漏掉哪行代码？它应该放在哪里？
2. 缓动函数的输入、输出范围各是什么？easeOutCubic 与 linear 的体感差异是什么？
3. 实现拖拽时间轴预览时，action 状态、time 与 mixer 调用分别怎么处理？

<details>
<summary>答案</summary>

1. 漏掉 **`mixer.update(deltaTime)`**；必须在**渲染循环中每帧调用**，把距上一帧的真实秒数喂给混合器，关键帧插值才会推进。
2. 输入与输出都是 **0~1**（输入为 elapsed/duration 的线性进度）。linear 匀速推进；**easeOutCubic 起步快、接近终点时减速**，适合"落下/归位"类动作。
3. 设 **`action.paused = true`** 防止自动推进；拖动时把滑块值写入 **`action.time`**；随后调用一次 **`mixer.update(0)`**，强制混合器按当前时间重算姿态并立即刷新画面。
</details>
