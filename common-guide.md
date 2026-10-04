# 通用指南

四套服务**共用的部分**：环境前置、MCP 配置的通用形态、验证方法、共用踩坑、更新卸载。

各服务的具体安装步骤见其独立文档（[browser-skill.md](browser-skill.md) · [everything-mcp.md](everything-mcp.md) · [wiztree-mcp.md](wiztree-mcp.md) · [bcompare-mcp.md](bcompare-mcp.md)）。

> 本文**不绑定任何宿主**。DSH 专用的挂载写法（`cordis.patch.yml` + `insert` 条目）单独放在 [dsh-mcp-setup.md](dsh-mcp-setup.md)。

- 实测环境：Windows x64，2026-10-04
- 文档原则：**每套服务一份独立文件**，新增服务只加文件，不动其他文档

---

## 1. 环境前置

| 组件 | 要求 | 说明 |
|---|---|---|
| 操作系统 | Windows 10 / Server 2016 以上，x64 | 四套服务里三套依赖 Windows 软件（Everything、WizTree、Beyond Compare）；browser-skill 的 CLI 也仅提供 Windows/macOS/Linux 的 x64·ARM64 |
| Node.js | ≥ 18（everything-mcp）；≥ 24（bcompare-mcp） | **MCP 子进程必须用真正的 `node.exe`**，见第 4.2 节 |
| Python | ≥ 3.10（wiztree-mcp） | 用**真实安装**的 Python，不要用 Microsoft Store 的 `python.exe` 占位 stub |
| 浏览器 | Chromium 内核 **125+** | 仅 browser-skill 需要（Chrome / Edge） |
| Everything | voidtools Everything，**已建 NTFS 索引** | 仅 everything-mcp 需要，且须**启用 HTTP 服务器** |
| WizTree | 便携版或安装版均可 | 仅 wiztree-mcp 需要 |
| Beyond Compare | **5**（项目要求） | 仅 bcompare-mcp 需要；**试用版即可**（见 bcompare-mcp.md 第 8.1 节） |
| MCP 宿主 | 任何支持 stdio MCP 的客户端 | Claude Code / Cursor / Codex / DSH 等 |
| 网络 | 能访问 GitHub Releases / npm registry / PyPI | 若下载需要代理，命令行要**显式带 `--proxy`**（`curl.exe` 不读系统代理设置） |

> 只装某套服务时，只需要该服务对应的那几行前置。

---

## 2. MCP 配置的通用形态

三套 MCP 服务器（everything-mcp / wiztree-mcp / bcompare-mcp）是**标准 stdio MCP 服务器**，与宿主无关。接入任何宿主都是同一件事：**填一个「命令 + 参数 + 环境变量」三件套**。

### 2.1 三件套

| 字段 | 填什么 | 要点 |
|---|---|---|
| `command` | **解释器的绝对路径** | node 用 `<Node 安装目录>\node.exe`；python 用项目里的 `.venv\Scripts\python.exe` |
| `args` | 服务入口脚本的**绝对路径** | 如 `...\dist\index.js` 或 `...\wiztree_mcp.py` |
| `env` | 服务专属环境变量 | 如 Everything 的 `EVERYTHING_PORT`、WizTree 的 `WIZTREE_EXE` |

**唯一容易出错的地方**：不能用「先激活虚拟环境」或「PATH 里有 node」这种依赖——MCP 客户端启动子进程时的环境往往是干净的，必须写绝对路径 + 显式 env。

### 2.2 各宿主的写法

**通用 JSON 形态**（Claude Code 的 `.mcp.json`、Cursor 的 `mcp.json`、多数 MCP 客户端都类似）：

```json
{
  "mcpServers": {
    "<服务器名>": {
      "command": "<解释器绝对路径>",
      "args": ["<服务脚本绝对路径>"],
      "env": { "<变量>": "<值>" }
    }
  }
}
```

**Claude Code**（命令行注册，会自动合并原有配置）：

```bash
claude mcp add --scope user <服务器名> -- <解释器绝对路径> <服务脚本绝对路径>
```

`--scope` 决定注册位置：`user`（所有项目）/ `project`（写 `.mcp.json` 到当前目录）/ `local`（仅当前目录、仅自己）。
环境变量需要另外注入，或用带环境变量的包装脚本。

**Cursor**：`Settings → MCP → Add new MCP server`，或直接编辑 `%USERPROFILE%\.cursor\mcp.json`（全局）／项目内 `.cursor\mcp.json`。

**DSH**：不适用上面的 JSON——见 [dsh-mcp-setup.md](dsh-mcp-setup.md)。

**其他宿主**：把三件套填进它的 MCP 配置即可。只要它能填「解释器绝对路径 + 显式 env」，就能接上。

### 2.3 browser-skill 不走 MCP

browser-skill 是 CLI + 浏览器扩展，**不是 MCP 服务器**，接入方式不同：

- **CLI 型宿主**：`bsk install-skill --harness <id>` 装技能（详见 [browser-skill.md](browser-skill.md) 第 9 节）
- **不支持技能的宿主**：直接调 `bsk` 命令
- **DSH**：用它的专用插件，提供原生 `browser_*` 工具

---

## 3. 验证方法

### 3.1 先用协议探针独立验证服务器

**接入宿主之前**先确认服务器自己能跑通——这样出问题就能立刻判断是「服务器坏了」还是「宿主配置错了」。

用 `initialize` → `notifications/initialized` → `tools/list` → `tools/call` 四步握手，按行收发 JSON-RPC 即可。最小版本：

```powershell
# 只看能否 initialize（把路径换成你的）
'{"jsonrpc":"2.0","id":1,"method":"initialize","params":{"protocolVersion":"2024-11-05","capabilities":{},"clientInfo":{"name":"probe","version":"1.0"}}}' |
  & "<解释器绝对路径>" "<服务脚本绝对路径>"
```

期望在 stderr 看到服务自己的就绪信息，stdout 返回 `serverInfo`。完整的四步探针见 [everything-mcp.md](everything-mcp.md) 第 4 节。

### 3.2 接入后看子进程是否存在

```powershell
Get-CimInstance Win32_Process -Filter "Name='node.exe' OR Name='python.exe'" |
  Where-Object CommandLine -match '<服务关键词>' | Select-Object ProcessId,CommandLine
```

**注册成功 ≠ 连接成功**：工具名出现了但调用报 `server is disconnected`，多半就是子进程没起来。

---

## 4. 共用注意事项

### 4.1 端口/路径不一致是最常见的失败原因

每个服务的「服务端」都有自己的配置（Everything 的 HTTP 端口、WizTree 的 exe 位置、Beyond Compare 的 exe 位置），而 MCP 侧的默认值**不一定与你的实际一致**。**先确认服务端实际情况，再写配置**——各服务文档的「本服务专属注意事项」里都有各自的查法。

### 4.2 MCP 子进程必须用真正的解释器可执行文件

- **Node**：不要用宿主自带的包装脚本（如某些宿主提供的 `node.cmd`，它依赖环境变量，而 MCP 桥接可能清洗掉那些变量）→ 直接写系统 `node.exe` 的绝对路径。
- **Python**：不要用 Microsoft Store 的 `python.exe`（占位 stub）→ 用真实安装路径建 venv，然后指向 `.venv\Scripts\python.exe`。

**症状**（两者相同）：服务器注册了但一直 `server is disconnected`，且没有任何子进程。

> DSH 用户请额外看 [dsh-mcp-setup.md 第 4.2 节](dsh-mcp-setup.md#42-子进程必须用真正的解释器可执行文件)——那里解释了 DSH 清洗 `DSH_*` 变量导致的同类问题。

### 4.3 受限沙箱会拦截 IPC 与工作区外读写

在带沙箱的受限会话里：

- 工作区之外的目录可能**不可写**；
- 走**命名管道**的 IPC 可能被拒绝（`拒绝访问 (os error 5)`）。

这是沙箱的预期隔离，不是服务的 bug。应对：把常驻进程起在非沙箱环境、让沙箱与宿主共享同一数据目录、客户端命令带 `BSK_AUTO_START=0` 以免误拉起新 daemon。
（browser-skill 官方指南：<https://github.com/Tencent/BrowserSkill/blob/main/docs/sandboxed-agents.md>）

### 4.4 配置与排错通用提示

- **改配置前先备份**（惯例：留 `.bak-*`）。
- **YAML 里 Windows 路径不要用双引号**——`"C:\Users\..."` 的 `\U` 会被当转义序列导致解析失败，用单引号或裸写。
- **改了服务源码后要重启对应的子进程**——MCP 子进程在启动时缓存模块，重新构建后若进程还在跑，用的仍是旧代码。
- **找不到原因时**：按「服务端是否正常 → 探针能否跑通 → 子进程是否存在 → 宿主配置」的顺序逐段排除，不要一上来就改宿主配置。

---

## 5. 更新与卸载

**更新**

| 组件 | 命令 |
|---|---|
| browser-skill CLI | `bsk update --yes` |
| 浏览器扩展 | 商店版自动更新；解压版需重新下载新包并重新加载 |
| MCP 服务器本体 | 各自项目目录里 `git pull` 后重新构建 |
| DSH 插件 | `dsh plugin --profile <profile> update <包名> --latest` |

保持 bsk 的 CLI、daemon、扩展三者版本一致（`bsk doctor` 会检查 `version_skew`）。

**卸载**

- 从宿主的 MCP 配置里删掉对应的服务器条目（DSH 是从 `cordis.patch.yml` 删 `insert` 条目）
- browser-skill 另有：`bsk daemon stop`、删除 `%USERPROFILE%\.bsk`（daemon 运行时数据）、在浏览器里移除扩展

---

## 6. 路径写法说明

本文档用 `<Node 安装目录>`、`<MCP 根目录>`、`<便携软件根目录>` 这类**占位符**表示位置，因为四套服务装在哪里由你决定。看到占位符时替换成你自己的实际路径即可。

本机（文档作者的机器）的对照关系记录在各服务文档的「本机实测」小节里，仅作参考——**你不需要与之一致**。
