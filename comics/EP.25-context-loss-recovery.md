# EP.25 显卡打了个嗝：上下文丢失急救

> 对应正文：[25.1 基础入门：WebGL 上下文丢失与自愈](../guide/25-monitoring-resilience/basics.md)
> 延伸阅读：[25.2 进阶应用：容错分级与监控上报体系](../guide/25-monitoring-resilience/advanced.md) · [25.3 实战案例：给 3D 应用装上"黑匣子"](../guide/25-monitoring-resilience/practice.md)
> 动手演示：[context-loss-lab 上下文丢失急救台](../public/demos/monitoring/context-loss-lab.html) · [telemetry-pipeline 遥测管道](../public/demos/monitoring/telemetry-pipeline.html) · [resilience-ladder 黑匣子时间线](../public/demos/monitoring/resilience-ladder.html)

## 剧情回顾

| 格 | 场景 |
| --- | --- |
| ① 困境 | 园区大屏突然黑屏，标签君在现场怎么都复现不了，像素酱一句"刷新又好了"把问题变得更玄。 |
| ② 揭秘 | 码叔把 WebGL 上下文比作 GPU 的办公桌：桌子被收走，JS 对象还在，但所有显存句柄都失效了。 |
| ③ 实证 | 两个事件接上，`preventDefault()` 报名恢复，`rebuildWorld()` 整组重建，状态机从 lost 走回 running。 |
| ④ 结案 | 坑点清单：只阻止不重建、恢复直接回 L0、不落快照。码叔总结：状态机管生死，黑匣子管证据。 |

## 知识点拆解

### ① 黑屏不是 JS 报错：上下文丢失是静默的

- 控制台不报红、`try/catch` 抓不到、`renderer.render()` 也不抛错，只是画布不再刷新；
- 用户看到的是整块黑屏，页面其它模块（图表、告警列表）往往还正常；
- "刷新一下又好了"是最典型的线索：刷新等于重新申请上下文，但现场也被一起清掉了。

### ② 两个事件：lost 是通知，restored 是机会

| 事件 | 含义 | 必须做的事 |
| --- | --- | --- |
| `webglcontextlost` | 浏览器/驱动收走了上下文 | 同步调用 `preventDefault()`，停循环、落快照、切兜底 |
| `webglcontextrestored` | 浏览器把上下文还回来了 | 重建全部 GPU 资源，再重启渲染循环 |

`preventDefault()` 只是"我愿意恢复"的报名，不是"一定会恢复"的保证。restored 可能几毫秒后来，也可能永远不来；所以既要接事件，也要有超时兜底。Three.js 内部已经替我们阻止了默认行为，但**重建资源的责任始终在业务侧**。

### ③ 恢复要用"重建函数"，不要打补丁

上下文一丢，geometry、material、texture、program、render target 的 GPU 副本全部作废。补丁式修复会漏掉后处理、GPGPU、环境贴图这些"业务自己持有"的资源，表现就是场景回来了但泛光没了、粒子不动了、贴图发黑。

正确纪律是两段式：`disposeWorld()` 只负责销毁，`buildWorld()` 只负责创建，中间用一个 `rebuildWorld()` 串起来。初始化、上下文恢复、热重载、切场景全走同一个入口，天然幂等，也天然可测试。

### ④ 状态机 + 黑匣子：让自愈和自证同时成立

容错逻辑一旦散落在 resize、事件回调、定时器里，就会互相踩脚。把它们收敛成五个状态：`healthy`、`degraded`、`suspended`、`recovering`、`failed`，每次迁移都写事件日志。

同时用环形缓冲保留最近 30 秒帧时间，挂起/失败时把 `renderer.info`、当前档位、最近事件一起落成快照。恢复时回到崩溃前的档位，而不是直接回 L0——刚出过事的设备，经不起全画质的二次冲击。

## 坑点清单（FAQ）

- **Q：调了 `preventDefault()` 还是回不来？** 它只代表"有资格恢复"，显存耗尽、GPU 进程崩溃、驱动不可恢复都会让 restored 永远不来；要加恢复超时和静态兜底。
- **Q：恢复后贴图发黑、泛光消失？** 旧纹理与 render target 的 GPU 副本已经失效。不要复用 `dispose()` 过的对象，整组重建最省心。
- **Q：恢复时直接回 L0 行不行？** 不行。回到崩溃前档位先站稳，再按迟滞规则逐级升回，否则很容易二次崩溃。
- **Q：用户说黑屏但本地复现不了？** 先查快照里的 `state`、`frame.p95`、`renderer.info`，再对齐 GPU、DPR、是否切过后台；最后用 `WEBGL_lose_context` 主动触发一次，验证恢复路径本身是否通。

## 自测（Quiz）

1. 为什么 `webglcontextlost` 里的 `preventDefault()` 必须同步调用？Three.js 已经调用了它，业务侧还需要做什么？
2. 上下文恢复后，哪些资源必须重建？为什么"补丁式修复"容易漏掉后处理与粒子系统？
3. 黑匣子的环形缓冲为什么选 30 秒？快照里为什么要带 `renderer.info` 和最近事件？

<details>
<summary>答案</summary>

1. 规范规定：如果 lost 事件的默认行为没有被阻止，浏览器会认为页面不打算恢复，上下文永久丢失。`preventDefault()` 必须写在事件回调的第一行，不能 `setTimeout` 之后再补。Three.js 的 `WebGLRenderer` 内部已经阻止了默认行为，但那只解决"有没有资格恢复"，恢复之后 geometry、material、texture、program、render target 的重建仍然要业务侧自己做。
2. 所有 GPU 侧对象都要重建：几何体缓冲、材质与着色器程序、纹理、render target/FBO，以及业务自己持有的 VAO、GPGPU ping-pong 缓冲、后处理链。补丁式修复只盯着"看得见的模型"，很容易漏掉 `EffectComposer` 内部的 render target 和粒子系统的浮点纹理，于是出现"场景回来了但泛光没了、粒子不动了"的半恢复状态。整组 `disposeWorld()` + `buildWorld()` 可以一次性覆盖。
3. 30 秒足够覆盖"劣化 → 触发 → 崩溃"的完整过程，1800 个 `Float32` 只占约 7KB，内存恒定且永远保留最近现场。快照带 `renderer.info` 是为了回答"卡在哪"：draw call、三角形数、几何体/纹理数量能区分合批失败、LOD 失效和资源泄漏；带最近事件是为了回答"为什么卡"，把用户操作、降级、上下文事件和帧时间对齐成一条时间线。
</details>
