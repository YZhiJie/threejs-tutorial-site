# EP.01 第一个三维场景

> 对应正文：[1.1 基础入门：Three.js 是什么与第一个场景](../guide/01-intro/basics.md)
> 延伸阅读：[进阶篇](../guide/01-intro/advanced.md) · [实战篇](../guide/01-intro/practice.md)
> 动手演示：[hello-cube 第一个场景](../public/demos/intro/hello-cube.html) · [template 最小模板](../public/demos/intro/template.html) · [debugging 黑屏排查台](../public/demos/intro/debugging.html)

## 剧情回顾

| 格 | 场景 |
| --- | --- |
| ① 困境 | 标签君抄完 Hello Cube，屏幕一片黑：Mesh 建了、灯也"觉得"加了，就是什么都没有。 |
| ② 揭秘 | 码叔摆出渲染管线四件套：**Scene 场景、Camera 相机、Renderer 渲染器、Mesh 物体**，少一环都不亮。 |
| ③ 实证 | 像素酱把相机挪到 (3,2,4)、补上 `scene.add(cube)`，再用一盏灯配合 Standard 材质，立方体终于转了起来。 |
| ④ 结案 | 码叔总结黑屏排查清单：没 add、相机扎进物体、材质要灯没灯、near 设成 0。 |

## 知识点拆解

### ① 黑屏四件套：任何 Three.js 程序的最小闭环

Three.js 不是游戏引擎，而是 WebGL 的高层封装。一个能出画面的程序必须同时具备：

1. `Scene`——场景树的根，所有可见物体的容器；
2. `Camera`——观察者，决定从哪里看、怎么投影；
3. `WebGLRenderer`——把场景画到 `<canvas>` 上；
4. `Mesh`——真正摆在场景里的物体，由 **Geometry（形状）+ Material（外观）** 组成。

四者的连接关系固定：`renderer.render(scene, camera)` 每帧执行一次，而 mesh 必须已经 `scene.add()` 进场景树。

### ② Mesh 创建了不会自动出现

```js
const cube = new THREE.Mesh(
  new THREE.BoxGeometry(1.4, 1.4, 1.4),
  new THREE.MeshStandardMaterial()
)
scene.add(cube) // 漏掉这行 = 屏幕一片黑
```

Three.js 只渲染场景树上的节点。new 出来的对象只是游离在内存里，`add` 才是把它挂到树上。

### ③ 相机不能和物体重合

立方体中心默认在原点 (0,0,0)，相机也在原点就是"扎"进几何体内部，看到的只有内表面（通常被背面剔除，画面全黑）。起步固定动作：

```js
camera.position.set(3, 2, 4)
camera.lookAt(0, 0, 0)
```

### ④ 要灯的材质，没灯就是黑的

`MeshStandardMaterial` / `MeshLambertMaterial` 完全依赖灯光计算颜色，场景里没有任何光源时输出纯黑。两条路：

- 学习起步用 `MeshNormalMaterial`（按法线自动配色）或 `MeshBasicMaterial`（纯色），不需要灯；
- 用 Standard 材质就加一盏 `HemisphereLight` 或 `DirectionalLight`。

另外两个出厂设置建议：像素比钳制 `renderer.setPixelRatio(Math.min(devicePixelRatio, 2))`，避免高分屏 GPU 过载；窗口缩放后必须 `camera.updateProjectionMatrix()`。

## 坑点清单（FAQ）

- **Q：near 能设成 0 吗？** 不能。near 过小会让深度缓冲精度崩塌，出现 z-fighting 闪烁条纹。常规场景用 `0.1`，far 按实际需要设，别一上来 10000。
- **Q：setAnimationLoop 和 requestAnimationFrame 选哪个？** 优先 `setAnimationLoop`，它额外处理了页面隐藏暂停与 WebGL 上下文丢失恢复。
- **Q：resize 后画面拉伸？** 只改 `camera.aspect` 不够，必须调用 `camera.updateProjectionMatrix()`，再 `renderer.setSize()`。
- **Q：文件双击打开（file://）演示页白屏？** ES Module + importmap 在 file 协议下会被 CORS 拦截，请用本地静态服务器（`python3 -m http.server`）打开。

## 自测（Quiz）

1. 下列哪种情况**不会**导致黑屏？（A. 忘记 `scene.add(mesh)` B. 相机放在原点 C. near 设为 0.1、far 设为 100 D. 用 Standard 材质但场景无灯）
2. Mesh 由哪两个部分组成？各自负责什么？
3. 窗口大小变化时，除了 `renderer.setSize()` 还必须对相机做哪两步？

<details>
<summary>答案</summary>

1. **C**。0.1/100 是合理的近远裁面；A、B、D 都是经典黑屏原因。
2. **Geometry 几何体**负责形状（顶点/三角面数据），**Material 材质**负责外观（颜色、如何与光互动）。
3. 重算 `camera.aspect = innerWidth / innerHeight`，然后调用 **`camera.updateProjectionMatrix()`**。
</details>
