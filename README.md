# 粒子星球 · Particle Planet

> 单文件网页应用：用摄像头识别手势，实时驱动一个 Three.js 粒子星球。

**English summary** — A single-file, dependency-light web page that turns your webcam into a 3D controller. MediaPipe Hands tracks **21 hand landmarks locally in the browser (WASM)**; hand-written geometry (not a trained classifier) turns five gestures into changes in planet shape, colour and motion. **Nothing is uploaded, nothing is stored** — there is no backend, no model API and no API key. Open `index.html` and it just runs.

**▶ 在线体验：<https://guludaren.github.io/particle-planet/>**（需要摄像头权限与联网加载 CDN）

---

## 运行方式

三种方式任选其一，无需安装任何依赖、无需构建。

### 1. 直接双击打开

双击 `index.html` 用浏览器打开即可。

> `file://` 协议下通常可以工作，但部分浏览器会收紧摄像头权限。如果摄像头打不开、或一直停在「待机中」，请改用下面第 2 种方式。

### 2. 本地 HTTP 服务器（推荐）

在本目录下执行：

```bash
python -m http.server 8000
```

然后浏览器访问 <http://localhost:8000>。经过 HTTP 协议加载时摄像头授权最稳定。任何静态服务器都可以（`npx serve`、VS Code Live Server 等）。

### 3. 在线体验（GitHub Pages）

本仓库已开启 GitHub Pages，**打开就能玩**：<https://guludaren.github.io/particle-planet/>

自己部署一份：把仓库推到你的 GitHub，在 **Settings → Pages** 中选择从分支（如 `main` / 根目录）发布即可。由于入口文件已更名为 `index.html`，Pages 会自动以它作为首页。

> **关于文件名**：原始文件名是 **`星球.html`**，为了兼容 GitHub Pages 和各类静态服务器自动识别的入口约定，发布时复制并重命名为 `index.html`，文件内容完全一致（逐字节相同）。如果你更喜欢原名，直接改名回 `星球.html` 也能正常使用，只是需要手动在网址里指明文件名。

---

## 权限与隐私

**这一节是本文档最重要的部分，请先读完再授予权限。**

本应用会请求你的摄像头权限，理由是：**手势识别的输入源就是摄像头画面**。

- **它不采集屏幕，也不截屏。** 代码中没有任何 `getDisplayMedia` 调用，也没有任何截图逻辑。输入只有摄像头这一路。
- **它不上传任何东西。** 视频帧只在本机内存中传给 MediaPipe 做推理，处理完即被丢弃。
- **它不持久化任何东西。** 没有后端、没有数据库、没有 `localStorage` / `sessionStorage` / IndexedDB 写入，也没有任何埋点。
- **它没有 API Key，也不调用任何模型 API。** 手部识别模型（MediaPipe Hands 的 WASM 与模型文件）是从 jsDelivr CDN 下载后**在你的浏览器里本地运行**的。
- 页面里有一个 `<video>` 元素承载摄像头画面，但它是**隐藏的**（`width:2px;height:2px;opacity:0.01`，见 `index.html` 第 13 行），它纯粹被当作「取帧源」使用，不会显示在界面上。你**不会**在页面上看到自己。

**唯一的联网行为**是页面的 5 个 CDN `<script>` 标签，以及 MediaPipe 运行时从同一 CDN 拉取 WASM / 模型资源。因此：

- **完全离线无法使用**：首次加载需要网络。
- **不想授权摄像头时**：直接关闭浏览器标签页即可，页面不会保留任何状态。也可以在浏览器地址栏的权限设置里随时撤销。

关闭页面时，脚本会在 `beforeunload` 中主动停止摄像头轨道（`getTracks().forEach(t=>t.stop())`），释放摄像头。

---

## 手势对照表

把手放在摄像头前约一臂距离，保持手势稳定即可触发（表格中的「触发」为手势需要保持的时间，用于防抖）：

| 手势 | 效果 | 触发 |
|---|---|---|
| 🖐 五指张开 | 星球扩张 `expandTarget = 2.5` | 220 ms |
| ☝ 单指 | 🌌 银河：`rotTarget = 2.3`，光环层 scale ×2.5 | 300 ms |
| ✌ 两指 | 💕 爱心：变形为心形 `morphTarget = 1` + 生成爱心粒子 | 400 ms |
| 🤏 捏合 | 闪耀坍缩 `expandTarget = 0.25` | 700 ms |
| 无手 | 5 秒后自动回中 | — |

补充说明（均可在 `index.html` 第 497–521 行核对）：

- 手势判定是**手写几何规则**，不是训练出来的分类器：
  - `isFingerUp`：指尖到手腕的距离 > 对应指关节到手腕的距离 × 1.12；
  - `isThumbUp`：同样的思路，阈值系数 × 1.06；
  - 捏合：拇指尖（landmark 4）与食指尖（landmark 8）的归一化距离 < 0.045。
- 星球跟随的是**手掌中心**（landmark 0 与 landmark 9 的中点），而不是指尖：`planetTX = (0.5 - pcx) * 9`。
- 手离摄像头越近，星球越大：`distScaleTarget = 0.5 + pcy * 1.3`（约 0.5× ~ 1.8×）。
- 「银河」状态下两个光环层的缩放为 `2.5 + sin(t × 3) × 0.4`，即约 2.5–2.9 倍的呼吸式扩张。

---

## 技术栈

| 项 | 说明 |
|---|---|
| 3D 渲染 | Three.js **v0.160.0**（CDN 引入），`THREE.Points` + `BufferGeometry` |
| 发光效果 | `AdditiveBlending` 叠加混合 + 在 JS 里用 canvas 现场生成的径向渐变发光贴图 |
| 手部追踪 | **MediaPipe Hands**（jsDelivr CDN），浏览器内 **WASM** 本地推理，输出 21 个手部关键点 |
| 识别参数 | `{maxNumHands: 1, modelComplexity: 1, minDetectionConfidence: 0.7, minTrackingConfidence: 0.5}` |
| 手势逻辑 | 纯手写几何判定（见上一节），无模型训练、无分类器 |
| 取帧方式 | 自己写的 `requestAnimationFrame` 循环，等上一帧处理完再送下一帧 |
| 粒子规模 | 6 个粒子层，初始共 **26,000** 个粒子；滑块可在 5,000–40,000 之间调节 |
| 构建 | 无。单文件、零构建、零 npm 依赖 |
| 后端 | 无 |

---

## 项目结构

整个项目只有**一个** HTML 文件。

```
particle-planet/
├── index.html    # 全部内容：样式 + 场景 + 手势逻辑
├── README.md
├── LICENSE
└── .gitignore
```

`index.html`（726 行）内部大致按以下区块组织：

| 行号（约） | 区块 | 内容 |
|---|---|---|
| 7–30 | `<style>` | 深色 UI、隐藏的摄像头 `<video>`、全屏按钮、控制面板 |
| 34–47 | DOM | `#three-container`、`#cam-video`、`#panel`（FPS / 手势名 / 粒子数 / 扩张幅度 / 换色按钮） |
| 49–53 | CDN | 5 个 `<script>`：Three.js + 4 个 MediaPipe 脚本 |
| 59–73 | 发光贴图 | `glowTex()` 现场生成径向渐变贴图 |
| 75–98 | 场景 | `WebGLRenderer`、相机、星空背景 |
| 100–127 | 粒子层 | `createLayer()`、6 个层的创建 |
| 129–161 | 反馈与粒子系统 | 扩张指示环、核心光晕、指向喷射粒子系统（**已废弃**，见已知问题） |
| 163–255 | 层数据 | 各层参考坐标（球体 / 球面 / 环面）、爱心变形目标、位置与缩放状态 |
| 257–311 | 颜色 | `distColor()`（按到中心距离渐变）+ 4 套主题与 `themedColor()` |
| 313–435 | 动画 | 手势状态对象、`updateAllLayers()` 逐粒子变换与着色 |
| 437–563 | 手势识别 | `isFingerUp` / `isThumbUp` / `pinchDist`、`onHandResults()` 手势映射、爱心粒子生成 |
| 565–588 | 摄像头 | `startCam()`：`getUserMedia` + MediaPipe Hands 初始化 + 自建取帧循环 |
| 590–668 | 主循环 | FPS 统计、5 秒自动回中、`animate()` |
| 670–723 | UI 与启动 | 全屏 / 滑块 / 换色 / 面板自动变暗 / 窗口自适应 / 启动 |

**粒子层构成**（`index.html` 第 119–126 行）：

| 层 | 初始粒子数 | 形态 |
|---|---|---|
| 0 核心 | 8,000 | 体积球 |
| 1 内壳 | 5,000 | 球面 |
| 2 外壳 | 4,000 | 球面 |
| 3 外晕 | 3,500 | 球面 |
| 4 光环 1 | 3,000 | 3D 环面 |
| 5 光环 2 | 2,500 | 3D 环面（更大半径） |

拖动「粒子总数」滑块时，总数按 `[0.30, 0.20, 0.17, 0.13, 0.12, 0.08]` 的比例分配到 6 个层。

**颜色主题**（4 套，点击「🎨 切换色调」循环切换）：暖金核心 / 紫罗兰 / 翡翠 / 烈焰。着色方式为按粒子到中心的距离做 HSL 渐变。

**界面元素**：FPS 计数器（低于 35 时变为警示色）、全屏按钮、控制面板（鼠标移开 2 秒后淡至 20% 不透明度，悬停恢复）、粒子总数滑块、「扩张幅度」滑块、换色按钮。

---

## 已知问题

以下问题在 `index.html` 中如实存在，未做修复，先记录在此以免误解：

1. **「扩张幅度」滑块是无效控件。** 它的 `input` 事件只更新旁边的数字标签（`v-expand`），从未写回 `expandTarget`，所以拖动它**没有任何视觉效果**（`index.html` 第 689–691 行）。
2. **粒子喷射系统是死代码。** `sprayPos` / `sprayActive` 等定义于第 150–161 行，但 `sprayActive` 全程只可能为 `0`，从未被写入有效值；早期设计注释里「食指指向 → 粒子束喷射」的说法已经过时——现在单指映射到的是 **🌌 银河**。
3. **5 个 CDN 脚本里有 3 个未被使用。** `camera_utils`、`control_utils`、`drawing_utils` 都没有被调用；应用用的是自己写的 `requestAnimationFrame` 取帧循环，不依赖 MediaPipe 的 `Camera` 工具类。留着它们只是无害的冗余请求。
4. **存在两套互相竞争的配色函数。** 逐帧动画走的是 `themedColor()`（跟随当前主题），而拖动「粒子总数」滑块会调用 `updateLayerColors()`，后者用的是写死的 `distColor()`（暖金 → 冷蓝的默认渐变）。因此**拖一次粒子数滑块，画面会瞬间回到默认主题配色**，直到下一帧逐帧着色再按当前主题覆盖回来，观感上就是一次可见的颜色跳变。
5. **需要联网 + 需要摄像头。** 渲染库和识别模型都来自 CDN，断网无法使用。推荐使用 **Chrome / Edge**（对 MediaPipe WASM 与摄像头权限支持最好）。`file://` 直开一般可用，但本地 HTTP 服务器在摄像头授权上更可靠。
6. **单层粒子数上限会「削顶」。** `MAX_PER_LAYER = 15000`，而滑块总量可到 40,000；按上面的比例分配时，核心层（0.30）在总量超过 50,000 时才会触顶，因此在当前量程内不会实际发生，但该上限确实存在。
7. **「粒子总数」的初始值前后不一致。** 滑块与标签的初始值是 24,000（第 42–43 行），而实际创建的 6 层粒子总数是 26,000（第 119–126 行）；只有在你第一次拖动滑块之后，两者才会由同一条比例规则统一。

---

## 许可与致谢

本项目采用 **MIT License**，详见 [LICENSE](LICENSE)。

- Copyright (c) 2026 杨钊文 (github.com/guludaren)

致谢与第三方组件（各自遵循其原始许可）：

- [Three.js](https://threejs.org/) — MIT License
- [MediaPipe Hands](https://github.com/google/mediapipe) — Apache License 2.0

本项目为单文件原创实现，未包含上述库的源码，仅在运行时通过 CDN 引用。
