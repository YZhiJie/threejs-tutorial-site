# EP.18 把场景拆成乐高积木

> 对应正文：[18.1 基础入门：声明式 3D 与第一个 R3F 场景](../guide/18-react-three-fiber/basics.md)
> 延伸阅读：[进阶篇](../guide/18-react-three-fiber/advanced.md) · [实战篇](../guide/18-react-three-fiber/practice.md)
> 动手演示：本模块为 React 技术栈，演示以 StackBlitz 外链呈现（见 [18.1 运行效果展示](../guide/18-react-three-fiber/basics.md#运行效果展示)）；本地占位页：[r3f placeholder](../public/demos/r3f/placeholder.html)

## 剧情回顾

| 格 | 场景 |
| --- | --- |
| ① 困境 | 像素酱的原生 three 项目一个文件揉了 800 行：场景、灯光、模型、动画全在一起，产品一句"换个颜色"就得翻半天。 |
| ② 揭秘 | 码叔点破命令式 vs 声明式：命令式一步步指挥"怎么做"，声明式只描述"要什么"——JSX 嵌套就是 scene.add，React 负责比对更新。 |
| ③ 实证 | 场景拆成 `<Lights/>`、`<Model/>`、`<Effects/>` 三块积木后，需求变更只改一行 props，800 行大文件瘦身成十几个小组件。 |
| ④ 结案 | 坑点清单：args 传构造参数、useFrame 里别 setState、高频状态走 transient 读取；码叔总结"把怎么做交给渲染器"。 |

## 知识点拆解

### ① 命令式代码的失控临界点

原生 three 的写法是"一步步指挥"：new 对象、设属性、add 进场景、在循环里手动驱动。小场景很直观，但对象一多，**创建、布局、动画、事件**四类逻辑全部揉在同一个文件里——改一处要先读懂八百行。问题不在 three，在于组织方式：没有任何机制帮你把"灯光""模型""特效"分开。

### ② 声明式：只描述"要什么"

R3F 把 three 的每个类映射成一个小写驼峰标签，JSX 的结构就是场景图的结构：

```jsx
// 命令式：亲自指挥每一步
const m = new THREE.Mesh(geo, mat)
scene.add(m)

// 声明式：写出"场景里有个 mesh"即可
<mesh><boxGeometry /></mesh>
```

React 的 reconciler 负责比对前后 JSX 的差异并调用底层 three API：标签消失就 remove、属性变了就重新赋值。你从"指挥者"变成"描述者"。映射四条规则：驼峰标签 = three 类、属性 = 实例赋值、`args` = 构造参数、`attach` = 挂载到父对象的命名属性。

### ③ 组件化 = 乐高积木

声明式的真正红利是**组件复用**。把灯光、模型、特效各自封装成组件后：

```jsx
<Canvas shadows>
  <Lights /> <Model url={url} />
  <Effects bloom />
</Canvas>
```

需求变更不再是"翻 800 行找那一行"，而是改一行 props：换模型改 `url`、关特效删掉 `<Effects />`。组件还能跨项目搬运——这正是 18.3 实战篇封装 `<SceneCanvas />` 与 `<ProductViewer />` 的动机。

### ④ 三条高频坑点

1. **`args` 传构造参数**：`<boxGeometry args={[1,1,1]} />` 等价于 `new THREE.BoxGeometry(1,1,1)`；写错位置（写成 props）构造参数拿不到。
2. **useFrame 里别 setState**：每帧 setState = React 每秒重渲染 60 次，帧率崩塌；高频动画一律 `ref.current` 直改 three 对象。
3. **高频状态走 transient 读取**：转速这类"每帧要读但不进 JSX"的值存 zustand，在 useFrame 里 `getState()` 直读，不订阅、不触发渲染。

## 坑点清单（FAQ）

- **Q：构造函数的参数在 JSX 里怎么传？** 用 `args` 数组：`args={[1, 1, 1]}` 按位对应构造参数；无参构造可省略。
- **Q：useFrame 里能 setState 吗？** 低频可以、每帧禁止。要反映到 JSX 的低频状态用 store/useState；高频动画用 ref 直改。
- **Q：转速这种每帧读的值放哪？** zustand 存值，useFrame 里 `useStore.getState().speed` transient 直读——不订阅所以零重渲染。
- **Q：R3F 创建的对象能当原生 three 用吗？** 能，挂 `ref` 拿到的就是货真价实的 `THREE.Mesh`；`useThree()` 还能拿到 scene/camera/gl。
- **Q：Vue 用户有没有对应方案？** 有，TresJS——同样的声明式理念，换成 Vue 模板语法。

## 自测（Quiz）

1. 原生 three 的 `scene.add(mesh)` 在 R3F 里对应什么写法？JSX 属性又对应什么？
2. 为什么在 `useFrame` 里调用 `setState` 会导致卡顿？正确做法是什么？
3. `<boxGeometry args={[1, 1, 1]} />` 中的 `args` 起什么作用？如果想给场景加雾，JSX 该怎么写？

<details>
<summary>答案</summary>

1. 把 `<mesh>` 作为 JSX 子节点写进 `<Canvas>` 即完成 add（嵌套关系 = 场景图父子关系）；JSX 属性对应实例属性赋值，如 `position={[0,1,0]}` 等价于 `mesh.position.set(0,1,0)`，连字符属性可深入嵌套（`shadow-mapSize`）。
2. setState 会触发 React 重渲染，useFrame 里每帧调用等于每秒重渲染 60 次，reconciler 开销直接压垮帧率。正确做法：高频动画用 `ref.current` 直改 three 对象（不经过 React）；需要状态的用 zustand transient 读取。
3. `args` 是构造函数参数数组，等价于 `new THREE.BoxGeometry(1, 1, 1)`。加雾：`<fogExp2 attach="fog" args={['#05070c', 0.022]} />`——雾是 scene 的命名属性，必须显式 `attach="fog"`。
</details>
