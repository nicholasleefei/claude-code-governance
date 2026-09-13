# 2026-09-13-02 · product-shots 安装与本地出图接线（交接纪要）

> 类型：跨项目安装/接线工作交接纪要（Z-08）。只增不改。
> 项目仓：`E:\my projects\product-shots`（git clone 自 github.com/motiful/product-shots，MIT，alpha，提交 063c508）。

## 一、任务

飞总要求调研并安装 motiful/product-shots（单张白底商品图 → 亚马逊/TikTok 等平台带货图的 7 个 Claude Code Agent Skills），装到当时的空文件夹 `E:\my projects\新建文件夹` 并按项目改名；出图后端**不用官方云网关，尝试接入本机生图模型**。

## 二、成果（均已验证）

1. **仓库就位**：clone 到 `E:\my projects\product-shots`（52MB，7 个 skill 齐全）。
   - ⚠ 文件夹改名遗留：会话进程占用 `新建文件夹`，Windows 不允许 rename；全部内容已移入新夹，**空旧夹 `E:\my projects\新建文件夹` 待关闭 Claude Code 后删除**（每次试删均报 in use）。
2. **技能激活**：7 个 skill 以**目录联接（junction）**在 `C:\Users\pc\.claude\skills\product-shots*` 激活，指向仓库 `skills\*`；不复制文件，`git pull` 即更新（Z-03 不搬文件）。
3. **本地出图链路打通并冒烟通过**（2026-09-13）：
   - skill generate.py（只会 OpenAI/Gemini 云 API 方言）→ **自写兼容网关** `E:\my projects\product-shots\local-gateway\server.py`（纯标准库，:8190，仅监听 127.0.0.1）→ ComfyUI 0.34（:8188，`E:\ComfyUI\.venv`）→ **M-002 Qwen-Image-2512 FP8**（20 步/cfg4/euler-simple/shift3.1，对齐官方 2512 蓝图核心节点）。
   - 冒烟命令经 skill 脚本端到端出图（黑保温杯白底 1024²，330KB，首跑 235s 含权重加载），成品 `local-gateway\smoke-test.png`。
4. **环境变量**（飞总批准，用户级 setx，重开终端后生效）：
   - `PRODUCT_SHOTS_IMAGEGEN_BASE_URL=http://127.0.0.1:8190/v1`
   - `PRODUCT_SHOTS_IMAGEGEN_API_KEY=local-no-auth`（占位假值，网关不校验、不出本机，非凭据，不触 Z-09）。
5. **启动器**：双击 `local-gateway\start-local-stack.cmd` 自动拉起 ComfyUI+网关（已在跑则跳过）。
6. **台账已登记**：E:\02-skill 新增 **S-028～S-034**（7 条，外部下载/junction 激活），计数 27→34，变更记录已追加，S-029 与模型台账 M-002 互链。

## 三、已知边界 / 待办（飞总已拍板：编辑模型先不下）

- **图生图未开通**：M-002 只有 2512 文生图权重；`/v1/images/edits` 现返回明确 501。
  待办：下载 `qwen_image_edit_2509_fp8_e4m3fn.safetensors`（约 20GB，Comfy-Org/Qwen-Image_ComfyUI）到 `E:\01-model\M-002-Qwen-Image-2512\diffusion_models\` → 登记台账 → 在网关补 edit 工作流（参考 ComfyUI 蓝图「Image Edit (Qwen 2509)」：UNETLoader+TextEncodeQwenImageEditPlus+FluxKontextImageScale+VAEEncode）→ 真实白底图冒烟。
  **影响**：产品核心卖点「保留真实商品换背景/换角度」与 S-032 多角度九连拍，在补齐前本地跑不了；文生图类（凭空生成场景图）可用。云端备选：OmniMaaS（gemini-3-pro-image-preview ≈¥1/张）。
- **模型名是伪装**：skill 只允许 gpt-image-2/gemini-* 等名字，统一用 `--model gpt-image-2`，网关忽略模型名一律路由 Qwen-2512；产品图技能默认调 gemini 名时会被 skill 拒识——各业务 skill 内默认模型名日后若需本地全兼容，可能要在调用层统一改传 gpt-image-2（待实际用业务 skill 时验证一轮）。
- 服务常驻占用：ComfyUI 空闲约 1–2GB 显存；不用时关两个最小化窗口即可。
- 仓库自带更新直接 `git pull`；**自写的 local-gateway 在仓库内非上游目录，pull 不冲突，但属未跟踪文件，日后可考虑独立备份**。

## 四、关键路径速查

| 项 | 路径/值 |
|---|---|
| 项目仓 | `E:\my projects\product-shots` |
| 网关代码+说明 | `…\product-shots\local-gateway\server.py` / `README.md` / `start-local-stack.cmd` |
| ComfyUI | `E:\ComfyUI`（.venv Python 3.13，端口 8188） |
| 模型 | M-002 `E:\01-model\M-002-Qwen-Image-2512`（经 E:\ComfyUI\extra_model_paths.yaml 接线） |
| 技能台账 | S-028～S-034 @ E:\02-skill\index.html |
| 环境变量 | PRODUCT_SHOTS_IMAGEGEN_BASE_URL / _API_KEY（用户级） |

## 五、合规自查

无凭据落盘（仅占位符与环境变量名）；未改上游 skill 源文件；模型权重未落入项目目录（Z-02）；junction 而非复制（Z-03）；系统级变更仅用户级 env，已经飞总当轮批准（Z-02 注）。
