# 过程记录 2026-10-02-01 · Flova CLI v0.0.10 装机与台账登记

> **Z-08 「只增不改」纪要** · 编号 2026-10-02-01 · 主题：Flova CLI 接入与功能研究项目（`E:\my projects\flova-ai-research`）立项 → CLI v0.0.10 装机 → 台账 CLI-010 / S-049 双登记 → junction 待用户授权。

---

## 0. 一句话

按 Z-19 给 Flova 视频创作云接一个本机研究项目；按 Z-18 装 CLI；按 Z-04 + Z-03 登记台账；junction 一步被 harness 拦了权限，命令已写好在项目说明书里等用户授权跑。

---

## 1. 立项背景

飞总 2026-10-02 14:04 起手：「我想开一个新的项目是结合 Flova ai 的，你帮我新建一个项目在 E:\my projects，请帮我安装 Flova CLI: https://cli.flova.ai/flovaCLI-setup-cn.md，并开始研究一下他的功能。给我一个列表。」随后补充：「这应该是代理的问题，你绕过代理去这里安装就行。」

按 Z-19「进场自建」要求，无则当场建项目：`E:\my projects\flova-ai-research\`。

---

## 2. 安装路径

### 2.1 失败尝试
- `WebFetch` 拉 `https://cli.flova.ai/flovaCLI-setup-cn.md` 失败（Socket is closed）
- `curl` 直接走 `https://cli.flova.ai/` 失败（`curl: (35) schannel: failed to receive handshake`）
- **根因**：本机 `HTTP_PROXY=127.0.0.1:10808` / `HTTPS_PROXY=127.0.0.1:10808`（xray），与 cli.flova.ai 域名的 TLS 握手被拦；Telegram 之类国内镜像代理对境外 S3 桶不友好。

### 2.2 解决
所有后续 `curl` / `Invoke-WebRequest` 必须带 `--noproxy "*"`（curl）或 `-Proxy $null`（PowerShell）。

### 2.3 实际安装（绕过代理 + 手动）
1. 拉桶 list 拿到所有 release tag 与 checksums；当前 latest = **v0.0.10**（built 2026-08-19T08:03:42Z / commit 4258717 / go1.26.5 windows-amd64）。
2. 拉 `releases/v0.0.10/flova-cli_v0.0.10_windows_amd64.zip` + `releases/v0.0.10/checksums.txt`。
3. **SHA256 校验通过**：`89f9d319b3581df4e64e03f7b33bedb9242750b41e51aac6bcf187879ea3e746`，与 `checksums.txt` 同名行一致。
4. 解压内容：`flova.exe`（8.4 MB）+ `skills/flova/SKILL.md` + `skills/flova/references/{commands,model-knowledge,runtime-contract,skill-builder}.md`。
5. 装 `flova.exe` → `E:\tools\flova\`（**Z-18 默认落点**，install.ps1 自定义变量 `FLOVA_INSTALL_DIR` 可改）。
6. 装 SKILL → `C:\Users\pc\.flova\`（install.ps1 默认 `FLOVA_HOME`，符合 setup 文档第 3 步「不要搬 `$HOME/.flova`」要求）。
7. 加 `E:\tools\flova` 到用户 PATH（PowerShell `[Environment]::SetEnvironmentVariable('Path', ..., 'User')`）—— 实测 `user PATH now:` 已含 `E:\tools\flova` 末位。

### 2.4 不替用户做的事
- ❌ 不登录（`flova auth login` 是用户的活；不替用户决策账号、不替用户点浏览器）
- ❌ 不替用户在 `C:\Users\pc\.claude\skills\flova` 建 junction（**harness 分类器拦了 `New-Item -ItemType Junction`**；命令写好在项目说明书 ③.1 等用户手敲）

---

## 3. 台账登记

### 3.1 CLI-010（E:\04-cli\index.html）
- **编号**：CLI-010
- **位置**：E:\tools\flova\flova.exe（Z-18 默认落点，**用户 PATH 已加**）
- **能力**：14 个一级子命令（account / auth / chat / completion / download / export / profile / project / resource / run / skill / update / upload / version）；核心 `project create → upload → run → run resume → export video / download all`；SKILL.md 强约束 Active Run Contract / Pending Action / Branch & Restore。
- **状态**：在编
- **互链**：→ S-049；项目说明书；本纪要
- **计数**：9 → 10（在编 9 / 退场保留 1）

### 3.2 S-049（E:\02-skill\index.html）
- **编号**：S-049
- **类型**：外部下载（厂商 CLI 自带 Skill）—— bundled 在 release zip `skills/flova/`
- **位置**：真身 `C:\Users\pc\.flova\SKILL.md` + `references/`（4 篇 ~89KB）；Claude Code 索引 junction **待用户授权建**
- **能力**：驱动 CLI 走完整 Flova 视频项目工作流（按 SKILL.md Active Run Contract / Pending Action / Branch & Restore 三件事的强约束）
- **状态**：在编（junction 待建）
- **互链**：→ CLI-010
- **计数**：48 → 49

### 3.3 总台账（E:\my projects\index.html）
- **#17 flova-ai-research**：meta、toc、表格行、#17 卡片四处同步更新；体量 3 文件（项目说明书 1 + notes/空 + 输出物/空）
- **计数**：16 → 17

---

## 4. 功能列表（飞总说"给我一个列表" → 已落项目说明书附录）

14 个一级子命令，按工作流分组：

| 工作流环节 | 命令 |
|---|---|
| **账号 / 环境** | `flova account user` · `flova auth login/logout/status` · `flova profile add/list/remove/show/use` · `flova version` |
| **项目生命周期** | `flova project create/list/info/branch/restore/restore-plan` |
| **项目读** | `flova project storyboard/document/resources` · `flova chat history/pending-actions` · `flova resource info` |
| **上传** | `flova upload <file> [--timeout 600]` |
| **运行（核心）** | `flova run "$PROJECT_ID" --content "..." [--file-from upload.json] [--skill-id ID] [--skill-share-code CODE] [--timeout N] [--no-progress-previews]`（聚合，**不走 --no-wait**） |
| **运行恢复** | `flova run current "$PROJECT_ID"` · `flova run status --stream-chat-id ID` · `flova run result --stream-chat-id ID` |
| **Pending Action** | `flova run resume "$PROJECT_ID" --message-id ID --action-id ID [--option ID]`（**option 必须用户明确选**） |
| **导出** | `flova export readiness "$PROJECT_ID"` · `flova export video "$PROJECT_ID"` · `flova export current/status` |
| **下载** | `flova download all "$PROJECT_ID" [--types images,videos,audios,music]`（**不能替代最终视频文件**） |
| **Skill 管理** | `flova skill list/feed/categories/get/copy/install/uninstall/create/update/delete`（NFKC 严格匹配名） |
| **升级** | `flova update [--check] [--force]`（**唯一升级入口**） |
| **补全脚本** | `flova completion bash/zsh/fish/powershell` |

完整表格版含「分类 / 命令 / 用途」三列落 `E:\my projects\flova-ai-research\项目说明书\index.html#附--给用户的列表功能列表你说要的功能列表`。

---

## 5. Junction 这件事

### 5.1 为什么需要
官方 setup 文档第 3 步：「If the client supports local Skills, keep `$HOME/.flova` as the only source directory and symlink it as `flova` into the location the current Agent can index.」——Claude Code 扫的是 `C:\Users\pc\.claude\skills\`，里面没有 `flova` 子夹就加载不到 `SKILL.md`，SKILL 就只是个孤儿文件。

### 5.2 本轮做的与被卡的
- ✅ 装好了真身 `C:\Users\pc\.flova\`（SKILL.md + references/）
- ❌ 在 `C:\Users\pc\.claude\skills\flova` 建 junction → `C:\Users\pc\.flova` —— **harness Stage 2 classifier 报「deny」，定性为 modification of outcome**。

### 5.3 不绕过
按系统消息「don't pursue the same outcome through another tool, interpreter, host, encoding, sub-agent or later turn」——我没有用 `cmd /c mklink /J`、`ln -s`、sub-agent 等等替代路径再试。

### 5.4 等用户
命令（含 PowerShell / cmd 兜底两条）写在 `E:\my projects\flova-ai-research\项目说明书\index.html#③.1` 等用户手动 PowerShell 执行。建好后无需重启，新 bash / 新 Claude Code 会话即可加载。

> **2026-10-02 复跑小坑**：第一次给的命令用 `powershell -NoProfile -Command "..."` 把脚本包成一行，撞 PowerShell 双引号字符串里 `\c` `\s` 等被当转义符（"Unrecognized escape"），加上 `>>` 续行把字符串截断成多条命令，于是 `New-Item -Path` 没了参数、后续行被当 cmdlet 名解析。**修法**：让用户直接在 PS 里跑三行，不要再包 powershell.exe 子进程前缀。三行版与 `cmd /c mklink /J` 备选已同步回项目说明书 ③.1。

### 5.5 不影响什么
- ❌ junction 没建 ≠ `flova` 命令不能用 —— 命令行直接 `flova` 全功能。
- ❌ junction 没建 ≠ 项目不能跑 —— 用户可以手动开 Claude Code 在 `E:\my projects\flova-ai-research\` 目录，agent 会通过 `/skill flova` 临时加载或纯靠命令行。
- ❌ junction 没建 ≠ 后续登录与冒烟用例卡住 —— `flova auth login` + `flova project create` 都不依赖 Skill。
- ✅ junction 没建 **唯一影响**：Claude Code agent 不会自动按 SKILL.md 工作流跑；需用户在提示词里明示「按 flova skill 工作流」或主动调用 `/skill flova`。

---

## 6. 边界与红线（先认后踩，已落项目说明书 4.3）

- 🔴 **云端计费**：Flova 是云服务，agent 不替用户买、不在台账记费用；`membership_required` / `insufficient_credits` 错误含 `data.upgrade_url`，原样转给用户。
- 🔴 **Active Run Contract**：`flova run` 聚合，不起额外轮询、不用 `--no-wait`、不合并 stdout/stderr。
- 🔴 **本地不能直生成**：Flova 不开放裸生成接口；agent 活是「提需求→审核→纠偏」，不是「画图/合成视频」。
- 🟡 **Z-09 凭据零明文**：API key / token 不写日志 / 纪要 / 台账 / 消息；只走环境变量或浏览器登录一次性回执。
- 🟡 **Signed URL 不持久化**：`export_url` / `resource_url` 含签名，不进 notes / 消息 / 项目文档；存本地路径即可。

---

## 7. 文件清单（本轮产出）

| 路径 | 性质 | 说明 |
|---|---|---|
| `E:\my projects\flova-ai-research\` | 项目根 | 新建（Z-19 进场自建） |
| `E:\my projects\flova-ai-research\项目说明书\index.html` | 项目说明书 | 四块格式 + 功能列表附录 |
| `E:\my projects\flova-ai-research\notes\` | 研究笔记 | 空（首份待写） |
| `E:\my projects\flova-ai-research\输出物\` | 本地产物 | 空（首份待冒烟用例出） |
| `E:\tools\flova\flova.exe` | Z-18 工具落点 | 8.4 MB · SHA256 通过 |
| `C:\Users\pc\.flova\SKILL.md` | SKILL 真身 | 13.9 KB |
| `C:\Users\pc\.flova\references\{4 篇}.md` | SKILL references | ~89 KB |
| `C:\Users\pc\.claude\skills\flova → C:\Users\pc\.flova` | junction | **未建，待用户授权** |
| `E:\04-cli\index.html` | CLI 台账 | CLI-010 行 + meta + h2 + 变更记录 |
| `E:\02-skill\index.html` | Skill 台账 | S-049 行 + meta + h2 + 变更记录 |
| `E:\my projects\index.html` | 项目总台账 | #17 四处同步 |
| `E:\claude code总则\过程记录\2026-10-02-01-flova-cli-install.md` | **本纪要** | Z-08 只增不改 |

---

## 8. 待办（按用户决定推进）

1. ⬜ 用户在 PowerShell 跑项目说明书 ③.1 的 junction 命令
2. ⬜ 用户跑 `flova auth login`（浏览器或环境变量 API key；不替登、不记 token）
3. ⬜ 用户与 agent 一起走冒烟用例（30 秒产品介绍短片），落首份本地视频到 `输出物/`
4. ⬜ 补 branch / restore / pending action（membership_required / insufficient_credits）/ Skill builder 闭环的笔记
5. ⬜ Z-07 周检把 CLI-010 / S-049 / junction 三件事的健康度纳入例行

---

## 9. 一句话回顾

CLI v0.0.10 装好 + 台账双登记完成；junction 一步让用户 30 秒在 PowerShell 里敲一行命令就能补上；功能列表按工作流 12 类列出 + 落项目说明书；Flova 与本机生态的边界与红线先认后踩。**没有发生任何凭据泄露、没有绕过任何权限。**

---

## 10. 闭环补记（2026-10-02 当日，用户手敲三行 PS 命令后回执）

- ✅ 用户在 PowerShell 直接跑三行（见项目说明书 ③.1），建链成功（Get-Item 返回 `LinkType=Junction / Target=C:\Users\pc\.flova`）。
- ✅ 新会话 Claude Code 自动加载 `flova` skill（harness system-reminder 中已列出）。
- ✅ S-049 状态由「在编（junction 待建）」→「在编（junction 已建 · 2026-10-02 用户手敲确认）」。
- ✅ 项目说明书 4.2 待办第 1 项划掉。
- 下一步：用户 `flova auth login`（浏览器登录或 `FLOVA_API_TOKEN` 环境变量；不替登、不记 token）→ 冒烟用例。

