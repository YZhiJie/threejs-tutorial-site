# 🎬 Three.js 漫画剧场 · 霓虹墨 Neon Ink

> 把学前预备与 11 个核心模块画成 12 话四格漫画：每话一幅 **1200×840 SVG 原画** + 一篇**讲解 MD**（剧情回顾 / 知识点拆解 / 坑点清单 / 自测题）。
> 视觉体系：深墨底 + 霓虹青/紫/品红 + 斜切光楔；三角色——**像素酱**（提问担当）、**标签君**（行动派）、**码叔**（教授总结）。规范见 [style-guide.md](./style-guide.md)，定妆稿见 [samples/character-sheet.svg](./samples/character-sheet.svg)。

## 📖 一键阅读（整合阅读器）

打开 [index.html](./index.html)（需经本地 HTTP 服务访问，命令见文末「本地预览」），在封面点击「一键打开完整漫画」即可：

- 按 `manifest.json` 组织全部 **12 话**资源（SVG 内联，矢量无损缩放）；
- **翻页 / 长卷双模式**：默认翻页模式一话一屏（←/→ 键盘、两侧悬停大按钮、顶部计数器、按需加载并预取相邻话），点「📜 长卷」或按 `M` 切换到全文连续滚动；
- **讲解抽屉**：每话「📖 讲解」按钮从右侧滑出知识点拆解，`Esc` 关闭、抽屉内 ←/→ 或底部导航切换上下篇、`md` 按钮新窗口打开原始文件；
- **阅读进度记忆**：自动记住话与位置，下次封面出现「继续阅读」；
- **章节导航**：当前话自动高亮、已读绿点标记；桌面固定侧栏，移动端抽屉目录；
- **错误处理**：单话加载失败显示独立错误卡片并可单独重试，不影响其他话。

## 建议读法

1. 先看原画 SVG（浏览器直接打开），跟着剧情把概念"看见"；
2. 再读同 slug 的讲解 MD，把每一格落成可运行的机制与代码；
3. 最后点开"正文"与"演示"链接，回到本项目主线深读并亲手跑一遍 `public/demos` 里的交互演示。

## 目录

| 话 | 主题 | 原画 | 讲解 | 对应正文 |
| ---- | ---- | ---- | ---- | ---- |
| EP.00 | 学前必备：开跑前的检查清单 | [SVG](./EP.00-prerequisites.svg) | [MD](./EP.00-prerequisites.md) | [0 学前预备](../guide/00-prerequisites/index.md) |
| EP.01 | 第一个三维场景 | [SVG](./EP.01-first-scene.svg) | [MD](./EP.01-first-scene.md) | [01 入门](../guide/01-intro/basics.md) |
| EP.02 | 场景图与层级矩阵 | [SVG](./EP.02-scene-graph.svg) | [MD](./EP.02-scene-graph.md) | [02 场景](../guide/02-scene/basics.md) |
| EP.03 | 相机与视锥体 | [SVG](./EP.03-camera-frustum.svg) | [MD](./EP.03-camera-frustum.md) | [03 相机](../guide/03-camera/basics.md) |
| EP.04 | 渲染循环与帧率自由 | [SVG](./EP.04-render-loop.svg) | [MD](./EP.04-render-loop.md) | [04 渲染器](../guide/04-renderer/basics.md) |
| EP.05 | 几何体与顶点流 | [SVG](./EP.05-buffer-geometry.svg) | [MD](./EP.05-buffer-geometry.md) | [05 几何体](../guide/05-geometry/basics.md) |
| EP.06 | 材质与 PBR 光影 | [SVG](./EP.06-material-pbr.svg) | [MD](./EP.06-material-pbr.md) | [06 材质](../guide/06-material/basics.md) |
| EP.07 | 纹理 UV 与流动 | [SVG](./EP.07-texture-uv.svg) | [MD](./EP.07-texture-uv.md) | [07 纹理](../guide/07-texture/basics.md) |
| EP.08 | 灯光与阴影映射 | [SVG](./EP.08-light-shadow.svg) | [MD](./EP.08-light-shadow.md) | [08 灯光](../guide/08-light/basics.md) |
| EP.09 | 缓动与动画混合器 | [SVG](./EP.09-animation-mixer.svg) | [MD](./EP.09-animation-mixer.md) | [09 动画](../guide/09-animation/basics.md) |
| EP.10 | 射线拾取与拖拽 | [SVG](./EP.10-raycaster-picking.svg) | [MD](./EP.10-raycaster-picking.md) | [10 交互](../guide/10-interaction/basics.md) |
| EP.11 | 综合实战与性能优化 | [SVG](./EP.11-capstone-performance.svg) | [MD](./EP.11-capstone-performance.md) | [11 收官实战](../guide/11-capstone/basics.md) |

## 目录结构

```
comics/
├── index.html                # 整合阅读器（翻页/长卷双模式 + 讲解抽屉 + 导航 + 进度记忆）
├── style-guide.md            # 霓虹墨风格规范 Three.js 版（唯一视觉标准）
├── samples/
│   └── character-sheet.svg   # 三角色定妆稿
├── manifest.json             # 12 话清单（校验脚本的依据）
├── check-comics.mjs          # 校验：manifest 一致性 + SVG 良构 + 文本溢出启发式 + MD 完整性
├── EP.NN-*.svg               # 每话原画（1200×840 四格）
└── EP.NN-*.md                # 每话讲解（剧情/拆解/坑点/自测）
```

## 本地预览

```bash
cd threejs-tutorial
python3 -m http.server 8899
# 整合阅读器（推荐）：http://127.0.0.1:8899/comics/index.html
# 单话原画直开：http://127.0.0.1:8899/comics/EP.01-first-scene.svg
```

> 阅读器依赖 `fetch` 合并资源，`file://` 直开会被浏览器拦截，页面会给出上述启动提示。

## 校验

```bash
node comics/check-comics.mjs
```
