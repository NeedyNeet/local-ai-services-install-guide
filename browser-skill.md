# browser-skill 安装与使用

腾讯开源的浏览器自动化：`bsk` CLI + 后台 daemon + 浏览器扩展，让代理在独立的 **Agent Window** 里读页面、点按钮、填表单、截图，全程复用你**已登录**的会话。

> 通用部分（环境前置、DSH 挂载机制、跨工具方式、共用踩坑、更新卸载）见 [common-guide.md](common-guide.md)。

- 官方安装说明：<https://raw.githubusercontent.com/Tencent/BrowserSkill/main/AGENT_INSTALL.md>
- 本机实测：Windows x64 + DSH `0.2.0-rc.2`，bsk CLI `0.3.2`、浏览器扩展 `0.3.2`、DSH 插件 `@wxg-prc-cpg/browser-skill-dsh-plugin@0.3.2`

---

## 1. 前置条件

只列 browser-skill 自己的前置（完整环境前置表见 [common-guide.md](common-guide.md#1-环境前置)）：

| 条件 | 要求 | 本机实测 | 说明 |
|---|---|---|---|
| 浏览器 | Chromium 内核 **125+** | Chrome `154.0.0.0`、Edge `154.0.0.0` | 官方只保证 Chrome / Edge，其他 Chromium 浏览器不保证 |
| 网络 | 能访问 GitHub Releases | 需**代理**才能访问 Google | 能否访问 Google **直接决定扩展的安装渠道**：能直连则 Chrome 商店可用，不能则走 Edge 商店或 GitHub Release（见第 3 节） |

> 扩展必须由**用户本人**在浏览器里安装，代理不能代劳。

---

## 2. 安装 CLI

```powershell
# 官方一键安装（会写入 ~/.local/bin 并把它加到用户 PATH）
irm https://raw.githubusercontent.com/Tencent/BrowserSkill/main/install.ps1 | iex
```

### 2.1 受限环境绕过：`irm` 失败时先用下载器落地脚本

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

## 3. 安装浏览器扩展

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
# 例：下载 0.3.2 的扩展包（如需代理请加 --proxy http://127.0.0.1:7897）
curl.exe -fL -o bsk-ext.zip `
  'https://github.com/Tencent/BrowserSkill/releases/download/ext-v0.3.2/browser-skill-extension-v0.3.2-chrome.zip'
Expand-Archive .\bsk-ext.zip -DestinationPath .\bsk-ext -Force
```

然后浏览器打开 `chrome://extensions` → 右上角打开**开发者模式** → **加载已解压的扩展程序** → 选择 `bsk-ext` 目录。（`edge://extensions` 同理。）

> 本机实际走的是 C：`chromewebstore.google.com` 直连超时，而 Edge 商店可用。

**安装后必做**：点浏览器工具栏的 BrowserSkill 图标 → 打开弹窗 → **启用本地连接**，确认状态为 connected 且端口与 daemon 一致。只装扩展不启用本地连接，daemon 里看不到这个浏览器。

---

## 4. 启动 daemon

```powershell
bsk daemon start            # 后台自管 daemon（推荐）
# 或在需要常驻的终端/任务里：
bsk daemon start --foreground
```

**本机的选择**：把 daemon 起在**非沙箱**环境、且采用自管方式（不带 `--foreground`），这样它独立于任何会话存活。默认 bsk home 是 `%USERPROFILE%\.bsk`；若该目录在沙箱内不可写，需要把 `BSK_HOME` 指到一个**两端都能访问**的目录。

> 为什么不能在沙箱会话里直接跑 `bsk` 客户端（`拒绝访问 (os error 5)`）、以及 `BSK_AUTO_START=0` 的用法，见 [common-guide.md](common-guide.md#53-沙箱会拦截-ipc-命名管道并可能挡住工作区外的读写)。官方沙箱指南：<https://github.com/Tencent/BrowserSkill/blob/main/docs/sandboxed-agents.md>

---

## 5. 接入 DSH（专用插件，自带技能）

```powershell
dsh plugin --profile desktop add @wxg-prc-cpg/browser-skill-dsh-plugin
```

配置（profile 补丁 `%USERPROFILE%\.dsh\profiles\desktop\cordis.patch.yml`，`bskPath` 建议写绝对路径，避免 PATH 差异）：

```yaml
- id: browserskill
  config:
    bskPath: 'C:\Users\<你>\.local\bin\bsk.exe'
    defaultTimeoutMs: 120000
    maxSessions: 5
    observationEnabled: true
    thumbnailIntervalMs: 1500
    idleIntervalMs: 8000
    lazyTools: true
```

插件提供 `browser_session` / `browser_page` / `browser_inspect` / `browser_interact` / `browser_tabs` / `browser_assist` 六个原生工具与 `browser-skill` 技能，**不需要**再执行 `bsk install-skill`。

> 本机 profile 是 `desktop`。改完配置的 YAML 校验与备份要求见 [common-guide.md](common-guide.md#54-配置与排错通用提示)。

---

## 6. 验收：`bsk doctor` 必须没有 fail

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

> **doctor 通过 ≠ 技能被宿主发现**。还要确认宿主能把 `browser-skill` 列出来（DSH 里它会出现在技能目录/`/browser-skill` 命令中），并跑一次真实任务（第 7 节）。

---

## 7. 首次实测任务

在 DSH 会话中（或在别的宿主里让代理加载 `browser-skill` 技能），三步走：

```text
browser_session({ action: "start", browser: "<实例id>", url: "https://example.com" })
browser_inspect({ action: "observe", session: "<id>" })
browser_session({ action: "stop", session: "<id>" })
```

本机实测成功读到 example.com 正文（多语言段落）并截图，会话正常停止。

> 多浏览器同时在线时**必须显式传 `browser`**，先用 `bsk browsers` 确认实例（本机 Chrome `85936e7f`、Edge `214c635c`，实例 id 会变化）。

---

## 8. 在本服务上继续验证的清单

- [ ] `bsk --version` 输出预期版本，且 `bsk` 在 PATH 中（新终端里 `Get-Command bsk` 指向 `~/.local/bin/bsk.exe`）
- [ ] 浏览器扩展已安装，弹窗显示 connected（**且是真的连上，不只是弹窗字样**，见 10.1）
- [ ] `bsk doctor` 无 fail（`N/A` 不算失败）
- [ ] `bsk browsers` 能看到浏览器实例，记录其 instance_id
- [ ] 宿主能列出 `browser-skill` 技能（DSH 里可在技能目录/命令中看到）
- [ ] 跑一次 example.com 任务并能正常 `stop`

---

## 9. 在其他 AI 工具里使用

### 9.1 CLI 型宿主用 `bsk install-skill`

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
bsk install-skill --list                 # 查看全部与检测状态
bsk install-skill --harness cursor --json # 明确指定宿主（推荐，检测失败也能装）
```

注意：

- `--yes` 不带 `--harness` 会**给所有检测到的宿主**都装，且检测不到就直接失败。
- 已存在的技能文件会被**跳过**；要覆盖用 `--force`（会覆盖该文件）。
- 未列出的宿主：把仓库里 `crates/bsk-cli/skill/` 整个目录（含 `references/`）复制成目标宿主的 `browser-skill/`。
- **DSH 走插件（第 5 节），不要跑 `bsk install-skill`**。
- 装完要在**新会话**里确认技能被发现。技能会在 daemon 启动 / `session start` / `doctor` 时自动同步（文件未被本地改动时）。

### 9.2 不适配技能的宿主，直接调 CLI

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

## 10. 本服务专属注意事项

### 10.1 Google 相关域名不可达导致的连锁问题

- Chrome 应用商店打不开 → 扩展装不上（**但弹窗可能仍显示"已连接"，具有误导性**）。
- **诊断方法**：直接看浏览器 profile 目录里有没有扩展、`chrome://extensions` 是否列出它：

```powershell
# Edge: <Edge User Data>\<Default|Profile N>\Extensions\<扩展ID>
# Chrome 商店版扩展 ID：hhcmgoofomhgciiibhipgmgkgnoenaoi
# Edge 商店版扩展 ID：emacgiaaaiojkkpdmmdfhmokgmnikg
```

- **解决**：优先用 **Edge 商店**（国内多可直连），或用 **3.C 方案**（GitHub Release 解压加载）。
- 若本机有代理（本机 `127.0.0.1:7897`，系统代理已开），命令行下载要**显式带 `--proxy`**，因为 `curl.exe` **不会自动读系统代理**。
- 扩展在**浏览器关闭时会断开**连接，daemon 里就看不到该浏览器了——这是正常现象，重开浏览器 + 弹窗连接即可。

### 10.2 `bsk doctor` 的 `N/A` 不是失败

DSH 用插件自带技能，所以 `agent skill matches bundled CLI` 这一项恒为 `N/A`。判据是**没有 fail**，不是"没有 N/A"。

### 10.3 daemon 空闲会自己退出

默认 10 分钟无浏览器/会话/IPC 客户端即退出，**下次命令会自动拉起**，不必手工干预。所以长时间空闲后再看 `bsk doctor` 显示 `daemon running` 属正常。

### 10.4 daemon 的更新方式

- 用安装脚本替换二进制后要 `bsk daemon restart`。
- **`--foreground` 起的 daemon 不会自我更新**，必须在它自己的那个终端里重启。
- 保持 CLI、daemon、扩展三者版本一致（`bsk doctor` 会检查 `version_skew`）。

---

## 11. 更新与卸载

仅列 bsk 相关命令（通用部分见 [common-guide.md](common-guide.md#6-更新与卸载)）。

**更新**

```powershell
bsk update --yes                                        # CLI（前台 daemon 需自行重启）
dsh plugin --profile desktop update @wxg-prc-cpg/browser-skill-dsh-plugin --latest
# 浏览器扩展：商店版自动更新；解压版需重新下载新包并重新加载
```

**卸载**

```powershell
dsh plugin --profile desktop remove @wxg-prc-cpg/browser-skill-dsh-plugin
bsk daemon stop
Remove-Item "$env:USERPROFILE\.bsk" -Recurse -Force     # daemon 运行时数据
# 再从 profile 的 cordis.patch.yml 删掉 browserskill 条目，并在浏览器里移除扩展
```

> 索引：[README.md](README.md)
