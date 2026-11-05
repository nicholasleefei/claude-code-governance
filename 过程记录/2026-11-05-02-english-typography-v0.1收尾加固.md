# 过程记录 2026-11-05-02 · english-typography v0.1 收尾加固（配对质量三修）

> **Z-08 「只增不改」续记** · 编号 2026-11-05-02 · 承接 `2026-11-05-01`
> 主题：引擎 v0.1 提交后，验收配对结果时发现三类错配，逐个修掉；测试 31 → 35。

---

## 0. 一句话

首版配对把**展示体当正文**、把**中文字体当英文字体**推荐，验收时被真实数据抓出来；
补上"用途适配折扣"与"cmap 汉字数判定"两道闸，并把踩到的三处坑写成测试钉住。

---

## 1. 问题怎么暴露的

首版（2026-11-05-01）配对结果里，给 Avara / D-DIN-PRO 推荐的正文字体是：

- `GenRyuMin2 TC` / `GenSekiGothic2 TC`（**中文字体**，只是带拉丁字形）
- `RocknRoll One`（**展示体**，x-height 匹配度高但字形夸张）

根因是：**配对只比度量，不比用途**。度量相近 ≠ 能当正文。

## 2. 三处坑（每处都写进测试）

### 坑一：CJK 判定只看族名与目录名，漏了文件名
本机字体库里 `ButKo/源柔黑wTC-B.otf` 这类：目录名是罗马字、文件名是中文、
**族名也是罗马字**（name 表里写 `GenRyuMin2 TC`）——三条命名线索只命中一条。
→ 命名检测补齐为「族名 / 文件名 / 上级目录名」三处都查。

### 坑二：`cjk_font` 的计算早于 cmap 统计（最隐蔽）
先算了 `cjk_font = cjk_named or cjk_glyph_count > 100`，**之后**才遍历 cmap 统计汉字数，
于是 `cjk_glyph_count` 恒为 0，判定永远只看命名 → 罗马字族名的 CJK 字体全部漏网。
证据：`GenSekiGothic2 TC` 的 `cjk_glyph_count` 实测 **27,558**，而 `cjk_font` 仍为 False。
→ 把综合判定**移到 cmap 统计之后**。这类"初始化顺序"bug 靠人眼很难发现，
所以补了断言：**任何 `cjk_glyph_count > 100` 的字体，`cjk_font` 必须为 True**。

### 坑三：同族多字重重复占位
语料里同族多个字重各占一条，推荐列表出现 `Public Sans` 两遍。
→ 结果按族去重，同族只留最高分。

## 3. 修正内容

| 改动 | 位置 | 说明 |
|---|---|---|
| 命名检测扩到文件名 | `metrics.py` | 族名 / 文件名 / 目录名三处查 CJK 字符 |
| `cjk_glyph_count` 统计 | `metrics.py` | cmap 里 U+4E00–9FFF / U+3400–4DBF 的个数 |
| `cjk_font` 顺序修正 | `metrics.py` | 移到 cmap 统计之后 |
| 中文字体默认排除 | `catalog.py` | `allow_cjk_fonts=False`；放行时给"目视确认拉丁部分"风险提示 |
| 用途适配折扣 | `catalog.py` | 展示 0.60 / 装饰 0.45 / 手写 0.55 / 等宽 0.35 / 未知 0.90 |
| 同族去重 | `catalog.py` | 按族保留最高分 |
| 测试 | `test_typolab.py` | 31 → **35**，新增 CJK 判定、拉丁字体反证、展示体折扣、CJK 排除四类断言 |

## 4. 验收结果（真实语料）

字体目录：**1,047 款**，其中判定为中文字体 **689 款**（默认不进候选），配对时排除 **320 款**/次。

| 展示字体 | 推荐正文字体（前 3） |
|---|---|
| D-DIN-PRO（sans） | LINE Seed Sans · Public Sans · Space Grotesk |
| Bungee（display） | D-DIN-PRO · LINE Seed Sans · Space Grotesk |
| Avara（判不出衬线） | Basteleur · Tektur Condensed · Tektur |

Space Grotesk 出现在 D-DIN-PRO / Bungee 的推荐里，与 bzchoice 品牌既有搭配
（Space Grotesk 标题 + Inter 正文）方向一致——这是度量驱动的结果，不是预先写死的。

## 5. 遗留

- 衬线判定仍有 448 款"未知"（PANOSE 无判定值且族名无线索）；配对时按中性处理。
- 展示体识别仍是启发式（关键词 + PANOSE），`RocknRoll One` 这类仍被归为 serif。
- superfamily 优先（同族多字重自动识别为"一套字"）尚未实现。
- 研究线仍待补（见 `2026-11-05-01` §6 与 `research/00-index.md` §5）。

## 6. 实体

- 项目：`E:\my projects\english-typography`（my projects #33）
- 本轮改动：`engine/typolab/metrics.py` · `catalog.py` · `engine/tools/scan_fonts.py` · `engine/tests/test_typolab.py`
- 数据：`data/font_catalog.json` 重新扫描（新增 `cjk_named` / `cjk_glyph_count` / `cjk_font` 三字段）
- 纪律：零明文凭据（Z-09）；未编造任何数值或出处。
