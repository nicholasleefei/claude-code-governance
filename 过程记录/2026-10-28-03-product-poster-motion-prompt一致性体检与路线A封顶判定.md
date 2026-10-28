# 过程记录 2026-10-28-03 · product-poster-motion prompt 跨层一致性体检 + 路线 A 文本路线封顶判定

> **Z-08「只增不改」纪要** · 编号 2026-10-28-03 · 主题：接 10-28-02，量化并修掉 prompt 的跨层自相矛盾 → **39/51 张参考图的 prompt 自相矛盾** → 但修掉并不涨分 → 再测"细节预算"肯定式改写 → **目标维度也不动** → 判定**路线 A 的文本路线已到天花板** → 同时用 VLM 打标补上 DNA 的**画种**与**含文字**两个字段，并把画种接进路线 D 的 caption 前缀。

---

## 0. 一句话

v4/v5/v6 三层是逐层叠上去的，**从没做过跨层一致性检查**——体检发现 **39/51（76%）张参考图的 prompt 自相矛盾**；
但按层优先级修掉冲突子句后 A/B **−0.009**，把治"太碎"的细节预算改成肯定式后**目标维度也不动**
（inner_variance 0.441→0.437）→ 加上前两轮（画种词 −0.002、留白叙述 +0.007），
**路线 A 的四次 prompt 层面改动全部落在 n=5 噪声带（±0.015）内或维度不动 → 文本路线封顶**，
画面的"表面"只能由像素（路线 C）或权重（路线 D）承载。

---

## 1. prompt 跨层一致性体检（新增工具）

- `scripts/prompt_consistency.py`：断言表（claim）+ 矛盾对 + 层优先级（解 > 构 > 题）。
  三层子句的词汇是**封闭集合**（每条都是代码里写死的字符串），所以能做逐条断言→两两比对。
- `scripts/check_prompt_consistency.py`：体检工具（`--dna` 看单张、`--repair` 看修复后）。

**结果：51 张里 39 张自相矛盾，只有 12 张干净。**

| 矛盾类型 | 张数 | 出处 |
|---|---|---|
| 勾线 vs 无轮廓线 | 6 | 解剖 `line_presence` vs 逻辑 `broken_edge_ratio` |
| 密度均匀 vs 主体更密 | 5 | 逻辑 `density_contrast` vs 母题 `subject_contrast` |
| 禁止碎点 vs 满布小点 | 3 | 解剖 `speckle_ratio` vs 母题 `motif_count` |
| 密笔满布 / 主体更密 vs 一层主导色块 | 2 | 解剖/母题 vs 逻辑 `layer_count` |

**注意后两类是测量层的分歧，不只是叙述问题**：`logic.density_contrast` 与 `motif.subject_contrast`
对**同一张图**给出相反结论（前者说密度均匀，后者说主体明显更密）——两个指标测的是同一件事。

## 2. 修复实验：修掉矛盾不涨分

`dna_txt2img.py --consistency repair`：冲突时按层优先级（解 > 构 > 题）丢弃低层子句。
5 张矛盾最多的 DNA（3b68cb2b / e89ac772 / 523a23f6 / 070316bf / 2defe0fa），同内容同 seed 42：

| DNA | 旧（带矛盾） | 修复后 | Δ | 矛盾数 |
|---|---|---|---|---|
| 3b68cb2b | 0.665 | 0.660 | −0.005 | 2 |
| e89ac772 | 0.576 | 0.566 | −0.010 | 1 |
| 523a23f6 | 0.473 | 0.461 | −0.012 | 3 |
| 070316bf | 0.662 | 0.640 | −0.022 | 2 |
| 2defe0fa | 0.591 | 0.594 | +0.003 | 1 |
| **均分** | **0.593** | **0.584** | **−0.009** | |

**矛盾是事实，但不是瓶颈**。且"丢哪一边"的规则本身武断（真正的分歧在测量层，见上）。

## 3. 治"太碎"的子句也不起作用（按维度读数）

路线 A 的 prompt 里本来就写着 `DETAIL BUDGET: minimal … avoid fine texture and speckling`。
把路线 A 的输赢点量出来（5 张均值，维度级直接测量）：

| 维度 | 参考 | 生成 | 判读 |
|---|---|---|---|
| inner_variance | 0.201 | 0.441 | 内部太碎（2.2 倍） |
| logic.interior_ratio | 0.220 | 0.074 | 内部太少 |
| logic.hard_edge_ratio | 0.069 | 0.144 | 边缘偏硬（2 倍） |
| anatomy.content_width | 100.3 | 160.7 | 笔触太宽（1.6 倍） |

把它改成**肯定式**（`--detail-form positive`："each shape is built from a few broad confident
strokes, its interior color smooth and largely even"），再量：

| 维度 | 参考 | 否定式 | 肯定式 |
|---|---|---|---|
| inner_variance | 0.201 | 0.441 | **0.437** |
| hard_edge_ratio | 0.069 | 0.144 | **0.160** |
| content_width | 100.3 | 160.7 | **154.5** |
| patch_count | 343 | 394 | **464** |

**目标维度也几乎不动**（patch_count 反而更多）→ 不是措辞问题，
是**模型先验压过文字约束**（与 10-23 已知结论一致，这次量到了具体倍数）。

## 4. 判定：路线 A 的文本路线封顶

| 改动 | 读数 | 结论 |
|---|---|---|
| 画种词 watercolor → "A painting" | −0.002 | 无效应 |
| 留白叙述拆开 | +0.007（4/5 同向） | 噪声内 |
| 跨层矛盾修复 | −0.009 | 无效应 |
| 细节预算肯定式 | 目标维度不动 | 无效应 |

四次全在 n=5 噪声带（±0.015）内或维度不动 → **路线 A 的分数不是被 prompt 措辞卡住的**。

**方法论沉淀**：当某条子句针对某个**可测维度**时，**直接量那个维度，别看综合分**——
综合分把 30 多项平均掉了，效应会被埋掉。本轮若只看综合分，会把"目标维度完全不动"
误读成"有点用但效果小"。

## 5. 顺带补上 DNA 的两个缺字段：画种 + 含文字（VLM 打标）

新增 `scripts/label_refs_vlm.py`（本机 Qwen3-VL-8B，与 `caption_refs.py` 同模型但**目的不同、落盘不同**）：
- `medium`：画种。**程序化测不出来**（透明晕染 vs 不透明厚涂的边界本身模糊），
  而 VLM 判媒材是强项。**实测：模板写的 "watercolor" 几乎全错**——
  前 15 张里 9 张 digital illustration、2 张 ink wash painting、1 张 oil painting、
  1 张 digital painting、1 张 photograph、1 张 watercolor painting。
- `has_text`：是否含文字/签名/logo。**本库至少 8 张含文字**
  （已确认 034547b5 / 070316bf / 21249e84 / 28567c50 / 2defe0fa / 3daec7e4 / 404e4ed0 / 4f70f516，
  另有 667a9ad6 已清洗过）。含文字的图**不能进 LoRA 训练**（模型会学会写字）。

**接线**：
- `dna_txt2img.py --medium auto`：路线 A 的画种词改用 DNA 的标签（不再写死 watercolor）。
- `flux_lora_train.py`：**画种前缀自动从簇内 DNA 标签多数票取**（簇内画种可能不一致——
  实测 3b68cb2b=oil painting 而 5fcf60dc=digital illustration，"同风格簇"横跨了不同媒材）；
  `flux_lora_gen.py` 从 `meta.json` 自动带上该前缀，用户仍然只填内容。
- `flux_lora_train.py`：含文字且无清洗版的成员**自动跳过**（`--allow-text` 可强训）。

## 6. 本轮改动清单

- 新增：`scripts/prompt_consistency.py`（规则+修复）、`scripts/check_prompt_consistency.py`（体检）、
  `scripts/label_refs_vlm.py`（VLM 打画种/含文字）
- 改：`dna_txt2img.py`（`--medium auto`、`--consistency repair`、`--detail-form positive`、
  `medium_phrase()`）、`flux_lora_train.py`（画种前缀多数票自动取、含文字跳过）、
  `flux_lora_gen.py`（前缀自动带上）
- 数据：DNA 新增 `medium` / `has_text` / `_medium_note` 字段（VLM 打标，逐张补齐中）

## 7. 遗留

1. **笔触表面（厚涂质感）仍未迁移**——路线 D 最硬的骨头。候选：提 LoRA 容量、换底模、
   用「表面描述」而非「画种名」做条件。
2. **测量层内部的分歧**（`density_contrast` vs `subject_contrast`）需要定口径。
3. `subject_contrast` 指标稳健性待查（带约束的臂上普遍崩到 0.08）。
4. VLM 打标覆盖 51 张需要约 1 小时（每张两次 VLM 问询）。
5. 治理仓三篇纪要（10-28-01/02/03）**尚未 push**（另有 10-02 的 flova 纪要仍未跟踪）。

相关：[[product-poster-motion-scope]]、[[style-cluster-lora]]、[[prompt-constraint-capacity-ceiling]]、[[dna-library-data-quality]]
