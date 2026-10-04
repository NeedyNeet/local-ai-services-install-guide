# 在 DeepSeek Harness（DSH）中挂载 MCP 服务器

> 🔵 **这是 DSH 专用部分。** 用其他宿主（Claude Code / Cursor / Codex 等）请看每份服务文档的「宿主无关配置」一节，不需要本文。

DSH 没有 `mcpServers` 那种 JSON 键，而是把每个 MCP 服务器挂成 **`@deepseek-ai/dsh-mcp-client` 的一个插件实例**，写在 profile 的补丁 YAML 里。

| | Claude Code / Cursor | DeepSeek Harness |
|---|---|---|
| 格式 | JSON | YAML |
| 文件 | `mcp.json` / `.mcp.json` | `cordis.patch.yml` |
| 结构 | `mcpServers` 对象 | 顶层是**列表**，每项一个 `insert` |
| 一个服务器 | 对象里的一个键 | 一个 `insert` 条目，`name: '@deepseek-ai/dsh-mcp-client'` |
| 传输方式 | 隐式 | 显式 `transport: stdio` |

---

## 1. 装 MCP 客户端插件（每个 profile 只需一次）

```powershell
# 版本必须与 DSH 主版本对齐！见第 4.1 节
dsh plugin --profile <profile> add @deepseek-ai/dsh-mcp-client@0.2.0-rc.2
```

> 该包**没有声明 `dsh.bundle`**，安装时 DSH 会提示 `installed as a plain dependency, not a profile layer` —— **这是正常的**，它本来就要靠下面的 `insert` 条目实例化。

## 2. 每加一个 MCP 服务器 = 加一个 insert 条目

追加到 `%USERPROFILE%\.dsh\profiles\<profile>\cordis.patch.yml`（**顶层是列表**，不要嵌套）：

```yaml
- insert:
    - id: mcp-<server>            # 唯一
      name: '@deepseek-ai/dsh-mcp-client'
      config:
        serverName: <server>      # 决定工具前缀 mcp__<server>__*
        transport: stdio          # 或 streamable-http
        command: <解释器绝对路径>
        args: [ <服务脚本绝对路径> ]
        env: { <服务专属变量>: <值> }
        cwd: <工作目录，可选>
        toolCallTimeoutMs: 60000  # 长任务要放大（如 WizTree 全盘扫描 900000）
```

改完保存即**热加载**（无需重启 DSH）：子进程被拉起，工具随之注册。

> **Windows 路径不要用双引号**——`"C:\Users\..."` 里的 `\U` 会被 YAML 当转义序列导致解析失败。用单引号或裸写。

## 3. 中间层机制（理解它能省很多排错时间）

- 桥接会在启动子进程前**清洗环境变量**：删掉所有 `DSH_*`，以及名字含 `KEY`/`TOKEN`/`SECRET`/`PASSWORD` 的变量；只保留你显式写在 `env:` 里的。所以**不要依赖继承的环境变量或 PATH**。
- 因此 `command` 必须写**可执行文件的绝对路径**；虚拟环境要指向 venv 内的解释器（见第 4.2 节）。
- `reconnect`：断线后按退避自动重连并刷新工具；连续失败上限后工具被注销，需重载配置。
- **服务器注册成功 ≠ 连接成功**：注册了但启动失败时，工具名可见但调用报 `server is disconnected`。排错先看**子进程是否存在**。

## 4. DSH 专属注意事项

### 4.1 MCP 客户端插件版本必须与 DSH 主版本对齐

`@deepseek-ai/dsh-mcp-client` 的 `latest` 标签是**旧版本**（`0.0.1-rc.1`），直接 `add` 会装错版本。
按 DSH 版本选：DSH `0.2.0-rc.2` → 用 `@0.2.0-rc.2`（该版本也挂在 `next` 标签上）。

```powershell
# 查当前 DSH 版本
dsh --version
# 按版本安装
dsh plugin --profile <profile> add @deepseek-ai/dsh-mcp-client@0.2.0-rc.2
```

### 4.2 子进程必须用真正的解释器可执行文件

两个典型陷阱：

**(a) 不要用宿主自带的包装脚本。** DSH 的 `resources\runtime\bin\node.cmd` 只是个包装：

```bat
set ELECTRON_RUN_AS_NODE=1
"%DSH_DESKTOP_NODE_EXECUTABLE%" --expose-internals %*
```

而桥接会清洗掉所有 `DSH_*`，`DSH_DESKTOP_NODE_EXECUTABLE` 变空 → **必然启动失败**。
**症状**：服务器注册了，但一直 `server is disconnected`，且没有任何子进程。
**解决**：`command` 写系统 node 的绝对路径（如 `<Node 安装目录>\node.exe`），并确认该文件**真实存在**——本机就曾误写成不存在的 `runtime\bin\node.exe`，症状完全相同。

**(b) 不要用 Microsoft Store 的 `python.exe`。** `Get-Command python` 可能指向 `WindowsApps\python.exe` 这个**占位 stub**，执行时可能转而弹出商店或行为异常。用真实安装路径建 venv，然后一律用 `.venv\Scripts\python.exe`。

### 4.3 沙箱会拦截 IPC 命名管道

在带文件沙箱的受限会话里：

- 工作区之外的目录可能**不可写**（例如 `~/.bsk` 建立失败 → `bsk doctor` 报 `bsk home writable` FAIL）；
- 某些 IPC 走**命名管道**，沙箱内客户端连接会 `拒绝访问 (os error 5)`。

这是沙箱的预期隔离，不是服务的 bug。应对：

- 把常驻进程起在**非沙箱**环境，或用宿主的托管后台任务；
- 让沙箱与宿主**共享同一个数据目录**，并允许访问该目录与 IPC 端点；
- 客户端命令带 `BSK_AUTO_START=0`，避免在沙箱里误拉起新 daemon；
- browser-skill 官方沙箱指南：<https://github.com/Tencent/BrowserSkill/blob/main/docs/sandboxed-agents.md>

> DSH 插件提供的原生工具（如 `browser_*`）由宿主进程拉起，**不受**会话沙箱影响；受影响的只是你在沙箱内直接用 shell 跑 CLI。

### 4.4 其他

- 改了 MCP 服务器源码后，除了重新构建**还要重载 MCP 子进程**（子进程启动时缓存模块），否则运行的仍是旧代码。改一下 `cordis.patch.yml` 里该条目的值即可触发重载。
- 改配置前先备份 `cordis.patch.yml`（惯例：留 `.bak-*`）。

---

## 5. 验证三连（每个 MCP 服务器都适用）

```powershell
# 1) 子进程是否存在（应指向你配置里的路径）
Get-CimInstance Win32_Process -Filter "Name='node.exe' OR Name='python.exe'" |
  Where-Object CommandLine -match '<server>' | Select-Object ProcessId,CommandLine

# 2) 不靠宿主，直接用协议探针验证服务器本身
#    依次发 initialize -> notifications/initialized -> tools/list -> tools/call

# 3) 宿主里真实调一次工具
```

## 6. 配置文件位置

| 用途 | 路径 |
|---|---|
| DSH profile 补丁（**所有服务配置都在这里**） | `%USERPROFILE%\.dsh\profiles\<profile>\cordis.patch.yml` |
| DSH profile 依赖/装配 | `%USERPROFILE%\.dsh\profiles\<profile>\package.json` |

---

各服务的具体 `insert` 条目见对应服务文档的「接入 DSH」一节：
[browser-skill.md](browser-skill.md) · [everything-mcp.md](everything-mcp.md) · [wiztree-mcp.md](wiztree-mcp.md) · [bcompare-mcp.md](bcompare-mcp.md)
