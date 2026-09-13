# 2026-09-08-01 · LichtFeld Studio（3D Gaussian Splatting）源码编译完成

## 结论
LichtFeld Studio 已在 **E:\3DGaussianSplatting** 从源码编译成功，生成 **LichtFeld-Studio.exe（79MB）**。

## 关键路径（本机）
- 成品：`E:\3DGaussianSplatting\LichtFeld-Studio\build\LichtFeld-Studio.exe`（同目录 80 个 DLL）
- 源码 + 全部依赖工具：`E:\3DGaussianSplatting\`（vcpkg、下载缓存、日志、_seed_retry.sh 等都在此）
- **工作区从原 `E:\my projects\3D Gaussian Splatting` 整体复制到 `E:\3DGaussianSplatting`（无空格）**。原目录仍在（被进程占用无法改名），本会话结束后可手动删除。
- 旧目录里 `vcpkg\buildtrees`、`packages`、`LichtFeld-Studio\build` 为可重建物已删除；`_setup` 内 CUDA 安装包 3.3GB、VS 日志可留可删。

## 环境
- VS2022 Build Tools：`E:\BuildTools`（MSVC 14.44、clang-cl、ninja）；CUDA 12.8.61：C 盘默认路径
- CMake 3.30.9：`E:\tools\cmake-3.30.9-windows-x86_64`
- vcpkg：`E:\3DGaussianSplatting\vcpkg`（full clone，HEAD 04a9d8e5，含 builtin-baseline 所需全历史）
- 显卡 RTX 4090 D 24GB，CUDA arch 锁定 sm_89，驱动 591.86

## 遇到的坑与解法（重要，防再踩）
1. **路径含空格 → 编译失败**：x264/ffmpeg 等 autotools 端口把 `-libpath:` 里的空格截断。解决：整盘搬到无空格路径。
2. **无法移动被占用目录**：运行中进程的 cwd 在目录内，Windows 禁止改名/移动。解决：用 robocopy 复制（`/E`，PowerShell 里跑，git bash 会把 `/E` 转成路径导致 exit 16）。
3. **vcpkg shallow clone → builtin-baseline read-tree 失败**：须 full history（`git fetch --unshallow`）。
4. **GitHub 直连不稳**：github.com 走本地代理 127.0.0.1:10808（vcpkg 用它），大源包必要时 ghfast.top 镜像；fatal SSL/GOAWAY 多为间歇性，重试或续传即可（CMake FetchContent 大文件用 `curl -C -` 断点续传）。
5. **CMake 内部 file(DOWNLOAD) 只认小写 `http_proxy/https_proxy`**：只设大写代理时 CMake 会直连被重置。须大小写都导出。
6. **CUDA "toolkit dir '' does not exist"**：本会话启动早于 CUDA 安装，继承环境缺 `CUDA_PATH(_V12_8)`（注册表有）。configure 时显式 export。
7. **VS 生成器干净目录下 C/C++ 编译器探测失败**（CUDA 正常）：改用 **Ninja 生成器 + vcvars64** 解决——写 `_ninja_cfg.cmd`：`call vcvars64.bat` 后 `cmake -G Ninja -DCMAKE_BUILD_TYPE=Release -DBUILD_TESTS=OFF -DCMAKE_CUDA_COMPILER=...nvcc`。
8. **FetchContent 下载目标**：nvjpeg2k_headers 的 zip 放 `build\_deps\nvjpeg2k_headers-subbuild\nvjpeg2k_headers-populate-prefix\src\` 且哈希匹配即跳过下载。

## 构建/运行方法
- 重配置：`cmd /c E:\3DGaussianSplatting\_ninja_cfg.cmd`
- 重新编译：`cmd /c E:\3DGaussianSplatting\_ninja_build.cmd`（= cmake --build build）
- 运行：直接双击 `LichtFeld-Studio.exe`（Vulkan GUI 程序）
