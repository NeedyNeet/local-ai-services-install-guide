# 四套 MCP / 浏览器自动化服务 · 安装集成指南

把下面四套能大幅提升 AI 代理干活能力的服务一次装好，**不用一个一个去搜、去踩坑**。

面向 **Windows**（四套里有三套依赖 Windows 软件）。每一套都独立，**可以只装你要的**。

| 服务 | 它让 AI 能做什么 | 前置软件 | 文档 |
|---|---|---|---|
| **browser-skill** | 在你**已登录**的 Chrome / Edge 里读页面、点按钮、填表单、截图、调试网页 | 无（自带 CLI） | [browser-skill.md](browser-skill.md) |
| **everything-mcp** | **毫秒级**全盘文件检索（按名字、路径、扩展名、时间、大小） | voidtools Everything | [everything-mcp.md](everything-mcp.md) |
| **wiztree-mcp** | 分析磁盘占用：最大的文件/文件夹、类型分布、重复文件、树状图 | WizTree | [wiztree-mcp.md](wiztree-mcp.md) |
| **bcompare-mcp** | 文件/文件夹比较、三方合并、目录同步 | Beyond Compare 5 | [bcompare-mcp.md](bcompare-mcp.md) |

没有 IDE 插件、没有云服务、不需要账号——全部在本机运行，用的是你现有的软件与登录状态。

---

## 我应该装哪个

| 你的痛点 | 装这个 |
|---|---|
| "AI 帮我找一下那个文件在哪" | everything-mcp |
| "C 盘满了，看看什么占空间" | wiztree-mcp |
| "这个改动影响了哪些文件 / 帮我把两个目录同步一下" | bcompare-mcp |
| "帮我打开这个网页看看 / 帮我填个表单 / 抓一下页面内容" | browser-skill |
| 全都想要 | 四个都装，它们互不冲突 |

---

## 快速开始（以 everything-mcp 为例）

四套服务的流程是同一套：

```text
1. 装好前置软件（如 Everything）并确认它能用
2. 装/构建 MCP 服务器本体
3. 先用协议探针确认服务器自己能跑通
4. 填进你的 MCP 客户端配置（三件套：command + args + env）
5. 在 AI 里调一次工具，确认端到端可用
```

第 4 步是最容易卡住的地方：**三套 MCP 服务器都是标准的 stdio 服务器，与宿主无关**，填的都是「解释器绝对路径 + 脚本绝对路径 + 环境变量」。
通用写法（含 Claude Code / Cursor / 其他宿主的示例）见 **[common-guide.md 第 2 节](common-guide.md#2-mcp-配置的通用形态)**。

> 如果你用 **DeepSeek Harness（DSH）**：它不用 `mcpServers` 那种 JSON，而是 YAML 里的 `insert` 条目 —— 见 **[dsh-mcp-setup.md](dsh-mcp-setup.md)**。

---

## 建议的阅读顺序

1. **[common-guide.md](common-guide.md)** —— 通用部分：环境前置、MCP 配置的通用形态、验证方法、**共用踩坑**（先读这个，很多坑能一次避开）
2. 你要装的那套服务的独立文档（上面表格里的链接）
3. DSH 用户额外看 [dsh-mcp-setup.md](dsh-mcp-setup.md)

每份服务文档的结构都一样：

- 前置条件 → 安装 → **先验证服务器本身** → 环境变量 → 接入宿主（含 DSH 与宿主无关两种） → 工具一览 → **本服务专属注意事项** → 更新与卸载

**"本服务专属注意事项"是最值得看的一节**——那里是真实踩过的坑（端口不一致、路径探测失败、权限不足导致扫描缓慢、退出码语义误解等），每条都写清了症状、原因和解决方式。

---

## 几个跨服务的关键提醒

| 提醒 | 说明 |
|---|---|
| **MCP 子进程要用真正的解释器绝对路径** | 不要依赖 PATH 或"已激活的虚拟环境"——MCP 客户端启动子进程时环境往往是干净的 |
| **先确认服务端实际情况，再写配置** | 例如 Everything 的 HTTP 端口、WizTree / Beyond Compare 的 exe 位置，MCP 侧的默认值不一定和你的实际一致 |
| **注册成功 ≠ 连接成功** | 工具名出现了但调用报 `server is disconnected`，通常是子进程没起来 → 先查进程是否存在 |
| **接入宿主前先用协议探针验证** | 这样出错时能立刻分清是"服务器问题"还是"宿主配置问题" |
| **路径因机器而异** | 本文档用 `<Node 安装目录>` 这类占位符，替换成你自己的路径 |

---

## 文件说明

| 文件 | 内容 |
|---|---|
| [common-guide.md](common-guide.md) | 通用指南：环境前置、MCP 配置通用形态、验证方法、**共用注意事项**、更新卸载 |
| [browser-skill.md](browser-skill.md) | 浏览器自动化：CLI 安装、扩展安装、daemon、技能安装（9 个宿主）、专属坑 |
| [everything-mcp.md](everything-mcp.md) | 文件检索：构建、协议探针、端口配置、多份 ini 的判断、专属坑 |
| [wiztree-mcp.md](wiztree-mcp.md) | 磁盘分析：venv 安装、17 个工具、管理员权限与扫描性能、GUI 协作 |
| [bcompare-mcp.md](bcompare-mcp.md) | 文件比较：**必须用 `BComp.com` 而非 `BCompare.exe`**、退出码语义、两个候选项目对比、已修复缺陷 |
| [dsh-mcp-setup.md](dsh-mcp-setup.md) | 🔵 **DSH 专用**：`cordis.patch.yml` 里怎么挂 MCP 服务器、DSH 特有的坑 |
| [LICENSE](LICENSE) | MIT |

---

## 实测环境

文档里的命令与输出都在以下环境验证过（作为参考，**你的版本可能不同**）：

| 组件 | 版本 |
|---|---|
| 操作系统 | Windows 10 x64 |
| Node.js | v24.19.0 |
| Python | 3.14（venv 内 3.14.6） |
| browser-skill CLI / 扩展 | 0.3.2 |
| Chrome / Edge | 154 |
| Everything | 1.4.1.1026 x64（便携版） |
| WizTree | 4.31.0.0（便携版） |
| Beyond Compare | 5（便携版） |
| MCP 宿主 | DeepSeek Harness 0.2.0-rc.2 |
| 最后更新 | 2026-10-04 |

---

## 反馈

发现文档里的**事实错误**（命令写错、退出码含义不对、步骤失效）或**有更好的办法**，欢迎提 issue / PR。
如果你只是目录结构或版本与我不同，那不算错误——本文档的路径都用占位符表示，替换成你自己的即可。
