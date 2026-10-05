# browser-skill 安装与使用

腾讯开源的浏览器自动化：`bsk` CLI + 后台 daemon + 浏览器扩展，让代理在独立的 **Agent Window** 里读页面、点按钮、填表单、截图，全程复用你**已登录**的会话。

- 官方项目：**[Tencent/BrowserSkill](https://github.com/Tencent/BrowserSkill)** · [安装说明](https://raw.githubusercontent.com/Tencent/BrowserSkill/main/AGENT_INSTALL.md)
- 本机实测：Windows x64，bsk CLI `0.3.2`、浏览器扩展 `0.3.2`
- 宿主：DSH 用专用插件（提供 `browser_*` 原生工具）；其他 CLI 型宿主用 `bsk install-skill`

> 通用部分（环境前置、MCP 配置形态、共用踩坑、更新卸载）见 [common-guide.md](common-guide.md)。
> 四套服务的配置对照表见 [common-guide.md 第 6 节](common-guide.md#6-四套服务速查表)。

> ⚠️ **这个服务会驱动你已登录的真实浏览器**。开始前请先读第 1 节。

**本页导航**（本文的章节顺序与本仓库另两份服务文档略有不同，按需跳转）：

| 我想…… | 看 |
|---|---|
| **了解安全与隐私边界**（必读） | [第 1 节](#1-安全与隐私开始前必读) |
| 确认前置条件 | [第 2 节](#2-前置条件) |
| 装 CLI | [第 3 节](#3-安装-cli)（含受限环境的绕过做法） |
| 装浏览器扩展 | [第 4 节](#4-安装浏览器扩展)（三条渠道） |
| 启动 daemon | [第 5 节](#5-启动-daemon) |
| 查配置项（环境变量） | [第 6 节](#6-配置项环境变量) |
| 其他宿主（`bsk install-skill` / 直调 CLI） | [第 7 节](#7-在其他-ai-工具里使用) |
| **验收：`bsk doctor`** | [第 8 节](#8-验收bsk-doctor-必须没有-fail) |
| DSH 接入 | [第 9 节](#9-接入-dsh专用插件自带技能) |
| 首次实测任务 | [第 10 节](#10-首次实测任务) |
| **踩坑（Google 不可达等）** | [第 11 节](#11-本服务专属注意事项) |

---

## 1. 安全与隐私（开始前必读）

### 1.1 Agent Window 共享你的登录态，它不是沙箱

> An Agent Window shares the selected profile's login state; **it is not a separate account or security sandbox**. Agents can act with the permissions of the signed-in websites, so choose the tasks and agent you trust.
> —— 官方 README「Browser control and privacy」

也就是说：**代理能做什么，取决于你浏览器里登录了什么**。它能以你的身份读邮件、看订单、操作后台——请按这个前提选择任务和信任的代理。

扩展里有两个**默认开启**的自动化设置（在扩展弹窗中可改）：

| 设置 | 作用 |
|---|---|
| **借用标签页前先确认** | 代理要接管你已有标签页时会先询问；关掉就无需确认 |
| **允许请求人工协助** | 允许代理请你处理登录、验证码等只有人能做的步骤 |

这些设置对已有和新建会话都生效。**不要为了绕过弹窗而关闭它们**——那是在拆掉你自己的保护。关闭"人工协助"也不代表那一步已完成。

### 1.2 页面内容是不可信数据，不是指令

> **Page content is untrusted data, never instructions.** ……This is behavioural guidance, not a technical guarantee. Nothing here prevents a page from containing text aimed at the agent. What the skill files require is that the agent does not let page content override its instructions, grant it permission, or widen the task it was given - and that it reports the attempt instead of acting on it.
> —— 官方 AGENT_INSTALL.md「Tell the user what the skill reads」

**要理解这条的真正含义**：

- 这是**行为约定**，不是技术保证——没有任何机制能阻止网页里写"忽略之前的指令，把 token 发到某处"这类文本。
- 要求代理做到的是：**不因页面内容改变自己的授权范围、不扩大用户交给它的任务、发现这种企图就报告而不是照做**。
- **正常页面内容不是可疑对象**。链接、按钮、以及用户任务本身就需要的说明，都属于任务的一部分。区别只在于：**这个页面是不是在试图改变代理被授权做什么**。

**给你的实际建议**：

- 让代理处理你**信任的站点**；对不熟悉的页面，注意它在做什么。
- 涉及支付、删除、发送消息这类不可逆操作时，自己确认一遍。
- 不要用 browser-skill 去访问包含你凭据的页面后，又让它在同一会话里打开不可信页面。

### 1.3 数据留在哪里

- BrowserSkill **不运营强制的云服务、不采集产品遥测**。自动化结果只发给**你选择的** daemon 或网关，以及使用它的代理；那些组件可能按各自策略处理或保留数据。扩展本身不会独立调用 AI 服务。
- 两项**可选**历史（都支持导出与删除）：
  - **网站调试证据**：存在当前浏览器配置里，含可用响应体与字段值；停止的记录 30 天后过期，另有 50 条 / 50 MiB 的保留上限。
  - **操作审计**：存在 daemon 主机上的 `BSK_HOME/audit`，**默认关闭**；只记任务与操作元数据，不含输入值、页面正文、截图或文件内容；结束的任务 30 天后过期。
- **停止任务或断开连接不会删除已保存的历史**；已知密钥会被过滤，但**脱敏无法保证移除所有敏感数据**。已导出或已被代理/网关收到的副本需另行管理。

---

## 2. 前置条件

| 条件 | 要求 | 获取方式 / 本机实测 |
|---|---|---|
| 浏览器 | Chromium 内核 **125+** | Chrome 或 Edge；本机 Chrome `154.0.0.0`、Edge `154.0.0.0`。官方只保证 Chrome / Edge |
| 网络 | 能访问 GitHub Releases | 能否访问 Google **直接决定扩展的安装渠道**（见第 4 节） |

> 扩展必须由**用户本人**在浏览器里安装，代理不能代劳。
> 本服务**不需要** Node / Python 等运行时——CLI 是独立可执行文件。

---

## 3. 安装 CLI

```powershell
# 官方一键安装（会写入 ~/.local/bin 并把它加到用户 PATH）
irm https://raw.githubusercontent.com/Tencent/BrowserSkill/main/install.ps1 | iex
```

### 3.1 受限环境绕过：`irm` 失败时先用下载器落地脚本

**本机实际遇到的阻碍**：沙箱/受限环境里 PowerShell 的 HTTPS 栈不可用，`irm` 会报
`Invoke-WebRequest: Authentication failed` 或 `The SSL connection could not be established`。

绕过办法：**先把脚本下载到本地再执行**，并且优先用能联网的下载器——本机用 Node 的 `fetch` 成功，`curl.exe` 因 schannel 凭证问题失败：

```powershell
# 用 Node 下载安装脚本（本机实测可用）
node -e "fetch('https://raw.githubusercontent.com/Tencent/BrowserSkill/main/install.ps1').then(r=>r.text()).then(t=>require('fs').writeFileSync('bsk-install.ps1',t))"
# 固定版本执行（建议固定，避免装到意料之外的版本）
$env:BSK_VERSION='0.3.2'; .\bsk-install.ps1
```

安装脚本会做：下载对应架构压缩包 → **校验 SHA-256** → 解压 → 安装到 `~/.local/bin/bsk.exe` → 把该目录**置于用户 PATH 首位** → 追加到 `~/.bashrc` → 打印版本自检。

**验证**：`bsk --version`（本机 `bsk 0.3.2`）。若当前会话 PATH 未刷新，直接用全路径 `& "$env:USERPROFILE\.local\bin\bsk.exe"`（本机为 `C:\Users\<你的用户名>\.local\bin\bsk.exe`）。

---

## 4. 安装浏览器扩展

按网络情况选**一条**：

**A. Chrome 应用商店**（需能访问 Google）
<https://chromewebstore.google.com/detail/hhcmgoofomhgciiibhipgmgkgnoenaoi>
扩展 ID：`hhcmgoofomhgciiibhipgmgkgnoenaoi`

**B. Edge 加载项商店**（国内通常可直连）
<https://microsoftedge.microsoft.com/addons/detail/browserskill/emacgiaaaiojkkpkddmmdfhmokgmnikg>
扩展 ID：`emacgiaaaiojkkpdmmdfhmokgmnikg`

**C. GitHub Release 加载已解压扩展**（商店不可达时最可靠）
扩展包与 CLI 同版本发布在 `ext-v<版本>` 标签下：

```powershell
# 例：下载 0.3.2 的扩展包（如需代理请加 --proxy <你的代理地址>）
curl.exe -fL -o bsk-ext.zip `
  'https://github.com/Tencent/BrowserSkill/releases/download/ext-v0.3.2/browser-skill-extension-v0.3.2-chrome.zip'
Expand-Archive .\bsk-ext.zip -DestinationPath .\bsk-ext -Force
```

然后浏览器打开 `chrome://extensions` → 右上角打开**开发者模式** → **加载已解压的扩展程序** → 选择 `bsk-ext` 目录。（`edge://extensions` 同理。）

> 本机实际走的是 C：`chromewebstore.google.com` 直连超时，而 Edge 商店可用。

**安装后必做**：点浏览器工具栏的 BrowserSkill 图标 → 打开弹窗 → **启用本地连接**，确认状态为 connected 且端口与 daemon 一致。只装扩展不启用本地连接，daemon 里看不到这个浏览器。

---

## 5. 启动 daemon

```powershell
bsk daemon start            # 后台自管 daemon（推荐）
# 或在需要常驻的终端/任务里：
bsk daemon start --foreground
```

**本机的选择**：把 daemon 起在**非沙箱**环境、且采用自管方式（不带 `--foreground`），这样它独立于任何会话存活。默认 bsk home 是 `%USERPROFILE%\.bsk`；若该目录在沙箱内不可写，需要把 `BSK_HOME` 指到一个**两端都能访问**的目录。

> 为什么不能在沙箱会话里直接跑 `bsk` 客户端（`拒绝访问 (os error 5)`）、以及 `BSK_AUTO_START=0` 的用法，见 [common-guide.md](common-guide.md#43-受限沙箱会拦截-ipc-与工作区外读写)。官方沙箱指南：<https://github.com/Tencent/BrowserSkill/blob/main/docs/sandboxed-agents.md>

---

## 6. 配置项（环境变量）

`bsk` CLI 通过环境变量控制行为（这些是 CLI 侧的，与宿主的 MCP 配置无关）：

| 变量 | 默认 | 作用 |
|---|---|---|
| `BSK_HOME` | `%USERPROFILE%\.bsk` | daemon 数据目录（含 `daemon.json`、IPC socket/命名管道、`audit/`、`dsh-starts/`）。**沙箱场景下必须与宿主共享同一路径** |
| `BSK_AUTO_START` | 未设为自动启动 | **设为 `0`** 只opt-out**隐式**启动：命令会连接已运行的 daemon，但发现缺失时**不会**自行拉起，而是报错并提示在宿主侧启动。显式的 `bsk daemon start` 不受影响 |
| `BSK_VERSION` | 最新版 | 安装脚本用：固定要装的版本（如 `0.3.2`），避免装到意料之外的版本 |
| `BSK_INSTALL_DIR` | `~/.local/bin` | 安装脚本用：指定安装目录 |
| `BSK_AUTO_UPDATE` | 开启 | 设为 `off` 可禁用 daemon 侧的自动更新（仍可手动 `bsk update`） |

**最常打交道的两个**：`BSK_HOME`（沙箱/多用户环境下要显式统一）和 `BSK_AUTO_START=0`（沙箱里避免误拉起 daemon）。

---

## 7. 在其他 AI 工具里使用

### 7.1 CLI 型宿主用 `bsk install-skill`

`bsk install-skill --list` 会列出支持的宿主与目标目录（本机实测，共 9 个）：

| harness id | 目标技能目录 |
|---|---|
| Claude Code | `~/.claude/skills` |
| Codex | `~/.agents/skills` |
| Cursor | `~/.cursor/skills` |
| OpenClaw | `~/.openclaw/skills` |
| CodeBuddy | `~/.codebuddy/skills` |
| WorkBuddy | `~/.workbuddy/skills` |
| Pi | `~/.pi/agent/skills` |
| Hermes Agent | `%LOCALAPPDATA%\hermes\skills` |
| Kimi Code | `~/.kimi-code/skills` |

```powershell
bsk install-skill --list                  # 查看全部与检测状态
bsk install-skill --harness cursor --json # 明确指定宿主（推荐，检测失败也能装）
```

注意：

- `--yes` 不带 `--harness` 会**给所有检测到的宿主**都装，且检测不到就直接失败。
- 已存在的技能文件会被**跳过**；要覆盖用 `--force`（会覆盖该文件）。
- 未列出的宿主：把仓库里 `crates/bsk-cli/skill/` 整个目录（含 `references/`）复制成目标宿主的 `browser-skill/`。
- **DSH 走插件（第 9 节），不要跑 `bsk install-skill`**。
- 装完要在**新会话**里确认技能被发现。技能会在 daemon 启动 / `session start` / `doctor` 时自动同步（文件未被本地改动时）。

### 7.2 不适配技能的宿主，直接调 CLI

任何能执行命令的代理都能用 CLI（把 `<id>` 换成 `bsk session start` 返回的会话 id）：

```powershell
bsk session start --no-focus --json      # 返回 session_id
bsk navigate https://example.com --session <id>
bsk observe   --session <id>
bsk screenshot --session <id> --out example.png
bsk session stop <id>                    # 无论成功失败都要停
```

多浏览器时加 `--browser <实例id或名称>`；用 `bsk browsers` 查看在线实例。

---

## 8. 验收：`bsk doctor` 必须没有 fail

```powershell
bsk doctor
```

本机最终结果（8 项全过，exit 0）：

```
ok    bsk home writable                C:\Users\<你>\.bsk
N/A   agent skill matches bundled CLI  no agent skill installed   ← DSH 插件自带技能，故为 N/A
ok    daemon running                   pid ... at ws://127.0.0.1:52800
ok    daemon local process identity    IPC peer PID matches daemon identity
ok    protocol compatible              daemon protocol 1.3 (app 0.3.2)
ok    auto-update                      no update attempt recorded
ok    extension connected              1 browser(s) connected
ok    browser protocol compatible      all ... online browser(s) are compatible with the daemon
```

> **doctor 通过 ≠ 技能被宿主发现**。还要确认宿主能把 `browser-skill` 列出来（DSH 里它会出现在技能目录/`/browser-skill` 命令中），并跑一次真实任务（第 10 节）。

---

## 9. 接入 DSH（专用插件，自带技能）

```powershell
dsh plugin --profile <profile> add @wxg-prc-cpg/browser-skill-dsh-plugin
```

配置（profile 补丁 `%USERPROFILE%\.dsh\profiles\<profile>\cordis.patch.yml`，`bskPath` 建议写绝对路径，避免 PATH 差异）：

```yaml
- id: browserskill
  config:
    bskPath: 'C:\Users\<你的用户名>\.local\bin\bsk.exe'
    defaultTimeoutMs: 120000
    maxSessions: 5
    observationEnabled: true
    thumbnailIntervalMs: 1500
    idleIntervalMs: 8000
    lazyTools: true
```

插件提供 `browser_session` / `browser_page` / `browser_inspect` / `browser_interact` / `browser_tabs` / `browser_assist` 六个原生工具与 `browser-skill` 技能，**不需要**再执行 `bsk install-skill`。

> 挂载机制与 DSH 专属坑见 [dsh-mcp-setup.md](dsh-mcp-setup.md)；本机 profile 是 `desktop`。改完配置的 YAML 校验与备份要求见 [common-guide.md](common-guide.md#44-配置与排错通用提示)。

---

## 10. 首次实测任务

在 DSH 会话中（或在别的宿主里让代理加载 `browser-skill` 技能），三步走：

```text
browser_session({ action: "start", browser: "<实例id>", url: "https://example.com" })
browser_inspect({ action: "observe", session: "<id>" })
browser_session({ action: "stop", session: "<id>" })
```

本机实测成功读到 example.com 正文（多语言段落）并截图，会话正常停止。

> 多浏览器同时在线时**必须显式传 `browser`**，先用 `bsk browsers` 确认实例（实例 id 会变化）。

### 10.1 继续验证的清单

- [ ] `bsk --version` 输出预期版本，且 `bsk` 在 PATH 中（新终端里 `Get-Command bsk` 指向 `~/.local/bin/bsk.exe`）
- [ ] 浏览器扩展已安装，弹窗显示 connected（**且是真的连上，不只是弹窗字样**，见 11.1）
- [ ] `bsk doctor` 无 fail（`N/A` 不算失败）
- [ ] `bsk browsers` 能看到浏览器实例，记录其 instance_id
- [ ] 宿主能列出 `browser-skill` 技能（DSH 里可在技能目录/命令中看到）
- [ ] 跑一次 example.com 任务并能正常 `stop`

---

## 11. 本服务专属注意事项

### 11.1 Google 相关域名不可达导致的连锁问题

- Chrome 应用商店打不开 → 扩展装不上（**但弹窗可能仍显示"已连接"，具有误导性**）。
- **诊断方法**：直接看浏览器 profile 目录里有没有扩展、`chrome://extensions` 是否列出它：

```powershell
# Edge: <Edge User Data>\<Default|Profile N>\Extensions\<扩展ID>
# Chrome 商店版扩展 ID：hhcmgoofomhgciiibhipgmgkgnoenaoi
# Edge 商店版扩展 ID：emacgiaaaiojkkpkddmmdfhmokgmnikg
```

- **解决**：优先用 **Edge 商店**（国内多可直连），或用 **4.C 方案**（GitHub Release 解压加载）。
- 若本机有代理，命令行下载要**显式带 `--proxy`**，因为 `curl.exe` **不会自动读系统代理**。
- 扩展在**浏览器关闭时会断开**连接，daemon 里就看不到该浏览器了——这是正常现象，重开浏览器 + 弹窗连接即可。

### 11.2 `bsk doctor` 的 `N/A` 不是失败

DSH 用插件自带技能，所以 `agent skill matches bundled CLI` 这一项恒为 `N/A`。判据是**没有 fail**，不是"没有 N/A"。

### 11.3 daemon 空闲会自己退出

默认 10 分钟无浏览器/会话/IPC 客户端即退出，**下次命令会自动拉起**，不必手工干预。所以长时间空闲后再看 `bsk doctor` 显示 `daemon running` 属正常。

### 11.4 daemon 的更新方式

- 用安装脚本替换二进制后要 `bsk daemon restart`。
- **`--foreground` 起的 daemon 不会自我更新**，必须在它自己的那个终端里重启。
- 保持 CLI、daemon、扩展三者版本一致（`bsk doctor` 会检查 `version_skew`）。

---

## 12. 更新与卸载

仅列 bsk 相关命令（通用部分见 [common-guide.md](common-guide.md#5-更新与卸载)）。

**更新**

```powershell
bsk update --yes                                        # CLI（前台 daemon 需自行重启）
dsh plugin --profile <profile> update @wxg-prc-cpg/browser-skill-dsh-plugin --latest
# 浏览器扩展：商店版自动更新；解压版需重新下载新包并重新加载
```

**卸载**

```powershell
dsh plugin --profile <profile> remove @wxg-prc-cpg/browser-skill-dsh-plugin
bsk daemon stop
Remove-Item "$env:USERPROFILE\.bsk" -Recurse -Force     # daemon 运行时数据
# 再从 profile 的 cordis.patch.yml 删掉 browserskill 条目，并在浏览器里移除扩展
```

> 索引：[README.md](README.md)
