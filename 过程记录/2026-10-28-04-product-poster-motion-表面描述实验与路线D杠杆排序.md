# 过程记录 2026-10-28-04 · product-poster-motion 表面描述实验（负结果）+ 路线 D 杠杆排序 + DNA 画种/含文字 VLM 打标

> **Z-08「只增不改」纪要** · 编号 2026-10-28-04 · 主题：接 10-28-03（路线 A 文本路线封顶），把「DNA 能不能直接驱动表面」做成实验——把 DNA 的笔触维度翻成**一句集中的表面描述**当路线 D 的条件词 → **不比一个画种词好** → 顺带复现了「主体更密」指令在扩散模型上的副作用（subject_contrast 崩到 0.11）；同时用 VLM 给 51 张 DNA 补齐 **画种**与**含文字**两个字段。

---

## 0. 一句话

路线 D 的「DNA → 表面描述 → 生成」闭环试过了：**一句集中的表面描述（把笔触维度翻成
"countless small dabs of pigment, each dab is a small rounded blob of thick paint…"）
并不比一个画种词有效**（均值 0.696 vs 0.694），内部变异也不动；而且表面描述里
"主体更密"那句会**把整幅画都画密**，实测主体/背景对比崩到 0.11（第三次复现）。
至此路线 D 的杠杆排序也清楚了：**画种词（+0.09）≫ 簇 LoRA 本身（+0.02）＞ 其余全是 ±0**。

---

## 1. 表面描述实验

新增 `dna_txt2img.surface_phrase(dna)`：把 anatomy（块数/宽度/密度/勾线/叠色）+ motif
（母题形状/密度/主体对比/笔速）+ metrics（干笔）翻成**一句**表面描述。
动机：路线 A 是把维度**拆成很多条指令**写进 prompt（实测压不过模型先验），
一句集中的描述更像"给模型看一幅画"而不是"下十道工序"——正是本项目的命题。

条件词（簇首成员 3b68cb2b 的 DNA 推导，画种由 VLM 打标得到）：
`an oil painting with countless small dabs of pigment, each dab is a small rounded blob of
thick paint, the surface is broken into a multitude of small thick strokes, each shape is
ringed by a clear dark contour line, the colors are wet-blended into one another,
the subject carries much denser brushwork than its surroundings`

对照矩阵（2 内容 × 3 参考 DNA，簇均分，同 seed 123）：

| LoRA | 条件 | 内容001 | 内容000 | 均值 | inner_variance | subject_contrast |
|---|---|---|---|---|---|---|
| 内容only 训练 | 无词 | 0.691 | 0.485 | 0.588 | 0.524 | 0.07 |
| 内容only 训练 | 画种词 | 0.734 | 0.654 | **0.694** | 0.449 | **1.31** |
| 内容only 训练 | 表面词 | 0.688 | 0.705 | **0.696** | 0.477 | **0.15** |
| +oil painting 训练 | 画种词 | 0.746 | 0.633 | 0.689 | 0.451 | 1.05 |
| +oil painting 训练 | 表面词 | 0.693 | 0.698 | **0.696** | 0.470 | **0.11** |

**结论**：
1. **一句集中的表面描述 ≈ 一个画种词**（0.696 vs 0.694），没有额外增益；
2. **内部变异仍不动**（0.45→0.47，参考 0.201）——把"碎点/厚涂"写进 prompt 也换不来真的碎点；
3. **「主体更密」有反直觉副作用**（第 3 次复现）：说了它，模型把**整幅**都画密，
   实测 subject_contrast 崩到 0.11（参考 1.57），综合分跟着掉。
   → **别用"主体/背景密度差"这类指令去调扩散模型。**

**方法论警告（这条 A/B 的先天局限）**：表面描述是从**用于打分的同一个 DNA** 推出来的，
所以"分数变高"里有闭环成分——这个实验真正测的是**模型听不听表面描述**，
不是"描述对不对"。即便如此它也没测出增益，结论是稳的。

## 2. 路线 D 的杠杆排序（截至 2026-10-28）

| 手段 | 效果 | 判读 |
|---|---|---|
| **画种词**（`an oil painting of`） | **+0.09~0.10** | 最大杠杆。提示词稀薄 → 照片先验压倒性 |
| 簇 LoRA 本身 | +0.02~0.03 | 真实但不大：迁来构图语言与配色 |
| 训练时也带画种词 | ±0 | 推理时带就够，不必重训 |
| 表面描述（更细） | ±0 | 不比画种词好 |
| DNA 量化约束搬给路线 D | ±0 | 且有副作用 |

**仍未解决**：厚涂的**笔触表面**。四轮尝试（提分辨率/保构图、画种词、表面描述、量化约束）
只解决了"风格搬过来"，没解决"表面像过来"。

## 3. DNA 补上两个缺字段（VLM 打标，`scripts/label_refs_vlm.py`）

- `medium`（画种）：**程序化测不出来**（透明晕染 vs 不透明厚涂边界本身模糊），
  而 VLM 判媒材是强项。**实测模板写死的 "watercolor" 几乎全错**——
  全库分布以 digital illustration / ink wash painting / digital painting 为主，
  oil painting 只有少数。**这意味着全库 route A 的 prompt 一直在说错画种**。
- `has_text`（含文字/签名/logo）：**本库有相当比例的参考图带文字**（含中文海报 070316bf、
  手写贺卡 667a9ad6 等）。含文字的图**不能进 LoRA 训练**（模型会学会写字），
  且其 DNA 的笔触指标被外来笔画污染。

**接线**：
- 路线 A：`--medium auto` 用 DNA 的画种标签（不再写死 watercolor）。
- 路线 D：画种前缀**由簇内 DNA 标签多数票自动取**（实测簇内画种不一致——
  3b68cb2b=oil painting 而 5fcf60dc=digital illustration，"同风格簇"横跨了不同媒材）；
  `flux_lora_gen` 从 `meta.json` 自动带上，用户仍然只填内容。
- 训练器：**含文字且无清洗版的成员自动跳过**（`--allow-text` 可强训）。

## 4. 顺带修的一个基础设施坑

`encode_texts` 批量编码在**内存紧张**时静默失败（子进程无输出、退出码非 0），
本轮撞上一次：VLM 打标占着 ~16GB 时并行加载 T5 bf16（9.8GB）→ 子进程部分初始化失败。
已把错误信息补全（带退出码 + 内存提示）。这是记忆里"反复加载大模型在内存紧张时会部分初始化失败"
那条的又一次复现，**教训：T5 编码与大模型加载不要同机并发**。

## 5. 本轮改动清单

- 新增：`dna_txt2img.surface_phrase()`（DNA → 表面描述）、`scripts/label_refs_vlm.py`（VLM 打标）
- 改：`dna_txt2img.py`（`--medium auto`、`medium_phrase()`）、`flux_lora_train.py`
  （画种前缀多数票自动取、含文字成员跳过）、`flux_lora_common.py`（编码失败信息补全）
- 数据：DNA 新增 `medium` / `has_text` / `_medium_note`

## 6. 遗留

1. **厚涂笔触表面**（路线 D 最硬的骨头）：本轮四条路都试过，只解决"风格搬过来"。
   剩余候选：提 LoRA 容量（rank/步数/更多同风格图）、换底模（FLUX.1-dev 的先验更强）、
   多图簇扩容（当前最大簇只有 3 张）。
2. **测量层分歧**（`density_contrast` vs `subject_contrast`）需要定口径。
3. `subject_contrast` 指标稳健性待查（"主体更密"指令下普遍崩到 0.11）。
4. 含文字的参考图需要批量清洗（`mask_reference.py` 目前要手工给矩形）。
5. 治理仓四篇纪要（10-28-01/02/03/04）**尚未 push**（另有 10-02 的 flova 纪要仍未跟踪）。

相关：[[product-poster-motion-scope]]、[[style-cluster-lora]]、[[prompt-constraint-capacity-ceiling]]、[[dna-library-data-quality]]
