# Orca 功能文档

> 版本基准：`orca` v1.4.197（`package.json`）。内容依据仓库内 `README.md` 与 `docs/site/content/docs/` 官方文档整理，面向想要完整了解 Orca 能力边界的用户与开发者。

---

## 1. Orca 是什么

Orca 是一个**并行运行多个 AI 编码 Agent 的桌面 IDE**。每个任务都会拿到属于自己的 git worktree、自己的 agent 终端、自己的浏览器标签，因此可以同时把一个需求分发给 Claude Code、Codex、Cursor CLI 等多个 Agent，而不需要 stash、切分支或打断心流。

- **平台**：macOS、Windows、Linux 桌面端 + iOS / Android 移动伴侣应用。
- **许可**：MIT 开源。
- **定位**：`Next-gen IDE for parallel agentic development`。

### Orca 不是什么

| 不是 | 说明 |
| --- | --- |
| 不是模型 | Orca 只编排你已经订阅的 Agent（Claude / Codex / OpenCode 等），自带订阅即可 |
| 不是 git 替代品 | 每个 workspace 都是真实的 git worktree，可以随时 `cd` 进去用原生 git |
| 不是托管 VPS 产品 | 默认跑在你自己的桌面；远程算力用的是你自己的机器和云账号 |
| 不是零代码工具 | 面向会读 diff、在意 commit 的职业开发者 |

### 典型使用场景

- 让三个 Agent 同时攻同一个 bug，再挑最优解合并。
- 认真逐行评审 AI 生成的 diff 之后再发布。
- 把已有的 Claude Code / Codex / Cursor CLI 订阅收敛到一个编排入口。
- 把 Agent 放到远端跑（SSH、自托管 Orca 服务器、按需 VM），同时保留本地 IDE 体验。

---

## 2. 核心模型

### 2.1 Worktree（工作区）

Orca 是 **worktree 原生**的：每个任务对应一份独立的磁盘检出，这正是并行 Agent 互不踩踏的基础。

**模型要素**

- 每个仓库有一个 **base ref**（通常是 `origin/main`）。
- 每个 worktree 有一个 **start-from ref**（从哪里分叉）。
- 每个 worktree 拥有独立分支、独立文件、独立 Agent 终端。
- 删除 worktree 会同时删除目录与分支（有确认）；若 git 因存在未合并提交而保留了本地分支，Orca 会提供「保留分支复核」入口。

**生命周期**：创建（任务名 + start-from + 可选关联 GitHub / Linear / Jira / GitLab 条目）→ 工作（Agent 终端、编辑器、浏览器、终端分栏都限定在该 worktree）→ 评审（对比 start-from 的 diff、Annotate AI Diff、Attribution）→ 交付（commit、push、开 PR、等 CI）→ 归档或删除。

**关键能力**

- **后台创建**：提交创建对话框后立即关闭，`git fetch` 与 `git worktree add` 在后台继续；标签页显示实时进度，可切走、可取消、失败可重试。
- **Start-from 选择器**：base ref、其它本地分支（可堆叠在评审中的 PR 之上）、指定 commit SHA、已有远程分支（自动 fetch）。
- **gitignore 文件继承**（三种互补机制）：
  1. `Settings → Repository → Worktree Shared Paths`（按用户配置，macOS 优先 APFS clone-copy，否则 symlink）；
  2. `orca.yaml` 的 `worktree.sharedDirectories`（仓库内声明，共享 `node_modules`、`.cache` 这类可重建大目录）；
  3. 仓库根的 `.worktreeinclude`（**复制**而非软链，适合 `.env`、本地配置；仅支持字面量路径）。
- **分支命名**：默认从 workspace 名或关联的 PR / Issue 推导；`Advanced` 抽屉可手填分支名，也可指定 **Parent workspace**（仅影响侧边栏层级，不改 git 历史）。从 Linear issue 创建时直接采用 Linear 提供的分支名。
- **Emoji 工作区名**：支持 Slack 风格 `:rocket:` 短码，生成分支名时自动转写为可读短码。
- **侧边栏**：按项目分组，支持过滤（休眠 / 默认分支 / 自动化创建 / CLI 创建 / 其它客户端创建 / detached HEAD）、置顶、多选（`Cmd`/`Shift` 点选）、拖拽排序、双击重命名、右键归档/休眠/删除，以及对嵌套子工作区的 **Sleep with Descendants** 与 **Delete with Descendants**。
- **Resource Manager → Clean up workspaces**：跨主机盘点工作区（含断连 SSH 主机上的），按状态、活跃度、体积、git 状态、关联评审筛选后批量清理。
- **多仓库项目组与文件夹工作区**：导入包含多个 git 仓库的父目录时可归为一个项目组，并创建绑定到其中某个仓库任务源的「文件夹工作区」。
- **外部 worktree**：手工 `git worktree add` 的目录默认不显示，可通过「Non-Orca worktrees」对话框逐个显示；`Settings → General → Workspace` 可设置全局的外部 worktree 来源策略。

### 2.2 标签页、面板与分栏

- 每个标签页承载一样东西：终端、编辑器、浏览器、diff、PR。
- 拖拽标签到面板**右边缘**左右分栏、拖到**下边缘**上下分栏，分栏可嵌套。
- 终端标签也可在标签内 **Split terminal right / down**。
- **面板边界固定**：调整窗口不会打乱布局，边界位置按 worktree 保存。
- 每个 worktree 拥有自己的整棵面板树，切换 worktree 会整体换页。
- 默认快捷键（新安装）：跨类型切换标签 `Cmd+Shift+]` / `[`；同类型切换 `Cmd+Option+]` / `[`；最近标签 `Ctrl+Tab`；关闭全部编辑器标签 `Cmd+Option+W`。

### 2.3 Agent 会话与状态

一个 **agent session** = 一个 worktree 里一个终端中运行的一个 CLI Agent。状态来自终端 OSC 标题序列与 agent hooks。

状态标记：**转圈**＝工作中；**琥珀问号**＝等待你（权限/提问）；**绿色对勾/绿点**＝完成或安静活跃；**红点**＝阻塞、被打断或失败；**灰点**＝空闲；**无标记**＝普通 shell。

- **Agent Dashboard**（Settings → Experimental 开启）：跨 worktree 的看板，列为 **Needs You / Working / Done / Idle**（Idle 默认隐藏）；支持搜索、按项目/工作区状态/PR 状态筛选；可窗口内或独立弹出窗口；卡片显示会话名、最近消息预览、项目、年龄、主机徽章，嵌套子 Agent 可展开。
- **启动默认值**：Orca 默认给每个 Agent 预置「完全自治」权限旗标（Claude `--dangerously-skip-permissions`、Codex `--dangerously-bypass-approvals-and-sandbox`、Gemini `--yolo` 等），把 worktree 本身当作隔离边界。可在 `Settings → Agents → Agent Permissions` 全局切换 **Yolo / Manual**，或逐个 Agent 覆写启动参数（有 Reset 还原）。
- **Restart 芯片**：Agent 退出（正常或崩溃）后标签出现一键重启，沿用同一工作目录；Codex 还会保留当前账号。

### 2.4 会话恢复

关闭 Orca 再打开，会恢复：打开的 worktree、每个 worktree 的分栏布局与聚焦标签、终端回滚缓冲（包括关闭期间产生的输出），**以及仍在运行的 Agent 进程**——后台守护进程持有 PTY，关窗不会杀掉正在跑的 Claude Code / Codex。

- 守护进程存活的场景（`Cmd-Q` 退出、自动更新重启、应用崩溃）→ Agent 继续跑，下次启动热重连。
- 守护进程死亡的场景（重启、系统更新、断电）→ 布局与回滚仍恢复，但 Agent 进程已结束，需要重新运行。

### 2.5 导航：Quick Open 与 Jump Palette

- **Quick Open（`Cmd-P`）**：当前 worktree 的文件搜索，按最近使用 + 匹配度排序；结果以文件名开头；gitignore 文件作为第二梯队出现。
- **新标签 omnibox（标签栏 `+`）**：一个输入框同时搜索已打开标签、文件、URL、Agent；输入非路径文本可直接用默认搜索引擎在 worktree 浏览器中搜索，`?` 前缀强制搜索。
- **Worktree Jump Palette（`Cmd-J`）**：跨 worktree 与标签跳转。空查询时展示「最近的聊天与终端」（`Cmd-1`~`Cmd-6` 直达）和「最近的 worktree」；支持按 PR 标题/编号 `#123`、GitLab MR `!123` 匹配；`Tab` 打开主机与仓库级过滤；`Shift-Enter` 在新分栏打开；查询无命中时提供「Create worktree」直接建工作区。

---

## 3. Agent 支持

### 3.1 覆盖范围

**任何 CLI Agent 都能跑**——只要它能在终端里运行。以下为内置预配置、可一键启动/自动安装的 Agent（深度集成项已标注）：

| Agent | 集成深度 |
| --- | --- |
| Claude Code | 用量、账号热切换、hooks |
| Claude Agent Teams | 默认关闭，可用 `orca claude-teams` 启动，每个成员一个原生面板 |
| Codex | 用量、账号热切换 |
| Cursor CLI | 深度集成 |
| MiniMax | 自动安装、用量与限流追踪 |
| Pi / OMP / Prime Agent / Droid / Antigravity | 自动安装、hooks、状态（Prime Agent 另含会话历史）|
| OpenCode / Ante / Command Code | 自动安装、状态 |
| Grok、GitHub Copilot CLI、Gemini、Aider、Goose、Amp、Kilocode、Kiro、Charm Crush、Auggie、Autohand、Cline、Codebuff、Continue、Devin、Kimi、Mistral Vibe、Qwen Code、Rovo Dev、Hermes、OpenClaw、Trae | 自动安装 |

### 3.2 Chat UI（原生聊天，实验特性）

在受支持的 Agent 终端之上叠加的聊天视图；**终端仍是事实来源**，Chat UI 只是同一个 PTY 的结构化转写 + 输入框。转写解码覆盖 Claude、Codex、Grok、OMP。

- 输入框支持发送/停止、附件、跨断连保留草稿与附件；`/` 唤出按 Agent 过滤的斜杠命令与已发现的 skills。
- **模型与选项药丸**：Claude 的模型列表取自该主机上已安装的 Claude CLI（不是硬编码）；Codex 支持直接选模型（发送 `/model <id>`）；Grok 暴露模型与推理强度（随模型变化）。
- Agent 抛出的 `AskUserQuestion` 等结构化提问会渲染成问题卡片，直接在输入路径回答，远程配对主机同样适用。

### 3.3 会话历史

Orca 扫描各 Agent CLI 在磁盘上留下的会话转写（`~/.codex/sessions`、`~/.claude`、Cursor 日志、OpenCode 的 `opencode.db` 等），在右侧栏 **Agent Session History** 面板列出。

- 作用域：Workspace / Project / All；可按标题、工作目录、分支、模型、预览文本搜索。
- 视图选项：按 Agent 开关扫描范围、按更新/创建时间排序、按项目/文件夹/Agent 分组、隐藏空会话。
- **Resume**：在同一 `cwd` 新开终端并执行该 Agent 的恢复命令（`claude --resume <id>`、`codex resume <id>`、`pi --session <path>` 等），无需手拼 `--resume`。还可复制恢复命令、会话 ID、日志路径，打开日志或 `cwd`。
- 远程工作区可浏览本地历史，但 Resume 只能在本地工作区执行。

### 3.4 Agent 休眠（实验特性）

空闲 Agent 会占着 PTY 与模型会话内存。开启后，Orca 会在满足**全部**条件时暂停后台 Agent 终端：Agent 处于 done 状态、不在当前前台 worktree、完成后没有新按键、属于可恢复会话的 Agent（Claude、Codex、Gemini、Antigravity、OpenCode、Pi、MiMo Code、Droid、Grok、Devin、OMP）、空闲超过阈值（默认 30 分钟，可设 1 分钟~24 小时）、没有移动端会话在驱动、没有未结算的编排 Dispatch、面板上没有存活的子 Agent。

重新打开 worktree 时自动用相同的 resume 参数与原始启动命令/环境恢复，无需点击。同一 worktree 的多个 Agent 面板作为一个整体休眠。

### 3.5 用量与限流追踪

读取 Claude Code、Codex、Gemini、OpenCode、Kimi Code、MiniMax 在本地磁盘上的用量状态（无额外 API 调用与鉴权），在状态栏展示：

- 当前账号套餐下的用量；
- 5 小时 / 每日 / 每周 / Claude Fable 周窗口的重置倒计时；
- 越过 80% 阈值的告警芯片；
- 多账号分别计账、用量花名册；移动端同样可见；`Settings → Appearance` 可切换显示「已用 %」或「剩余 %」。

### 3.6 账号热切换

Claude 与 Codex 支持在不重新登录的情况下挂载多个账号并即时切换（包含「系统默认账号 vs 额外账号」的区分）。Codex 的 Restart 芯片会保留当前账号；移动端账号页可看用量、重置倒计时，并在 Codex 攒到 **rate-limit reset** 额度时消费一次重置（带幂等日志，避免弱网重试重复扣减）。

### 3.7 Hooks 与记忆

支持按仓库的 hooks、worktree 创建时的 setup hooks、记忆文件，以及 **Agent status hooks**（把 working / waiting / done 状态送进 Orca，切换即时生效，含 Windows WSL hook 中继；CLI 为 `orca agent hooks on|off|status`）。

---

## 4. 终端

基于 xterm.js（与 VS Code 同源），面向 Agent 工作流做了增强。

- **面板与标签**：终端即标签，可任意分栏；Agent 终端标签显示身份与实时状态，Claude / Codex 还能显示 AI Vault 会话名（手动重命名优先）。
- **OSC 52 剪贴板**：默认允许 Zellij、tmux、Neovim、fzf、Grok 等 TUI 通过 PTY 写系统剪贴板，SSH 下同样生效；可在设置中关闭。
- **搜索**：`Cmd-F` 查找回滚内容，支持高亮、大小写、正则、逐项跳转。
- **链接动作**：普通点击弹出紧凑操作气泡（Orca 浏览器 / 系统浏览器 / 复制链接，含 OSC 8 隐藏目标）；`Cmd`/`Ctrl` 点击直接打开；`Shift+Cmd` 走链接路由的相反默认值。可预览的 HTML 文件链接能直接在 Orca 内打开，远程文件另有「下载并用默认应用打开」。
- **Copy Context**：右键复制该面板的有界转写，便于粘到别处。
- **主题**：内置多套主题可自定义；支持首次启动导入 **Ghostty** 配置（主题/字体/光标），以及从 **Warp** 主题目录或任意 YAML 目录导入主题。
- **Windows shell**：默认 shell 可选 PowerShell / CMD / WSL（`wsl.exe --status` 成功时自动提供）；WSL 文件系统仓库经 `wsl.exe -d <distro>` 启动，Windows 路径仓库会翻译为 `/mnt/<drive>/...`。
- **原生按键**：广播 kitty 键盘协议，终端应用能收到真实的 `Shift+Enter`、`Ctrl+Enter` 等修饰键组合；macOS 日文 JIS 键盘可开启「¥ 转反斜杠」。
- **浮动终端**：全局 shell 面板，`Cmd+Option+A`（macOS）/ `Ctrl+Alt+A` 呼出，自带标签页，可设置起始目录，支持发起编排任务而不占用 worktree 面板。
- **常用快捷键**：`Cmd-T` 新终端标签、`Cmd-Alt-T` 新 Agent 标签（macOS）、`Cmd-W` 关闭、`Cmd-\` 右分栏、`Cmd-Shift-\` 下分栏。

### Quick Commands（快捷命令）

保存常用终端命令（`npm run dev`、`pnpm test`、项目 setup 脚本）或可复用的 Agent 提示词。每条命令有标签、命令体、作用域（**Global** 或 **Project**）。可从标签栏按钮新开终端并执行，或从终端右键菜单插入到当前终端；行内有复制按钮。与 Remote Orca Server 配对时，选择器会并排显示**本地与远程**两套集合并标注 **Saved on** 主机；命令存在哪里与在哪里执行是两回事。移动端共用同一份列表并双向同步。

---

## 5. 编辑与预览

### 5.1 Monaco 编辑器

- **自动保存**：失焦与短暂空闲后保存，正常流程中没有「未保存」状态点。
- 多光标 `Cmd-D`、文件内/工作区查找 `Cmd-F` / `Cmd-Shift-F`、`Cmd-Click` 跳转定义。
- **Changes view mode**：在标签内把文件切成 HEAD vs 工作树的 diff，不丢光标位置。
- 自动换行（默认开，`Alt+Z` 切换，与 Diff 换行是两个独立设置）、可选 minimap、可单独设置编辑器字体（默认跟随终端字体）。
- 定位是 **editor-first 而非 IDE-first**：类型检查与 lint 请在终端面板里跑。

### 5.2 富文本 Markdown 编辑器

- 默认以富文本方式打开（`Cmd-Shift-M` 切回原始 Monaco）；大于 300 KB 的文件先开原始编辑器，可选择「仍然打开」。
- `/` 斜杠菜单：标题、列表、代码块、callout、图片、mermaid、折叠块（`/toggle-text`、`/toggle-h1`…，以可移植的 `<details>`/`<summary>` 存盘）。
- `[[` 触发 wiki 风格内部链接，自动补全 worktree 内路径并插入相对链接。
- 编辑器内搜索基于**渲染后文本**，`# Install` 与 `<h1>Install</h1>` 都能被「Install」命中。
- **评审批注**：选中渲染文本即可加批注，绑定到源码区间（`Cmd+Shift+A`）。
- Front matter（YAML/TOML）默认展示，可按文件隐藏；表格支持 Tab/Enter 导航、空行退格删除、工具栏与右键的行列增删；长文档可开 **Table of Contents** 大纲。
- **Share as artifact**：把当前 Markdown 发布为公开查看链接（需在设置中开启，默认关闭）。

### 5.3 内置查看器

| 类型 | 能力 |
| --- | --- |
| HTML | 沙箱浏览器标签渲染，按需授权读取额外目录，外链需确认，预览内禁用下载；支持本地 / SSH / 配对运行时 |
| Mermaid | Markdown 预览内联渲染；独立 `.mmd` 文件有带缩放平移的专用查看器 |
| PDF | 滚动、缩放、文本选择；会话内记忆滚动位置（含页内偏移）|
| 图片 | `.png/.jpg/.svg/.webp/.gif`，并支持同一文件两个版本的图片 diff |
| CSV / TSV | 表格视图，列排序 + 快速搜索，可切回原始文本 |
| Jupyter | `.ipynb` 渲染 markdown、代码高亮与已保存输出，编辑写回时保留 nbformat（Beta）|

### 5.4 文件浏览器

- 实时跟踪磁盘：创建、重命名、删除、移动都映射为文件系统操作，Agent 的外部改动即时显现；目录优先、名称按**自然序**（9、99、100）。
- **外部拖放**：从 Finder/Explorer 拖入复制；拖图片进 Markdown 编辑器按光标插入；拖文件到 Agent 终端粘贴其路径；SSH worktree 会先上传到远端再完成拖放。
- 按 git 状态着色，右键可丢弃、暂存、重命名、复制路径/相对路径，或把文件本身放进系统剪贴板。
- **远程下载**：SSH 工作区右键文件 → Download，文件夹 → Download Folder（需连接支持递归传输，桌面端专属）。
- **Find in Folder**：右键文件夹或选中后 `Cmd-Shift-F`，直接以该目录为作用域打开搜索。

---

## 6. 评审与交付

### 6.1 Diff 查看器

面向「认真评审 AI 代码」设计：合并 diff 覆盖已暂存/未暂存/未跟踪文件；双侧行号可切换；二进制图片支持并排、滑动、洋葱皮三种对比模式；合并冲突有三路视图与内联解决；可从 Source Control 暂存文件。默认对比 worktree 的 start-from ref，也可切换到任意 commit、分支或 base ref。支持 diff 自动换行、可折叠文件树（宽度记忆）、`F7` / `Shift+F7` 跳转变更。

### 6.2 Annotate AI Diff（AI Diff 批注）

针对 Agent 生成代码的内联评审闭环：

1. 悬停 diff 任意行，点击左槽 **+**（或按 `c`）写评论，支持 Markdown，`Cmd-Enter` 保存。
2. 评论钉在具体行上，diff 变动时会跟随。
3. 评审完点 **Send to agent**，Orca 把所有评论合成**一条**带行锚点的提示，通过 **Send notes to** 菜单选择要改的 Agent（也可从这里新开 Agent）。
4. 批量发送优于逐条发送：一轮思考、一轮修改，命中率更高。
5. 改完评论仍然钉在原处用于验收；**Resolve** 折叠会话，未解决的评论会进入下一批。

### 6.3 Attribution（归属追踪）

Orca 对它看到的每一行 Agent 改动记录来源，读 diff 时可一眼区分「人写的」与「AI 写的」。

### 6.4 提交与推送

- **Commit**：Source Control 面板暂存文件 → 写提交信息（或 **Generate with AI** 从暂存改动起草）→ `Cmd-Enter` 提交；仓库的 pre-commit hook 正常运行，失败时内联显示输出，并可用 **Fix with AI** 把 hook 输出、尝试的提交信息、暂存文件列表交给默认 Agent 修复（只让它修，不让它绕过 hook 或自行提交推送）。
- **Push**：推到 `origin` 并首次设置 upstream；落后时不会静默强推。历史被改写时，面板会单独给出 **Force push with lease**（标注被替换的提交数与上游分支名，使用 `--force-with-lease`）。
- **开托管评审**：推送后从 Source Control 创建 PR/MR，确认 base 分支、标题、描述、draft 状态；**Generate pull request details with AI** 可依据分支 diff 与提交起草标题与描述（生成的文案偏 ELI5 的问题/方案结构，并按关联 issue 给出 `Fixes` / `Refs` 指引）。
- **按仓库的 AI 动作配方**：`Generate with AI`、`Generate pull request details with AI`、`Fix with AI`、`Resolve with AI` 都由 **action recipe** 驱动（指定 Agent、CLI 参数、提示模板），可在 `Settings → Git & Source Control → Action recipes` 配置全局默认或仓库级覆写；模板变量包括 `{basePrompt}`、`{branch}`、`{stagedFiles}`、`{stagedPatch}`、`{linkedIssue}`，PR 场景另有 `{baseBranch}`、`{currentTitle}`、`{currentBody}`、`{commitSummary}`、`{changedFiles}`、`{patch}`。
- **Amend** 需显式触发；已推送的提交需确认才允许 amend。
- **Source Control 面板**：分支上下文行显示 `branch → base`，可用 merge base 时展示增删行数芯片（悬停可看 Source / Tests / Generated 拆分，基于路径启发）；底部主按钮随状态变化（Stage Files → Commit → Push / Pull / Sync）；冲突时提供 **Resolve with AI** 与 **Review conflicts**，以及 **Abort merge / Abort rebase**。

### 6.5 托管评审、Issue 与 CI

**支持的 Provider**：GitHub（最深）、GitLab、Bitbucket Cloud、Azure DevOps、Gitea。在 `Settings → Integrations` 连接。

- **评审**：关联的 PR/MR 在侧边栏显示状态（open / merged / closed），一键在浏览器打开评审页；GitHub 的 checks、reviews、comments 在 Orca 内的 PR 标签中打开；GitLab pipeline 的 **Checks** 面板含 bridge 与 child pipeline 作业，展开可加载作业日志。
- **评论**：可回复线程中的任意评论（不止根评论）；GitHub 评论支持 8 种 reaction；分组评论区按最新优先排序，Timeline 保持最旧优先。
- **Auto-merge**：GitHub 开放 PR 可启用自动合并（合并方式跟随仓库默认：squash / merge commit / rebase）；base 分支启用 merge queue 时显示 **Merge when ready**。
- **Stacked PR**：所选 base 已有开放 PR 时，创建器提供 **Stack this PR above #N**；PR 侧栏显示可折叠的 **Stack #N** 层级图与各层状态；堆叠合并显示 **Merge through #N · M PRs**，原子性失败则全部不合。
- **Issues**：GitHub / GitLab issue 抽屉可浏览、筛选、编辑；从 issue 或 PR 创建 worktree 会走交互式创建器并预填任务名与关联；GitHub issue 详情有 **Activity** 时间线。
- **Actions**：失败的 GitHub Actions 显示红色芯片，点进去看失败作业日志；PR 视图的 **Fix broken checks** 可把失败 check 名与链接交给 Agent。
- **Tasks**：侧边栏的 Tasks 提供完整的 GitHub Projects 视图，可跨仓库浏览卡片、按来源仓库筛选、查看 draft PR 状态，并从任意卡片创建 worktree。

### 6.6 Linear

- `Settings → Integrations → Linear` 粘贴个人 API token 并选择团队。
- 任务抽屉把 GitHub 与 Linear issue 合并展示；**Has Workspace** 模式只列已关联本地工作区的 issue。
- 从 Linear issue 创建工作区会使用 Linear 自己的分支名；issue 详情菜单可复制建议分支名；创建后还能在 **Edit Worktree Details** 的 Issue 字段改绑（GitHub / Linear 共用一个字段）。
- 可在抽屉内更新状态、负责人、优先级、标签、估点（优先级用 Linear 原生图标）。
- 从 Linear issue 启动 Agent 时，issue 描述、评论、子 issue 中的**内嵌图片与媒体**会一并进入提示上下文，无需手动粘截图。
- 列表/看板、分组、排序、可见列、属性筛选跨重启保留（属性筛选按 Linear workspace 存储）。
- Agent 可通过 `orca linear`（及 `orca-linear` skill）读写 Linear，含 MCP 兼容的创建/更新与列表过滤。

### 6.7 Jira

支持 **Jira Cloud**（站点 URL + Atlassian 邮箱 + API token）与**自托管 Server / Data Center**（PAT 或用户名密码）。可连接多个站点并用 **All sites** 合并。能在抽屉内查看描述、评论、元数据，内联编辑状态（按可用 transition）、优先级、负责人与自定义字段，添加评论；从 issue 创建工作区会预填并关联；创建工作区时也可直接粘贴 `…/browse/ABC-123` URL 或切到 Jira 搜索模式选取。凭据经 OS keychain 加密存于本地。

---

## 7. 内置浏览器与 Design Mode

### 7.1 每工作区浏览器

每个 worktree 拥有自己的 Chromium 浏览器面板（地址栏、历史、devtools）。

- 地址栏支持历史与模糊补全，非 URL 文本用默认搜索引擎搜索；前进/后退/刷新/停止，右键或长按刷新可选 **Hard Reload**（绕过缓存，适合前端资源迭代）。
- `Cmd-F` 页内查找、`Cmd-T` 新建（限定当前 worktree）、`Cmd-Shift-T` 恢复最近关闭的标签。
- **作用域**：标签按 worktree 过滤，切回时恢复该 worktree 的标签与滚动位置。
- **远程工作区**：配对 Remote Orca Server 的工作区，新页面默认在本机渲染而 HTTP(S)/WebSocket/DNS/loopback 流量仍走远端；可在设置中切成 **Server (streamed)**。SSH 工作区可选择浏览流量是否走该工作区的 SSH 主机。
- **链接路由**：终端、Markdown、编辑器里的 http(s) 链接默认在 Orca 浏览器还是系统浏览器打开可配置，并有「按住 Shift 反转一次」的开关。
- **下载**：工具栏下方有下载栏，可取消、打开、在文件夹中显示、关闭行。
- **Share as artifact**：本地 HTML 文件可发布为公开查看链接（相对资源不会上传）。
- **视口尺寸模拟**：基于 CDP 设备模拟，页面自身的 `window.innerWidth` 与媒体查询会看到模拟尺寸。
- **可被 Agent 脚本化**：`orca snapshot`、`orca click`、`orca fill` 等驱动的是同一个浏览器与同一批标签。

### 7.2 Design Mode

把浏览器变成「指哪打哪」的工具：打开开关后光标变为拾取器，点击页面上任意元素，Orca 会把以下内容作为一个附件送进当前 Agent 终端：

- 元素的 HTML（自身与一小片邻域）；
- 计算后的 CSS（颜色、字体、间距）；
- 该元素的裁剪截图；
- 如果有 dev-mode source map，还包括源文件与行号。

随后你只需描述想改成什么。Agent 改源码 → Orca 热重载 → 再点一次验证。

### 7.3 浏览器身份与 Profile

- **Profile**：为浏览器指定身份（登录态、cookie jar），可注入 cookie 与视口尺寸；每个 profile 有独立存储分区（cookie、localStorage、缓存互不泄漏）；Agent 驱动的浏览器命令继承当前 profile。
- **浏览器身份**：应用级选择 **Cleaned**（去掉 Orca/Electron 标识，保持 Chrome 形态，Google 登录走受限 Firefox 身份）或 **Native**（保留 Electron 身份，兼容部分 Cloudflare 站点但无法 Google 登录），切换需重启。
- **Cookie 导入**：可从 Chrome / Edge 或 cookie 文件导入，只替换导入涉及域名的 cookie；Google cookie 不导入，需在 Orca 内直接登录。外置 FIDO 安全密钥提供多账号 passkey 时会弹出账号选择器（尚不支持系统平台 passkey）。

---

## 8. 运行位置：四种模式

| 模式 | 文件与 Agent 在哪 | 机器归属 | 适合 |
| --- | --- | --- | --- |
| **本地桌面** | 你的电脑 | 你 | 日常编码、快速迭代 |
| **SSH 目标** | 通过 SSH 连接的远程主机 | 你 / 团队 | 开发机、GPU 机、常开 VPS |
| **Remote Orca Server** | 跑着 Orca 桌面端或 `orca serve` 的机器 | 你 / 团队 | 常驻共享运行时、移动端、自动化 |
| **Cloud VM / 按工作区环境** | 每个工作区一台一次性 VM/沙箱 | 你的云账号（自带 Provider）| 隔离、用完即弃的 Agent 算力 |

> Orca 不售卖托管 VPS。远程模式用的始终是你自己控制的机器与云账号。四种模式可在同一安装内混用。

### 8.1 SSH 工作区

- `Settings → SSH → Add Target` 填主机/用户/端口/身份文件，或从 `~/.ssh/config`（含 `Include`）选取预填；可 **Test** 验证。
- **主机密钥校验**：对照 OpenSSH `known_hosts` 与已保存记录；默认策略首次接触时接受并记录指纹并展示；密钥变更/吊销会在询问密码前直接拒绝，并给出对应的 `ssh-keygen -R` 恢复命令。
- 创建 worktree 时在 **Run on** 选 SSH 目标：git worktree 建在远端、Agent 在远端跑，而编辑器、diff、浏览器仍是本地手感。
- **高级连接**：代理 / 跳板机；默认开启 **SSH 连接复用**（OpenSSH multiplexing）以加速 setup。
- **状态**：工作区卡片显示绿/黄/红连接芯片；断连不会杀掉运行中的 Agent，重连后重新附着并重放输出、按渲染帧恢复全屏 TUI 面板；卡片上可直接内联重连。
- **跨应用关闭存活**：远程 PTY 通过远端 relay 租约持有，关闭桌面端不杀会话；重开后按 **attached** 状态还原标签与回滚。默认 **Keep terminals alive until reset** 常开（直到显式结束），也可为每个目标设置 60 秒~7 天的断连超时。
- **远程文件下载**、**VS Code Remote-SSH 打开**（仅 VS Code / Insiders）、**Kerberos / GSSAPI**（自动改用系统 OpenSSH）、**FIDO2 安全密钥**（同样走系统 OpenSSH）。
- **端口转发**：远程工作区右侧栏 **Ports** 标签（`Cmd+Shift+I`），扫描远端 `/proc/net/tcp` 列出监听端口一键转发，也可手工增删改；转发跨重启与重连保留，特权端口自动改映射（远端 80 → 本地 10080）。
- Linux 远端若缺少 make / C++ 编译器 / python3，文件、git、编辑器仍可用，但远程终端需要先装构建工具。

### 8.2 Remote Orca Server

一台机器干活、另一台提供 UI。**服务端**持有项目、worktree、终端、标签、Provider 账号与 Agent 会话；客户端只是 UI。

- 推荐路径：两端都装 Orca + Tailscale，在服务端 `Settings → Remote Orca Servers → Advertise this app as a server → New Link` 生成可吊销的访问链接，客户端 **Add Server** 粘贴即可。
- 无头 Linux 服务器用 `orca serve`（Linux 上命令名为 `orca-ide`）：`orca-ide serve --pairing-address <可达地址>`。
- 服务端需要自行安装并登录 Codex、Claude Code、OpenCode、`git` 等 CLI；无头主机用 `orca account add --agent claude|codex`、`orca skills install/update` 管理账号与技能。
- 与 SSH 的区别：SSH 的运行时属主是你的笔记本，Remote Server 的完整会话状态在服务端，且可被笔记本、Web、移动端、自动化**多客户端共享**。

### 8.3 Cloud VM / 按工作区环境（实验特性）

每个 worktree 可以从仓库内签入的 **recipe**（`orca.yaml` + 生命周期脚本）启动自己的按需环境——云沙箱、VM 或本地 Docker。创建时拉起，suspend / resume / destroy 负责回收。Orca 只是薄包装：Provider 账号、镜像与账单都归你。

- 已有人接的 Provider：Vercel Sandbox、Fly、Modal、纯 SSH 主机、本地 Docker。
- 连接方式二选一：**Orca server**（recipe 启动 `orca serve` 并返回配对 URL）或 **SSH**（recipe 返回连接信息由 Orca 拨号）。
- 落地步骤：`Settings → Experimental` 开启 **Cloud VM** → 安装 `orca-per-workspace-env` skill → 让 Agent 按 skill 配置 recipe（前置条件 → 基础快照 → Agent 鉴权 → `orca.yaml` → doctor 校验）→ recipe 出现在 **Recipes** 后即可在创建工作区时于 **Run on** 选择。
- 注意：只有当 `environmentRecipes` 位于项目**主检出**的 `orca.yaml` 上时才会出现在创建流程中（feature 分支上可以先跑 doctor 与实时预配）。

---

## 9. CLI 与自动化

### 9.1 Orca CLI

`orca` 命令随桌面端一起分发，在 `Settings → General → Orca CLI` 注册；**Linux 上命令名是 `orca-ide`**（`/usr/bin/orca` 被 GNOME Orca 读屏软件占用）。它面向 shell 脚本与 AI Agent，用来驱动**正在运行的 Orca**。

```bash
command -v orca && orca status --json

# 工作区
orca worktree ps --json
orca worktree create --repo id:<repoId> --name my-task --issue 123 --json
orca worktree current --json
orca worktree set --worktree active --comment "reproduced bug" --json
orca worktree rm --worktree id:<id> --force --json

# 终端
orca terminal list --json
orca terminal read --json
orca terminal send --text "continue" --enter --json
orca terminal wait --for tui-idle --timeout-ms 30000 --json
orca terminal create --worktree path:/projects/app --command "npm test" --json
orca terminal split --direction vertical --command "npm run dev" --json

# 文件与 diff
orca file open src/App.tsx
orca file diff src/App.tsx --staged
orca file open-changed --mode both

# 浏览器自动化（snapshot → 操作 → 再 snapshot）
orca goto --url https://example.com --json
orca snapshot --json          # 返回 @e1、@e3 之类的引用
orca click --element @e3 --json
orca fill --element @e1 --value "user@example.com" --json
orca screenshot --json
orca set device --name "iPhone 12" --json

# iOS 模拟器桥接
orca emulator list|attach|tap|type|gesture|rotate|kill

# 其它
orca tab profile list|create|set|clone|use-default
orca automations create|list|run|rm
orca artifacts share|update|list|delete
orca linear …
orca account add|list
orca skills install|update
```

### 9.2 编排（Orchestration，实验特性）

结构化的多 Agent 层：**Run**（持久命名空间 + 协调者收件箱）、**Task**（`pending / ready / dispatched / completed / failed / blocked`）、**Dispatch**（任务的一次执行尝试，持有 `worker_done` 与心跳的生命周期权威）、**Message**（收件箱邮件）、**Decision gate**（协调者持有、阻塞任务直到被裁决的问题）。

典型监督循环：

```bash
orca orchestration run-create --objective "…" --json
orca orchestration task-create --spec "…" --task-title "…" --json
orca orchestration worker-start --task <taskId> --worktree new-child --name billing-audit --agent codex --setup run --json
orca orchestration check --wait --types worker_done,escalation,question --timeout-ms 900000 --json
```

- 每个 worker 完成时必须**恰好一次**发送 `worker_done`（带 `--outcome succeeded|failed`、`taskId`、`dispatchId`、简短总结），长任务期间发 `heartbeat`，阻塞性问题用 `orca orchestration ask` 而不是本地 TUI 提问。
- 支持逐 worker 的 `--model` / `--effort` 覆写（仅 Claude、Codex、Cursor，且不能与 `--terminal` 同用）。
- 支持**联邦 worker**（`--on <host>` 在其它主机起 worker，之后按 Dispatch ID 路由）。
- 群组地址：`@all`、`@idle`、`@claude`、`@codex`、`@opencode`、`@gemini`、`@droid`、`@grok`、`@cursor`、`@worktree:<id>`（`worker_done` 与心跳不可用群组地址）。
- 恢复与回收：`worker-show` / `worker-read` / `worker-stop` / `worker-release` / `worker-retain`，`dispatch-show`、`task-list`、`task-update`、`gate-create` / `gate-resolve`，以及有意放弃状态时的 `reset`。
- 终端里打印的 `task_...` ID 是可点链接，点击会定位到该任务当前 Dispatch 所在终端（远程/SSH 运行时同样适用）。
- 何时用什么：轻量提示用 `orca terminal send`；需要完成追踪与提问回传用 `worker-start` / `dispatch --inject`；需要持久命名空间与监督循环用 `run-create` + task + worker。

### 9.3 定时自动化

`orca automations` 让重复性的巡检、评审、维护任务按计划自动跑，无需手动开工作区。

```bash
orca automations create \
  --name "Weekday triage" --trigger weekdays --time 09:00 \
  --prompt "Triage new issues and summarize blockers" \
  --provider codex --repo my-repo --disabled --json
```

- `--trigger` 支持 `hourly` / `daily` / `weekdays` / `weekly` 预设，也支持 cron 表达式与 RRULE；`--timezone` 指定 IANA 时区。
- `--repo` 让每次运行在仓库内创建/选择工作；`--workspace` 则在已有 worktree 内运行；都不给时从当前 shell 目录推断。
- 支持运行前 **precheck**（shell 探针非零退出则记为跳过）、项目主机/setup 目标、跨主机管理、错过运行的宽限窗口、复用已有会话、按需手动运行；建议先用 `--disabled` 调好再启用。

### 9.4 Computer Use（桌面操控）

让 Agent 通过无障碍树、截图与安全的 UI 动作驱动本机桌面应用，工作模式是 **snapshot → act → snapshot**，支持多窗口应用、选择目标应用、敏感输入处理与截图。适用于必须真实交互才能完成的流程。

### 9.5 Skills 与 MCP

可安装的 Orca 技能：`orca-cli`、`orchestration`、`computer-use`、`orca-linear`、`orca-emulator`、`orca-emulator-android`、`orca-per-workspace-env`（另有 `linear-tickets`）。

```bash
npx skills add https://github.com/stablyai/orca --skill orca-cli
# 无 Settings UI 的无头环境：
orca skills install --skill orca-cli
orca skills update --all
```

采用「**混合桩**」设计：本地桩保持轻薄，`orca skills get <skill> --full` 取回与当前版本匹配的完整指南，避免 Agent 拿到过期命令。技能可在后台更新（进度显示在状态栏），也支持在主机之间共享私有技能。同时支持配置 **MCP servers**。

### 9.6 Artifacts（公开分享链接）

通过已登录的 Orca 账号把 HTML 或 Markdown 发布成公开查看链接：`orca artifacts share|update|list|delete`，或在 Markdown 编辑器/浏览器工具栏用 **Share as artifact**。发布是**默认关闭**的设备级开关（`Settings → Artifacts → Allow publishing public artifact links`），可在侧边栏 **Artifacts** 页搜索、预览、复制、删除，并可要求删除前确认。

---

## 10. 移动伴侣应用（Beta）

iOS / Android 应用与桌面端配对，提供以读为主的 Agent 遥控能力；**桌面端始终是事实来源**。

**能做什么**

- 查看所有主机（本地桌面 + 远程 Orca 服务器）上的全部工作区及其 Agent 状态（工作中 / 完成 / 等待输入）。
- 浏览工作区完整文件树；拉取最近终端回滚查看 Agent 做了什么、问了什么（聊天视图可渲染 Mermaid）。
- 在 **Chat UI** 或原始终端间切换（设备默认值 + 单标签长按覆写）；聊天输入框支持 `@` 提及文件、按 Agent 过滤的 `/` 斜杠命令、图片附件、语音听写、模型与会话选项药丸。
- 终端视图支持长按选择复制粘贴、辅助键行（含 `Tab` / `Shift+Tab`）、**Live** 模式逐字符直通。
- Agent 等待输入时发送简短回复（`continue`、`yes`、自由文本），附照片或文件，或用麦克风口述。
- 从会话标签条运行 **Quick Commands**（与桌面同一份列表，双向同步）。
- 打开浏览器会话（**Web / Mobile** 视图切换）。
- 打开 Source Control：查看改动文件、暂存/取消暂存、提交；已有 PR 时可 **Link an existing PR**。
- 切换活跃 Agent 账号、查看用量与限流重置倒计时；Codex 攒到重置额度时可消费一次 **Use reset**。
- 创建工作区（Smart / GitHub / Linear / GitLab / Branch / Name 六种来源模式，多桌面时先选主机）。
- 编辑已配对主机的显示名或连接地址（无需重新配对，适合家庭 LAN 与 Tailscale 之间切换）。
- 接收 Agent 完成的推送通知。

**配对**：桌面端生成一次性配对码 → 手机粘贴（或走深链）。优先使用 **Orca Relay**（仅 Relay 需要登录），LAN 配对需要填地址。配对交换会为该手机建立设备令牌。

**终端设置**：文字大小 50%~200%（捏合缩放吸附到同一批档位，按设备保存）；自动补全/自动纠错默认关闭，避免系统改写命令、参数与路径。

移动端刻意不做成完整编辑器——它是你那台已经在跑的桌面的遥控器。

---

## 11. 通知与 Agents 动态

### 通知与收件箱

- **Agent 完成提醒**：Agent 从 working 转为 idle 时触发系统通知 + 声音 + 工作区芯片。
- **常驻铃铛**：头部铃铛汇总所有工作区未读，点击跳到对应工作区与面板；macOS 会同步到 Dock 角标。
- **标记未读**：右键通知标记未读，便于稍后回头处理。
- **分类开关**：系统通知 / 声音 / 仅芯片、PR 检查失败、有更新可用，均可单独关闭。
- **自定义声音**：按类别指定磁盘上的音频文件或内置音效（MP3、WAV、OGG、M4A、AAC、FLAC），并可设置播放音量。

### Agents 动态（Activity Feed）

侧边栏 **Agents** 入口，跨工作区的线程式事件流：Agent 完成一轮（idle 或被问题阻塞）、新工作区创建、等待输入过久而升级为阻塞、以及 Agent 最近回复的简短预览。按状态分组，运行中的置顶；有新事件时显示未读角标；点击条目跳转到对应工作区与面板。`Cmd+F` 聚焦过滤框（终端有焦点时该快捷键归终端）。它不替代铃铛和系统通知，而是离开一段时间后的「补课界面」。

---

## 12. 设置总览

`Cmd-,` 打开并可直接搜索。主要分组：

| 分组 | 要点 |
| --- | --- |
| **General** | 注册 Orca CLI；更新（Shift 点击含 RC 预发布，Cmd/Ctrl 点击 perf 预发布，macOS Option 点击选本地校验版本）；**Open in** 菜单的外部编辑器；UI 缩放；默认新工作区命名；编辑器自动换行 |
| **Appearance** | 主题、强调色、密度；UI 字体（编辑器字体可选覆写）；minimap；状态栏项（含 Resource Manager）；用量显示为已用/剩余；App 图标；**界面语言**（System / English / 简体中文 / 한국어 / 日本語 / Español）|
| **Git** | 默认 base ref 解析；提交签名；外部 git 工具编辑器；从工作内容自动重命名分支；**GitHub API Budget**（REST / Search / GraphQL 余量）|
| **Terminal** | 字体、主题、光标、内边距；Ghostty 导入；Warp 主题导入；JIS ¥ 转反斜杠；Windows 默认 shell；OSC 52 剪贴板开关 |
| **Quick Commands** | 保存的命令与 Agent 提示词预设、作用域过滤、多主机归属 |
| **Agents** | 已检测到的 CLI 启停；Agent 权限（Yolo / Manual）；Claude 与 Codex 账号列表；启动 hooks；Agent 状态 hooks；**Keep computer awake**（On / Agent / Off，状态栏 Caffeinate）；技能新鲜度与后台更新 |
| **Browser** | Profile；默认缩放；Design Mode 默认值；devtools；远程工作区渲染位置；是否经 SSH 主机浏览；链接路由；终端链接动作气泡；默认搜索引擎 |
| **Artifacts** | Orca 账号登录；公开链接发布总开关（默认关）；Artifacts 列表；删除前确认 |
| **Integrations** | GitHub OAuth、Linear token、Jira、Bitbucket Cloud、MiniMax（用量追踪用的会话 cookie）、MCP servers |
| **Notifications** | Agent 完成（系统/声音/芯片）、自定义声音、PR 检查失败、更新提醒 |
| **Voice** | 语音听写开关与麦克风选择；Toggle / Hold 两种模式；本地模型（Parakeet TDT v3/v2、Zipformer 系列含中英与韩语、Paraformer 中英、Parakeet TDT-CTC JA、SenseVoice 多语种自动识别、Whisper Tiny）或云端 OpenAI 模型（GPT-4o mini / GPT-4o Transcribe，需填 API key）|
| **SSH** | 目标、口令、默认身份文件；代理/跳板机；连接复用；Kerberos |
| **Remote Orca Servers** | 配对与连接远程运行时；把本机广播为服务器并生成可吊销访问链接；默认运行时选择 |
| **Shortcuts** | 全量键位表，全部可改键 |
| **Repository** | 按仓库的 base ref 与 hooks；工作区创建后自动运行的命令；仓库图标（内置图标 / emoji / 上传图片 / 网站 favicon / GitHub 头像 + 徽章颜色）；Source Control AI 覆写；**Worktree Shared Paths** |
| **Floating Workspace** | 浮动工作区开关、终端起始目录、切换按钮位置 |
| **Plugins（实验）** | 插件系统总开关 + 逐个授权；git 插件市场（面板、命令、语言包、VM recipe），可安装/更新/回滚；插件 worker 只在本机运行；第三方插件按不可信软件对待 |
| **Experimental** | Activity Page、紧凑工作区卡片、Agent 休眠、Agent Dashboard、Chat UI、Cloud VM |

---

## 13. 隐私与遥测

- **匿名**：事件以本机随机 ID 标识，不收集账号、邮箱、IP、用户名。
- **不含内容**：从不上传文件内容、提示词、Agent 输出、终端输出、仓库名、分支名、URL、路径、提交信息。
- **采集范围**：生命周期（应用打开）、仓库与工作区的**创建方式**（而非名字/路径）、启动了哪一类 Agent 及从何处启动、粗粒度错误类别、白名单内的设置开关变更、隐私开关本身的变化。随事件附带 Orca 版本、操作系统、CPU 架构、粗粒度系统版本、发布通道与匿名 ID。
- **三种关闭方式**（任一即可，可叠加）：应用内 `Settings → Privacy → Share anonymous usage data` 关闭；环境变量 `DO_NOT_TRACK=1`；环境变量 `ORCA_TELEMETRY_DISABLED=1`。
- **去向**：PostHog Cloud（美国区），保留期用其套餐默认值，访问权限限于少数维护者。

---

## 14. 安装与平台

### 桌面端

- 官网下载：<https://onorca.dev/download>；或直接取 macOS Apple Silicon / Intel `.dmg`、Windows `.exe`、Linux AppImage。
- 包管理器：
  ```bash
  # macOS
  brew install --cask stablyai/orca/orca
  # Arch Linux（AUR），或 stably-orca-git 从源码构建
  yay -S stably-orca-bin
  ```
- 无头 Linux 服务器上跑 `orca serve` 参见 `docs/reference/headless-linux-server.md`。
- **Linux 命令名为 `orca-ide`**，以避让 GNOME Orca 读屏软件。

### 移动端

- iOS：App Store 或 TestFlight 预览通道。
- Android：GitHub Releases 的 APK。

### 跨平台注意事项

Orca 同时面向 macOS、Linux 与 Windows，平台相关行为都在运行时判断：快捷键（Mac 的 `⌘` vs 其它平台的 `Ctrl`）、路径拼接、Windows 终端 shell 选择、Windows 子进程启动、WSL 命令构造、Linux 原生模块的 glibc 下限等，详见 `AGENTS.md` 与 `docs/reference/` 下的对应文档。

---

## 15. 快速上手路线

1. **安装**并首次启动，注册 Orca CLI。
2. **添加仓库**（文件夹选择或 clone URL）。
3. **创建工作区**：输入任务名、选 start-from、选 Run on（本地 / SSH / 服务器 / recipe）、选 Agent，可顺手关联 GitHub / Linear / Jira 条目。
4. **让三个 Agent 跑同一个任务**：在同一 worktree 或三个 worktree 分别启动不同 Agent。
5. **拖拽标签分栏**，同屏盯着它们跑。
6. **挑出赢家**：看 diff、逐行批注并批量回传给 Agent 修改，满意后提交、推送、开 PR、等 CI，全程不离开 Orca。

对应的完整教程在官方文档 `first-session` 与 `recipes/` 下的五个配方（并行 Agent、逐行评审 AI diff、跨 10 个 worktree 跳转、用 Design Mode 修 UI bug、通过 SSH 在远程机器工作）。

---

## 附：文档与资源

| 资源 | 位置 |
| --- | --- |
| 官方文档站 | <https://www.onorca.dev/docs> |
| 文档源码 | `docs/site/content/docs/` |
| 中文 README | `docs/readme/README.zh-CN.md` |
| 设计规范 | `docs/STYLEGUIDE.md` |
| 平台专项参考 | `docs/reference/` |
| 变更日志（真正的功能清单）| <https://github.com/stablyai/orca/releases> |
| 社区 | Discord / X `@orca_build` / 微信群 |
