# 过程记录 2026-10-27-01 · product-poster-motion 生成端"纯噪点出图"事故定位与修复

> **Z-08「只增不改」纪要** · 编号 2026-10-27-01 · 主题：风格 LoRA（路线 D）生成端出图是纯噪点 → 定位为编码环境错（transformers 5.x 静默加载随机 CLIP）→ 修编码链路 + 加四道护栏 → 重跑生成并做 LoRA/基模对照。

---

## 0. 一句话

上一会话留下的最后一张簇 LoRA 产物是蓝绿噪点；根因不在模型、不在 LoRA，而在**生成端用错解释器编码**——transformers 5.x 下 `clip_l.safetensors` 的 196 个权重静默全部 missing，CLIP 随机初始化 → `pooled` 是随机向量 → Flux 出图纯噪声。已定位并实测验证，修复后出图正常。

---

## 1. 事故现象

- `renders/LORA_cluster_3b68cb2b_5fcf60dc_d43afa51_9515_seed123.png`（2026-10-26 12:41，上一会话收尾产物）= 蓝绿噪点，无任何画面结构。
- 全程**无任何报错**——子进程退出码 0、文件正常落盘。这是它危险的地方。

## 2. 根因定位（实测，非推断）

| 环节 | 事实 | 证据 |
|---|---|---|
| `flux_lora_gen.py` 生成新内容嵌入 | 用 `sys.executable`（= py-lora venv，**transformers 5.20.2**）调 `encode_caption.py` | 代码 + 版本实测 |
| transformers 5.20.2 加载 `clip_l.safetensors` | **missing 196 项 / unexpected 196 项** → CLIP 完全随机初始化 | `load_state_dict` 实测 |
| 存下的 `_gen_tmp.pt` 的 pooled | 与"正确加载的 CLIP"余弦 **−0.040**；与随机实例余弦 0.025 → 确系随机向量 | 与 py-storyline(4.57) 正确 CLIP 对比 |
| 训练侧的簇嵌入 | 与正确 CLIP 余弦 **0.9999** → 训练数据**未被污染**，LoRA 权重本身没问题 | 三张成员逐一验证 |
| T5 | 5.x 下键名未变 → 正常加载 → 只有 CLIP 坏 | 5.x 子进程能跑完并落盘 |

**因果链**：transformers 5.x 重构 `CLIPTextModel`（去掉 `text_model.` 前缀、删 `text_projection`）→ `strict=False` 吞掉 196 个 missing → CLIP 停在随机初始化 → Flux 的 `pooled_projections` 是垃圾 → 出图纯噪点。训练脚本因为硬编码用 py-storyline 编码而幸免——**坑是按"谁调用了文本编码器"逐个咬的，训练对不代表生成对**。

## 3. 修复（`E:\my projects\product-poster-motion\scripts\`）

1. `flux_lora_common.py`：新增 `ENCODER_PY` 常量 + `encode_text()` / `encode_texts()` 统一入口（训练与生成共用，杜绝两边各写一套）；CLIP 加载断言 `missing==0`，并**把 CLIP 提到 T5 之前装**（错环境秒级失败，而不是等 9.8GB T5 读盘 3 分钟）。
2. `encode_caption.py`：加 transformers 版本闸门（≥5 直接拒）+ **基准探针**（探针文本编码与 `_probe_golden.pt` 余弦 <0.99 即中止）；新增 `--texts/--out-dir` 批量模式。
3. `flux_lora_gen.py`：改走 `encode_texts`；`--scale` 之前是**静默空参数**（LoRA 恒按 1.0 合并），现已在 `flux_lora_common.load_lora()` 里真按比例缩放 `lora_B`；输出文件名改用稳定 slug（原来用内置 `hash()`，按进程随机化，且不同内容会撞名互相覆盖——本轮就踩到）。
4. `flux_denoise.py`：末端潜变量 std 护栏（正常 ≈1，条件坏掉会跑飞）。

**护栏设计的教训**：出图的图像统计量（块平均比、梯度比）**分不出**这类故障——噪点图反而更平滑。护栏必须打在**条件向量**这一步，不能指望出图侧兜底。

## 4. 修复后实测

出图正常（末端 latents std 0.68–1.11，健康）。同 seed（123）、同内容、同参数（28 步 / CFG 3.5 / 768×1344）对照，评分器为 10-25 改版后的 `dna_score.py`：

| 图 | 3b68cb2b | 5fcf60dc | d43afa51 | 簇均分 |
|---|---|---|---|---|
| 簇 LoRA · 密花地毯 | 0.531 | 0.515 | 0.553 | **0.533** |
| 基模（无 LoRA）· 密花地毯 | 0.479 | 0.460 | 0.515 | 0.485 |
| 簇 LoRA · 小径 | 0.705 | 0.594 | 0.708 | **0.669** |
| 基模（无 LoRA）· 小径 | 0.662 | 0.635 | 0.670 | 0.656 |
| 路线 C（USO 注入）参照 | 0.772 | 0.696 | 0.767 | 0.745 |

**读出来的两件事**：
- LoRA 相对基模确有增益（+0.048 / +0.013），且**把"密花框住小径"的构图语言与粉紫配色搬过来了**——簇训练方向有效。
- 但**厚涂的笔触表面没学到**（出图像照片不像油画），与"笔触是命门"的判断一致：300 步 @512、rank 16 学不动表面质感。这是下一步要攻的点。

## 5. 顺带记下的口径与遗留

- **评分器口径**：`dna_score.py` 10-25 改版后综合分含解剖+逻辑+母题三层 29 项 → 与 10-24 README 的 0.571 / 0.486 / 0.502（v4 评分器）**不是同一把尺**，跨版本绝对分不可比（此坑与"消融前先核对数据版本"同源）。
- **文档漂移未清**：README 与 `项目说明书\index.html` 停在 10-24，路线 D 与簇 LoRA 未入档；本纪要之外，文档补课是独立任务。
- **数据卫生**：`captions/` 与 `dna/` 的 id 集合错开 3 个 —— `0eca49fa`、`667a9ad6` 两张参考图无 caption（文件名非哈希规范），`81dM+Beq.txt` 是按文件名取的错 id → 这两张进不了 LoRA 训练。
- **纪要缺口**：10-24、10-26 两轮项目工作均未留纪要，本篇是本项目第一篇。

## 6. 本次改动清单（项目内文件）

`scripts/flux_lora_common.py`（ENCODER_PY / encode_text / encode_texts / load_lora / CLIP 断言 / 装载顺序）、`scripts/encode_caption.py`（版本闸门 / 基准探针 / 批量模式）、`scripts/flux_lora_gen.py`（批量编码 / 真 scale / 稳定文件名）、`scripts/flux_lora_train.py`（改用统一编码入口）、`scripts/flux_denoise.py`（latents 护栏 / scale 透传）、`lora_data/embeds/_probe_golden.pt`（基准探针，由 py-storyline 生成）。

相关：[[product-poster-motion-scope]]、[[flux-lora-training-pitfalls]]
