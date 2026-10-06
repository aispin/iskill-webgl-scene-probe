# iskill-webgl-scene-probe

**无头探测 three.js / React-Three-Fiber 场景里「看不见也听不见」的东西。** 截图能证明画面，但证明不了「这个音效到底播了没」「这个网格到底渲染了没有」。手写一遍探针要踩一堆坑（挂钩时机、音效指纹、SwiftShader 等待、事件 vs 射线、透明排序），本 skill 把这些固化成可直接抄的脚本。

由一次真实的 3D 个人站点排查沉淀而来，内置了当时踩到的全部坑。

---

## 它能回答什么

| 症状 | 本 skill 给的答案 |
|---|---|
| 音效没声 | **哪一个**音效被触发了（buffer 时长指纹）、有效增益是多少 |
| 网格「组件在、场景里一个都没有」 | 按几何尺寸普查场景图，分清「没渲染」与「没加载」 |
| 某个元素盖住了另一个 | 透明排序 / `renderOrder` 陷阱 + 屏幕空间 AABB 相交判定 |
| 地板闪出别的材质 | 两个平面**完全共面**（不是普通 z-fighting） |
| 点击没反应 | 先分清**事件没到 canvas** 还是**射线没命中**，再查应用层守卫 |
| 偶发卡死、刷新就好、一旦卡住就永久失效 | 「不稳定回调写进依赖数组 → cleanup 杀掉动画时间轴」这一模式 |
| 改一个常量要等 170s 编译 | 免编译快速验证：直接核对 dev server 的编译产物 |

> **音频的分工**：本 skill 做「哪个音效触发了」的**指纹级**判断。
> 要证明「**真的出声了**」（信号级时域 RMS）、抓音频 404 与循环泄漏，
> 请配合 [`iskill-audio-verify`](https://github.com/aispin/iskill-audio-verify)。

---

## 内容结构

- **一、音频探测** —— `AudioBufferSourceNode.start` 挂钩 + buffer 时长指纹 + 判读表
- **二、响度诊断** —— 算 `PositionalAudio` 的有效增益（含距离模型衰减）
- **三、免编译快速验证** —— 不等 SwiftShader 的 170s
- **四、驱动 R3F 场景** —— 调试桥、相机投影点击、按几何尺寸定位网格、事件 vs 射线、
  选目标对象的四条判据、`Vector3.project()` 把相机背后的点镜像到屏幕中央的坑
- **四·四/四·五** —— 「偶发 + 刷新就好 + 一旦触发就持续」这一症状模式的通用心法
- **四·六** —— `renderOrder` 会推翻透明排序
- **四·七** —— 地面闪现别的材质 = 共面
- **四·八** —— canvas 贴图做地贴时的几何与合成陷阱
- **五、踩坑清单** —— 17 条，每条都标了「症状 → 根因」
- **六、完整脚本骨架**

---

## 与相邻 skill 的分工

| skill | 负责 |
|---|---|
| `iskill-ui-verify` | 普通网页截图 / DOM 断言 / 多语言×多主题矩阵 |
| **本 skill** | WebGL 场景：音频指纹、场景驱动、渲染层陷阱、免编译验证 |
| `iskill-audio-verify` | 音频**信号级**验证：RMS、手势门控、音频 404、循环泄漏 |

---

## 安装

本 skill 放在 `~/Workbuddy/ISkills/iskill-webgl-scene-probe`，通过软链接暴露给 WorkBuddy：

```bash
ln -s ~/Workbuddy/ISkills/iskill-webgl-scene-probe ~/.workbuddy-ai/skills/iskill-webgl-scene-probe
```

---

## License

MIT
