# 通用指南（common-guide）

所有本地服务的**共用部分**：环境前置、DSH 如何挂载服务、在其他 AI 工具里怎么接、更新卸载、路径规范与配置文件位置。

各服务的具体安装步骤见其独立文档（[browser-skill.md](browser-skill.md) · [everything-mcp.md](everything-mcp.md) · [wiztree-mcp.md](wiztree-mcp.md)）。

> **范围声明**：本文的「DSH 如何挂载服务」（第 2 节）与「共用注意事项」里的 DSH 相关条目（如 `DSH_*` 变量清洗、沙箱 IPC）是 **DSH 专用**；
> 第 1 节环境前置、第 4 节跨工具接入、第 6 节更新卸载对**任何宿主**都适用。
> 完整区分见 [README 的「哪些是通用的，哪些是 DSH 专用的」](README.md#先看这里哪些是通用的哪些是-dsh-专用的)。

- 实测环境：Windows x64 + DeepSeek Harness（DSH `0.2.0-rc.2`），2026-10-04
- 文档结构：**每套服务一份独立文件**，新增服务只加文件，不动其他文档

---

## 1. 环境前置

| 组件 | 要求 | 本机实测 | 说明 |
|---|---|---|---|
| 操作系统 | Windows 10 / Server 2016 以上，x64 | Windows 10 x64 | bsk 仅提供 x64 与 ARM64；Everything / WizTree 的深度扫描能力是 Windows 专属 |
| Node.js | ≥ 18（everything-mcp 要求） | `v24.19.0` | **MCP 子进程要用真正的 `node.exe`**，见 5.2 |
| Python | ≥ 3.10（wiztree-mcp 要求） | `3.14.6`（venv 内） | 用**真实安装**的 Python，不要用 Microsoft Store 的 `python.exe` 占位 stub |
| 包管理器 | npm / pnpm | npm `12.2.0`，pnpm `11.7.0` | DSH 管理插件用 pnpm |
| 浏览器 | Chromium 内核 **125+** | Chrome `154.0.0.0`、Edge `154.0.0.0` | 仅 browser-skill 需要 |
| Everything | voidtools Everything，**已建 NTFS 索引** | `1.4.1.1026 x64` 便携版 | 仅 everything-mcp 需要，须启用 HTTP 服务器 |
| WizTree | WizTree 便携版或安装版 | `4.31.0.0` 便携版 | 仅 wiztree-mcp 需要 |
| AI 宿主 | 任一 shell 型代理；或 DSH | DSH `0.2.0-rc.2` | 见第 4 节跨工具用法 |
| 网络 | 能访问 GitHub Releases / npm registry / PyPI | 访问 Google 需**代理**（`127.0.0.1:7897`） | 命令行下载要显式带 `--proxy` |

> 只装其中某套服务时，只需要该服务对应的那几行前置。

---

## 2. DSH 如何挂载服务

DSH 有两种接入方式，别混：

| 方式 | 适用于 | 表现 |
|---|---|---|
| **原生插件**（`dsh plugin add`） | 服务提供了 DSH 插件包（本机只有 browser-skill） | 提供原生工具（`browser_*`），自带技能 |
| **MCP 客户端实例**（`@deepseek-ai/dsh-mcp-client` + `cordis.patch.yml` 里的 `insert` 条目） | 任何标准 stdio / streamable-http MCP 服务器 | 工具以 `mcp__<serverName>__<tool>` 暴露 |

### 2.1 装 MCP 客户端插件（只需一次）

```powershell
# 版本必须与 DSH 主版本对齐！见 5.1
dsh plugin --profile desktop add @deepseek-ai/dsh-mcp-client@0.2.0-rc.2
```

### 2.2 每加一个 MCP 服务器 = 加一个 insert 条目

追加到 `%USERPROFILE%\.dsh\profiles\<profile>\cordis.patch.yml`（**顶层是列表**，不要嵌套）：

```yaml
- insert:
    - id: mcp-<server>            # 唯一
      name: '@deepseek-ai/dsh-mcp-client'
      config:
        serverName: <server>      # 决定工具前缀 mcp__<server>__*
        transport: stdio          # 或 streamable-http
        command: <可执行文件绝对路径>
        args: [ <脚本绝对路径> ]
        env: { <服务专属变量>: <值> }
        cwd: <工作目录，可选>
        toolCallTimeoutMs: 60000  # 长任务要放大
```

改完保存即**热加载**（无需重启 DSH）：子进程被拉起，工具随之注册。

### 2.3 中间层机制（理解它能省很多排错时间）

- 桥接会在启动子进程前**清洗环境变量**：删掉所有 `DSH_*`，以及名字含 `KEY`/`TOKEN`/`SECRET`/`PASSWORD` 的变量；只保留你显式写在 `env:` 里的。所以**不要依赖继承的环境变量或 PATH**。
- 因此 `command` 必须写**可执行文件的绝对路径**，虚拟环境要指向 venv 内的解释器。
- `reconnect`：断线后按退避自动重连并刷新工具；连续失败上限后工具被注销，需重载配置。
- 服务器**注册成功 ≠ 连接成功**：注册了但启动失败时，工具名可见但调用报 `server is disconnected`。排错先看**子进程是否存在**。

### 2.4 验证三连（每个 MCP 服务器都适用）

```powershell
# 1) 子进程是否存在（应指向你配置里的路径）
Get-CimInstance Win32_Process -Filter "Name='node.exe' OR Name='python.exe'" |
  Where-Object CommandLine -match '<server>' | Select-Object ProcessId,CommandLine

# 2) 不靠宿主，直接用协议探针验证服务器本身
#    依次发 initialize -> notifications/initialized -> tools/list -> tools/call
#    用 child_process.spawn 起服务器、按行解析 stdout 的 JSON-RPC（见 5.3）

# 3) 宿主里真实调一次工具
```

---

## 3. 路径规范与目录规划（本机约定）

MCP 服务器是**需要构建、需要长期维护**的代码，不要留在下载目录（临时区会被清理）。

> **本节的路径是「本机的约定」，不是通用要求。** 全仓库文档统一用下面的占位符表示本机的实际位置——这样文档不绑定某台机器的盘符，换台电脑或分享给别人时只需知道自己对应哪个目录。
> 想改成自己的布局，把下表的位置换掉，并把文档里的占位符按同样含义理解即可。

| 类型 | 本机位置 | 说明 |
|---|---|---|
| GitHub 代码仓库 | `<Dev 根目录>\Github\<repo>` | 新建或克隆的仓库都放这里 |
| MCP 服务器 | `<MCP 根目录>\<server-name>` | everything-mcp、wiztree-mcp、beyondcompare-mcp 都在这里 |
| 本仓库（文档/约定） | `<Dev 根目录>\Github\dsh-local-services\` | 本目录 |
| 新项目起步模板 | `<Dev 根目录>\Github\_templates\` | 复制 `AGENTS.md` 到新项目根目录，即继承本机服务约定 |
| 其他本地项目 | `<Dev 根目录>\Projects\` | 不打算推到 GitHub 的实验/本地项目 |
| 包管理器缓存 | `<Dev 根目录>\Cache\` | npm / pip / gradle / nuget 等 |
| 免安装软件 | `<便携软件根目录>\` | Everything、WizTree、Beyond Compare |
| 已安装软件 | `<已安装软件根目录>\` | 如 DeepSeek Harness |
| Node 运行时 | `<Node 安装目录>\node.exe` | MCP 子进程必须指向这个真实可执行文件 |
| Python 运行时 | `<Python 安装目录>\python.exe` | 用它建 venv，不要用 Store 版 stub |

### 3.1 占位符对照（本机实际值）

文档里出现这些占位符时，本机对应的真实位置如下（**只在需要照抄命令时替换**）：

| 占位符 | 本机实际值 |
|---|---|
| `<Dev 根目录>` | `D:\Dev` |
| `<MCP 根目录>` | `D:\Dev\MCP` |
| `<便携软件根目录>` | `D:\Apps\Portable` |
| `<已安装软件根目录>` | `D:\Apps\Installed` |
| `<Node 安装目录>` | `D:\Node.js` |
| `<Python 安装目录>` | `D:\Dev\Python314` |
| `<本仓库>` | `<Dev 根目录>\Github\dsh-local-services` |
| `C:\Users\<你的用户名>` | `C:\Users\liuhongda` |

> 之前存在的「归档资料」目录（`_archive`，存放迁移前的旧配置与搜索历史）**已从仓库移除**：其中的 Everything 搜索历史包含本机完整文件路径，不适合随仓库分发。备份保留在仓库外。

**迁移已有安装时**，只需改 `cordis.patch.yml` 里该条目的 `args` / `cwd` / `command`，保存后热加载生效：

```powershell
# 验证迁移成功：子进程应指向新路径
Get-CimInstance Win32_Process -Filter "Name='node.exe' OR Name='python.exe'" |
  Where-Object CommandLine -match '<server>' | Select-Object ProcessId,CommandLine
```

---

## 4. 在其他 AI 工具里使用

### 4.1 标准 MCP 服务器（everything-mcp / wiztree-mcp）

它们是**与宿主无关**的标准 stdio MCP 服务器。通用接法是「命令 + 参数 + 环境变量」三件套：

- `command`：解释器**绝对路径**（node：`<Node 安装目录>\node.exe`；python：项目 `.venv\Scripts\python.exe`）
- `args`：`[<服务脚本绝对路径>]`
- `env`：服务专属变量（如 Everything 的 `EVERYTHING_PORT`、WizTree 的 `WIZTREE_EXE`）

举例：

- **Claude Code**：`claude mcp add --scope user <名字> -- <解释器> <脚本>`（env 需另外注入，或用带环境变量的包装脚本）
- **Cursor**：写 `%USERPROFILE%\.cursor\mcp.json`（`mcpServers` 对象结构，含 `command`/`args`/`env`）
- **DSH**：见 2.2 的 `insert` 补丁
- **其他宿主**：填进它的 MCP 配置（多数支持 `command`/`args`/`env`/`cwd` 的 JSON）

**关键点**：宿主的 MCP 配置必须能用「解释器绝对路径 + 显式 env」。凡是依赖"先激活虚拟环境"或"PATH 里有 node/python"的接法，在清洗环境后都会失败。

### 4.2 browser-skill

见 [browser-skill.md](browser-skill.md#9-在其他-ai-工具里使用)（它有自己的 CLI 技能安装机制 `bsk install-skill`）。

---

## 5. 共用注意事项

### 5.1 MCP 客户端插件版本必须与宿主主版本对齐

`@deepseek-ai/dsh-mcp-client` 的 `latest` 标签是**旧版本**（`0.0.1-rc.1`），直接 `add` 会装错版本。
按 DSH 版本选：DSH `0.2.0-rc.2` → 用 `@0.2.0-rc.2`（该版本也挂在 `next` 标签上）。

```powershell
dsh plugin --profile desktop add @deepseek-ai/dsh-mcp-client@0.2.0-rc.2
```

另外：该包**没有声明 `dsh.bundle`**，安装时 DSH 会提示 `installed as a plain dependency, not a profile layer` —— **这是正常的**，它本来就要靠 `cordis.patch.yml` 里的 `insert` 条目实例化。

### 5.2 子进程必须用真正的解释器可执行文件

两个典型陷阱：

**(a) 不要用宿主自带的包装脚本。** DSH 的 `resources\runtime\bin\node.cmd` 只是个包装：

```bat
set ELECTRON_RUN_AS_NODE=1
"%DSH_DESKTOP_NODE_EXECUTABLE%" --expose-internals %*
```

而桥接会清洗掉所有 `DSH_*`，`DSH_DESKTOP_NODE_EXECUTABLE` 变空 → 必然启动失败。
**症状**：服务器注册了，但一直 `server is disconnected`，且没有任何子进程。
**解决**：`command` 写系统 node 绝对路径（本机 `<Node 安装目录>\node.exe`），并确认该文件**真实存在**（本机一度误写成不存在的 `runtime\bin\node.exe`，症状完全相同）。

**(b) 不要用 Microsoft Store 的 `python.exe`。** `Get-Command python` 可能指向 `WindowsApps\python.exe` 这个**占位 stub**，它执行时可能转而弹出商店或行为异常。用真实安装路径（本机 `<Python 安装目录>\python.exe`）创建 venv，之后一律用 `.venv\Scripts\python.exe`。

### 5.3 沙箱会拦截 IPC 命名管道，并可能挡住工作区外的读写

在带文件沙箱的受限会话里：

- 工作区之外的目录可能**不可写**（例如 `~/.bsk` 建立失败 → `bsk doctor` 报 `bsk home writable` FAIL）；
- 某些 IPC 走**命名管道**，沙箱内客户端连接会 `拒绝访问 (os error 5)`。

这是沙箱的预期隔离，不是服务的 bug。应对：

- 把常驻进程（如 `bsk` daemon）起在**非沙箱**环境，或用宿主的托管后台任务；
- 让沙箱与宿主**共享同一个数据目录**，并允许访问该目录与 IPC 端点；
- 客户端命令带 `BSK_AUTO_START=0`，避免在沙箱里误拉起新 daemon；
- browser-skill 官方沙箱指南：<https://github.com/Tencent/BrowserSkill/blob/main/docs/sandboxed-agents.md>

> 注意：DSH 插件提供的原生工具（如 `browser_*`）由宿主进程拉起，**不受**会话沙箱影响；受影响的只是你在沙箱内直接用 shell 跑 CLI。

### 5.4 配置与排错通用提示

- **改配置前先备份** `cordis.patch.yml`（本机每次改动都留了 `.bak-*`）。
- **YAML 缩进错了宿主可能静默忽略**。注意：**Windows 路径不要用双引号**（`"C:\Users\..."` 里的 `\U` 会被当转义序列导致解析失败），用单引号或裸写。
- **探针脚本**：在没有宿主的情况下验证 MCP 服务器——用 `child_process.spawn` 起服务器，按行解析 stdout 的 JSON-RPC，依次发 `initialize` → `notifications/initialized` → `tools/list` → `tools/call`。本机用它定位了 5.2(a) 的问题。
- **注册了却没连接**：先查子进程是否存在（2.4 第 1 步），再看服务器自己的 stderr。

---

## 6. 更新与卸载

**更新**

```powershell
bsk update --yes                                              # browser-skill CLI
dsh plugin --profile desktop update @wxg-prc-cpg/browser-skill-dsh-plugin --latest
dsh plugin --profile desktop update @deepseek-ai/dsh-mcp-client@<与 DSH 对齐的版本>
# 浏览器扩展：商店版自动更新；解压版需重新下载新包并重新加载
# MCP 服务器本体：各自项目目录里 git pull / 重新构建
```

保持 bsk 的 CLI、daemon、扩展三者版本一致（`bsk doctor` 会检查 `version_skew`）。

**卸载**

```powershell
dsh plugin --profile desktop remove @wxg-prc-cpg/browser-skill-dsh-plugin
dsh plugin --profile desktop remove @deepseek-ai/dsh-mcp-client
bsk daemon stop
Remove-Item "$env:USERPROFILE\.bsk" -Recurse -Force     # daemon 运行时数据
# 再从 cordis.patch.yml 删掉对应 insert 条目 / 浏览器里移除扩展
```

---

## 7. 配置文件位置速查

| 用途 | 路径 |
|---|---|
| DSH profile 补丁（**所有服务配置都在这里**） | `%USERPROFILE%\.dsh\profiles\<profile>\cordis.patch.yml` |
| DSH profile 依赖/装配 | `%USERPROFILE%\.dsh\profiles\<profile>\package.json` |
| bsk daemon 数据与日志 | `%USERPROFILE%\.bsk\`（`daemon.json`、`daemon.log.*`） |
| bsk CLI | `%USERPROFILE%\.local\bin\bsk.exe` |
| everything-mcp 项目 | `<MCP 根目录>\everything-mcp`（产物 `dist/index.js`） |
| wiztree-mcp 项目 | `<MCP 根目录>\wiztree-mcp`（`.venv\Scripts\python.exe` + `wiztree_mcp.py`） |
| wiztree 快照缓存 | `%LOCALAPPDATA%\wiztree-mcp`（`scans.json`、`scan-*.csv`） |
