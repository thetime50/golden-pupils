# 鹰眼系统调研

本文只做调研与量级估计，不代表已测精度。第一版产品范围：Web 手机摄像头，不做本地文件导入、不做 IP 摄像头接入。后两者仅作对比或预留说明。

稠密场景 3D（点云/网格）预留，本期不做。

---

## 1. 问题解答

### 1.1 纯前端能否向 IP 摄像头 App 要 FLV/RTSP？App 能否当服务端？

[IP Camera（沈垚，`com.shenyaocn.android.WebCam`）](https://play.google.com/store/apps/details?id=com.shenyaocn.android.WebCam) **可以当服务端**：内建 RTSP + HTTP，可到 4K / 60fps，Android 9+ 可同时开两路镜头，另有 RTMP/SRT 推流、ONVIF 查看、用户名密码（默认 admin）。


| 协议       | 浏览器能否直连               | 抽帧分析   | 说明                                         |
| -------- | --------------------- | ------ | ------------------------------------------ |
| RTSP     | 否                     | 否      | 必须网关（MediaMTX / go2rtc）转 WebRTC 或 HTTP-FLV |
| HTTP-FLV | 可播放（`flv.js`，需 H.264） | 需 CORS | 无 CORS 则 canvas 污染，读不到像素                   |
| MJPEG    | `<img>` 可预览           | 需 CORS | 带宽大、帧率不稳                                   |
| RTMP/SRT | 否                     | 否      | 推给直播服务器，不是浏览器拉流                            |


额外限制：HTTPS 页面拉 HTTP 局域网流会被混合内容拦截；该 App 通常不带 CORS。预览可以，分析帧需要同源代理或原生采集。

**第一版不做 IP 摄像头。** 后续若做：预览可走 LAN HTTP-FLV；分析走代理或 App 原生。

### 1.2 没有标定板，只用场地模型能不能校正？

能做到「一定程度」的外参和弱畸变，不能替代完整内参标定。

- 已知场地/棋盘尺寸当作标定物，检测线或角点，用 PnP / 单应估计相机相对场地的位姿。
- 足球转播常用 TVCalib 一类方法：用场地线段优化位姿、焦距，甚至估一个径向系数 k1。
- 内参（焦距、主点、完整畸变）仍弱于棋盘/ChArUco；手机广角误差更大。
- 棋类、魔方：近距、平面、特征密，无标定板也够用。
- 羽毛球高度：单机只能假设球在某平面附近；要高度必须多机或强运动模型。

### 1.3 纯 Web / 手机支不支持 2～3 路？（含 USB 与网络视频流）

这里的「路」指同时进分析管线的画面，不单指 `getUserMedia`。来源分三类：本机 `getUserMedia`、USB 摄像头、网络视频流（HTTP-FLV / MJPEG / WebRTC；RTSP 浏览器不能直连）。**预览**和**抽帧分析**要分开看：能播不等于能读像素。

#### 按接入方式


| 接入                                      | 1 路                                              | 2 路                                                                                                     | 3 路                                                                   |
| --------------------------------------- | ------------------------------------------------ | ------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------- |
| `getUserMedia`（手机/笔记本自带镜）               | Web/App 都稳。第一版用这个                                | 手机浏览器多数只给 1 路；少数可前后同时且分辨率下降。桌面可开 2 个设备（自带+USB 也走这条 API）                                                 | 浏览器基本开不出 3 个本机设备                                                      |
| USB 摄像头                                 | 桌面 Web 当普通摄像头，稳                                  | 视总线：USB2 上 2×1080p@60 未压缩易满；MJPEG/H.264 摄像头 2×1080p@30 常见能扛。手机浏览器几乎用不了 UVC；App + OTG 部分机型可「本机+USB」共 2 路 | 桌面需 USB3 / 多控制器，或降到 720p。手机 USB 3 路不现实                                |
| 网络视频流（IP 摄像头 App、ONVIF、转码后的 FLV/WebRTC） | 浏览器可播 HTTP-FLV/MJPEG（要 CORS/非混合内容才能分析）。RTSP 必须网关 | **能支持**：局域网 2 路编码流（各约 4–8Mbps）带宽够。分析必须同源代理或 App 原生拉流，否则 canvas 污染                                       | **勉强能支持预览**；3 路 1080p 解码+推理，手机 Web 算力不够，桌面 Web 可试但卡顿/不同步会明显。App/服务更合适 |


RTSP 本身不是「Web 多路」方案，要先变成 WebRTC 或 HTTP-FLV，见 1.1。

#### 按端（2～3 路怎么凑）


| 端         | 2 路是否可行                                                                    | 3 路是否可行                                                         | 较现实的凑法                    |
| --------- | -------------------------------------------------------------------------- | --------------------------------------------------------------- | ------------------------- |
| Web 手机浏览器 | 弱。双 `getUserMedia` 基本不行；**1 路本机 + 1 路 IP 流**在 Wi‑Fi LAN 上预览可以，分析仍卡 CORS/算力 | 基本不行（解码+WASM 推理）                                                | 不要用手机页扛 3 路原流             |
| Web 桌面    | **可以**：2×USB，或 1×USB + 1×IP 流。USB 带宽和双路推理是瓶颈，建议 720p/1080p@30              | **有条件可以**：2×USB + 1×IP，或 3×IP（经网关）。3×USB 不稳。分析 3 路建议降分辨率或只推理关键帧 | 多路 IP 流 + 本机代理，比硬堆 USB 更稳 |
| App       | **可以**：本机 1～2 路，或本机 + USB OTG，或本机 + IP 流（无浏览器 CORS）                        | **可以但吃力**：第 3 路用另一台手机/IP 摄像头；需软同步（一期二阶段）                        | 多机用网络流，不要指望一只手机出 3 路高清    |


同时开 2～3 路时，限制依次是：USB 总线、浏览器 CORS/混合内容、解码路数、推理算力、时间戳对齐（见 1.4、3.3）。局域网编码流的带宽通常不是第一瓶颈。

第一版只做 **1 路 Web 手机摄像头（`getUserMedia`）**。USB 多路和 IP 视频流接入放后续阶段。

### 1.4 多机帧对齐、卡顿、时间戳怎么处理？

第一版单机，无此问题。一期二阶段再用「时间窗对齐」，不要「等齐再算」。

- 每路保留 `capture_ts` / `encode_ts` / `recv_ts` / `process_ts`。
- 启动用闪光、击掌或音频峰做一次偏移标定。
- 丢帧用轨迹插值，标低置信，不阻塞其他路。
- 量级：Web 软同步约 20–80ms；App NTP 约 5–20ms；服务 PTP/硬件触发才到亚毫秒。
- 羽毛球约 300km/h 时，30ms 错位约 2.5m。二阶段须写明：只适合落点/慢球，不做法官级。
- 时间窗对齐是在各路里取 **接近同一时刻 t** 的帧（窗口内缺路就跳过或插值），不是取「现在最新的」帧，也不是等所有路到齐再算。
- `getUserMedia`、USB、网络流都有时间戳，但多半是本流相对 PTS 或驱动收帧时刻，跨设备不能直接当同一时刻。
- 要对齐，先用闪光/击掌/音频峰标定各路 `offset`，再用 `t_common = pts + offset` 在窗口内配对，实时的检测 修正时间偏移并记录；重投影误差突然变大则多半对错了时刻，作为后部验证。

### 1.5 纯前端资源不够，App 套壳会不会更好？

Web 瓶颈：相机 API、CORS、后台节流、难用 NPU、视频拷贝次数。棋牌第一版：**VideoFrame 尽量零拷贝进 WebGPU/WASM，禁止 canvas 来回 draw**。

App 套壳放在 **一期二阶段**：同一套 Web 页面，两种模式。

- 浏览器模式：用浏览器 WASM/WebGPU。
- App WebView 模式：同一套 UI，经兼容接口用手机原生/NPU 做采集和推理，Web 只显示。

模块名、接口、数据协议不变，只换 `platform` 和算力后端。套壳就是 WebView 出 UI，采集和推理走手机原生/NPU。

纯前端 1 路 1080p@30（本机预览、码率约 4–8Mbps）时 WASM/WebGPU 检测大约 10–20fps，2 路 1080p 解码加推理常掉到个位数 fps，还受 CORS 和后台节流。App 原生用 Camera2/NPU（旗舰约数十 TOPS），1 路 1080p 检测可到约 30fps，2 路 720p@30（各约 2–4Mbps）仍可试实时，原流不必经浏览器再编码。

### 1.6 树莓派、小型机值不值得做？

裸 Raspberry Pi 5 推理明显弱于旗舰手机（YOLOv8n 常见数 FPS 级），**不考虑**。闲置手机当摄像头更合适。

若必须做低成本边缘盒：Pi 5 + Hailo，或 Jetson Orin Nano（约 20–40 TOPS INT8）。不要为「比手机贵但更慢」的板子单独开一条产品线。

### 1.7 给定 2D/3D 模型，能不能做特征识别和绑定？

可行。引擎只做检测绑定框架（插件加载器在引擎，场景资源不在引擎项目里）。场景尺寸、识别模型、业务逻辑、交互页放在独立插件仓。流程：关键点 → 2D-3D 对应 → PnP/单应 → 绑到标准模型。详见设计文档。

---

## 2. 技术框架（软硬件）

### 2.1 三端同一模块、不同实现深度


| 模块      | Web 第一版               | App（一期二阶段）       | 服务/PC（二期）        |
| ------- | --------------------- | ---------------- | ---------------- |
| 采集      | 手机 `getUserMedia` 1 路 | 本机 1～2 路 + 后续 IP | 1～n 路 USB/IP/工业  |
| 同步      | 单机，无多机                | 软同步 / NTP        | NTP → PTP / 硬件触发 |
| 标定+去畸变  | 场地/棋盘 PnP，弱畸变         | 同左，可加棋盘标定        | 完整内参 + 复标        |
| 场地/盘面特征 | 线、格、角点                | 同左，NPU 更快        | 更大模型、更高分辨率       |
| 非线性扭曲   | 纸面棋盘透视+轻度弯曲           | 同左               | 更重的 warp         |
| 物体追踪    | 棋子/色块；球类预留            | 羽毛球开始做           | 多路球追踪            |
| 稀疏 3D   | 单机平面假设                | 2 机三角            | 多视三角             |
| 稠密 3D   | 预留                    | 预留               | 预留               |
| 置信度     | 检测分 + 几何残差            | 同左               | 再加多机一致性          |
| 绑定显示    | Three.js / 2D 叠加      | 同一套 Web          | 无头服务只出 JSON/glTF |


PC 与服务器是同一类引擎，差别是包装：实时走 HTTP/WebSocket，非实时走文件；PC UI 用 Web 套壳调本机 loopback。
置信度： 每个阶段 每个模块产生的误差都要记录下来

### 2.2 采集与推理栈（建议，非本期落地）


| 层   | Web                               | App WebView                            | 服务                               |
| --- | --------------------------------- | -------------------------------------- | -------------------------------- |
| 采集  | `getUserMedia` + WebCodecs        | Camera2 / AVFoundation                 | Video4Linux / DirectShow / ONVIF |
| 抽帧  | `VideoFrame` → GPU，少拷贝            | 原生 buffer → NPU                        | FFmpeg / DeepStream              |
| 推理  | ONNX Runtime Web / TF.js / WebGPU | ONNX Runtime Mobile / TFLite / Core ML | ONNX / TensorRT                  |
| 几何  | 自研或 OpenCV.js                     | OpenCV 原生                              | OpenCV                           |
| 显示  | 同一套 Vue/Nuxt 页面                   | 同一套页面                                  | 无，或 PC 套壳                        |


不要把 TF.js 当主格式，ONNX 三端能直接吃同一份权重。
所以几何仍用 OpenCV.js，像素操作用 WebGPU，检测用 ONNX；

### 2.3 硬件档位


| 档       | 设备                           | 用途                       |
| ------- | ---------------------------- | ------------------------ |
| 低成本离线   | 近年中旗舰手机                      | 第一版 Web；二阶段 App          |
| 标准单机    | 游戏本 RTX 4060 级，16–32GB       | 二期本地基础引擎                 |
| 标准增强    | 单机 RTX 4080/4090             | 多路、更高分辨率                 |
| 边缘盒（可选） | Jetson Orin Nano 或 Pi5+Hailo | 场馆盒子，不推荐裸 Pi             |
| 云开发     | 阿里云 PAI-DSW（A100 等）          | 会话式训练/重算，用完关机；不当 7×24 采集 |
| 云生产     | 阿里云 PAI-EAS                  | 检测/追踪按量推理，闲时缩到 0         |


口径不同，数字不能直接横比。`TOPS（峰值）` 列优先写 dense/sparse；手机 NPU 多为单一 INT8 峰值；GeForce 40 系多为 INT8/FP8 Tensor，50 系宣传峰值多为 FP4。


| 类别     | 产品                           | 芯片                                     | TOPS（峰值）                                                                                                                                     | 口径                                                                                       | 来源                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                     |
| ------ | ---------------------------- | -------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 手机     | 华为 P60 Pro                   | Snapdragon 8+ Gen 1（7th Gen AI Engine） | **27**                                                                                                                                       | 发售 2023-03；AI Engine / Hexagon（与 8 Gen 1 同档）                                             | [Lantronix 写明 8 Gen 1 = 27 TOPS](https://www.lantronix.com/products/snapdragon-8-gen-1-mobile-hardware-development-kit/)；[Beebom 对照表同为 27 TOPS](https://beebom.com/snapdragon-8-gen-1-vs-snapdragon-8-plus-gen-1/)                                                                                                                                                                                                                                                                                                                     |
| 手机     | 小米 17 / 17 Ultra             | Snapdragon 8 Elite Gen 5（第五代骁龙 8 至尊版）  | **89**                                                                                                                                       | 发售 2025-12；INT8 推算：8 Gen 3 = 45 → Elite ×1.45 = 65 → Elite Gen 5 官方再快 37% → 65×1.37 ≈ 89 | [Forbes：8 Gen 3 = 45 TOPS](https://www.forbes.com/sites/moorinsights/2023/10/25/ai-dominates-qualcomm-snapdragon-summit-with-new-snapdragon-products/)；[Elite Product Brief：NPU +45%](https://docs.qualcomm.com/doc/87-83196-1/87-83196-1_REV_D_Snapdragon_8_Elite_Mobile_Platform_Product_Brief.pdf)；[Qualcomm：Elite Gen 5 Hexagon +37%](https://www.qualcomm.com/news/releases/2025/09/snapdragon-8-elite-gen-5--the-world-s-fastest-mobile-system-on-a)；[PChome：小米 17 Ultra 发售与芯片](https://article.pchome.net/content-2192164.html) |
| 手机     | OPPO Find X8 / Find X8 Pro   | Dimensity 9400（NPU 890）                | **50**                                                                                                                                       | 发售 2024-10；NPU INT8 峰值                                                                   | [DIGITIMES：NPU 890 = 50 TOPS](https://www.digitimes.com/news/a20241011PD210/mediatek-dimensity-mobile-flagship-npu.html)；[NanoReview：theoretical 50 TOPS](https://nanoreview.net/en/soc-compare/mediatek-dimensity-9400-plus-vs-mediatek-dimensity-9300)                                                                                                                                                                                                                                                                               |
| 手机     | 华为 Mate 70 Pro / Pura 80 系列  | Kirin 9020（Da Vinci NPU）               | **未公布**                                                                                                                                      | Mate 70 发售 2024-11、Pura 80 发售 2025-06；华为/海思规格页不给峰值 TOPS                                  | [TechInsights：Mate 70 Pro+ = Kirin 9020](https://www.techinsights.com/blog/huawei-mate-70-pro-exploring-hisilicon-kirin-9020-processor)；[海思麒麟规格页无 TOPS 数字](https://www.hisilicon.com/cn/products/kirin/kirin-flagship-chips/kirin-9000)                                                                                                                                                                                                                                                                                                |
| 手机（对照） | 华为 Mate 40 代（麒麟 9000）        | Kirin 9000 Da Vinci 2.0                | **24**                                                                                                                                       | 发售 2020-10；INT8 NPU 峰值（历史对照；非现款）                                                         | [21IC 技术资料：Kirin 9000 = 24 TOPS INT8](https://dl.21ic.com/download/kirin_npu-929616.html)                                                                                                                                                                                                                                                                                                                                                                                                                                              |
| 游戏卡    | GeForce RTX 4060             | Ada AD107                              | INT8 dense/sparse = **121 / 242**                                                                                                            | 发售 2023-06；NVIDIA 对比页标称 242 为含稀疏                                                         | [NVIDIA GeForce 对比页](https://www.nvidia.com/en-us/geforce/graphics-cards/compare/)                                                                                                                                                                                                                                                                                                                                                                                                                                                     |
| 游戏卡    | GeForce RTX 4080             | Ada AD103                              | INT8 dense/sparse = **390 / 780**                                                                                                            | 发售 2022-11；NVIDIA 对比页标称 780 为含稀疏                                                         | [NVIDIA GeForce 对比页](https://www.nvidia.com/en-us/geforce/graphics-cards/compare/)；[RTX 4080 产品页](https://www.nvidia.com/en-us/geforce/graphics-cards/40-series/rtx-4080-family/)                                                                                                                                                                                                                                                                                                                                                      |
| 游戏卡    | GeForce RTX 4090             | Ada AD102                              | INT8 dense/sparse = **660.6 / 1321.2**                                                                                                       | 发售 2022-10；产品页 1321 = sparse                                                             | [NVIDIA GeForce 对比页](https://www.nvidia.com/en-us/geforce/graphics-cards/compare/)；[Ada 架构白皮书](https://images.nvidia.com/aem-dam/Solutions/geforce/ada/nvidia-ada-gpu-architecture.pdf)；[RTX 4090 产品页](https://www.nvidia.com/en-us/geforce/graphics-cards/40-series/rtx-4090/)                                                                                                                                                                                                                                                        |
| 顶级消费卡  | GeForce RTX 5090             | Blackwell GB202                        | FP4 dense/sparse = **1676 / 3352**；INT8 dense/sparse = **838 / 1676**                                                                        | 发售 2025-01；宣传 3352 = FP4 sparse（非 INT8）                                                  | [NVIDIA CES 新闻稿：3352 AI TOPS](https://nvidianews.nvidia.com/_gallery/download_pdf/677c9a9c3d6332e22e6c8571/)；[Blackwell 白皮书口径说明](https://aivideosensei.com/guides/rtx-5090-ai-tops-explained)                                                                                                                                                                                                                                                                                                                                          |
| 专用卡    | NVIDIA A100 80GB（SXM / PCIe） | Ampere GA100                           | INT8 dense/sparse = **624 / 1248**                                                                                                           | 发布 2020-05                                                                               | [NVIDIA A100 产品页](https://www.nvidia.com/en-us/data-center/a100/)；[A100 Datasheet PDF](https://www.nvidia.com/content/dam/en-zz/Solutions/Data-Center/a100/pdf/nvidia-a100-datasheet-us-nvidia-1758950-r4-web.pdf)                                                                                                                                                                                                                                                                                                                     |
| 专用卡    | NVIDIA H100 80GB             | Hopper GH100                           | SXM INT8 dense/sparse = **1979 / 3958**；PCIe INT8 dense/sparse = **1513 / 3026**                                                             | 发布 2022-03                                                                               | [NVIDIA H100 产品页](https://www.nvidia.com/en-us/data-center/h100/)；[H100 Datasheet PDF](https://www.pny.com/File%20Library/Company/Support/Product%20Brochures/NVIDIA%20Data%20Center%20GPUs/english/nvidia-h100-datasheet.pdf)；[Hopper 架构白皮书](https://www.techpowerup.com/gpu-specs/docs/nvidia-gh100-architecture.pdf)                                                                                                                                                                                                                |
| 专用卡    | NVIDIA H200 141GB            | Hopper GH100                           | SXM INT8 dense/sparse = **1979 / 3958**；NVL INT8 dense/sparse = **1671 / 3341**                                                              | 发布 2023-11；算力同代 H100，显存升至 141GB HBM3e                                                    | [NVIDIA H200 产品页](https://www.nvidia.com/en-us/data-center/h200/)                                                                                                                                                                                                                                                                                                                                                                                                                                                                      |
| 专用卡    | NVIDIA B200                  | Blackwell                              | INT8 dense/sparse = **4500 / 9000**；FP4 dense/sparse = **9000 / 18000**                                                                      | 发布 2024-03，量产约 2024-Q4；单位为 TOPS（= POPS×1000）                                             | [Lenovo HGX B200 规格：4.5/9 POPS、9/18 PFLOPS](https://lenovopress.lenovo.com/lp2226-thinksystem-nvidia-b200-180gb-1000w-gpu)；[NVIDIA B200 Datasheet](https://www.megware.com/fileadmin/user_upload/LandingPage%20NVIDIA/nvidia-b200-datasheet.pdf)                                                                                                                                                                                                                                                                                       |
| 专用卡    | NVIDIA GB200                 | Grace + 2×Blackwell                    | Superchip：INT8 dense/sparse = **10 / 20 POPS**，FP4 dense/sparse = **20 / 40 PFLOPS**；NVL72：INT8 **360 / 720 POPS**，FP4 **720 / 1440 PFLOPS** | 发布 2024-03；Superchip=1 Grace+2 GPU，NVL72=36 Grace+72 GPU 机柜                              | [NVIDIA GB200 NVL72 规格页](https://www.nvidia.com/en-us/data-center/gb200-nvl72/)                                                                                                                                                                                                                                                                                                                                                                                                                                                        |
| 边缘盒    | Jetson Orin Nano Super（8GB）  | Orin                                   | INT8 dense/sparse = **33 / 67**                                                                                                              | 原版 Orin Nano 发售 2023-03、Super 软件升档 2024-12；MAXN_SUPER                                    | [NVIDIA Orin Nano Super 页](https://www.nvidia.com/en-us/autonomous-machines/embedded-systems/jetson-orin/nano-super-developer-kit/)；[NVIDIA 技术博客 dense/sparse 对照](https://developer.nvidia.com/blog/nvidia-jetson-orin-nano-developer-kit-gets-a-super-boost/)                                                                                                                                                                                                                                                                         |
| 边缘加速   | Hailo-8（含 Pi5+Hailo 方案）      | Hailo-8                                | INT8 = **26**                                                                                                                                | 量产上市约 2021                                                                               | [Hailo-8 产品页](https://hailo.ai/products/ai-accelerators/hailo-8-ai-accelerator/)；[Hailo-8 Datasheet：up to 26 TOPS](https://www.farnell.com/datasheets/4746516.pdf)                                                                                                                                                                                                                                                                                                                                                                     |



|            | RTX 4090     | A100 80GB        |
| ---------- | ------------ | ---------------- |
| INT8 dense | ~661 TOPS    | 624 TOPS         |
| 显存         | 24 GB GDDR6X | **80 GB HBM2e**  |
| 带宽         | ~1008 GB/s   | **~2 TB/s**      |
| 多卡         | 基本无 NVLink   | **NVLink / MIG** |
| ECC / 稳跑   | 消费级          | 数据中心级            |
| 典型用途       | 单机推理、本地微调    | 云训练、大 batch、长时任务 |


- 显存 带宽 多卡
- INT8：8 bit 整数精度，检测/分类等视觉推理最常用的量化口径，生态成熟。
- FP4：4 bit 浮点，位数更少、峰值吞吐更高，但不是「比 INT8 高级」，精度更粗且依赖新软件栈。
- dense：权重大致全参与计算时的峰值，更接近多数真实模型能摸到的上限。
- sparse：启用 2:4 结构化稀疏后的峰值（常约为 dense 的 2 倍），模型未按该格式剪枝时达不到。
- SXM-INT8：数据中心卡（如 H100）SXM 板型的 INT8 Tensor 峰值，功耗/互联更高，云主机常见。
- PCIe INT8：同一代卡的 PCIe 板型 INT8 Tensor 峰值，插槽兼容更好，但峰值通常低于同代 SXM。

生产云各一句话：EAS 把模型做成按量服务，有请求再扩、闲时缩容省钱；DSW 用 A100 做开发或偶发重算，用完关机，不当采集节点。可再预留异步队列、ICE/直播转码。

有服务器时：相机直推服务，终端只收轨迹，减本地算力和上行原流。

---

## 3. 参数标准

量级估计，不是实测。球与场地（或棋盘/物体）分开。三种输入：Web 摄像头、App、文件（后续对比，第一版产品不做文件）。

### 3.1 手机摄像头常见指标


| 项     | 常见值                     | 对分析的影响               |
| ----- | ----------------------- | -------------------- |
| 拍照像素  | 1200 万级                 | 分析用视频，不看静态千万像素       |
| 视频    | 1080p@30/60，部分 4K@30/60 | 第一版建议 1080p@30，省电省算力 |
| 快门    | 卷帘快门                    | 快球拖影、倾斜              |
| AE/AF | 自动                      | 棋盘尚可；球类会曝、会跟焦抖       |
| 视场    | 主摄约 70–80°              | 太近裁场地，太远 GSD 变差      |


GSD 粗算：1080p 横框约 16m 宽的羽毛球场地 → 约 8mm/px；40cm 棋盘占画面约 70% → 约 0.3mm/px。

### 3.2 精度量级（业余、标定尚可）


| 场景           | 输入              | 场地/盘面        | 球或物体           |
| ------------ | --------------- | ------------ | -------------- |
| 围棋/象棋/魔方 1 机 | Web 摄像头         | 盘面毫米～厘米，格线可辨 | 子/色块：格级或色块级    |
| 同上           | App（NPU，可更高分辨率） | 略好于 Web，仍是格级 | 同左，帧率更稳        |
| 同上           | 文件 1080p 离线     | 与拍摄时分辨率绑定    | 可多帧投票，略稳       |
| 羽毛球 1 机平面    | Web             | 线 2–10cm     | 球 5–30cm，快球易丢  |
| 羽毛球 1 机      | App             | 线 2–8cm      | 球 5–20cm       |
| 羽毛球 1 机      | 文件 60fps+       | 线 2–8cm      | 球好于实时，仍无高度     |
| 羽毛球 2 机三角    | App/服务          | 线 1–5cm      | 球 2–10cm（慢球更好） |
| 专业 Hawk-Eye  | 高速多机            | 约 2.6–3.6mm  | 约 2.6–3.6mm    |


### 3.3 同时工作的资源上限（后续多路时）


| 组合                  | 带宽              | 算力                      | 建议  |
| ------------------- | --------------- | ----------------------- | --- |
| Web 1 路 1080p30     | 本机，无上行          | 中端手机 WASM 约 10–20fps 检测 | 第一版 |
| Web 2 路 1080p60 USB | 易满              | 浏览器难扛双路推理               | 不建议 |
| App 2 路 720p30      | 本机或 LAN 数 Mbps  | 旗舰 NPU 可试               | 二阶段 |
| 服务 4 路 1080p + 轨迹回传 | 原流在服务端；终端数 KB/帧 | RTX 4060 级起             | 二期  |


---

## 4. 协议调研

自研协议对齐这些标准，场景格式（SGF 等）只从插件进出，不进引擎核心。


| 标准                                                                    | 用途                               | 自研如何对齐                         |
| --------------------------------------------------------------------- | -------------------------------- | ------------------------------ |
| [ONVIF Profile S](https://www.onvif.org/profiles/profile-s/)          | IP 相机发现、码流、配置                    | 后续 IP 适配器按 Profile S 拉流；第一版不用  |
| OpenCV camera YAML                                                    | `camera_matrix`、`dist_coeff`、分辨率 | 会话里的标定块与 OpenCV 字段同名，便于互导      |
| [glTF 2.0](https://registry.khronos.org/glTF/specs/2.0/glTF-2.0.html) | 场地/物体 3D                         | 场景包用 `.glb`                    |
| SMPTE ST 12 时间码                                                       | 广电帧对齐                            | 服务端预留字段；Web 用 UNIX ms + 帧序号    |
| [SGF](https://www.red-bean.com/sgf/)                                  | 围棋棋谱                             | `board.go` 插件导入导出              |
| FEN / PGN                                                             | 象棋/国际象棋                          | `board.xiangqi` 等插件            |
| WCA 色序约定                                                              | 魔方还原                             | `cube.rubik` 插件                |
| HTTP + WebSocket + NDJSON                                             | 控制与实时观测                          | 通信协议正文；消息头带 `protocol_version` |


---

## 5. 产品调研

先分类，再逐条：功能、特点、参数、价格、链接。价格为公开报道或商店标价，赛事租赁常需询价。

### 5.1 专业鹰眼 / 判罚

#### Sony Hawk-Eye（网球）

- 功能：多机追踪、电子司线、挑战回放、转播 CG。
- 特点：转播级；美网等已无司线；约 3 天装一场。
- 参数：约 10–12 路追踪 + 脚误相机；约 340fps；宣称误差约 2.6–3.6mm。美网量级：17 片场地约 204 路相机。
- 价格：单片场地设备公开报道约 10 万美元量级；需询价。
- 链接：[Hawk-Eye 简介（Sony）](https://www.sony.com/en/SonyInfo/technology/stories/entries/Hawk-Eye/) · [CNBC 美网](https://www.cnbc.com/2023/09/09/how-sonys-hawk-eye-works-at-the-us-open.html) · [Wikipedia](https://en.wikipedia.org/wiki/Hawk-Eye)

#### Sony Hawk-Eye（羽毛球）

- 功能：边线/底线挑战、球速等数据。
- 特点：球头轻、受风影响大，轨迹仿真与网球不同；2014 年 BWF 引入。
- 参数：常见 8 路高速机，报道约 660fps；延迟约 2s 量级供裁判看回放。
- 价格：只租不卖；报道有单场约 10 万美元以上、或约 5000–7000 美元/天/片；另有单片约 14 万美元租赁报道。需询价。
- 链接：[VICTOR 文](https://www.victorsport.com/blog/article/a-badminton-hawkeye-system-that-is-different-to-the-tennis-one) · [亚洲锦标赛费用报道](https://www.thesabamynews.com/news/the-hawk-eye-system-at-the-badminton-asia-championships-is-a-bit-expensive/)

#### Sony Hawk-Eye / FIFA GLT（足球门线）

- 功能：球是否整体过线，1 秒内通知裁判表。
- 特点：IFAB 认证；门线专用机，不兼半自动越位。
- 参数：世界杯报道 14 路高速机；判决时限 1s。
- 价格：需询价，仅顶级联赛负担得起。
- 链接：[FIFA GLT](https://inside.fifa.com/innovation/world-cup-2022/goal-line-technology)

#### Stupa IRS（羽毛球即时回看）

- 功能：BWF 认证线路挑战，约 12–22s 出结果。
- 特点：比 Hawk-Eye 轻、便宜；无线部署。
- 参数：报道准确率满足 BWF「约 99%」认证门槛。
- 价格：报道约 1000–1500 美元/天/片。
- 链接：[报道](https://www.onfieldnews.com/indias-stupa-takes-on-hawk-eye-bringing-affordable-precision-to-badmintons-line-calls/)

#### Reveal Lens（羽毛球 IRS）

- 功能：BWF 批准的即时回看之一；可配套记分与转播字幕。
- 特点：无线、约 1 小时内部署。
- 参数：未公开相机路数与 fps。
- 价格：未公开；宣传对标「传统 IRS 单场超 10 万美元」。
- 链接：[报道](https://weirdkaya.com/malaysia-unveils-worlds-first-ai-powered-review-system-for-badminton/)

### 5.2 开源框架 / 论文实现

#### TrackNetV3（羽毛球 2D）

- 功能：广播画面羽毛球定位、轨迹修补。
- 特点：热力图，不适合用 IoU 评小目标；测试集约 97.51% Accuracy、F1 约 98.56%、约 25fps（论文环境）。
- 参数：输入为视频帧序列；判对标准为 4 像素内。
- 价格：开源，无授权费。
- 链接：[GitHub](https://github.com/qaz812345/TrackNetV3) · [论文](https://people.cs.nycu.edu.tw/~yushuen/data/TrackNetV3.pdf)

#### YOLO + 多视三角（排球等）

- 功能：每路检测 → 极线匹配 → RANSAC 三角 → 3D 轨迹。
- 特点：研究原型，需同步与标定文件。
- 参数：常见 2～10 路；精度随标定与同步变。
- 价格：开源。
- 链接：[labvisio 多目标三角](https://github.com/labvisio/Multi-Object-Triangulation-and-3D-Footprint-Tracking) · [10 路排球标定追踪](https://github.com/pietrolechthaler/MultiViewCalibration-BallTracking)

#### TTNet（乒乓球）

- 功能：球检测、事件（击球/出界等）、台面分割，多任务。
- 特点：CVPR 2020；偏广播 2D，不是多机 3D。
- 参数：实时级（论文/实现依赖 GPU）。
- 价格：开源实现。
- 链接：[PyTorch 实现](https://github.com/maudzung/TTNet-Real-time-Analysis-System-for-Table-Tennis-Pytorch) · [论文](https://arxiv.org/pdf/2004.09927.pdf)

#### Uplifting Table Tennis

- 功能：球台 13 点、相机标定、3D 轨迹与旋转估计。
- 特点：用台面关键点当标定物，贴近「无棋盘、用模型校正」。
- 参数：关键点 `(N,13,3)`；3D 误差需看论文场景。
- 价格：开源/权重见仓库。
- 链接：[HF 台面关键点](https://huggingface.co/KieDani/upliftingtabletennis_tablekeypoints)

#### 双目乒乓球业余方案

- 功能：两路手机 720p30，角点手点 + 三角。
- 特点：验证「2 机业余 3D」可行，未产品化。
- 参数：测试配置 720p / 30fps。
- 价格：开源。
- 链接：[tt_tracker](https://github.com/ckjellson/tt_tracker)

#### SoccerNet / TVCalib

- 功能：足球场线段检测、相机参数、场地配准。
- 特点：无关键点对应也能优化位姿；SoccerNet 挑战常用。
- 参数：论文 Acc@5/10/20 等，见项目页。
- 价格：开源（TVCalib MIT）。
- 链接：[TVCalib](https://mm4spa.github.io/tvcalib/) · [GitHub](https://github.com/mm4spa/tvcalib) · [SoccerNet Calibration](https://github.com/SoccerNet/sn-calibration)

### 5.3 消费级 App / 网站

#### SwingVision（网球/匹克球）

- 功能：手机支架拍摄，击球、线路、速度等。
- 特点：单机消费级「鹰眼感」；不是多机判罚。
- 参数：依赖手机 60/120fps 与固定机位。
- 价格：商店订阅，报道 Pro 约数十至约 180 美元/年量级，以商店为准。
- 链接：[swingvision.com](https://swingvision.com/)

#### HomeCourt（篮球）

- 功能：投篮识别、命中、训练统计。
- 特点：单机、室内筐为已知模型。
- 参数：未公开算法精度。
- 价格：订阅，以商店为准；未在此固化数字。
- 链接：App Store 搜 HomeCourt

#### Chessvision.ai / 各类棋盘识别

- 功能：摄像头或截图识别棋盘，导出 FEN。
- 特点：与第一版棋类最接近；平面、格级。
- 参数：一般 1 机、静态或慢速。
- 价格：有免费与订阅；未统一公开企业价。
- 链接：[chessvision.ai](https://chessvision.ai/)

#### 魔方识别还原 App（Cube Solver 等）

- 功能：扫六面颜色，给出还原步骤。
- 特点：物体小、颜色块、近距；与第一版魔方接近。
- 参数：通常一次一面或展开图，不是连续 3D 追踪。
- 价格：免费 + 去广告内购，未公开。
- 链接：各应用商店「Cube Solver」

消费级羽毛球记分/训练 App 多为记分和慢动作，少有公开的多机 3D 落点，价格未公开，不单列以免编造。

---

## 6. 场景与物体


| 场景   | 识别对象        | 要否稀疏 3D     | 安排                  |
| ---- | ----------- | ----------- | ------------------- |
| 围棋   | 盘、星位、黑白子    | 否，平面即可      | 第一版（Web 插件，独立于引擎项目） |
| 象棋   | 盘、楚河汉界、红黑子  | 否           | 第一版（Web 插件，独立于引擎项目） |
| 魔方   | 六面色块、姿态     | 弱 3D（立方体姿态） | 第一版（Web 插件，独立于引擎项目） |
| 数独   | 格、印刷/手写数字   | 否           | 待规划                 |
| 羽毛球  | 场地线、球、人（可选） | 高度要多机       | 待规划                 |
| 乒乓球  | 台、网、球       | 建议双目        | 待规划                 |
| 台球   | 桌、袋、球号      | 平面为主        | 待规划                 |
| 足球   | 场地线、球、门     | 专业要多机       | 待规划                 |
| 网球   | 场地、球        | 同羽毛球        | 待规划                 |
| 排球   | 场地、球        | 多机          | 待规划                 |
| 篮球投篮 | 筐、球、人       | 单机可估        | 待规划                 |
| 飞盘   | 场地、盘        | 可单机平面       | 待规划                 |
| 冰壶   | 壶、营垒        | 平面          | 待规划                 |
| 飞镖   | 盘、镖         | 近距平面        | 待规划                 |
| 棒球   | 本垒、球        | 高速多机        | 待规划                 |
| 体操落地 | 垫、人关键点      | 姿态          | 待规划                 |
| 乐高   | 积木色与孔       | 近距          | 待规划                 |
| 书法临摹 | 纸、字轨迹       | 平面扭曲        | 待规划（可复用 warp）       |
| 桌面卡牌 | 牌面、区域       | 平面          | 待规划                 |


场景与插件放在独立项目/目录，不放进引擎仓库。引擎只提供加载接口。

---

## 7. 现有仓库（一句）

<!-- 当前 play-site 是 Nuxt 4 的 `app/` 加 `support/` 旁路服务，适合围棋评分。鹰眼宜：站点与协议在本仓，重引擎在 support（不含场景/插件），场景插件另仓，Android/PC UI 另仓；不要把 CV 引擎塞进 Nuxt server。结构只在设计文档规划，本次不改仓库。 -->