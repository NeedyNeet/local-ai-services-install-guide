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

## 从零开始：你需要准备什么

**本指南默认你只有一台 Windows 电脑**，下面每一样都给出获取方式。装哪套服务就装哪几样，不必全装。

### 前置软件

| 软件 | 哪些服务需要 | 获取方式 |
|---|---|---|
| **Git** | 全部（克隆项目本体） | `winget install --id Git.Git --exact` · 或 <https://git-scm.com/download/win> |
| **Node.js ≥ 18** | everything-mcp（≥18）、bcompare-mcp（≥24） | `winget install --id OpenJS.NodeJS --exact` · 或 <https://nodejs.org/> |
| **Python ≥ 3.10** | wiztree-mcp | `winget install --id Python.Python.3.14 --exact` · 或 <https://www.python.org/downloads/windows/> |
| **voidtools Everything** | everything-mcp | `winget install --id voidtools.Everything --exact` · 或 <https://www.voidtools.com/downloads/> |
| **WizTree** | wiztree-mcp | `winget install --id AntibodySoftware.WizTree --exact` · 或 <https://diskanalyzer.com/> |
| **Beyond Compare 5** | bcompare-mcp | `winget install --id ScooterSoftware.BeyondCompare.5 --exact` · 或 <https://www.scootersoftware.com/download> |
| **Bun ≥ 1.4.2** | bcompare-mcp（仅构建时需要） | `winget install --id Oven-sh.Bun --exact` · 或 <https://bun.sh/> |
| **Chrome 或 Edge**（Chromium 125+） | browser-skill | 系统一般已有；没有就装 <https://www.google.com/chrome/> 或 Edge |

**一次装齐**（只需其中几个就把不要的删掉）：

```powershell
# 逐个安装，无需管理员权限；装完必须重开终端，新命令才会进入 PATH
foreach ($id in 'Git.Git','OpenJS.NodeJS','Python.Python.3.14',
                'voidtools.Everything','AntibodySoftware.WizTree',
                'ScooterSoftware.BeyondCompare.5','Oven-sh.Bun') {
    winget install --id $id --exact --accept-package-agreements --accept-source-agreements
}
```

> **为什么用循环而不是一行塞多个 ID**：winget 的 `install` **不接受重复的 `--id`**（会报 `提供的参数超过允许的参数: "id"`），
> 所以只能用循环逐个装。上面的 `--accept-*` 参数用于免交互确认。

> **装完必须重开终端**：winget 修改 PATH 后，当前已打开的窗口读到的还是旧 PATH（winget 自己会提示 `已修改路径环境变量；重启 shell 以使用新值`）。
> 不重开的话，`node -v` / `python --version` 会报"找不到命令"——这不是装失败。

> 上表的 ID 在本机实测（winget 源，2026-10）：7 个 ID 均可解析、都有 x64 安装程序与 SHA256 校验，
> 并已用另一个包做过完整的"安装 → 验证可运行 → 卸载"端到端验证。
> 若某个 ID 搜不到（源同步差异），用右侧官网链接手动安装。

### 项目本体从哪来

这是**最容易被忽略的一步**——四套服务里有三套的"服务器本体"是别人的开源项目，需要先克隆到本地：

| 服务 | 项目本体 | 获取 |
|---|---|---|
| browser-skill | `Tencent/BrowserSkill` | **不需要克隆**——官方安装脚本会自动下载 CLI（见其文档第 2 节） |
| everything-mcp | `Josephur/everything-mcp` | `git clone https://github.com/Josephur/everything-mcp.git` |
| wiztree-mcp | `AlessandroBonomo28/wiztree-mcp` | `git clone https://github.com/AlessandroBonomo28/wiztree-mcp.git` |
| bcompare-mcp | `danielsimonjr/beyondcompare-mcp` | `git clone https://github.com/danielsimonjr/beyondcompare-mcp.git` |

> **注意分支名不统一**：有的项目默认分支是 `main`，有的是 `master`（如 wiztree-mcp）。
> **直接 `git clone` 不加 `-b` 就会自动用默认分支**，这也是推荐做法——别手写 `-b main`，否则遇到 `master` 的项目会失败。

克隆到哪里由你决定，文档里用 `<MCP 根目录>` 表示。建议放在一个专门的目录（不要放下载目录，会被清理）。
克隆下来后**必须按各服务文档完成构建/虚拟环境**，才能被宿主调用——源码不等于可运行。

### 就绪检查

```powershell
git --version          # 期望 2.x
node -v                # 期望 v18+（bcompare-mcp 要 v24+）
python --version       # 期望 3.10+（注意不要是 Microsoft Store 的占位 stub）
bun --version          # 只用 bcompare-mcp 时需要
```

前置软件本身能跑起来（如 Everything 能搜到文件、WizTree 能打开），再去装对应的 MCP 服务器——否则会在后面分不清是哪一层出的问题。

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
