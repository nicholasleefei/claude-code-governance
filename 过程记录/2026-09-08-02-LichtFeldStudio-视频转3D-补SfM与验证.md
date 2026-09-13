# 2026-09-08-02 · 视频→3DGS 链路补齐(SfM/ffmpeg)与用户素材验证结论

## 背景与结论一句话
用户目标 =「上传一段视频 → 用 3D 高斯泼溅做 3D 模型」。已补齐 SfM 环(COLMAP+ffmpeg)并与已编译的 LichtFeld Studio(训练端)接通。
用户素材(E:\3d资产\测试\IMG_0091.mov / IMG_0092.mov + 顶视 IMG_0093.jpeg)经多轮验证**无法重建**——因拍摄为"物体转、相机基本不动"的近正面自转,画面缺乏刚性 3D 视差。已请用户**补拍**(人绕静止物体走两圈)。

## 新部署的工具(全免费、本地)
- **COLMAP 4.2.0 Windows CUDA 版**:`E:\tools\colmap\bin\colmap.exe`(从官方 4.2.0 release 下载,备份包 `E:\tools\_downloads\colmap-x64-windows-cuda.zip`,363MB)
  - GPU 实测通过(SIFT GPU 提取 / matcher 绑 device 0)。选项前缀已变:**`--FeatureExtraction.use_gpu`、`--FeatureMatching.use_gpu`**;`SiftExtraction` 旧名已废。
- **ffmpeg 9.0.1 essentials**:`E:\tools\ffmpeg\ffmpeg-9.0.1-essentials_build\bin\ffmpeg.exe`(备份 `E:\tools\_downloads\ffmpeg-release-essentials.zip`)。
  - ffmpeg 9 注意:`-vsync` 已废,用 `-fps_mode vfr`。
- **系统 Python 3.13.2**(C:\Users\pc\AppData\Local\Programs\Python\Python313)带 numpy2.2/cv2 4.13/PIL——本机已有,可直接用。
- 两工具都不改系统 PATH;脚本内全路径/env 覆盖。

## 关键文件
- `E:\3DGaussianSplatting\video2dataset.cmd` — 一键 视频→COLMAP 数据集(抽帧 fps 默认2 → extractor(single_camera,OPENCV,GPU) → sequential_matcher → mapper → `数据集/sparse/0`)。拖视频上去即可。
- `E:\3DGaussianSplatting\salvage_mask.py` — 转台式素材抢救:时间中值抠背景 → 逐帧前景 mask → COLMAP mask_path。**实测对用户素材无效**(见下)。
- `E:\3DGaussianSplatting\视频转3D流水线使用说明.md` — 中文全流程 + 拍摄要点 + FAQ。

## 验证过的坑与事实(防再踩)
1. COLMAP 4.2.0 mapper 对"相机固定、物体转"的素材判死:所有两视图几何校验不通过。抠背景+提特征(560/帧)+放宽 init(inliers100→30, tri_angle16°→4°)后仍 0 注册、273 次丢弃。→ **该素材物理上无可用视差,软件无解**。
2. mapper 初始对门槛默认较高:init_min_num_inliers=100、init_min_tri_angle=16°;转台/慢速素材可先放宽。
3. 相位相关(phaseCorrelate)在这类含大旋转物体的画面上会出 ±200px 假位移(NCC 弱);判真平移要用**背景角区模板匹配(NCC≈1.000 才是对的)**。
4. 0092 素材背景仅 33% 像素稳定(相机在动)→ 中值背景法不适用;0091 66% 稳定(可用)。
5. 顶视 IMG_0093(4032×3024)特征正常(2382) → 物体不缺纹理;缺的是视差。

## 下一步(等用户)
1. 用户补拍「人绕静止物体走 2 圈(上下各一)」20–40s 的视频。
2. 拿到后跑 `video2dataset.cmd` 或让 Claude 代跑 → 出 COLMAP 数据集。
3. 在 LichtFeld Studio 加载该数据集目录 → 训练(mcmc)→ 导出 .ply/.licht/HTML。
4. 旧的 `E:\my projects\3D Gaussian Splatting`(带空格)工作区仍待本会话结束后手动删除;新工作区在无空格 `E:\3DGaussianSplatting`。
