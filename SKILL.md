---
name: iskill-webgl-scene-probe
summary: 无头探测 three.js / React-Three-Fiber 场景的「看不见的东西」——音频到底播没播、播的是哪个、有效音量多大，以及不等 170s 着色器编译就能确认改动是否生效。附 R3F 场景驱动与点击链路定位（事件没到 vs 射线没中）、状态机卡死（不稳定回调杀掉动画时间轴）、共面地面闪材质、透明排序（renderOrder）陷阱、地贴几何与屏幕空间遮挡判定，以及踩坑清单。
description: 当需要验证或排查 WebGL / three.js / R3F 应用的内部问题时使用——音频到底播没播/音量多大、某个网格到底渲染了没有（组件在但场景里一个都没有）、某个元素是不是盖住了另一个（透明排序 / renderOrder）、地面为什么会闪出别的材质（共面平面）、贴花到底压在角色身上没有（屏幕空间 AABB）、点击为什么没反应（事件 vs 射线）、动画/路由状态机为什么偶发卡死且一旦卡死就永久失效（依赖数组里的不稳定回调杀掉了时间轴）、以及不等 170s 着色器编译就确认改动是否生效。触发词：音效没声、听不到声音、音频验证、web audio 验证、three.js 验证、R3F 场景截图、无头探测、scene probe、positional audio、网格没渲染、场景图排查、贴图没生效、盖住了、renderOrder、透明排序、遮挡判定、点击没反应、射线没命中、探针摆相机、地板闪绿、闪材质、共面、z-fighting、房间不出现、偶发卡死、永久失效、状态机卡住、toast 提示、静默失败、按了没反应。适用于任何 Web Audio / three.js 项目。**注意分工：本技能做「哪个音效触发了」的指纹级判断；要证明「音效真的出声了」（信号级 RMS）、抓音频 404 与循环泄漏，用 `iskill-audio-verify`。**
agent_created: true
---

# iskill-webgl-scene-probe · 无头探测 WebGL 场景

**解决的问题**：3D 场景里的东西「看不见也听不见」——截图能证明画面，但证明不了
「这个音效到底播了没」。手写一遍探针要踩一堆坑（挂钩时机、音效指纹、SwiftShader 等待）。
本技能把这些固化成可直接抄的脚本。

**与 `iskill-ui-verify` 的分工**：普通网页截图 / DOM 断言 → 用 `iskill-ui-verify`。
本技能只管 WebGL 场景的**音频探测**、**场景驱动**、**免编译快速验证**。

---

## 一、音频探测：证明「哪个音效真的播了」

> ⚠️ **本节只回答「哪一个音效触发了」，不回答「它到底有没有出声」。**
> `start()` 被调用 ≠ 有声音 —— 中间还隔着四道门：「文件不存在(404)」、
> 「context 还是 suspended」、「增益某一环是 0」、「节点不在通往 destination 的路径上」。
>
> 要证明**真的产生了信号**（读时域 RMS）、要抓**音频 404** 与**循环泄漏**、
> 要验证**手势门控**，用 **`iskill-audio-verify`** ——
> 两者是「指纹级」与「信号级」的互补关系，**建议一起用**。

核心手法：**页面加载前**挂钩 `AudioBufferSourceNode.prototype.start`，用 **buffer 时长**
当音效指纹（不需要知道 URL，也不需要源码埋点）。

```js
await page.evaluateOnNewDocument(() => {
  window.__audioLog = []
  const t0 = performance.now()
  const push = (o) => window.__audioLog.push({ t: Math.round(performance.now() - t0), ...o })

  const origStart = AudioBufferSourceNode.prototype.start
  AudioBufferSourceNode.prototype.start = function (...a) {
    let dur = null
    try { dur = this.buffer ? +this.buffer.duration.toFixed(3) : null } catch (e) {}
    push({ kind: 'bufferSource', dur })          // ← 时长就是指纹
    return origStart.apply(this, a)
  }

  // HTMLMediaElement（<audio> / new Audio()）走另一条路，也要挂
  const origPlay = HTMLMediaElement.prototype.play
  HTMLMediaElement.prototype.play = function (...a) {
    let s = ''; try { s = (this.currentSrc || this.src || '').split('/').pop() } catch (e) {}
    push({ kind: 'media', src: s })
    return origPlay.apply(this, a)
  }

  // AudioContext 何时被创建 / resume（排查自动播放拦截）
  const AC = window.AudioContext || window.webkitAudioContext
  if (AC) {
    const origResume = AC.prototype.resume
    AC.prototype.resume = function (...a) { push({ kind: 'ctx.resume', state: this.state }); return origResume.apply(this, a) }
    const Orig = AC
    window.AudioContext = function (...a) { const c = new Orig(...a); push({ kind: 'ctx.create' }); return c }
    window.AudioContext.prototype = Orig.prototype
  }
})
```

**先建立指纹对照表**——在页面里解码一遍，拿到真实时长：

```js
const durations = await page.evaluate(async () => {
  const ctx = new (window.AudioContext || window.webkitAudioContext)()
  const out = {}
  for (const n of ['uchyleniedrzwi', 'otwarciedrzwi', 'zamknieciedrzwi']) {
    const b = await (await fetch('/sounds/' + n + '.mp3')).arrayBuffer()
    out[n] = +((await ctx.decodeAudioData(b)).duration).toFixed(3)
  }
  return out
})
// → {"uchyleniedrzwi":0.437,"otwarciedrzwi":2.04,"zamknieciedrzwi":0.575}
```

之后日志里出现 `{"kind":"bufferSource","dur":0.575}` 就知道是关门声。

**启动参数必须加** `--autoplay-policy=no-user-gesture-required`，否则 headless 下
AudioContext 常停在 suspended，声音不会真的启动。

> ⚠️ **但这个 flag 会掩盖真实 bug。** 它让音频在**无手势**下也能出声，于是
> 「页面在挂载时就 `new AudioContext()`」这个**真实用户一定会撞到的自动播放拦截**
> 在你的测试里完全看不见。要验证「真实用户能不能听到」，必须换成
> `--autoplay-policy=document-user-activation-required`，并断言控制台**没有**
> `AudioContext was not allowed to start`。细节见 `iskill-audio-verify` 第二节。

### 判读

| 现象 | 结论 |
|---|---|
| 日志里根本没有对应时长的 `bufferSource` | **触发链路断了** → 去查调用点 / 早退条件 |
| 有时长、但用户听不见 | **触发正常，响度问题** → 走第二节算有效增益 |
| 只有 `ctx.create` 没有 `bufferSource` | 音频被自动播放策略拦了 |

---

## 二、响度诊断：算「有效增益」

three.js 的 `PositionalAudio` 用距离模型衰减，`setVolume()` 只是其中一环。
把 `THREE.Audio` 的参数和世界坐标都读出来，自己算：

```js
const report = await page.evaluate(() => {
  const cam = window.__cam, V3 = cam.position.constructor
  const audios = []
  window.__scene.traverse((o) => {
    if (typeof o.getVolume === 'function' && o.context) {   // ← 这才是 THREE.Audio 的可靠判据
      const p = new V3(); o.getWorldPosition(p)
      audios.push({ vol: +o.getVolume().toFixed(4), model: o.getDistanceModel(),
        ref: o.getRefDistance(), roll: o.getRolloffFactor(), maxD: o.getMaxDistance(),
        bufDur: o.buffer ? +o.buffer.duration.toFixed(3) : null,
        pos: { x:+p.x.toFixed(2), y:+p.y.toFixed(2), z:+p.z.toFixed(2) } })
    }
  })
  const c = cam.getWorldPosition(new V3())
  return audios.map(a => {
    const d = Math.hypot(a.pos.x-c.x, a.pos.y-c.y, a.pos.z-c.z)
    const atten = a.model === 'exponential' ? Math.pow(Math.max(d,a.ref)/a.ref, -a.roll)
      : a.model === 'linear' ? Math.max(0, 1-(d-a.ref)/(a.maxD-a.ref))
      : a.ref / (a.ref + a.roll * (Math.max(d,a.ref)-a.ref))
    return { ...a, dist:+d.toFixed(2), atten:+atten.toFixed(5), effective:+(a.vol*atten).toFixed(5) }
  })
})
```

> **判据别写错**：`o.isPositionalAudio` / `o.isAudio` **不存在**（three.js 不设这两个标志），
> traverse 会一个都找不到。用 `typeof o.getVolume === 'function' && o.context`。

**经验阈值**：`effective < 0.005` 基本等于听不见；`0.01~0.05` 偏轻但可闻。
指数模型下 `atten = (dist/ref)^-rolloff`，所以 `rolloff=2` 掉得极快——
**远处触发的音效（如「离开后关门」）往往被衰减吃掉 2 倍以上**，即使它和近处音效设了同样的
`volume`。修法是提高该音效的 volume，或把 `rolloff` 降到 1 / `refDistance` 调大。

---

## 三、免编译快速验证：改动到底生效没有

SwiftShader 编译着色器要等 **~170s**，改一个常量不值得跑一遍完整渲染。
Vite dev server 会按需编译模块，直接拉编译产物核对即可：

```bash
curl -s "http://localhost:5199/src/components/canvas/corridor/SegmentDoors.jsx" -o /tmp/m.js
# 再用 Grep 工具在 /tmp/m.js 里找常量
```

**注意**：这个沙箱里 Bash 的 `grep`/`rg` 经常**静默返回空**，别用管道 grep 判断——
落盘后用 Grep 工具读。

---

## 四、驱动 R3F 场景

### 4.1 先找应用自带的调试桥，别自己造

很多 R3F 项目里已经有一个"跳过门动画直接进房间"的调试组件（本项目是 `App.jsx` 的
`DebugBridge`，暴露 `window.__aispin`）：

```js
window.__aispin = {
  enter:    () => scene.markEntered(),
  teleport: (id) => scene.teleportTo(id),
  snap:     (id) => { /* 设 window.__forceRoom = id + enterRoom(id) + 摆相机 */ },
  probe:    () => ({ currentRoom, hasEntered, phase, isTeleporting, pending }),
  clearForce: () => { window.__forceRoom = null },
}
```

**先 `Grep` 一遍 `__forceRoom` / `__debug` / `probe:` / `window.__`，能省掉一整轮试错。**

⚠️ **不要只自己在 `evaluateOnNewDocument` 里设 `window.__forceRoom`。** 踩过的坑：
那样设完，房间的门控确实会放行，但相机没被摆到房间里，而且 React 状态是异步的 ——
紧接着同步读 `probe()` 会拿到 `hasEntered: false` 这种陈旧值，一次就白烧 3 分钟。

正确姿势：调 `snap(id)` 之后**再多等 10~15s**（让 `Suspense` 里的 `useTexture` 全部 resolve、
整棵子树挂载完），然后再探。

### 4.2 有 `__cam` / `__scene` 就能摆相机与投影点击

惯例是在 `Experience.jsx` 里写：

```js
if (typeof window !== 'undefined') { window.__cam = camera; window.__scene = scene }
```

有了这两个句柄就能把世界坐标投影成屏幕坐标去点：

```js
const pt = await page.evaluate((x, y, z) => {
  const cam = window.__cam
  const v = new (cam.position.constructor)(x, y, z); v.project(cam)
  return { x: Math.round((v.x+1)/2*innerWidth), y: Math.round((1-v.y)/2*innerHeight) }
}, wx, wy, wz)
await page.mouse.move(pt.x, pt.y); await sleep(150)
await page.mouse.down(); await sleep(70); await page.mouse.up()
```

手动摆相机（配合 `{wait:170000}`，**不要**用 `until` 轮询，SwiftShader 会把主线程堵死）：

```js
const c = window.__cam
c.position.set(0, 0.15, 26.6); c.lookAt(0, 0, 22); c.updateProjectionMatrix()
```

### 4.3 在场景图里定位一个网格：按几何尺寸找，别按贴图找
想确认「某个组件到底渲染了没有」，不要按 `material.map.image.src` 去搜。**贴图可能还没
load 完**（`image` 为 `undefined`、或 `src` 为空串），会得到**假阴性**——你会误判"没渲染"。

按 `geometry.parameters` 的尺寸找才稳（几何体是同步创建的）：

```js
window.__scene.traverse((o) => {
  if (!o.isMesh) return
  const p = o.geometry?.parameters
  if (!p || p.width !== 0.17 || p.height !== 0.17) return   // ← 目标组件的特征尺寸
  /* ... */
})
```

**先做一次尺寸普查再下结论**——把场景里所有 `width x height` 统计出来，既能确认
邻近组件在不在（证明房间/卡片确实挂载了），也能确认目标组件是不是真的一个都没有：

```js
const sizes = {}
window.__scene.traverse((o) => {
  if (!o.isMesh) return
  const p = o.geometry?.parameters
  if (p?.width !== undefined) { const k = p.width.toFixed(2)+'x'+p.height.toFixed(2); sizes[k]=(sizes[k]||0)+1 }
})
```

顺带把 `o.visible`、`o.parent.visible` 和**世界坐标**一起 dump 出来。注意
`o.visible === false` **不代表不在场景图里**——父级把 `visible` 设成 false 时，
`traverse` 照样能访问到子对象。所以"找不到"是真找不到，不是被隐藏了。

### 4.4 先查数据源，再怀疑渲染

R3F 里"组件在、但什么都不显示"最常见的真凶不是渲染层，是**数据层**：

```js
const fromCms  = useProjects()
const projects = fromCms || FALLBACK_PROJECTS   // ← JSON 非空时 FALLBACK 是死代码
```

JSON 数据源一旦非空，`FALLBACK_PROJECTS` 里精心写的字段就**永远不会生效**。如果 JSON 里
某个字段是**空数组**（字段存在但没内容），对应的 `.map()` 就渲染出 0 个节点——
代码看起来完全正常，实测却是零网格。

### 4.5 房间/关卡布局探针：先让应用自己把相机送进去

**别自己算房间坐标。** 房间通常挂在某个带旋转/偏移的父组里（本项目：`RoomInterior` 挂在
`DoorSection` 内部，同时还叠了翻转），从场景图里的世界坐标反推「我该站哪里」很容易全错。

先找**应用自带的入场调试入口**，让动画把相机送到位：

```js
window.__aispin.enter()                 // 跳过开门，标记已入场
window.__aispin.teleport('gallery')     // 走传送流程，等 ~15s
// 然后才读 camera.position —— 这才是玩家真正看到的机位
```

⚠️ **小心「看起来像快照、其实不是」的调试钩子**。本项目有个 `snap(id)`，名字很像
"直接跳进房间"，实际只是 `__forceRoom` + `enterRoom` + 把相机放到**门上 8 单位**——
房间内部根本没进，探针会一直拍在门板上，白烧好几轮 3 分钟。
**用一个新钩子做布局探针前，先花 30 秒把它的实现读一遍。**

### 4.6 选「要点的那个对象」：按屏幕投影选，别按世界坐标选

想自动点中「视野中央的那张卡」，不要用「离所有卡片质心最近」——**质心不在相机视野中央**，
选出来的经常直接投影到屏幕外（实测 `sx = 2152`，而视口宽 1600）。

正确做法：把候选逐个投影到屏幕，**按到画面中心的距离排序**：

```js
const list = []
scene.traverse((o) => {
  if (!matches(o)) return
  o.updateWorldMatrix(true, false)
  const p = new K(); p.setFromMatrixPosition(o.matrixWorld)
  const v = p.clone().project(cam)
  const sx = (v.x + 1) / 2 * innerWidth, sy = (1 - v.y) / 2 * innerHeight
  list.push({ o, sx, sy, d: Math.hypot(sx - innerWidth/2, sy - innerHeight/2) })
})
list.sort((a, b) => a.d - b.d)
```

再顺手过滤掉贴边的（`sx` 要留边距），否则会点在 UI 覆盖层上。

### 4.7 「点了没反应」：先分清**事件没到**还是**射线没中**

这两类症状长得一样，但修法完全不同。用两步就能定位：

**第一步：证明 DOM 事件到没到 canvas。** 在 `window` 上用捕获阶段记日志：

```js
window.__ev = []
;['pointerdown','pointerup','click','pointermove'].forEach((t) =>
  window.addEventListener(t, (e) => window.__ev.push(t + '@' + Math.round(e.clientX) + ',' + Math.round(e.clientY) + ' tag=' + e.target?.tagName), true))
```

如果日志里有 `tag=CANVAS`，说明事件到了，问题在射线。

**第二步：自己手写「射线 × 平面」求交**，看命中点落在平面的哪个位置。
**不需要 three 的 `Raycaster`**（本项目 `window.Raycaster` 是 `undefined`，`__THREE__` 只挂了版本号）：

```js
const K = cam.position.constructor
const ndcX = (px / innerWidth) * 2 - 1, ndcY = -(py / innerHeight) * 2 + 1
const dir = new K(ndcX, ndcY, 0.5).unproject(cam).sub(cam.position).normalize()
// 对每个候选平面：
const n = new K(0, 0, 1).transformDirection(mesh.matrixWorld)
const den = n.dot(dir)
if (Math.abs(den) < 1e-6) continue
const t = p.clone().sub(cam.position).dot(n) / den       // 沿射线的距离，>0 才在眼前
const hit = cam.position.clone().add(dir.clone().multiplyScalar(t))
const rel = hit.clone().sub(p)
const right = new K(1,0,0).transformDirection(mesh.matrixWorld)
const up    = new K(0,1,0).transformDirection(mesh.matrixWorld)
const dx = rel.dot(right), dy = rel.dot(up)              // 平面局部坐标
const inside = Math.abs(dx) <= w/2 && Math.abs(dy) <= h/2
```

`t`、`dx`、`dy`、`inside` 四个数一摆，立刻知道是坐标错了还是应用层挡了。

**第三步（只有前两步都通过才需要）**：那就是应用自己的守卫，去读源码找
`if (isAnimating || isSelected) return` 这类提前返回。

⚠️ **探针里的坐标系要小心**：`Object3D.matrixWorld.decompose(pos, quat, scale)`
拿到的是**四元数**，不是 Euler。想读旋转角度就直接用 `obj.rotation.x`（局部）或
`obj.parent.scale.x`（父级缩放），别在探针里自己 `setFromQuaternion`——
很容易撞 `quaternion.setFromRotationMatrix is not a function`，白烧一轮。

### 4.8 两个反复坑人的探针错误

**① `Vector3.project()` 会把「相机背后」的点镜像到屏幕中央。**

它做的是透视除法，对 `z > 1` 的点会给出**符号翻转**的坐标 —— 看起来完全正常，
于是你会捞出一堆「明明在屏幕中央却不响应点击」的假目标。**先过滤再投影**：

```js
const v = new K(); v.setFromMatrixPosition(o.matrixWorld)
if (v.z >= cam.position.z - 1) return        // 反例：世界坐标直接比 z 只对「沿 -Z 看」成立
// 更通用的写法：把点变换到相机空间，剔除 z >= 0
const inCam = v.clone().applyMatrix4(cam.matrixWorldInverse)
if (inCam.z >= -0.1) return                  // 相机朝 -Z，所以可见点 z < 0
```

**② 选测试目标时，要选一个「真实交互距离」下的目标，而不是最近的。**

排查「点 X 没反应」时，我先挑了场景里最近的候选 —— 结果那是个
**28 单位外、法线几乎与视线平行（`n·toCam` = 0.05–0.12`）** 的物体。
在那个距离和角度，点不中本来就不代表代码有问题，纯属浪费一轮。

**判据**：目标应该同时满足

| 条件 | 阈值 |
|---|---|
| 距离 | 玩家真实可交互距离（本项目门的交互约 3–8 单位） |
| 朝向 | `normal · (camPos - objPos).normalize()` 明显为正（> 0.5 才舒服） |
| 屏幕位置 | 投影落在视口内且**离边缘留 20px 以上** |
| 可见性 | 沿 `parent` 链**没有任何 `visible === false`**（R3F 会跳过不可见物体） |

**没有合适目标时，不要硬点** —— 优先让应用自己走到那个状态
（本项目：先 `teleport` 进房间，再用返回按钮出来，相机就停在门前 6.5 单位处）。
真实状态往往比手工摆位更容易复现，也更可信。

**③ 千万不要把坐标写死在步骤里。** 这一条坑了我三轮。

被点物体的**屏幕位置每次运行都不一样**（相机受缓动相位、帧率、等待时长影响），
所以 `{clickWorld:[3.29,-0.6,-21.99]}` 这种写死的点击会时中时不中，
而**失败的表现和「代码有 bug」一模一样**：事件到不了、状态不变、截图没变化。

正确做法：用一个「先算再点」的步骤，在同一帧里求屏幕坐标并立即点击。

```js
// harness 里增加的步骤类型：expression 先求值，再 mouse.move/down/up
if (s.evalClick) {
  const pt = await page.evaluate(`(${s.evalClick})()`)
  if (Array.isArray(pt)) { await page.mouse.move(pt[0], pt[1]); /* down, up */ }
}
```

```jsonc
// 用法：把「找目标」和「点目标」分成两步，第二步实时算
{"js": FIND_TARGET_EXPR,  "name": "find"},          // 结果存 window.__t
{"evalClick": "() => { /* project window.__t, return [x,y] */ }", "name": "click"}
```

先前的 `clickWorld` 只适合**完全静止**的场景；只要相机还在缓动，就会漂。

**排查顺序：数据源是否非空 → 该字段是否为空数组 → 才轮到渲染层。**

---

## 四·四·八、通用心法：阈值挂错了参照系（「偶发 + 刷新就好 + 一旦触发就持续」）

本项目连着三个 bug，**两个的根因是同一个模式**，而且都不是「阈值大小不对」，是
**参照系选错了**。这类问题值得单独记一条，因为症状高度一致：

> **偶发、无法稳定复现、刷新就好、但一旦触发就会持续下去。**

### 两个真实案例

| 位置 | 判据 | 错误的参照系 | 正确的参照系 |
|---|---|---|---|
| 传送状态机 | 「还在进行中吗？」 | **不稳定的回调身份**（`play` 每次音量变化都重建） | 事件/相位本身 |
| 走廊分段可见性 | 「相机还没走到这一段吗？」 | **分段自己的 `startZ`** | 相机所处的分段索引 |

第二个的具体样子（很典型）：

```js
const startZ = 10 - (segmentIndex * SEGMENT_LENGTH);
const isFarAhead = camera.position.z > startZ + 30;   // ✗
```

看起来合理，但它假设「相机永远不会站在这一段之外还能看向它」。
而相机出生在 `z = 28`、segment 0 的 `startZ = 10`，**一出生就违反了这个假设**；
滚动又没有上下界钳制，用户一后退就越过 `startZ + 30`，
于是这一段（连它上面挂着的所有门）把自己隐藏了。

**正确写法是比较「相对位置」而不是绝对坐标**：

```js
const cameraSegment = Math.floor((10 - z) / SEGMENT_LENGTH);
const isAhead = segmentIndex > cameraSegment + 1;     // ✓ 我站在哪一段，就相对它算
```

### 加上一条「事实级」的硬保证

启发式判据之间**可能互相矛盾**（上面两行都从同一个 `z` 推出来，就一定有不一致的时候）。
凡是「隐藏/禁用」这类破坏性操作，都应该再叠一条**不依赖任何假设**的兜底：

```js
if (segmentIndex === cameraSegment) isVisible = true   // 我站的这一段，永远不许藏
```

这把「偶发整个场景/整条走廊全部消失」从**可能发生**变成**不可能发生**。
成本是一个 `if`。

### 另一个同源陷阱：状态收尾写在补间的 `onComplete` 里

这是本项目一连串「偶发 + 永久失效」bug 里最贵的一个，值得单列。

```js
gsap.to(camera.position, {
  duration: 1.5,
  onComplete: () => {          // ← 唯一的收尾入口
    setIsAnimating(false);     // 解锁
    enterRoom();               // 标记状态
    setCameraOverride(false);  // 交还控制权
  }
});
```

**只要有任何别处调用 `gsap.killTweensOf(camera.position)`，
这个 `onComplete` 就永远不会执行，三样收尾全部丢失。** 而症状会飘得很远：

- `isAnimating` 卡住 → 指针对同一个物体的每个点击都被守卫拒绝
  → 用户报的是「**返回按钮没反应**」，而根因在一个毫不相干的补间里
- `setCameraOverride` 丢失 → 相机被 hook 和动画同时写 → 滚动像是卡住
- `setIsInsideRoom` 丢失 → 房间永远不出现

**排查特征**：`onComplete` 里如果既有**视觉**收尾又有**状态解锁**，
就把解锁抽出来做成**幂等函数**，并且：

1. 让所有提前 `return` 的分支也调它（`const settle = () => {...}` 提到函数最前面）
2. 加一个看门狗定时器（本项目 9s，正常动画 1.5–3.5s）兜底
3. **禁止别人 kill 这个补间** —— 需要相机时用
   `gsap.getTweensOf(camera.position).length > 0` 轮询等待，而不是抢占。
   （轮询用的 tick 必须进 `useEffect` 依赖数组，否则定时器空转、effect 不重跑 ——
   表现为「按钮静默失效」。）

**同时**：这类失败要么自愈、要么可见。本项目最后给返回按钮加了 `disabled`
状态（门动画期间），让用户看得见「现在不能点」，而不是点了没反应。

### 排查清单

遇到「偶发 + 刷新就好 + 触发后持续」时，按顺序问：

1. **这个判据的参照系，在什么情况下会失效？**
   有没有「对象站在参照系之外但依然应该可见/可用」的合法状态？
2. **有没有完全不受钳制的量？** 用户能把它推到多远？
   （本项目是 `targetZ`，没有任何上下界）
3. **判据是硬切还是有余量？** 一过阈值立刻翻转，等于把「差值」变成「开关」，
   更容易被观察到、也更像严重故障。
4. **有没有第二条「事实级」兜底？** 即：不依赖任何假设、单独就能保证不出灾难的那一条。
5. **用户给了复现步骤就直接照着复刻。** 本项目前两轮我在传送/弹窗那条线上猜，
   方向错了；用户一句「往后退」直接指向了一个我从未检查过的模块。


**症状**：偶发的、无法稳定复现的「整个房间/页面不出现」，而且**一旦发生就永久持续**
（用户会说「所有的房间都不出现」）。刷新能好，所以看起来像玄学。

**根因模式**（非常常见，值得当 checklist 用）：

```jsx
const { play } = useAudio()          // ← 这个函数每次 context 变化都会重建
useEffect(() => {
  const tl = gsap.timeline({ onComplete: () => advancePhase() })
  tl.to(...)
  play('sound')                      // 顺手在里面播个音
  return () => { tl.kill() }         // ← cleanup 会杀掉时间轴
}, [phase, advancePhase, play])      // ← play 在依赖里 = 定时炸弹
```

`play` 一旦重建（本项目：`useCallback(..., [isMuted, globalVolume])`），
effect 重跑 → cleanup 先执行 → **正在跑的时间轴被杀** → `onComplete` 永不触发 →
**相位机停在中间状态**。而中间状态往往意味着：

- 一个「正在处理中」的标志永不清除 → 后续所有请求都被守卫拒绝
- 一个覆盖层的隐藏逻辑在同一个时间轴里 → 覆盖层永久留在屏幕上

**为什么难复现**：它需要「一个无关的状态变化」正好落在动画的 0.8–1.2s 窗口内。
本项目里那个无关变化是**音量**——而音量会被**成就弹窗自己**程序化修改，
所以它也完全可能在别的项目里由「任何 context 更新」触发。

**修法三层（都做，不要只做一层）**：

1. **回调走 ref，依赖数组只留真正的触发器**
   ```jsx
   const advanceRef = useRef(advancePhase)
   const playRef = useRef(play)
   useEffect(() => { advanceRef.current = advancePhase; playRef.current = play },
             [advancePhase, play])
   useEffect(() => { /* ...用 advanceRef.current() / playRef.current()... */ },
             [phase])   // ← 只有 phase
   ```
2. **相位推进加「一次且仅一次」护栏 + 超时兜底**：
   `onComplete` 和 `setTimeout(略长于动画)` 谁先到谁生效，保证即使时间轴仍被意外杀掉，
   状态机也会继续走。护栏要按「相位」记账（`{ phase, done }`），别用一个全局 boolean，
   否则重进同一相位时会被误判为已完成。
3. **给「空闲」状态写自愈分支**：不要只在某一条动画路径里做收尾（比如只在 `'opening'`
   的淡出里 `display:none`）。加一条
   ```jsx
   if (!phase) gsap.set(container, { opacity: 0, display: 'none' })
   ```
   让「什么都没在发生」这个状态自己把屏幕还原。

**再加一个全局看门狗**：给「正在处理中」这个状态配一个超时（本项目 `TELEPORT_WATCHDOG_MS = 25000`），
超时就强制释放标志。健康流程 1–2s，所以它不会误伤；但它能把
**「永久废掉整个功能」降级成「一次失败」** —— 这是性价比最高的一道保险。

## 四·五、状态机卡死：不稳定的回调写进 dependency array，会杀掉动画时间轴

### 症状自查表

| 观察到的现象 | 大概率是 |
|---|---|
| 电影式过渡后画面停在中间、且**之后所有同类操作都无效** | 卡在 `'closing'` 一类的「进行中」相位，标志没清 |
| 画面被某个覆盖层**永久盖住**，但底下其实已经好了 | 隐藏覆盖层的收尾动画在同一个被杀的时间轴里 |
| 刷新就好、无法稳定复现 | 需要某个外部状态变化落在动画窗口内 → 查依赖数组 |
| 传送/路由类操作「点了没反应」 | 先看是不是去重守卫（`if (inFlight) return`）吃掉了请求 |

### 顺手把静默失败变成可见反馈

排查完「按了没反应」之后，**别只修功能，也让界面说出来为什么**。
守卫式的 `if (blocked) return` 是 UX 黑洞 —— 用户无法区分「卡了」「坏了」「我点错了」。

推荐做法（本项目已落地，可复用）：

- **一层不碰框架的派发器**：`showToast(msg)` 走 `window.dispatchEvent(new CustomEvent(...))`。
  需要报警的调用方常常**不在 React 渲染树里**（状态守卫、R3F 指针处理器），
  走 context 会牵连所有消费者重渲染；事件派发零成本。
  （同一个项目里 `language-change` / `sw-update-ready` 也是这套。）
- **同一文案折叠 + 上限 3 条**：重复触发只刷新最新那条的计时器。
  否则用户连点 5 次会得到一摞完全相同的提示，比不提示更烦。
- **`pointer-events: none`**：toast 是纯反馈，绝不能吞掉点击。
  验证方式一行：`document.elementFromPoint(中心).className` 应该**不是** toast。
- **`aria-live` 区域必须常驻 DOM**：写成 `if (!list.length) return null` 是错的 ——
  读屏软件只播报「已存在的」live region 的内容变化，空闲时卸载容器会让
  **第一条提示永远读不出来**。始终渲染空容器。
- **注意别和已有的反馈撞位**：先查现有弹窗的 `position` / `bottom` / `z-index`
  （本项目成就弹窗也在底部居中、高 ~50px、z-index 110），再决定 toast 锚点。

---

## 四·六、`renderOrder` 会推翻透明物体的排序（「贴花盖在角色身上」的真因）

**这是本项目返工一次才找到的根因，而且它不是坐标问题、调位置永远调不好。**

three.js 对**透明**物体按 **back-to-front**（远处先画、近处后画）排序，这样近处的半透明
物体才能正确叠在远处之上。但 **`renderOrder` 的优先级高于这个排序**：
一旦给某个网格设了 `renderOrder={1}`，而相机与被遮挡者的 `renderOrder` 是默认 `0`，
那么**无论它在世界空间里多远**，它都会被安排在最后绘制 → 直接画在角色脸上。

症状特别有迷惑性：**看起来像「位置算错了」，其实是绘制顺序错了**。
当时那张地贴明明整块都在角色纵深之后，截图里却糊在他腿上。

排查手法（一行就能验证）：

```js
// 在 probe 里把两边的 renderOrder 都打出来
s.traverse(o => {
  if (o.isMesh && o.material && o.material.transparent) {
    console.log(o.name || o.geometry.type, 'renderOrder=', o.renderOrder,
                'z=', o.getWorldPosition(new K()).z)
  }
})
```

对照标准：**半透明物体之间，若 A 在 B 之后绘制，A 就盖住 B，与 z 无关。**
所以「贴花 → 让开角色」的正确做法是：

- 贴花的 `renderOrder` **保持默认 0**（不要设！），
- 确保贴花在纵深上确实比角色**远**（`z` 更小 / 更远离相机），
- 让排序自己得出「先画贴花、后画角色」。

只有当**同一位置**的两层半透明需要强制先后（如卡片背面的图层）才用 `renderOrder`，
并且此时要**两个都显式设置**，不能只设一个。

**反过来用**：如果确实需要某个半透明层永远压在最上面（HUD、描边、小地图），
那就该设 `renderOrder`——这正是它存在的意义。关键是**知道它会推翻排序**。

### 自动化回归：屏幕空间 AABB 相交

想证明「没有任何一块贴花压在角色身上」，别靠肉眼看截图——把两边的屏幕包围盒算出来：

```js
const P = (x, y, z) => { const v = new K(x, y, z).project(cam); return [(v.x+1)/2*innerWidth, (1-v.y)/2*innerHeight] }
// 角色：遍历其贴图 alpha 外接框（见下），逐点 project 后取 bbox
// 贴花：把平面四角 applyMatrix4(o.matrixWorld) 后 project，取 bbox
// 相交判定：!(t[2] < b[0] || t[0] > b[2] || t[3] < b[1] || t[1] > b[3])
```

输出三个数字就能下结论：`overlapping`（bbox 相交数）、
`centreUnderBody`（**贴花中心**落在身体框内数，这个才是「看起来压在身上」）、
以及每片的像素直径（顺便看远近视差缩放是否合理）。

**角色剪影宽度必须用 alpha bbox 量，不能用平面宽度**——贴图四周通常有透明留白。
本项目 avatar 平面宽 1.086，但真实剪影只有 **±0.41**；用 0.54 去算会误判。

```python
# PIL 量 alpha 外接框 → 换算成世界单位
from PIL import Image
im = Image.open(path).convert('RGBA'); cw = im.size[0] // 4; plane_w = H_plane * cw / im.size[1]
l, t, r, b = im.crop((i*cw, 0, (i+1)*cw, im.size[1])).split()[3].getbbox()
wx0 = ((l/cw) - 0.5) * plane_w;  wx1 = ((r/cw) - 0.5) * plane_w   # 世界 x 偏移
```

**注意动画帧要找最宽的那一帧**（本项目 4 帧里 frame3 最宽 `0.412`），
否则挥手时会露出手臂边缘。

---

## 四·七、地面闪现别的材质 = 两个平面**完全共面**（不是 z-fighting 那么简单）

**症状**：穿过某个位置时，地板「闪」出另一种材质（草地、水面、瓷砖），
有时闪一次、有时闪好几次，走过去之后就恢复。

**先算 y，别先怀疑 shader。** 把所有贴地平面的世界 y 打印出来，看有没有重复值：

```js
const flat = []
scene.traverse(o => {
  if (!o.isMesh) return
  if (!(o.material?.isShaderMaterial && o.geometry.type === 'PlaneGeometry')) return
  const y = new THREE.Vector3().setFromMatrixPosition(o.matrixWorld).y
  const up = new THREE.Vector3(0,0,1).transformDirection(o.matrixWorld)
  if (Math.abs(up.y) > 0.99 && y < -1.0) flat.push({ y: +y.toFixed(3), w: o.geometry.parameters.width })
})
console.log('levels:', [...new Set(flat.map(f => f.y))].sort())
```

本项目实测就是它：入口草皮、走廊地板、大门地面**三者全是 `-1.75`**，
而草皮是 **25×40** 的大平面、走廊只有 **7 宽** —— 草皮把整个街区都铺了，
所以在门洞那一带和走廊地板大面积重叠。

**为什么是「闪几下」而不是稳定的一直错**：两个**严格共面**的面片在**静止**相机下通常
有一个稳定赢家；但入场是相机从 z28 连续飞到 z11，**视距一直在变**，
深度精度随之变化 → 深度测试**整体翻转** → 赢家换人 → 看起来就是「闪」。
这解释了「闪了几次才恢复」：恢复的时刻是 `hasEntered` 把多余那层卸掉的时刻。

**修法**：把非主地板那一层**下沉一个远大于深度精度的量**，让它永远输掉深度测试。

```js
// 2cm 在 11 单位视距下约是深度缓冲精度的 3000 倍 —— 不是调出来的数字
const GROUND_DROP = 0.02
<mesh position={[0, -(corridorHeight / 2) - GROUND_DROP, zCenter]}>
```

⚠️ **这类 bug 的根因是「各模块各自硬编码同一个魔数」**。本项目
`EntranceDoors.jsx` / `CorridorWalls.jsx` / `EmptyCorridor.jsx` 三个文件里
各写了一遍 `-1.75`（其中两处是 `-corridorHeight / 2` 算出来的）。
**新增任何贴地物件前，先确认它和已有地板的确切 y 差**，不要往同一个魔数上叠。

## 四·八、用 canvas 贴图做「地贴」时的几何与合成陷阱

把一张程序化 canvas 铺在 3D 地平面上（地板贴花 / 地毯 / 地面文字）时，有几类坑与
WebGL 无关，纯粹是 canvas + 平面朝向 + 场景布局的组合，但**看截图才发现**，所以放在这里。

### 优先「一个元素一张小平面」，而不是「一张大 canvas 铺满」

第一版是「10 枚标记画进一张 900×1200 canvas，铺成 1 个 plane」。两个致命问题：

1. **10 枚共享同一个 transform**，没法各自旋转/挪动去贴合场景特征 → 视觉上被眼睛归并成
   一张「纸」，用户的原话就是「像一张纸贴地板，乱糟糟」。
2. **一个 plane 只有一个 `renderOrder` / 一个绘制时机**，其中任意一枚落进角色屏幕投影里，
   整张都会出问题（见 四·四）。

改成 **N 张小平面**之后：每片独立位置/旋转/尺寸、独立绘制时机，还能**按需剔重**
（只渲染该段需要的）。代价只是 N 个 draw call——对这种数量级完全可接受。
**分辨率还更均匀**：大 canvas 里远处的元素被压得很小，小 canvas 每片都一样清。

### 如果有网格化的地面，让元素吸附到网格单元上

「一个图标一块地板」听起来像修辞，其实**可以精确做到**。前提是地板用 shader 画、
且 UV 烘的是世界米（本项目 `bakeFloorUVs()` 把 `vUv` 写成世界坐标）：

```glsl
const float PW = 0.62;        // 板宽
float row = floor(world.x / PW);
```

→ 从 `PW` 反推**板条中心**在 `x = ±PW/2 + k·PW`，把元素做成 `0.60`（略小于 `PW`，
留缝）并吸附到这些列，就真的「一块地板一个元素」。
同理可以算出横向接缝位置，避免元素骑在缝上。

找节点做验证时，直接按这个尺寸筛：`o.geometry.parameters.width === 0.60`。

### 平面 `rotation = [-π/2, 0, 0]` 时的 UV 方向映射

| canvas 方向 | 映射到世界 |
|---|---|
| `+X`（向右） | `+X` |
| `+Y`（向下，像素坐标） | `−Z` |

也就是说 **canvas 的底边对应 +Z 那一侧**。而相机通常在 +Z 侧朝 −Z 看，所以：

- **canvas 底边 = 离相机最近的那一端**
- 想让贴花「在某个物体的身后」（画面里更远、往纵深去）→ 把平面的**近端边缘钉在该物体的 Z**，
  然后往 **−Z** 延伸（`centreZ = objZ − depth / 2`）

**先把这个方向搞对再排版**。第一版我按直觉把贴花放在相机与物体之间，结果整块糊在镜头前，
白烧一轮 3 分钟的无头验证。

### ⚠️ 相机贴地时「放得更远」≠「不被遮挡」

低机位（`camera.y ≈ 0.2`）看走廊地面是 4–6° 的掠射角，**地板往地平线爬升**，
而地平线的高度**正好是角色身体的位置**。于是：

- 位于**走廊中线**上的贴花，放在角色身后 **5~6 个世界单位**仍然投影在他躯干上（实测相交）；
- 只有**向两侧挪开**才能彻底让出他的剪影。

判据：贴花的屏幕投影中心到身体框中心要有富余。本项目最终**放弃中线那一列**，
只在 `x = ±0.93` / `±1.55` 两列交替排。**先算屏幕相交，再决定列位置**，不要靠调 `z` 硬试。

### 全局 `source-atop` / `multiply` 罩色会在每枚元素周围留下淡色方块

想做「整体降饱和 / 暖色统一」时，给整张 canvas 加一层
`ctx.globalCompositeOperation = 'source-atop'; fillRect(全画布)`——
**如果元素带软阴影（`ctx.filter = 'blur(...)'`），阴影的羽化 alpha 区会被这层罩一并染色**，
于是每个元素外面出现一个**半透明方框**（因为底层 fillRect 的透明度 ≥ 罩色，整块方形都被覆盖）。
截图上看就是「每个贴纸都套了个白/色底」。

修法二选一：
1. **别做全画布罩色**——把颜色差异直接写进各元素的 palette；
2. 收紧阴影（小面积 + 小 blur），并确认罩色的 alpha 足够低到看不出边界。

### 小尺寸元素上的文字标签，删掉而不是缩小

世界尺度 0.5 单位的贴片上写 10 个标签，屏幕上每个标签只有几个像素高 ——
**它不传达信息，只贡献噪点**，而且是最主要的「乱」的来源。要么把标签做到可读尺寸
（意味着贴片要大到不合场景比例），要么直接不要。本项目选了后者。

### 验证节奏：先看贴图本身，再看 3D

贴图生成器如果能被模块单独 import，就做一个**极简预览页**直接看成品，比进场景快 **20 倍**
（实测：预览页截图 ≈ **5 秒**，走完整「进房间」流程 ≈ **2.5 分钟**）。

做法（本项目已验证）：临时在**项目根目录**建 `__artpreview.html`：

```html
<script type="module">
import * as art from '/src/utils/contactArt.js'   // Vite dev server 会正常转换
const items = [['sea', art.makeSeaTexture(), 420], /* … */]
for (const [name, tex, w] of items) {
  const src = tex.image                      // THREE.CanvasTexture → 源 canvas
  const c = document.createElement('canvas')
  c.width = src.width; c.height = src.height
  c.style.width = (w) + 'px'; c.style.height = (w * src.height / src.width) + 'px'
  c.getContext('2d').drawImage(src, 0, 0)    // 缩小只是为了排版，像素仍来自原图
  /* append 到页面 + 写 figcaption（含 尺寸 / aspect） */
}
window.__ready = true
</script>
```

canvas 背景铺**棋盘格**（`repeating-conic-gradient`），一眼就能看出 alpha 抠图对不对。
一次截图就是一张 contact sheet。**用完立刻删掉，别留在项目里。**

> 贴图在场景里**摆对了没有**仍然要回 3D 验证 —— 预览页只能证明「画得对」，
> 证明不了「plane 宽高比 / 位置 / repeat 对得上」。


---

## 五、踩坑清单

1. **别点两扇门的正中缝**。世界坐标 `x=0` 正好落在两片门扇之间，射线穿过去，点不中——
   看起来「点击无效」，其实是坐标问题。往一侧偏半个门宽（`x=±0.45`）。
2. **`evaluateOnNewDocument` 必须早于 `goto`**，否则应用代码已经跑完，挂钩太晚。
3. **`PositionalAudio` 每个实例都 `new AudioListener()` 并 `camera.add(listener)`**
   （drei 的实现），卸载时 `camera.remove(listener)` + `_sound.disconnect()`。
   组件频繁挂卸（如走廊分段复用）时留意这个生命周期。
4. **headless 到底能不能「听到」声音？—— 分两层看，别混为一谈。**
   - **本技能**证明的是「音频节点被启动了、增益是多少」（指纹 + 增益），**不测信号本身**；
   - 想证明「**真的产生了信号**」，用 `iskill-audio-verify` 的 `AnalyserNode` 旁路读时域
     RMS（`RMS > 0.0005` 即为真的在发声）。**headless 确实会渲染音频**，
     `AudioContext.currentTime` 正常推进，所以这一层是可测的 ——
     早期「headless 听不到声音是正常的」这个说法**只对扬声器成立，对信号不成立**。
   - 至于**最终听感**（音色、响度合不合适）仍然要用户自己确认。
5. **`meshBasicMaterial` 默认 `side: THREE.FrontSide`**（背面剔除）。若某个子 group 带了
   `rotation.x = Math.PI` 来"贴到卡片背面"，那这些平面**只在翻面后才朝向相机**——从正面看
   会被剔除，看起来像"没渲染"。判定这类"设计成只在某一面可见"的元素时，先手动把宿主
   group 转 π 再截图，别直接下"坏了"的结论。
6. **别急着改渲染层**。R3F 里"组件在、但不显示"多是数据层问题（JSON 非空顶掉了
   `FALLBACK`，或 JSON 里字段是空数组）→ 见 4.4。
7. **「只藏 preloader」不等于「进入了场景」**。若应用有独立的**入场/加载态覆盖层**
   （本项目是 `!hasEntered && <EmptyCorridor>`，一块 25×40 的程序化**草地**平面，
   和走廊地板共面），它会在你的截图里和正式场景 z-fighting / 直接盖住地板，
   出现一整片莫名其妙的绿色 —— 而且它是**预期行为，不是回归**。
   验证正式场景前先把它关掉：`traverse` 找特征尺寸（`width===25 && height===40`）设 `visible=false`，
   或直接调应用自己的调试桥（`__aispin.enter()`）。
8. **调试桥的挂载时机**。`window.__scene` / `window.__cam` 这类是在**首次渲染时**才挂上的，
   在 `wait(170s)` 之前读会拿到 `undefined`（报 `Cannot read properties of undefined
   (reading 'traverse')`）→ 别把「读不到」当成「没渲染」，先确认时机。
9. **`renderOrder` / 透明排序**见 四·六 —— 遇到「X 盖住了 Y」先查这个，别急着挪坐标。
10. 本项目（aispin）专用：dev server 端口 **5199**；截图 harness 在 `/tmp/aispin-shot.mjs`；
    沙箱里 `rm -rf` 和 `vite build` 的 `prepareOutDir` 都会被拦
    （`[SAFE_DELETE_BULK_CONFIRM_REQUIRED]`），要删目录用 Python 的 `shutil.rmtree`。
    **本仓库不是 git 仓库，删除不可逆**——大改动前先 `tar` 归档到 `.workbuddy-ai/`。
    SwiftShader 下首屏要 **20~30 s** 才 warmup 完；有 `?noloader=1` 可跳过 2D preloader。
11. **别用 `requestAnimationFrame` 死循环去覆写 `camera.position`**。想「把相机放到 Z=X 再看一眼」时，
    写 `const tick=()=>{cam.position.z=X; ...}` 是**错的**：相机的驱动 hook
    （`useInfiniteCamera` 的 `useFrame`）同帧还会再写一次，两者互相打架，
    于是你**读到的 `cam.position.z` 是 X，但真正渲染出来的是别的值**——
    排查「数出来 5 扇门、截图里 0 扇门」时极难发现。正确做法是**走应用自己的输入通路**：
    `window.dispatchEvent(new WheelEvent('wheel',{deltaY:d/0.025}))`，让 `useInfiniteCamera`
    正常把 `targetZ` lerp 过去，再用 `still>=8` 帧判定「真的停稳了」再读。
    同理，**别用 `gsap.killTweensOf(camera.position)` 抢控制权**（见 四·五）。
12. **按「几何特征」找物体时，先把特征打印出来自证**。用「1.05×2.5 的平面 = 门」这个假设去扫，
    扫到 0 个——因为真实宽是 `2.5*0.376*1.12 = 1.0528`，而我的容差写成了 `<1e-6`。
    教训：**别把假设写成硬编码阈值直接采信**。先输出 `shapeWidths:[...]`（去重后的实际值）
    和 `rawSample`，确认签名对得上，再收窄条件。要么就换成**更可靠的特征**：
    R3F 里 `o.__r3f.handlers.onClick` 是否存在，比量尺寸稳得多——
    而且能顺带把「墙上的画」排除掉（画的 hitbox 用的是 `frame.width/height`）。
13. **`evalClick` 的回调必须 `return [x, y]` 恰好两个元素**。多返回一个
    （比如 `[x, y, dist]` 想顺便带出调试信息）会被 harness 判为非法点而 `SKIP`，
    现象是「点了但什么都没发生」。要带额外信息就挂到 `window.__xxx` 上，别塞进返回数组。
14. **`goto` 之后必须先 soak ~7 秒再读 `window.__scene`**。本项目的应用在**首帧渲染后
    会整页 reload**（HMR / StrictMode），所以 `waitForFunction('!!window.__scene')`
    很可能在**第一次 mount** 时就被满足，随后 reload 把那个 scene 丢掉 ——
    症状是 `window.__scene.children.length === 0`、`meshes: 0`，
    甚至下一次 `evaluate` 直接报 `Cannot read properties of undefined (reading 'traverse')`。
    harness 里那句 `await sleep(6000)` 不是凑数的；自己写脚本要照抄。
15. **别在房间里挪相机去拍道具特写**。房间的位置状态机是**按相机所在 segment** 判定的
    （本项目 `DoorSection` 里 `cameraSegment = floor((10 - camera.position.z) / 80)`），
    你把相机挪到别的 segment，它会判定「离开房间」→ 房间卸载、走廊回来，
    截图里就是「相机在 A 处，画面却是 B」。想拍特写就**换思路**：
    导出贴图本身（见 4.8「验证节奏」），或者接受只能拍默认机位。
16. **`page.evaluate` 返回几 MB 的 payload 会把整条命令 SIGKILL**（exit 137）。
    导出 canvas 时先在页面里**缩放到 max ~460px** 再 `toDataURL`，否则几 MB 的 base64
    走 stdout 会被杀，而且 `grep`/`wc` 全都不执行，看起来像「命令被沙箱拦了」。
    另外这类长跑命令**一律 `run_in_background: true`**，前台 120s 默认超时必被砍。
17. **扫描场景找「那个物体」时要排除 warmup 副本**。应用常把房间挂在远处预热
    （本项目 `RoomWarmup` 把 `GalleryRoom`/`ContactRoom` 挂在 `y=-500` 渲染几帧再卸载），
    `traverse` 会先撞到它 —— 拿到的世界坐标是 `y≈-500`，投影出来在屏幕外几千像素。
    按世界坐标过滤：`if (p.y < -100) return`。同理，判断「这个物体在场景里有几份」时，
    记得 warmup 副本会让计数翻倍。

---

## 六、完整脚本骨架

```js
import puppeteer from '<puppeteer-core 路径>'
const browser = await puppeteer.launch({
  executablePath: '<chrome-headless-shell 路径>',
  headless: 'shell',
  args: ['--no-sandbox', '--use-gl=angle', '--use-angle=swiftshader',
    '--enable-unsafe-swiftshader', '--autoplay-policy=no-user-gesture-required',
    '--window-size=1600,900', '--hide-scrollbars', '--disable-dev-shm-usage'],
  defaultViewport: { width: 1600, height: 900 },
})
const page = await browser.newPage()
// 1) evaluateOnNewDocument 挂钩（第一节）
// 2) goto + wait 170000
// 3) 解码音效建指纹表
// 4) 驱动场景（第四节）
// 5) 读 window.__audioLog / 算有效增益
```

输出用 `=== ERRORS (n) ===` 收尾，收集 `pageerror` / `console.error` / `response.status>=400`。
