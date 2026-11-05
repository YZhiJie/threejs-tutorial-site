# EP.12 自定义着色器 GLSL

> 对应正文：[12.1 基础入门：GLSL 与第一个自定义着色器](../guide/12-shader/basics.md)
> 延伸阅读：[进阶篇](../guide/12-shader/advanced.md) · [实战篇](../guide/12-shader/practice.md)
> 动手演示：[first-shader 第一个自定义着色器](../public/demos/shader/first-shader.html) · [uniforms-playground 调参游乐场](../public/demos/shader/uniforms-playground.html) · [hologram-scan 扫描线与全息投影](../public/demos/shader/hologram-scan.html)

## 剧情回顾

| 格 | 场景 |
| --- | --- |
| ① 困境 | 像素酱照着教程写下人生第一段 GLSL，刷新页面却只剩一片黑——没有报错弹窗、没有红色提示，球直接消失。 |
| ② 揭秘 | 码叔打开浏览器控制台，指着 `THREE.WebGLProgram: Shader Error` 报错块：冒号中间的数字就是出错行号，对着源码数行就能定位。 |
| ③ 实证 | 逐行排查揪出两个经典杀手：片元里混进的全角分号 `；` 和一句中文注释。改成英文半角后，法线渐变球应声点亮。 |
| ④ 结案 | 码叔总结 GLSL 三铁律：只写英文半角、行号看控制台、中间结果画成颜色当调试。 |

## 知识点拆解

### ① 着色器黑屏 ≠ JS 报错

`ShaderMaterial` 的 GLSL 源码是一段字符串，**JS 引擎不会检查它**——字符串能正常创建，语法错误要到 GPU 编译阶段才暴露。表现就是：页面不崩、控制台没有 JS 异常、但物体不渲染（黑屏/消失）。所以写 shader 的第一反应不是刷新重试，而是**打开控制台找编译日志**。

### ② 控制台上的 THREE.WebGLProgram 报错

```
THREE.WebGLProgram: Shader Error 0 - VALIDATE_STATUS false
  ERROR: 0:15: '；' : unexpected token
```

- `0:15` 中第二个数字 **15 是出错行号**（从该段着色器源码第 1 行数起）；
- 报错块下方还会打印完整源码，行号可能因引擎注入的前置声明偏移几行，在附近找拼写/符号错误即可。

### ③ 两个新手杀手：全角标点与中文注释

GLSL 编译器按 **ASCII** 解析源码：

- 全角分号 `；`、全角括号 `（）`、中文逗号 `，` 都是非法记号；
- 中文注释文字同样非法——`/* glsl */` 模板字符串里**只允许英文 + 半角符号**；
- 中文解释请写在 GLSL 字符串**外面**的 JS 注释里。

### ④ 修复链路：法线渐变球

```js
const vertexShader = /* glsl */ `
  varying vec3 vNormal;
  void main() {
    vNormal = normal;
    gl_Position = projectionMatrix * modelViewMatrix * vec4(position, 1.0);
  }
`
const fragmentShader = /* glsl */ `
  varying vec3 vNormal;
  void main() {
    vec3 color = normalize(vNormal) * 0.5 + 0.5;
    gl_FragColor = vec4(color, 1.0);
  }
`
new THREE.ShaderMaterial({ vertexShader, fragmentShader })
```

顶点阶段递出法线，光栅化自动插值，片元阶段归一化后映射到 0~1 当颜色——无需灯光，球面随视角转动呈现连续的红绿蓝渐变。

## 坑点清单（FAQ）

- **Q：黑屏了第一步做什么？** F12 打开控制台，搜 `THREE.WebGLProgram`，报错里的行号直接定位问题。
- **Q：报错行号和源码对不上？** 引擎会在你的源码前注入内置声明，行号可能偏移几行，上下找即可。
- **Q：GLSL 里能写中文注释吗？** 不能。编译器只认 ASCII，中文与全角标点都会编译失败；注释一律英文。
- **Q：片元着色器必须写 precision 吗？** `ShaderMaterial` 默认注入 `precision mediump float;` 无需手写；RawShaderMaterial 或独立 .glsl 文件必须显式声明。
- **Q：shader 里怎么调试中间值？** 没有 console.log，把变量塞进颜色输出：`gl_FragColor = vec4(vec3(myValue), 1.0);`，用眼睛看。

## 自测（Quiz）

1. `ShaderMaterial` 渲染黑屏但页面没有任何 JS 报错，原因是什么？第一步排查动作是什么？
2. 控制台报错 `ERROR: 0:23: '；' : unexpected token`，最可能的原因是什么？
3. 为什么 GLSL 字符串里写中文注释会导致编译失败，而写在字符串外的 JS 注释就没事？

<details>
<summary>答案</summary>

1. GLSL 源码是字符串，语法错误只在 **GPU 编译阶段**暴露，不经过 JS 异常机制，所以页面不崩但物体不渲染。第一步：打开控制台找 **`THREE.WebGLProgram: Shader Error`**，按报错行号定位。
2. 该行混入了**全角分号 `；`**（或其它全角/中文字符）。GLSL 只接受 ASCII 半角符号，逐个检查该行的标点即可。
3. GLSL 编译器按 ASCII 解析，中文字符是非法记号；而字符串外的 JS 注释由 **JS 引擎**处理，天然支持 Unicode，互不相干。规则：GLSL 内只写英文，中文解释放 JS 注释里。
</details>
