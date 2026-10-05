# everything-mcp 安装与使用

把 **voidtools Everything** 的 NTFS 索引能力通过 MCP 暴露给 AI，实现全盘文件「毫秒级」检索（工具名 `mcp__everything__everything_search`）。

> 通用部分（环境前置、MCP 配置形态、共用踩坑、更新卸载）见 [common-guide.md](common-guide.md)。
> 四套服务的配置对照表见 [common-guide.md 第 6 节](common-guide.md#6-四套服务速查表)。

- 实测环境：Windows x64 + DeepSeek Harness（DSH `0.2.0-rc.2`），2026-10-04
- 实测版本：Everything `1.4.1.1026 x64` 便携版（HTTP 端口 **80**）、Node `v24.19.0`、MCP 客户端 `@deepseek-ai/dsh-mcp-client@0.2.0-rc.2`
- 项目本体（本机）：`<MCP 根目录>\everything-mcp`，运行时入口 `dist\index.js`

**本页导航**（本文的章节顺序与本仓库另两份服务文档略有不同，按需跳转）：

| 我想…… | 看 |
|---|---|
| 确认前置条件 | [第 1 节](#1-前置条件只列本服务需要的) |
| **先确认 Everything 本身能用** | [第 2 节](#2-前置验证everything-http-服务必须真的可用)（含端口查法） |
| 装并构建 | [第 3 节](#3-安装与构建) |
| **接入宿主前独立验证服务器** | [第 4 节](#4-先用协议探针独立验证服务器再接入宿主)（含探针脚本） |
| 查环境变量 | [第 5 节](#5-配置项环境变量) |
| DSH 接入 | [第 6 节](#6-接入-dsh) |
| 其他宿主（Claude Code / Cursor） | [第 9 节](#9-在其他-ai-工具里使用) |
| 验收清单 | [第 7 节](#7-验收) · [第 8 节](#8-在本服务上继续验证的清单) |
| **踩坑（端口不一致等）** | [第 10 节](#10-本服务专属注意事项) |

---

## 1. 前置条件（只列本服务需要的）

| 组件 | 要求 | 本机实测 |
|---|---|---|
| 操作系统 | Windows 10 / Server 2016 以上，x64 | Windows 10 x64 |
| Node.js | ≥ 18（该项目要求） | `v24.19.0` |
| Everything | voidtools Everything，**已完成 NTFS 索引**（首次可能要几分钟） | `1.4.1.1026 x64` 便携版 |
| Everything HTTP 服务器 | **必须启用** | 已启用，端口 `80`（默认是 `54321`） |
| AI 宿主 | 任一支持 MCP 的宿主；DSH 需装 MCP 客户端插件 | DSH `0.2.0-rc.2` |

启用 HTTP 服务器：Everything 主界面 **工具 → 选项 → HTTP 服务器 → 勾选「启用 HTTP 服务器」**。

- 同一页可以看到/修改**端口**，默认 `54321`，本机改成了 **`80`** —— 后续配置必须显式指定端口（见 [10.1](#101-端口不一致最常见的失败原因)）。
- 若在那一页设了用户名/密码，MCP 侧当前实现**不支持带认证的请求**，需要自行改造。
- 只装本服务时，其余前置（Python、浏览器等）都不需要；完整前置总表见 [common-guide.md](common-guide.md#1-环境前置)。

---

## 2. 前置验证：Everything HTTP 服务必须真的可用

**先确认这一层通了，再去碰 MCP**；否则后面分不清是 Everything 的问题还是 MCP 的问题。

```powershell
curl.exe -sS "http://127.0.0.1:80/?search=everything-mcp&json=1&count=2&path_column=1&size_column=1&date_modified_column=1"
```

期望：**HTTP 200**，返回 `{"totalResults":N,"results":[...]}`，且每条结果含 `name` / `path` / `size` / `date_modified` 四个字段。

本机实测（搜索 `everything-mcp`；结构保留，路径已泛化）：

```json
{
	"totalResults": 3,
	"results": [
		{
			"type": "folder",
			"name": "everything-mcp",
			"path": "<MCP 根目录>",
			"size": "",
			"date_modified": "134355807077193639"
		},
		{
			"type": "file",
			"name": "everything-mcp.md",
			"path": "<本仓库>",
			"size": "18903",
			"date_modified": "134355808510268293"
		}
	]
}
```

> `size` 对文件夹是空字符串（正常）；`date_modified` 是 Windows FILETIME 的十进制字符串——MCP 服务器会转成可读时间再返回给你。

判定标准：返回 200 **且能同时看到 `path` / `size` / `date_modified`** 才算前置就绪。少了这些字段，说明请求里没带对应的 `*_column=1`（MCP 服务器自己会带），或者你连的根本不是 Everything 的 HTTP 服务。

### 2.1 查 Everything 实际监听哪个端口

端口经常被改，**写配置前先问出真实端口**（不要照抄 `54321`）：

```powershell
Get-NetTCPConnection -State Listen |
  Where-Object OwningProcess -in (Get-Process Everything).Id |
  Select-Object LocalAddress,LocalPort,OwningProcess
```

本机实测（Everything 监听 `80`）：

```text
LocalAddress LocalPort OwningProcess
------------ --------- -------------
::                  80         15756
0.0.0.0             80         15756
```

也可以直接读 ini 里的 `http_server_port` —— 但便携版/迁移过的机器上可能有多份 ini，**务必先按 [10.2](#102-机器上可能存在多份-everythingini--everythingdb) 判断哪份是活的**。

---

## 3. 安装与构建

MCP 服务器是**需要构建、需要长期维护**的代码，不要留在下载目录（临时区会被清理）；本机固定在 `<MCP 根目录>\everything-mcp`（路径规范见 [common-guide.md](common-guide.md#7-路径写法说明)）。下面命令里的目录请替换成你自己的：

```powershell
# 1) 取源码（任选其一）
git clone https://github.com/Josephur/everything-mcp.git
#   或下载 ZIP 解压

# 2) 安装依赖并编译
npm install
npm run build

# 3) 确认产物存在（该项目是 TypeScript，必须 build 才有 dist/index.js）
Test-Path .\dist\index.js
```

**关键点**：这是 TypeScript 项目，`npm install` 之后**必须 `npm run build`**，`dist/index.js` 才会生成。宿主配置里 `args` 指向的是**构建产物**，不是源码 `src/`；跳过 build 的典型症状是子进程刚起来就退出（找不到入口文件）。

---

## 4. 先用「协议探针」独立验证服务器，再接入宿主

这一步是整个排错流程的关键：**在接入任何 AI 宿主之前，先确认这个 MCP 服务器自己能跑通**。这样后面出问题，可以立刻判断是「服务器坏了」还是「宿主配置错了」。

### 4.1 最小探针（一条命令，看能否 initialize）

```powershell
$env:EVERYTHING_HOST = '127.0.0.1'
$env:EVERYTHING_PORT = '80'      # 换成本机实际端口
'{"jsonrpc":"2.0","id":1,"method":"initialize","params":{"protocolVersion":"2024-11-05","capabilities":{},"clientInfo":{"name":"probe","version":"1.0"}}}' |
  node .\dist\index.js
```

期望：stderr 出现 `Everything MCP Server running on stdio`，stdout 返回包含 `serverInfo` 的 JSON。任何一项缺失都说明**服务器本身**没起来，先修这里，不要动宿主配置。

### 4.2 完整探针（四步握手 + 真实调用）

最小探针只验证了 `initialize`。要验证工具链，需要依次发 `initialize` → `notifications/initialized` → `tools/list` → `tools/call`。用脚本按行解析 stdout 的 JSON-RPC 最省事：

```js
// probe.js —— 不依赖宿主，直接按 MCP stdio 协议验证服务器
const { spawn } = require('child_process');
// 把下面两个路径替换成你自己的（见 README 的「路径写法」）
const NODE  = '<Node 安装目录>\\node.exe';
const ENTRY = '<MCP 根目录>\\everything-mcp\\dist\\index.js';
const p = spawn(NODE, [ENTRY], {
  env: { ...process.env, EVERYTHING_HOST: '127.0.0.1', EVERYTHING_PORT: '80' },
  stdio: ['pipe', 'pipe', 'pipe'],
});
p.stderr.on('data', (d) => process.stderr.write('[server] ' + d));
let buf = '';
p.stdout.on('data', (d) => {
  buf += d;
  let i;
  while ((i = buf.indexOf('\n')) >= 0) {
    const line = buf.slice(0, i).trim(); buf = buf.slice(i + 1);
    if (line) console.log('[<-]', line.slice(0, 400));
  }
});
const send = (o) => p.stdin.write(JSON.stringify(o) + '\n');
send({ jsonrpc: '2.0', id: 1, method: 'initialize',
       params: { protocolVersion: '2024-11-05', capabilities: {}, clientInfo: { name: 'probe', version: '1.0' } } });
setTimeout(() => send({ jsonrpc: '2.0', method: 'notifications/initialized' }), 300);
setTimeout(() => send({ jsonrpc: '2.0', id: 2, method: 'tools/list' }), 600);
setTimeout(() => send({ jsonrpc: '2.0', id: 3, method: 'tools/call',
       params: { name: 'everything_search', arguments: { query: 'ext:md', count: 3 } } }), 1200);
setTimeout(() => p.kill(), 3000);
```

```powershell
node probe.js
```

期望：`tools/list` 列出检索工具（原始工具名形如 `everything_search`，宿主侧再加前缀变成 `mcp__everything__everything_search`），`tools/call` 返回真实的文件结果。

**探针里的 `EVERYTHING_PORT` 必须和 MCP 配置里的一致**：探针通了而宿主不通 → 问题在宿主配置（命令路径 / env）；探针本身就不通 → 问题在 Everything 或项目构建。

> 探针脚本的通用形态在 [common-guide.md](common-guide.md#44-配置与排错通用提示) 里也记了一份；本机正是用它把「端口不一致」定位出来的。

---

## 5. 配置项（环境变量）

| 变量 | 默认 | 本机设置 | 作用 |
|---|---|---|---|
| `EVERYTHING_HOST` | `localhost` | `127.0.0.1` | Everything HTTP 服务器地址 |
| `EVERYTHING_PORT` | `54321` | **`80`** | 端口；**本机必须覆盖**，否则连不上（见 10.1） |
| `EVERYTHING_MAX_RESULTS` | `255` | `255` | 单次返回结果上限 |
| `PROJECT_PATH` | 空 | 未设置 | 默认项目作用域；每次调用也可用 `project_path` 参数覆盖它 |

---

## 6. 接入 DSH

### 6.1 装 MCP 客户端插件（每个 MCP 服务器只需一次）

```powershell
# 版本必须与 DSH 主版本对齐！见 common-guide.md 5.1
dsh plugin --profile desktop add @deepseek-ai/dsh-mcp-client@0.2.0-rc.2
```

版本对齐的原因、以及 `installed as a plain dependency, not a profile layer` 提示为何正常，见 [dsh-mcp-setup.md](dsh-mcp-setup.md#41-mcp-客户端插件版本必须与-dsh-主版本对齐)。

### 6.2 加 insert 条目

在 profile 补丁 `%USERPROFILE%\.dsh\profiles\<profile>\cordis.patch.yml` 末尾追加（**顶层是列表**，不要嵌套；`insert` 条目的机制说明见 [dsh-mcp-setup.md](dsh-mcp-setup.md#2-每加一个-mcp-服务器--加一个-insert-条目)）：

```yaml
- insert:
    - id: mcp-everything
      name: '@deepseek-ai/dsh-mcp-client'
      config:
        serverName: everything
        transport: stdio
        # 必须是真正的 node.exe；不要用宿主自带的包装脚本，原因见 [dsh-mcp-setup.md](dsh-mcp-setup.md#42-子进程必须用真正的解释器可执行文件)
        command: '<Node 安装目录>\node.exe'
        args:
          - '<MCP 根目录>\everything-mcp\dist\index.js'
        env:
          EVERYTHING_HOST: 127.0.0.1
          EVERYTHING_PORT: '80'
          EVERYTHING_MAX_RESULTS: '255'
        cwd: '<MCP 根目录>\everything-mcp'
        toolCallTimeoutMs: 60000
        reconnect:
          enabled: true
          initialDelayMs: 500
          maxDelayMs: 30000
          maxAttempts: 10
```

保存后 DSH 通常**热加载**（本机无需重启）：子进程被拉起，工具以 `mcp__<serverName>__<tool>` 暴露，即 **`mcp__everything__everything_search`**。

> 上面的 `command` / `args` / `cwd` 是本机真实绝对路径；换机器或迁移目录时**这三处都要改**（见第 11 节）。路径别用双引号（Windows 路径里的 `\U` 会被 YAML 当转义序列）。

---

## 7. 验收

### 7.1 子进程检查（应指向你配置的路径）

```powershell
Get-CimInstance Win32_Process -Filter "Name='node.exe'" |
  Where-Object CommandLine -match 'everything-mcp' |
  Select-Object ProcessId,CommandLine
```

本机实测：

```text
ProcessId   : 32260
CommandLine : <Node 安装目录>\node.exe <MCP 根目录>\everything-mcp\dist\index.js
```

**进程不存在**就回到第 4 节：先用探针证明服务器能跑，再检查 `command` 路径是否真实存在、`args` 指向的 `dist\index.js` 是否构建出来了。

### 7.2 工具调用实测

在会话里直接调用 `mcp__everything__everything_search`，例如 `query: "ext:md"`、`project_path` 指向 `<本仓库>`。本机实测输出：

```text
Found 5 results (showing 1-3) in 47ms

<本仓库>\bcompare-mcp.md   12.0 KB  2026-10-04T14:41:23.100Z
<本仓库>\common-guide.md   10.2 KB  2026-10-04T14:40:58.837Z
<本仓库>\everything-mcp.md 18.6 KB  2026-10-04T14:41:44.951Z
```

判定标准：**返回真实路径 + 大小 + 修改时间，且耗时在几十毫秒量级**。若报 `Cannot connect to Everything HTTP server at localhost:54321`，直接看 10.1。

---

## 8. 在本服务上继续验证的清单

- [ ] Everything 已完成 NTFS 索引，且 HTTP 服务器处于启用状态
- [ ] `curl.exe` 请求 Everything HTTP 返回 **200**，且含 `path` / `size` / `date_modified`
- [ ] 已确认 Everything **实际监听端口**（本机 `80`），配置里的 `EVERYTHING_PORT` 与之相同
- [ ] `Test-Path .\dist\index.js` 为 `True`（`npm run build` 已跑过）
- [ ] 第 4 节探针能独立跑通 `initialize` → `tools/list` → `tools/call`
- [ ] 子进程存在，且 `CommandLine` 指向正确的 node 与 `dist\index.js`
- [ ] `mcp__everything__everything_search` 能返回真实结果（几十毫秒）
- [ ] 用项目内作用域（`project_path`）搜一次，确认作用域参数生效
- [ ] 需要全盘检索时，确认能开启全盘范围并接受越界警告

---

## 9. 在其他 AI 工具里使用

everything-mcp 是**标准 stdio MCP 服务器**，与宿主无关（通用说明见 [common-guide.md](common-guide.md#2-mcp-配置的通用形态)）。通用接法就是「命令 + 参数 + 环境变量」三件套：

- `command`：**真实 node 可执行文件的绝对路径** → `<Node 安装目录>\node.exe`
- `args`：`["<MCP 根目录>\\\\everything-mcp\\\\dist\\\\index.js"]`
- `env`：`EVERYTHING_HOST=127.0.0.1`、`EVERYTHING_PORT=<你的端口>`（本机 `80`）、`EVERYTHING_MAX_RESULTS=255`

### 9.1 Claude Code

```powershell
claude mcp add --scope user everything-search -- <Node 安装目录>\node.exe <MCP 根目录>\everything-mcp\dist\index.js
```

`claude mcp add` 这条 CLI **不直接收 `env`**，所以端口这类变量需要另外注入：或者用带环境变量的包装脚本，或者直接改 Claude Code 的 MCP 配置（JSON 结构里支持 `env`）：

```json
{
  "mcpServers": {
    "everything-search": {
      "command": "<Node 安装目录>\\\\node.exe",
      "args": ["<MCP 根目录>\\\\everything-mcp\\\\dist\\\\index.js"],
      "env": { "EVERYTHING_HOST": "127.0.0.1", "EVERYTHING_PORT": "80" }
    }
  }
}
```

### 9.2 Cursor

写 `%USERPROFILE%\.cursor\mcp.json`（`mcpServers` 对象结构）：

```json
{
  "mcpServers": {
    "everything": {
      "command": "<Node 安装目录>\\\\node.exe",
      "args": ["<MCP 根目录>\\\\everything-mcp\\\\dist\\\\index.js"],
      "env": { "EVERYTHING_HOST": "127.0.0.1", "EVERYTHING_PORT": "80", "EVERYTHING_MAX_RESULTS": "255" }
    }
  }
}
```

### 9.3 要点

**必须用真实 node 的绝对路径 + 显式 env**：

- 不要写 `node` 裸命令（依赖 PATH，很多宿主清洗环境后 PATH 里没有 node）；
- 不要用宿主自带的包装脚本（如 DSH 的 `runtime\bin\node.cmd`）；
- Windows 路径在 JSON 里要用双反斜杠 `\\` 转义；
- `EVERYTHING_PORT` 必须显式写，别指望默认值 `54321` 对得上（见 10.1）。

只要宿主能用「真实 node 绝对路径 + 显式 env」，就能接上；其他 MCP 宿主把上面三件套填进它的 MCP 配置即可。

---

## 10. 本服务专属注意事项

### 10.1 端口不一致（最常见的失败原因）

everything-mcp **默认连 `54321`**，而 Everything 的端口可能被改成别的（本机是 **80**）。

- **症状**：工具调用报 `Cannot connect to Everything HTTP server at localhost:54321`。
- **原因**：配置里没设 `EVERYTHING_PORT`，服务器退回默认值 `54321`。
- **解决**：显式设 `EVERYTHING_PORT`。**先确认 Everything 实际监听端口（2.1），再写配置。**

探针不通时也要先怀疑这一条：探针里的 `EVERYTHING_PORT` 与 Everything 实际端口必须一致。

### 10.2 机器上可能存在多份 `Everything.ini` / `Everything.db`

便携版、或曾经迁移过安装位置的机器上，常常残留**旧的数据目录**（含 `Everything.db` + `Everything.ini`），而且旧 ini 里的 `http_server_enabled` 可能是 **`0`**，与正在运行的那份**恰好相反**。本机就踩到了：`D:\` 根目录下有一份 **72 MB 的旧数据（HTTP 服务为关）**，真正在 `<便携软件根目录>\` 下那份才是开着的。

**判断哪份是「活的」**：

```powershell
# 1) 谁在监听 Everything 的 HTTP 端口
Get-NetTCPConnection -State Listen -LocalPort 80 |
  Select-Object LocalAddress,LocalPort,OwningProcess
# 2) 哪些 ini 分别是什么状态（注意 LastWriteTime，越新越可能是活的那份）
Get-ChildItem -Path D:\ -Recurse -Depth 4 -Filter Everything.ini -ErrorAction SilentlyContinue |
  ForEach-Object { $en=(Select-String -Path $_.FullName -Pattern '^http_server_enabled').Line
                   $pt=(Select-String -Path $_.FullName -Pattern '^http_server_port').Line
                   "{0}  [{1} {2}]  改动={3}" -f $_.FullName,$en,$pt,$_.LastWriteTime }
# 3) 确认某份 db 是否被进程占用（决定能否安全删除）
$db='D:\旧目录\Everything.db'
try { $fs=[System.IO.File]::Open($db,'Open','ReadWrite','None'); $fs.Close(); '未被占用，可删' }
catch { '被占用：可能有 Everything 正在使用它' }
```

**要点**：`Everything.ini` 与 `Everything.db` 都跟**数据目录**走，而数据目录取决于启动方式（**便携版**把 ini 放在 exe 旁边；**安装版**通常在 `%APPDATA%\Everything`）。判断依据优先看第 1 步的**端口占用进程**，其次比对 ini 的 `LastWriteTime`，第 3 步用来确认某份 db 没被占用。**删旧数据目录前先跑上面三步**；`Run History.csv` 里是搜索历史，想保留就先备份。

### 10.3 `command` 必须是真正的 node 可执行文件

本机用 `<Node 安装目录>\node.exe`；**不要**用 DSH 自带的 `resources\runtime\bin\node.cmd`。症状：服务器注册了但一直 `server is disconnected`，且没有任何子进程。完整原因（桥接清洗 `DSH_*` 变量）与通用应对见 [common-guide.md](common-guide.md#42-mcp-子进程必须用真正的解释器可执行文件)。

### 10.4 其他

- **HTTP 服务器没开就是白配**：`http_server_enabled=0` 时，探针和工具调用都会连不上，表现与端口写错一样，两件事都查一遍。
- **带认证的 HTTP 服务器不被支持**：在 Everything 里设了用户名/密码，当前实现不会带认证头，需要改造源码。
- **`dist/` 是构建产物**：改动请改源码后重新 `npm run build`，不要手写 `dist/index.js`；也不要随意移动项目目录（配置里是绝对路径，见第 11 节）。

---

## 11. 更新与卸载（本服务）

**更新服务器本体**

```powershell
cd <MCP 根目录>\everything-mcp
git pull
npm install
npm run build
# 重启生效：改一下 cordis.patch.yml 触发热加载，或重启 DSH
```

只更新 MCP 客户端插件时用 `dsh plugin --profile desktop update @deepseek-ai/dsh-mcp-client@<与 DSH 对齐的版本>`；插件与通用的更新/卸载流程见 [common-guide.md](common-guide.md#5-更新与卸载)。

**改端口**（Everything 那边换了端口，或你想换）

1. 按 2.1 确认新端口；
2. 改 `cordis.patch.yml` 里该条目的 `EVERYTHING_PORT`；
3. 保存后热加载；探针（第 4 节）用新端口再验一次。

**迁移项目目录**：只需改 `cordis.patch.yml` 里该条目的 `args` / `cwd`（`command` 一般不动），保存后热加载生效，再用 7.1 确认子进程指向新路径。

**卸载**

```powershell
# 1) 从 cordis.patch.yml 删掉 id 为 mcp-everything 的 insert 条目，保存（热加载后工具消失）
# 2) 确认没有别的 MCP 服务器还要用它时，才移除客户端插件：
dsh plugin --profile desktop remove @deepseek-ai/dsh-mcp-client
# 3) 可选：删除项目本体（确认不再需要后再删）
#    Remove-Item '<MCP 根目录>\everything-mcp' -Recurse -Force
```

Everything 本身可以留着——它不依赖 MCP，照常当文件搜索工具用。

> 索引：[README.md](README.md)
