# 过程记录 2026-10-28-02 · product-poster-motion 留白叙述层修正（4/5 同向，幅度在噪声内）

> **Z-08「只增不改」纪要** · 编号 2026-10-28-02 · 主题：承 2026-10-28-01（画种词效应），在 DNA 的**叙述层**里又抓到一处把测量讲错的实例并修正 —— 留白叙述把"低密度区"说成"没画的纸"，导致路线 A 被指示在满幅厚涂图上留出 1/3 空白。

---

## 0. 一句话

DNA 的数值测得对，不代表转成话的说法对：`whitespace.effective_ratio` 是"裸纸 + 低密度区"的混合值，
叙述层一律叫它 "untouched paper"，于是路线 A 被指示在**满幅厚涂**的参考图上留出 29% 空白。
已改为分开叙述；A/B 结果 4/5 同向为正、均分 +0.007，**方向对但幅度在噪声内**。

---

## 1. 缺陷

`dna_txt2img.py` 的留白子句（修前）：

```python
space = _band(ws, [(0.15, "... little empty space"),
                   (0.30, "a breathing composition with about a third of the canvas as open paper"),
                   (0.45, "generous open empty paper surrounding the subject"),
                   (1.01, "a minimalist composition dominated by large areas of empty paper")])
if ws and ws > 0.15:
    space += f" (roughly {round(ws*100)}% untouched paper)"
```

而 `ws = whitespace.effective_ratio` = **paper_like_ratio + sparse_low_density_ratio × 内容区占比**，
是**两个不同物理量**的混合：

| DNA | paper_like（真·裸纸） | sparse_low（低密度区） | effective | 旧 prompt |
|---|---|---|---|---|
| 3b68cb2b | **0.068** | 0.241 | 0.293 | 「roughly 29% untouched paper」 |
| 5fcf60dc | 0.076 | 0.266 | 0.322 | 同上 |
| d43afa51 | 0.096 | 0.276 | 0.346 | 同上 |
| 00dd75ea | 0.795 | 0.561 | 0.910 | 同上 |

前 3 张是**满幅厚涂、几乎没有裸纸**的画，却被指示"留出约三分之一画面当空白纸"。
低密度区在厚涂里其实是"安静些的薄涂段落"，不是没画的纸。

## 2. 修正

`dna_txt2img.py` 新增 `--space-mode {new,legacy}`（默认 new）：
- `paper_like_ratio` ≥ 0.20 → "around N% of the canvas is bare paper or bare ground"；
  ≥ 0.08 → "only a little bare paper shows through"；
- `sparse_low_density_ratio` ≥ 0.20 → "the quieter passages are thinly painted rather than bare"；
- `_band` 的中档措辞由 "as open paper / empty paper" 改为 "quieter and less densely painted / quiet passages"。

## 3. A/B（同 5 张测试 DNA · 同内容 · 同 seed 42 · 同参数 · v7 评分器）

| DNA | 旧留白写法 | 新留白写法 | Δ |
|---|---|---|---|
| 00dd75ea | 0.542 | 0.542 | +0.000 |
| e89ac772 | 0.548 | 0.576 | +0.028 |
| 3b68cb2b | 0.635 | 0.665 | +0.030 |
| d73779c3 | 0.553 | 0.565 | +0.012 |
| f8397adf | 0.572 | 0.536 | −0.036 |
| **均分** | **0.570** | **0.577** | **+0.007** |

**判读**：4/5 同向为正，但 **+0.007 落在本项目已知的噪声带（±0.015）内**（见 10-24 的消融教训：
n=5 分辨力不足）。→ **修正保留（依据是旧指令事实性错误，不是分数）**，但**不宣称性能胜利**；
要分辨需更大样本或配对同图对照。

## 4. 由此抽出的通则（值得写进项目方法论）

**DNA 的测量层与叙述层必须分开审。**
数值对 ≠ 话说对。叙述层一旦掺进"画种""留白"这类**带隐含画法**的词，
就会变成对生成模型的错误指令。本项目已在叙述层抓到两例：
- 画种名写死 watercolor（10-28-01 纪要）；
- 留白把低密度区说成"untouched paper"（本篇）。
两者都是**模板/混合口径问题**，不是测量公式问题——测量层反而是对的。

## 5. 遗留（接 10-28-01）

1. **prompt 内部一致性检查**尚缺（解剖 vs 逻辑 vs 母题层的矛盾子句）——很可能是 v5/v6 消融落在噪声里的原因。
2. **画种名仍无程序化测法**：目前只有 VLM 打标（`caption_refs.py` 已有 VLM 在环）这一条路，
   尚未做「VLM 打画种 → 存进 DNA → 供 prompt 使用」的闭环。
3. `subject_contrast` 指标稳健性待查（带约束的臂上普遍崩到 0.08）。
4. 笔触表面（厚涂质感）仍未迁移，是路线 D 最硬的骨头。

相关：[[product-poster-motion-scope]]、[[prompt-constraint-capacity-ceiling]]
