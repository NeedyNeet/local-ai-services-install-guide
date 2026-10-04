# bcompare-mcp 安装与使用

用 **Beyond Compare** 做文件/文件夹比较、三方合并、目录同步的 MCP 服务器。

本机采用的是 **[danielsimonjr/beyondcompare-mcp](https://github.com/danielsimonjr/beyondcompare-mcp)**（TypeScript）。
另一个候选项目（`beyondcompare-mcp`，Python 版）**已评估并放弃**，原因见第 7 节——它的退出码判断是错的，核心比较功能不可用。

- 项目：`<MCP 根目录>\beyondcompare-mcp`
- 本机实测：服务器 `1.1.0`、Bun `1.4.2`、Node `v24.19.0`、Beyond Compare `5`（便携版）
- 工具前缀：`mcp__bcompare__*`（5 个）

> 通用部分（环境前置、DSH 挂载机制、跨工具方式、共用踩坑、更新卸载）见 [common-guide.md](common-guide.md)。

---

## 1. 前置条件

| 项 | 要求 | 本机实测 |
|---|---|---|
| 操作系统 | **仅 Windows** | Windows 10 x64 |
| Beyond Compare | **5**（项目 README 明确要求 BC5） | 便携版 `<Beyond Compare 安装目录>` |
| 构建工具 | **Bun ≥ 1.4.2**（`bun install` + `bun run build`） | Bun `1.4.2` |
| 运行时 | **Node ≥ 24** | `<Node 安装目录>\node.exe` v24.19.0 |
| 许可 | 试用版即可 | **试用/已激活均可，见第 8 节** |

### 1.1 关键：必须用 `BComp.com` 而不是 `BCompare.exe`

Beyond Compare 目录下有两个可执行文件，**这个选择决定整套服务能不能用**：

| 文件 | 性质 | 命令行行为 |
|---|---|---|
| `BCompare.exe` | **GUI 程序** | ❌ 从命令行调用时**不返回退出码**，还会留下常驻窗口进程 |
| `BComp.com` | **控制台程序** | ✅ 正确返回退出码，是 BC 官方推荐的脚本/自动化入口 |

本服务器的核心逻辑就是「读退出码 → 判定 SAME/SIMILAR/DIFFERENT」，所以**必须指向 `BComp.com`**。这是本机实测确认的（详见第 7 节的项目对比）。

---

## 2. 安装

```powershell
# 1) 克隆源码（该仓库默认分支是 main）
git clone https://github.com/danielsimonjr/beyondcompare-mcp.git '<MCP 根目录>\beyondcompare-mcp'

# 2) 安装 Bun（若尚无）
irm https://bun.sh/install.ps1 | iex
# 默认装到 %USERPROFILE%\.bun\bin\bun.exe
# 或用 winget：winget install Oven-sh.Bun

# 3) 装依赖
& "$env:USERPROFILE\.bun\bin\bun.exe" install --cwd '<MCP 根目录>\beyondcompare-mcp'

# 4) 类型检查 + 构建（产物 dist/index.js）
& "$env:USERPROFILE\.bun\bin\bun.exe" run --cwd '<MCP 根目录>\beyondcompare-mcp' typecheck
& "$env:USERPROFILE\.bun\bin\bun.exe" run --cwd '<MCP 根目录>\beyondcompare-mcp' build
```

> Bun 只在**构建**时需要；**运行**只需要 Node。若不想装 Bun，理论上可用 `npx tsc` 代替 `bun run build`（本项目 devDependency 里就有 typescript），但未实测。

构建后确认 `dist\index.js` 存在。

---

## 3. 先验证服务器本身，再接入宿主

用协议探针（`initialize` → `notifications/initialized` → `tools/list` → `tools/call`）独立验证，避免把「服务器问题」误判成「宿主配置问题」。关键是设好 `BCOMP_PATH`：

```powershell
$env:BCOMP_PATH = '<Beyond Compare 安装目录>\BComp.com'
node '<MCP 根目录>\beyondcompare-mcp\dist\index.js'
# 期望 stderr: Beyond Compare MCP server 1.1.0 running on stdio
```

本机探针实测（用两个临时文件比较）：

```
serverInfo: {"name":"beyondcompare-mcp","version":"1.1.0"}
tools: compare_files / compare_folders / merge_files / sync_folders / run_script

[相同文件]            Comparison: SAME        Exit code: 1 (Binary same)
[仅大小写不同的文件]   Comparison: SIMILAR     Exit code: 12 (Similar)
```

---

## 4. 接入 DSH

追加到 `%USERPROFILE%\.dsh\profiles\<profile>\cordis.patch.yml`（**先备份**）：

```yaml
- insert:
    - id: mcp-bcompare
      name: '@deepseek-ai/dsh-mcp-client'
      config:
        serverName: bcompare
        transport: stdio
        command: '<Node 安装目录>\node.exe'
        args:
          - '<MCP 根目录>\beyondcompare-mcp\dist\index.js'
        env:
          # 必须是 BComp.com（控制台版）；BCompare.exe 不返回退出码
          BCOMP_PATH: '<Beyond Compare 安装目录>\BComp.com'
        cwd: '<MCP 根目录>\beyondcompare-mcp'
        toolCallTimeoutMs: 181000
```

保存后热加载（本机实测：改动该条目内的值会重建子进程）。

验证：

```powershell
Get-CimInstance Win32_Process -Filter "Name='node.exe'" |
  Where-Object CommandLine -match 'beyondcompare' |
  Select-Object ProcessId,CommandLine
```

> **改动源码后必须重载子进程**：MCP 子进程在启动时缓存模块，重新 `bun run build` 之后不改配置的话，运行的仍是旧代码。

---

## 5. 环境变量

| 变量 | 默认 | 本机取值 | 作用 |
|---|---|---|---|
| `BCOMP_PATH` | `%LOCALAPPDATA%\Programs\Beyond Compare 5\BComp.com` | `<Beyond Compare 安装目录>\BComp.com` | **必设**，自动探测只找 `LOCALAPPDATA` 下的默认安装位置，认不出便携版 |

---

## 6. 工具一览（5 个）

| 工具 | 用途 |
|---|---|
| `compare_files` | 比较两个文件，返回 `SAME` / `SIMILAR` / `DIFFERENT` / `ERROR`；可选 `fileViewType`（text/hex/table/mp3/picture/registry/version）、`silent`（默认 true）、`readOnly` |
| `compare_folders` | 比较两个文件夹；`silent: true` 生成结构化差异报告，`silent: false` 打开 GUI；支持 `criteria`（binary/rules-based/timestamp/size/CRC）、`filters`、`showMatches` |
| `merge_files` | 三方合并（left/right/center → output），`automerge` 可自动合并非冲突修改 |
| `sync_folders` | 目录同步：`mode`（update 非破坏性 / mirror 镜像可能删除）、`direction`（left->right / right->left / all）、**`dryRun` 预览** |
| `run_script` | 执行 Beyond Compare 脚本文件，做批量比较/同步/合并 |

### 6.1 退出码语义（判定等级的依据）

Beyond Compare 的退出码比"0=相同 / 1=不同"细得多，项目内置了完整映射：

| 退出码 | 含义 | 判定 |
|---|---|---|
| 0 | Success | SAME |
| 1 | Binary same | SAME |
| 2 | Rules-based same | SAME |
| 11 | Binary differences | DIFFERENT |
| 12 | **Similar** | SIMILAR |
| 13 | Rules-based differences | DIFFERENT |
| 14 | Conflicts detected | ERROR |
| 100 | Error | ERROR |
| 101 | Conflicts detected, merge output not saved | ERROR |

`SIMILAR`（12）夹在 11–13 之间，`verdictFor()` 特意先判 12，否则会被当成 DIFFERENT。

**本机实测确认**：两个文件内容相同但**行尾不同**时返回 `12 (Similar)`，完全相同的返回 `1 (Binary same)`，内容不同的返回 `13 (Rules-based differences)`，文件不存在返回 `100 (Error)`。

---

## 7. 为什么用这个项目，而不用另一个（实测对比）

本机曾同时评估两个同名项目，结论基于实测而非文档：

| 维度 | `beyondcompare-mcp`（Python/FastMCP） | **`beyondcompare-mcp`（TypeScript/Bun）** |
|---|---|---|
| 调用的可执行文件 | ❌ **`BCompare.exe`（GUI 版）** → 实测不返回退出码、残留进程 | ✅ **`BComp.com`（控制台版）** |
| 退出码处理 | ❌ 假定「0=相同 / 1=有差异 / ≥2=错误」，与 BC 实际语义不符 | ✅ 完整 10 项映射 + `verdictFor()` |
| 路径探测 | ❌ 只找 `C:\Program Files\...` | ⚠️ 只找 `LOCALAPPDATA\...`，但可用 `BCOMP_PATH` 覆盖 |
| 工具数 | 13 个（含多媒体扫描、仓库备份、重复文件清理等与 BC 无关的附加功能） | **5 个聚焦工具** |
| 体积 | 276 文件 / 2.9 MB | **22 文件 / 0.1 MB** |
| 工程状态 | 目录里散落 `*_COMPLETE.md`、`*.bak-*`、`ANTIGRAVITY_*.md` 等过程文档 | 干净（src 6 文件 + tests + CHANGELOG），注释解释到位 |

**决定性问题**：Python 项目把「有差异」（退出码 11/13）当错误抛出，同时把「相同」（1/2）当成有差异——即它的核心判定逻辑与实际退出码语义不兼容。这不是试用版或许可问题，是代码缺陷。

---

## 8. 本服务专属注意事项

### 8.1 关于试用版 / 未激活许可：不影响使用

本机在**试用状态下**完成了全部验证（7 次调用，含文件夹比较、脚本执行、三方合并），结论：

- **命令行接口与许可状态无关**。Beyond Compare 试用到期后是「持续提醒 + 停用右键菜单集成」，命令行比较/同步照常工作。
- **弹窗风险已被抑制**：项目调用时传 `/silent`，BC 文档中该参数正是用于抑制对话框与消息框。
- **有超时兜底**：`executeBComp` 默认 60 秒、脚本执行 120 秒超时，即使真卡住也只会返回超时错误，不会挂死服务器。
- **"定期重置试用"不影响本服务配置**——重置的是许可状态，与 `BCOMP_PATH`、工具调用无关。

### 8.2 已修复的缺陷：文件夹报告的"仅一侧存在"被误判为差异（本机修复）

**症状**：`compare_folders` 的汇总行把「仅存在于一侧」的文件计入 `different`，且 `left-only` / `right-only` 恒为 0。明细行本身正确，但汇总数字会误导（做同步前按汇总判断会高估差异量）。

**根因**（已取证）：Beyond Compare 5 写出的 XML 状态值是 **`ltonly` / `rtonly`**，而解析器判断的是 `status === "left"` / `"right"`。不匹配 → 落入"未识别状态"兜底分支 → 计入 `diff` 并以大写原样输出（这就是输出里出现 `LTONLY:` / `RTONLY:` 字样的原因）。

用 `folder-report layout:xml` 导出的真实 XML 取证：

```
<filecomp status="same">   <lt><name>one.txt</name></lt><rt><name>one.txt</name></rt>
<filecomp status="ltonly"> <lt><name>three.txt</name></lt>
<filecomp status="diff">   <lt><name>two.txt</name></lt><rt><name>two.txt</name></rt>
<filecomp status="rtonly"> <rt><name>four.txt</name></rt>
```

**为什么原来的测试没发现**：`tests/report.test.ts` 用的状态值是编造的 `left`/`right`（BC 从不写出这两个值），所以测试全绿而真实报告出错。

**修复**（本机已改并验证）：`src/report.ts` 同时接受两种写法，并补了基于真实 XML 的回归测试。

```ts
} else if (status === "left" || status === "ltonly") {
  leftOnly++;
  lines.push(`  LEFT ONLY: ${name}`);
} else if (status === "right" || status === "rtonly") {
  rightOnly++;
  lines.push(`  RIGHT ONLY: ${name}`);
```

修复前后对比（同一份真实数据，A/B 两目录：1 相同、1 不同、1 左独有、1 右独有）：

| | 修复前 | 修复后 | BC 原生报告 |
|---|---|---|---|
| same | 1 | **1** | 1 |
| different | **3** ❌ | **1** ✅ | 1 |
| left-only | **0** ❌ | **1** ✅ | 1 |
| right-only | **0** ❌ | **1** ✅ | 1 |

测试从 18 个增至 20 个（含 1 条真实报告回归测试 + 1 条向后兼容测试），typecheck 0 错误。

> 改动文件：`src/report.ts`、`tests/report.test.ts`；改动前的备份为同目录 `.bak-before-ltonly-fix`，便于回滚。
> 本项目未提交上游，属本机修改——**升级上游时注意这几处会被覆盖**。

### 8.3 其他

- **`merge_files` 报冲突（101）通常是正确行为**：BC 会把相邻修改视作同一 hunk，左右各改一行即构成冲突。重点是它**拒绝写出半成品文件**，退出码语义准确。
- **`sync_folders` 用 `mirror` 模式可能删除目标侧文件**，执行前先用 `dryRun: true` 预览。
- **改动源码后要重载子进程**（见第 4 节），否则跑的还是旧构建。

---

## 9. 更新与卸载

**更新**

```powershell
cd <MCP 根目录>\beyondcompare-mcp
git pull
& "$env:USERPROFILE\.bun\bin\bun.exe" install
& "$env:USERPROFILE\.bun\bin\bun.exe" run build
# 然后重载 MCP 子进程（改一下 cordis.patch.yml 里该条目的值，或重启 DSH）
```

> 升级上游前先看第 8.2 节的修复是否被覆盖；若被覆盖，需重新应用。

**卸载**

```powershell
# 1) 从 cordis.patch.yml 删掉 id: mcp-bcompare 条目（保存后热加载，子进程退出）
# 2) 可选：删除项目本体
Remove-Item '<MCP 根目录>\beyondcompare-mcp' -Recurse -Force
# 3) 可选：卸载 Bun（仅当没有其他项目用它）
Remove-Item "$env:USERPROFILE\.bun" -Recurse -Force
```

> 索引：[README.md](README.md)
