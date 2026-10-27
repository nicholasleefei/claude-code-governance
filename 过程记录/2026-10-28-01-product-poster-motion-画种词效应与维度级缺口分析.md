# 过程记录 2026-10-28-01 · product-poster-motion 画种词效应 + 维度级缺口 + 约束搬运（负结果）

> **Z-08「只增不改」纪要** · 编号 2026-10-28-01 · 主题：上一纪要（尺度诊断与重训）之后继续攻"笔触表面" → 发现**画种词是模板常量** → 分路线测其效应（路线 A 无效应 / 路线 D 大效应 +0.092）→ 维度级缺口分析定位共同短板 → 把路线 A 的量化约束搬给路线 D（**三条路都不通**）。

---

## 0. 一句话

"笔触表面迁不过去"这件事，找到的第一个真凶不是 LoRA 而是**画种词写错了**：全库 51/51 的 DNA 都写着 "watercolor"（模板常量，不是测出来的），而参考图多是厚涂油画。但这个错词**只对提示词稀薄的路线 D 有害**（+0.092），对已经塞满作画指令的路线 A 无效（−0.002）；顺着这个线索做维度级分析，定位到真正的共同短板是**内部变异（太碎）与主体-背景笔触密度比**，但把路线 A 的量化约束搬给路线 D 并没有收益。

---

## 1. 事实缺陷：画种词是模板常量

- `dna_extract.py`：`s = "A poster painted in watercolor"` —— **全库 51/51 的 `style_prompt_en` 都是这句**（正则统计：37 张带后缀、14 张纯 watercolor，无一例外）。
- `dna_txt2img.py`：`medium = "A watercolor painting"` —— 路线 A 的 prompt 也带着它。
- 而参考图大多是**厚涂油画/丙烯/数字插画**（如 3b68cb2b：完全不透明厚涂、无透明晕染、无露纸）。
- 后果：prompt 自相矛盾——同一条里既有 "A watercolor painting"（透明晕染），又有
  "visible loose strokes" / "water-bloom marks"（厚涂特征）。

**注意**：DNA 的 `style_prompt_en` 说这些图 "painted in watercolor … 29% of the canvas breathes as open space"，
与图像事实不符——说明这不只是措辞问题，**DNA 的"画种/留白"叙述层与实测指标也有出入**（待查）。

## 2. 分路线测这个错词到底伤不伤

新增 `dna_txt2img.py --medium {watercolor,generic,none}`；路线 A 用项目既有的 5 张测试 DNA
（内容按 scene_scale 逐图匹配）、同 seed 42、同步数同参数：

| 臂 | 00dd75ea | e89ac772 | 3b68cb2b | d73779c3 | f8397adf | 均分 |
|---|---|---|---|---|---|---|
| watercolor（写死） | 0.542 | 0.548 | 0.635 | 0.553 | 0.572 | **0.570** |
| generic（"A painting"） | 0.480 | 0.598 | 0.661 | 0.545 | 0.555 | **0.568** |

**路线 A：换掉画种词毫无效果（−0.002，噪声内）。**

路线 D（FLUX-lite + 触发词 + 内容，提示词近乎裸），2 内容 × 3 参考 DNA 矩阵：

| 格 | 内容001 | 内容000 | 均值 |
|---|---|---|---|
| 基模 · 无词 | 0.659 | 0.485 | 0.572 |
| 基模 · 有词（`an oil painting of`） | 0.721 | 0.606 | **0.663** |
| LoRA(内容only 训练) · 无词 | 0.691 | 0.489 | 0.590 |
| LoRA(内容only 训练) · 有词 | 0.734 | 0.654 | **0.694** |
| LoRA(+oil painting 训练) · 有词 | 0.746 | 0.633 | 0.689 |

**路线 D：画种词 +0.092（两内容同向 +0.062 / +0.121），是本轮最大的杠杆。**

**结论（机理）**：**画种词只在提示词稀薄时才起作用**。路线 A 的 prompt 已塞满
笔触/解剖/色锁/细节预算/取景指令，模型早就在"画"；路线 D 的 prompt 只有触发词+内容，
照片先验压倒性，一个词就能撬动一大块。

**两条附带结论**：
1. 同条件下 **LoRA 相对基模只有 +0.02~0.03**（此前 n=1 时报的 +0.032 是高估）。
2. **「训练时也带画种词」无效应（−0.005）** → 推理时带就够了，不必重训。
   前缀已存进 `meta.json`，`flux_lora_gen.py` 自动带上，用户仍只填内容。

## 3. 维度级缺口分析：真正的靶子

把最好的路线 D 产物与路线 C 并排，逐维度比"与参考图的贴合度"（3b68cb2b）：

| 指标 | 参考 | 路线D | 路线C | 判读 |
|---|---|---|---|---|
| inner_variance | 0.225 | **0.451** | 0.195 | 路线D **内部太碎（2 倍）** |
| subject_contrast | 1.57 | 1.05 | 0.98 | 两条路都**主体不突出** |
| patch_count | 204 | 602 | 641 | 块数差 3 倍（碎片结构没复现） |
| hard_edge_ratio | — | 0.79 | **0.00** | 这是**路线 C 的老毛病**（边缘偏硬），非路线 D |

→ 靶子：**内部变异 + 主体-背景笔触密度比**。

## 4. 把路线 A 的量化约束搬给路线 D（**负结果**）

抽出 `constraint_clauses()`（细节预算 + 主体突出）接到路线 D（`flux_lora_gen.py --constraints <hash>`），
三种写法各 2 内容：

| 臂 | 内容001 | 内容000 | 均值 | inner_variance | subject_contrast |
|---|---|---|---|---|---|
| 不加约束 | 0.746 | 0.633 | **0.690** | 0.451 | 1.05 |
| 否定式（原写法） | 0.704 | 0.623 | 0.664 | 0.489 | 0.08 |
| 肯定式改写 | 0.668 | 0.705 | 0.687 | 0.539 | 0.11 |
| 肯定式·仅主体强调 | 0.726 | 0.663 | 0.695 | 0.482 | 0.11 |

**三种写法都不优于"不加"，两个内容上方向相反、方差很大** → 更像
**文字约束与 LoRA 权重里的风格互相拉扯**，不是某一句式的问题。
（`subject_contrast` 在所有带约束的臂上都崩到 0.08~0.11，怀疑该指标在
"主体/背景密度差极大"的图上不稳，需另做稳健性检查。）

**自我修正**：上一轮我推测"否定句在扩散模型里反噬"，**本轮的肯定式改写实验没有证实它**——
肯定式同样没救回来。该推测不成立，已在记忆里改掉。

## 5. 顺带发现：路线 A 的 prompt 内部自相矛盾（未修）

路线 A 的 prompt 是 v4/v5/v6 逐层叠上去的，**从没做过跨层一致性检查**。实测同一张图里就有：
- 解剖层「clear dark contour lines drawn around each shape」 vs 逻辑层「with no drawn contour lines」
- 逻辑层「marks evenly distributed with no contrast in density」 vs 母题层「the subject carries much denser brushwork」

矛盾子句互相稀释，**很可能是 v5/v6 消融效应落在噪声里的原因之一**。
（库一致性已有 `scripts/check_library.py`；**prompt 内部一致性检查尚缺**。）

## 6. 本轮改动清单

- `scripts/dna_txt2img.py`：`--medium {watercolor,generic,none}`；抽出 `_detail_clause/_subject_clause/constraint_clauses`（支持否定式/肯定式、只留某一项）；`dna_to_style_prompt` 改为复用 `_detail_clause`（输出不变）。
- `scripts/flux_lora_train.py`：`--caption-prefix`（画种词插在触发词与内容之间）、`--embed-tag`（嵌入缓存变体标记）。
- `scripts/flux_lora_gen.py`：`--constraints <hash>`；从 `meta.json` 自动带 `caption_prefix`。
- `scripts/exp_medium_ab.py`（新）：路线 A 画种措辞 A/B。
- `scripts/exp_score_matrix.py`（新）：路线 D 画种词矩阵打分。
- 新产物：`lora/cluster_densefloral_paint_768/`（+oil painting 训练的簇 LoRA）。
- 文档：README 新增「画种词：这一轮最大的杠杆」「维度级缺口」「prompt 自相矛盾」三节。

## 7. 遗留

1. **笔触表面仍未迁移**（最硬的骨头）。下一步候选：① 提高 LoRA 容量（rank/步数/多图）；
   ② 换底模（FLUX.1-lite 的照片先验可能比 dev 更强）；③ 用「表面描述」而非「画种名」做条件。
2. **DNA 的叙述层与实测指标有出入**（如 3b68cb2b 被写成 watercolor + 29% 留白，实际是满幅厚涂）→ 待查是模板写死还是测量口径问题。
3. **prompt 内部一致性检查**尚缺（矛盾子句检测）。
4. `subject_contrast` 指标的稳健性待查。
5. 路线 D + 路线 C 叠加未验。

相关：[[product-poster-motion-scope]]、[[style-cluster-lora]]、[[prompt-constraint-capacity-ceiling]]、[[dna-library-data-quality]]
