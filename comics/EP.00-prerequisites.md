# EP.00 学前必备：开跑前的检查清单

> 对应正文：[0 学前必备：先修知识自查与补课指南](../guide/00-prerequisites/index.md)
> 动手演示：[vector-lab 点积叉积实验室](../public/demos/prerequisites/vector-lab.html) · [vector-lab-3d 三维版](../public/demos/prerequisites/vector-lab-3d.html) · [projection-cube 手写透视投影](../public/demos/prerequisites/projection-cube.html) · [render-pitfalls 渲染三坑实验室](../public/demos/prerequisites/render-pitfalls.html)

## 剧情回顾

| 格 | 场景 |
| --- | --- |
| ① 困境 | 标签君把官网示例存成 html 双击打开——白屏报错，控制台一条 CORS 报错看不懂。 |
| ② 揭秘 | 码叔摆出两条硬门槛：读得懂 ES Module 的 JavaScript + 会起本地静态服务器，importmap 只是浏览器的"路径别名"。 |
| ③ 实证 | 像素酱担心要啃完线性代数，码叔用两把尺打消顾虑：点积判夹角、叉积求法线，弧度混个脸熟就够。 |
| ④ 结案 | 码叔给出渲染常识三件套与坑点清单：三角形拼一切、管线三步走、PBR 要开灯，清单过完再开第一课。 |

## 知识点拆解

### ① file:// 是跨源：ES Module 必须跑在 http(s) 上

浏览器把 `file://` 协议当跨源对待，ES Module 的加载会被 CORS 策略直接拦截，典型报错是 `Access to script from origin 'null' has been blocked by CORS policy`。这不是 bug，是安全设计。解法三选一：

```bash
npx serve .                        # 零配置静态服务器
# 或 VSCode 插件 Live Server
# 或本教程仓库自带的 pnpm dev
```

### ② 两条硬门槛与 importmap

学前自查表有 8 项，但**硬性门槛只有两条**：

1. **JavaScript**：能读懂 `import`/解构/箭头函数/`async` 的代码——Three.js 每个示例都是十几行起跳的 ES Module；
2. **本地静态服务器**：`npx serve .` 一条命令的事。

importmap 是浏览器原生的"模块路径别名"，让裸导入 `import * as THREE from 'three'` 能被解析到 CDN 文件；`three/addons/` 带尾斜杠表示前缀映射。CDN 与 npm 两条路线的 `import` 语句完全相同，学习期用 CDN、做项目换 npm。

### ③ 向量两把尺：点积与叉积

三维图形只用到线性代数的一个小子集，先用两把尺建立直觉（可在 vector-lab 演示里拖出来）：

- **点积 A·B**：一个数，等于夹角余弦 × 两长度。`>0` 同向、`=0` 垂直、`<0` 反向——判断"敌人是否在视野内"就是它；
- **叉积 A×B**：一个向量，同时垂直于 A、B（右手定则：四指从 A 弯向 B，拇指即方向），模长等于两向量张成的平行四边形面积——求三角形面法线就是它。

另外记住换算：**180° = Math.PI**，Three.js 所有角度参数都是弧度。

### ④ 渲染常识：三角形、管线与 PBR

1. 一切 3D 形状都是三角形拼的——一个 BoxGeometry 是 12 个三角形，面数直接决定渲染开销；
2. 管线三步：**顶点着色器**（算位置）→ **光栅化**（拆像素）→ **片元着色器**（算颜色）；
3. `MeshStandardMaterial` 等 PBR 材质用"金属度 + 粗糙度"描述反光，**必须有灯光**才显形——这就是下一话黑屏坑的根源。

第④格的三个坑（关灯纯黑、忘 `scene.add`、弧度换算）都可在 [render-pitfalls 渲染三坑实验室](../public/demos/prerequisites/render-pitfalls.html) 里亲手复现：关掉灯光看画面瞬间变黑、取消勾选让绿色魔方"游离"、开线框模式数三角形。

## 坑点清单（FAQ）

- **Q：完全不会 JavaScript，能直接学 Three.js 吗？** 不建议。语言 + 图形 API + 数学三座山同时爬必崩，先花 1–2 周补 JS 语法与异步，再从 EP.01 开始。
- **Q：需要先系统啃完线性代数吗？** 不需要。用到哪个讲哪个：本话讲直觉，2.2 进阶篇展开层级矩阵，遇到陌生术语现场查。
- **Q：CDN 和 npm 会冲突吗？** 不会，`import` 语句完全一样。从 CDN 迁到 npm 只需删掉 importmap、`pnpm add three`。
- **Q：new 出来的 Mesh 为什么看不见？** Three.js 只渲染场景树上的节点，忘了 `scene.add()` 就是"游离在内存里"。

## 自测（Quiz）

1. 双击打开含 importmap 的 html 报 CORS 错误，根本原因是什么？至少说出一种解法。
2. `(1,0,0) · (0,1,0)` 的值是多少？说明两向量的位置关系。
3. `MeshStandardMaterial` 场景没有任何光源，画面是什么样？为什么？

<details>
<summary>答案</summary>

1. `file://` 协议被浏览器视为跨源，ES Module 加载被 CORS 策略拦截。解法：`npx serve .` / Live Server / `pnpm dev` 任一本地静态服务器。
2. 等于 **0**，两向量互相垂直（点积为 0 即 90° 夹角）。
3. **纯黑**。PBR 材质靠灯光计算颜色，零光源时输出黑色；改用 MeshBasicMaterial/NormalMaterial 或加一盏 HemisphereLight。
</details>
