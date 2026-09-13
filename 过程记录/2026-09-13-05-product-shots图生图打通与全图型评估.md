# 2026-09-13-05 · product-shots 图生图打通（M-010）与全图型样品评估（交接纪要）

> 类型：跨会话工作交接纪要（Z-08）。只增不改。接续 2026-09-13-02（安装与本地文生图接线）。
> 项目仓：`E:\my projects\product-shots`。

## 一、任务

飞总用手机实拍万圣节钻石画（黑猫巫师帽提灯+幽灵+南瓜，`测试\IMG_0122.jpeg`）实测整个 product-shots 技能组，并明确：**主战场是 TikTok Shop，亚马逊先不管；先把 Skill 支持的图全出一遍样图、充分了解后再改造**（改造未授权，本纪要只评估不动 skill）。

## 二、新成果（均已验证）

1. **图生图打通**：新增模型 **M-010 Qwen-Image-Edit-2509 FP8**（独立一夹 `E:\01-model\M-010-Qwen-Image-Edit-2509`，权重 20,430,698,424 字节；文本编码器/VAE 复用 M-002）。
   - HuggingFace 源当日不可用（直连超时、xray:10808 仅 91KB/s、hf-mirror 19KB/s），改用 **ModelScope 官方镜像 8 连接 Range 续传**（脚本 `M-010…\parallel-fetch-ms.sh`，聚合 ~12MB/s，逐段校验后 cat 拼接）。
   - ComfyUI `extra_model_paths.yaml` 新增 `qwen-image-edit-2509` 段（只挂 diffusion_models）；网关启动自动探测 UNETLoader 清单，`/health` 的 `edit_enabled:true`。
2. **网关 server.py 已扩展**（上轮会话完成）：`POST /v1/images/edits` multipart（image[] 最多 3 张）→ 官方 Image Edit (Qwen 2509) 蓝图（CFGNorm strength1 → FluxKontextImageScale → TextEncodeQwenImageEditPlus → VAEEncode → KSampler 20 步/cfg2.5/shift3.0）。负向词固定追加。
3. **全图型样品批量**：`测试\run-all-samples.py`（ComfyUI venv python 跑，PIL 预制各比例画布贴商品+羽化 → 逐张调 edits，断点续跑、日志 `run-all-samples.log`），输出 `测试\all-samples\`（含 `_refs\` 预制画布）。
   - 覆盖：主图套 8 张（m01–m08，1:1）、A+ 8 模块（a01–a08，21:9 用 1536×656、3:2 用 1024×688）、广告 3 代表（ad-tiktok 9:16 / ad-google 1.91:1 / ad-feed 4:5）、社媒 3 张（1:1/4:5/Reel 封面；9:16 Story 复用更早的 `out-01-tiktok-9x16.png`）。
   - **multi-angle 技能跳过**：它是服装模特人像九连拍（锁人脸/发型），对非服饰商品不适用。
   - 热身后单张约 65–165s。

## 三、评估初步发现（样图说话，改造时的输入）

- **白底主图/场景图可用**：商品保真约九成，模型会自加细黑框、轻微改写图案（提灯内猫脸等），需人工挑片。
- **图内文字半可用**：≤3 个短词的大号无衬线字成功率高（"EASY FOR BEGINNERS""FULL-DRILL CANVAS"完美）；长标题、小号字必乱码，DRILLS→DRALLS 类拼写错误常见。**信息图/A+ 文案模块的正确做法：出无字底图 → 后期排版加字**。
- **平面商品"多角度"是重灾区**：模型把三视图理解成台历（螺旋装订+厚书脊），并把紫夜配色漂成米黄。"细节特写"会把钻面猫美化成真毛猫（好看但失真）。
- 分辨率：网关钳 512–1536、16 取整；A+ Hero 正式 2388×1024 出不了，正式投放要放开钳制或换放大流程。
- Skill 本质（已向飞总通俗解释）：7 份写给执行者的 Markdown SOP，商品理解靠 Claude 看图，"视觉 DNA"是预写英文提示词桶；**它只有亚马逊上架图知识，对 TikTok Shop 上架图（9 张 1:1、首图规则等）零知识**，会拿亚马逊规矩套 TK。权威 TK 规则待查 S-008 卖家大学索引。

## 四、台账/文档同步（本纪要前已完成）

- M-010 建说明书 `index.html` 并登记 `E:\01-model\index.html`（M-010 行）。
- `E:\02-skill\index.html` S-028/S-029/S-032 修订：图生图 501/待权重等过时说法改为"已打通（M-010）"，S-032 标注仅适用服饰。
- `local-gateway\README.md` 已实现清单更新（edits 开通 + 三条已知局限）。

## 五、待办

1. 全量 22 张样图跑完后向飞总逐类点评，收敛 TK Shop 改造方案（未授权前不改 skill 文件；junction 随 `git pull` 更新，改造方式另议：本地补丁手册 / fork / 项目级覆盖 skill）。
2. 查 S-008 tts-seller-university 技能核实 TK Shop 上架图现行官方规则。
3. 正式尺寸出图方案（放开 1536 钳制 or 高清放大）。
4. 删除空旧夹 `E:\my projects\新建文件夹`（须关闭 Claude Code 后手动删）。
5. `local-gateway/`、`测试/` 在项目仓内未跟踪（git status 可见），暂不提交。
