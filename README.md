# dsh-local-services

本机（Windows）为 AI 代理配置的**本地能力服务**文档集合。目标：可以照着重装、也可以直接分享给别人。

**每套服务一份独立文档**——新增服务只加一个文件，不动其他文档，避免单文件越来越长。

## 先看这里：哪些是通用的，哪些是 DSH 专用的

本仓库是**「个人配置记录 + 可复用文档」的混合体**。分享或参考时请按下表区分——**不要把 DSH 专用内容当作通用教程**。

| 内容 | 性质 | 别人能用吗 |
|---|---|---|
| [project-conventions.md](project-conventions.md) 第 1–5、7、8 节 | 🟢 **通用** | ✅ git 配置、项目规范分层、目录结构、常见坑、决策表——与 DSH 无关，任何 Windows 开发者都适用 |
| [project-skeleton](../\_templates/project-skeleton/)（骨架与一键脚本） | 🟢 **通用** | ✅ 唯一前置是 git 和 PowerShell |
| [\_templates/AGENTS.md](../\_templates/AGENTS.md) | 🟢 **通用** | ✅ 只要你用支持 `AGENTS.md` 约定的 AI 工具（Claude Code / Cursor / Codex / DSH 等） |
| [browser-skill.md](browser-skill.md) 的 CLI 安装、扩展安装、`bsk doctor`、`bsk install-skill` | 🟢 **通用** | ✅ 官方工具本身的用法，与宿主无关 |
| [bcompare-mcp.md](bcompare-mcp.md) 第 7 节项目对比、8.2 缺陷修复 | 🟢 **通用** | ✅ 上游的真实 bug 与退出码语义，任何人都会踩 |
| [common-guide.md](common-guide.md) 的 DSH 挂载机制（`cordis.patch.yml` + `@deepseek-ai/dsh-mcp-client`） | 🔵 **DSH 专用** | ⚠️ 只有 DSH 用户用得上；其他宿主见该文第 4 节 |
| 各服务文档的「接入 DSH」章节 | 🔵 **DSH 专用** | ⚠️ 同上 |
| `browser_*` / `bsk` / `mcp__*` 工具名、`DSH_*` 变量清洗、沙箱 IPC | 🔵 **DSH 专用** | ⚠️ DSH 实现细节 |
| 各服务的「安装步骤 / 验证 / 踩坑」 | 🟡 **混合** | 前置条件、构建命令、协议探针验证是通用的；接入配置是 DSH 专用的 |
| 第 3 节的路径规范与占位符表 | 🟡 **本机约定** | 是「本机的目录习惯」而非通用要求，可改成自己的布局 |

> 路径写法：全仓库用 `<Dev 根目录>`、`<MCP 根目录>` 等占位符表示本机位置，对照表见 [common-guide.md](common-guide.md#31-占位符对照本机实际值)。

## 服务一览

| 服务 | 作用 | 调用形态 | 文档 |
|---|---|---|---|
| **browser-skill** | 用你**已登录**的 Chrome / Edge 干活：读页面、点按、填表、截图、调试 | DSH 原生插件（6 个 `browser_*` 工具）；其他 CLI 型宿主用 `bsk install-skill` | [browser-skill.md](browser-skill.md) |
| **everything-mcp** | voidtools Everything 的全盘 NTFS 索引 → **毫秒级**文件检索 | stdio MCP 服务器，工具名 `mcp__everything__everything_search` | [everything-mcp.md](everything-mcp.md) |
| **wiztree-mcp** | WizTree 磁盘占用分析：最大文件/文件夹、类型分布、重复文件、树状图 | stdio MCP 服务器，工具名 `mcp__wiztree__*`（17 个） | [wiztree-mcp.md](wiztree-mcp.md) |
| **bcompare-mcp** | Beyond Compare 文件/文件夹比较、三方合并、目录同步 | stdio MCP 服务器，工具名 `mcp__bcompare__*`（5 个） | [bcompare-mcp.md](bcompare-mcp.md) |

四者互相独立，可单独安装。**除 browser-skill 外，其余三个都是标准 MCP 服务器**，其他宿主也能接（接法见 [common-guide.md](common-guide.md#4-在其他-ai-工具里使用)）。

## 建议的阅读顺序

1. **[common-guide.md](common-guide.md)** —— 通用部分：环境前置、DSH 挂载 MCP 的机制、跨工具接入方式、**共用踩坑**（首次部署必读，很多坑能一次避开）
2. 对应服务的独立文档（上面表格里的链接）
3. **[AGENTS.md](AGENTS.md)** —— 给 AI 代理的调用约定：何时优先用这些服务、调用约束

## 文件

| 文件 | 说明 |
|---|---|
| [common-guide.md](common-guide.md) | 通用指南：前置条件、DSH MCP 挂载机制、其他 AI 工具接入、**共用注意事项**、更新卸载、路径规范与占位符对照 |
| [browser-skill.md](browser-skill.md) | browser-skill 安装与使用 |
| [everything-mcp.md](everything-mcp.md) | everything-mcp 安装与使用 |
| [wiztree-mcp.md](wiztree-mcp.md) | wiztree-mcp 安装与使用 |
| [bcompare-mcp.md](bcompare-mcp.md) | bcompare-mcp 安装与使用（含两个候选项目的对比结论、已修复缺陷说明） |
| [project-conventions.md](project-conventions.md) | **新建项目时看这个**：路径规范、git 配置、项目规范分层、目录结构、提效手段、常见坑（**通用性最高的一份**） |
| [AGENTS.md](AGENTS.md) | AI 代理工作区约定 |
| [CONTRIBUTING.md](CONTRIBUTING.md) | 反馈与提交修改的约定（含文档写法约定：路径用占位符、坑要写清症状/原因/解决） |
| [LICENSE](LICENSE) | MIT |
| [.gitattributes](.gitattributes) | 行尾规则（仓库内统一 LF，脚本保留 CRLF） |
| [.gitignore](.gitignore) | 文档仓库不进任何构建产物与备份文件 |
| [README.md](README.md) | 本文件：索引 |

> 注：独立的**通用骨架**不放在本仓库内，而在 `<Dev 根目录>\Github\_templates\`——
> 它被设计成可单独复制/发布，不依赖本仓库。

## 相关目录

| 路径 | 内容 |
|---|---|
| `<MCP 根目录>\everything-mcp` | everything-mcp 项目本体（TypeScript，需 `npm run build`；运行时用 `dist/index.js`） |
| `<MCP 根目录>\wiztree-mcp` | wiztree-mcp 项目本体（Python 单文件；运行时用 `.venv\Scripts\python.exe`） |
| `<MCP 根目录>\beyondcompare-mcp` | bcompare-mcp 项目本体（TypeScript，`bun run build`；运行时用 `dist/index.js`）。**已放弃的候选项目**（同名 Python 版，退出码判断有误）已从磁盘删除，对比结论见 [bcompare-mcp.md](bcompare-mcp.md#7-为什么用这个项目而不用另一个实测对比) |
| `<便携软件根目录>\` | 便携软件（Everything、WizTree、Beyond Compare） |
| `%USERPROFILE%\.dsh\profiles\desktop\cordis.patch.yml` | 所有服务的实际生效配置（DSH profile 补丁层） |
| `%USERPROFILE%\.local\bin\bsk.exe` | bsk CLI |
| `%USERPROFILE%\.bsk\` | bsk daemon 数据与日志 |
| `%USERPROFILE%\.bun\bin\bun.exe` | Bun（构建 TypeScript 型 MCP 服务器用） |
| `<Dev 根目录>\Github\_templates\` | 新项目起步模板（复制 `AGENTS.md` 到项目根目录即继承本机服务约定） |

## 快速验证（四套服务）

```powershell
# 1) browser-skill：应无 fail 项（N/A 不算失败）
& "$env:USERPROFILE\.local\bin\bsk.exe" doctor

# 2) 所有 MCP 子进程是否都在运行
Get-CimInstance Win32_Process -Filter "Name='node.exe' OR Name='python.exe'" |
  Where-Object CommandLine -match 'everything-mcp|wiztree|beyondcompare' |
  Select-Object ProcessId,CommandLine

# 3) everything-mcp 前置：Everything 的 HTTP 服务应返回 200
curl.exe -sS "http://127.0.0.1:80/?search=test&json=1&count=3"
```

再在 AI 会话里各调一次工具（`mcp__everything__everything_search`、`mcp__wiztree__wiztree_info`、`mcp__bcompare__compare_files`、`browser_session list`）即可确认端到端可用。

## 本机实测环境

Windows x64 · Node `v24.19.0` · Bun `1.4.2` · Python `3.14.6`（venv）· DSH `0.2.0-rc.2` ·
bsk CLI `0.3.2` · 浏览器扩展 `0.3.2`（Chrome / Edge 154）·
Everything `1.4.1.1026 x64` 便携版（HTTP 端口 **80**）· WizTree `4.31.0.0` 便携版 ·
Beyond Compare `5` 便携版（BComp.com）· MCP 客户端 `@deepseek-ai/dsh-mcp-client@0.2.0-rc.2`
