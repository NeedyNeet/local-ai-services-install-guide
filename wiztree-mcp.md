# wiztree-mcp 安装与使用

一个用 Python 写的 **MCP 服务器**，驱动 [WizTree](https://antibody-software.com/wiztree)（Windows 磁盘空间分析器）来回答"什么占了空间""哪里能清理"。第一次扫描生成 CSV **快照**并缓存，后续所有查询都在快照上流式读取——**不重复扫盘**。

- 项目：`<MCP 根目录>\wiztree-mcp`（单文件 `wiztree_mcp.py`）
- 本机实测：WizTree `4.31.0.0` 便携版、Python `3.14.6`、服务器 `wiztree-mcp 1.0.0`
- 工具前缀：`mcp__wiztree__*`（共 17 个）

> 通用部分（环境前置、DSH 挂载机制、跨工具方式、共用踩坑、更新卸载）见 [common-guide.md](common-guide.md)。

---

## 1. 前置条件

| 项 | 要求 | 本机实测 |
|---|---|---|
| 操作系统 | Windows（WizTree 是 Windows 程序） | Windows 10 x64 |
| Python | **≥ 3.10** | `<Python 安装目录>\python.exe`（3.14.7）→ venv 内 3.14.6 |
| WizTree | 便携版或安装版均可 | 便携版 `<WizTree 安装目录>\WizTree64.exe` |
| 依赖 | 仅 `mcp>=2.0.0,<3` | 已装入 venv |

**Python 的选择很重要**：系统 `python` 可能是 Microsoft Store 的占位 stub（`WindowsApps\python.exe`），不要用它建 venv。用真实安装路径，或用 `py -3.xx` 指定版本。

WizTree 的 exe 不需要在 PATH 里——由 `WIZTREE_EXE` 显式指定（见 4）。服务器自己会尝试自动发现（项目在便携版目录内时，或装在 `Program Files` 时），本机是显式指定的。

---

## 2. 安装

```powershell
# 1) 建虚拟环境（本机用真实安装的 Python，不是 Store stub）
& '<Python 安装目录>\python.exe' -m venv '<MCP 根目录>\wiztree-mcp\.venv'

# 2) 装依赖（requirements.txt 内容：mcp>=2.0.0,<3）
& '<MCP 根目录>\wiztree-mcp\.venv\Scripts\python.exe' -m pip install -r '<MCP 根目录>\wiztree-mcp\requirements.txt'

# 3) 确认解释器可用
& '<MCP 根目录>\wiztree-mcp\.venv\Scripts\python.exe' --version
```

> 装依赖时如遇网络问题，给 pip 加镜像源（例如 `-i https://pypi.tuna.tsinghua.edu.cn/simple`）。

**关键点**：配置里必须指向 **`.venv\Scripts\python.exe`**，不要指向系统 python，也不要指望"先激活虚拟环境"——MCP 桥接会清洗环境变量（见 [common-guide.md](common-guide.md#52-子进程必须用真正的解释器可执行文件)）。

---

## 3. 先验证服务器本身，再接入宿主

装完先不碰 DSH，直接跑一次自检——这样后面出错就能立刻判断是"服务器问题"还是"宿主配置问题"：

```powershell
$env:WIZTREE_EXE = '<WizTree 安装目录>\WizTree64.exe'
& '<MCP 根目录>\wiztree-mcp\.venv\Scripts\python.exe' '<MCP 根目录>\wiztree-mcp\wiztree_mcp.py'
```

正常时 stdout 会出现一行就绪信息（本机实测）：

```
wiztree-mcp 1.0.0 ready (cache: C:\Users\<你的用户名>\AppData\Local\wiztree-mcp)
```

想进一步验证工具调用，可用与 everything-mcp 相同的**协议探针**思路：`initialize` → `notifications/initialized` → `tools/list` → `tools/call`，按行解析 stdout 的 JSON-RPC。

---

## 4. 环境变量

| 变量 | 默认 | 本机取值 | 作用 |
|---|---|---|---|
| `WIZTREE_EXE` | 自动发现 | `<WizTree 安装目录>\WizTree64.exe` | `WizTree64.exe` 的完整路径 |
| `WIZTREE_DIR` | — | 未用 | exe 所在目录，`WIZTREE_EXE` 的替代写法 |
| `WIZTREE_MCP_CACHE` | `%LOCALAPPDATA%\wiztree-mcp` | 默认 | 快照 CSV 与树状图的存放目录 |
| `WIZTREE_MCP_MAX_SCANS` | `12` | 默认 | 保留多少个快照，超出后删最旧的 |

---

## 5. 接入 DSH

追加到 `%USERPROFILE%\.dsh\profiles\<profile>\cordis.patch.yml`（**先备份**）：

```yaml
- insert:
    - id: mcp-wiztree
      name: '@deepseek-ai/dsh-mcp-client'
      config:
        serverName: wiztree
        transport: stdio
        # 必须指向 .venv 里的 python：MCP 子进程环境会被清洗，无法依赖 PATH 或激活的虚拟环境
        command: '<MCP 根目录>\wiztree-mcp\.venv\Scripts\python.exe'
        args:
          - '<MCP 根目录>\wiztree-mcp\wiztree_mcp.py'
        env:
          WIZTREE_EXE: '<WizTree 安装目录>\WizTree64.exe'
        # 默认 60000ms 对全盘扫描太短（无管理员权限时可能数分钟）
        toolCallTimeoutMs: 900000
```

保存即热加载（无需重启 DSH），子进程会被拉起。

> **项目自带的 `dsh-cordis-patch.yml` 里路径是占位符**（`C:\path\to\...`），不能直接用，只适合当模板参考。注意里面提到的写法要点：Windows 路径**不要用双引号**（`"C:\Users\..."` 的 `\U` 会被 YAML 当转义序列而解析失败），用单引号或裸写。

验证：

```powershell
# 子进程应指向你配置里的 venv python 与脚本
Get-CimInstance Win32_Process -Filter "Name='python.exe'" |
  Where-Object CommandLine -match 'wiztree' | Select-Object ProcessId,ParentProcessId,CommandLine
```

然后在会话里调 `mcp__wiztree__wiztree_info`（本机实测输出）：

```
WizTree executable : <WizTree 安装目录>\WizTree64.exe
WizTree version    : 4.31.0.0
Elevated process   : no - whole-volume scans fall back to a slow directory walk unless admin=true is passed (UAC prompt)
Snapshot cache     : C:\Users\<你的用户名>\AppData\Local\wiztree-mcp
Cached snapshots   : 0
Server version     : wiztree-mcp 1.0.0 on Python 3.14.6
```

再做一次真实扫描确认端到端（本机用 `<MCP 根目录>\everything-mcp` 试的）：

```
Snapshot e637a0169eca | root <MCP 根目录>\everything-mcp | 4,354 rows | fresh, scan took 973 ms
Total size : 39.51 MB     On disk : 44.82 MB allocated
Contents   : 3,775 files in 578 folders
```

---

## 6. 工具一览（17 个）

| 工具 | 用途 |
|---|---|
| `wiztree_info` | 排错第一步：实际使用的 exe、版本、是否提权、缓存位置 |
| `wiztree_list_drives` | 各盘容量、已用、可用 |
| `wiztree_scan` | 扫描盘/文件夹并生成快照；返回总量、卷剩余空间、最大的顶层项 |
| `wiztree_top_files` | 最大文件，可按子目录 / 扩展名 / 最小体积过滤 |
| `wiztree_top_folders` | 最大文件夹（递归体积），`max_depth` 控制层级 |
| `wiztree_folder_breakdown` | **逐层下钻**：看某文件夹内部各项及其占比 |
| `wiztree_file_types` | 按扩展名汇总占用 |
| `wiztree_search` | 按 glob / 关键词 / 正则搜索，按体积排序 |
| `wiztree_duplicates` | 重复文件与可回收空间（只读同体积的文件，很快） |
| `wiztree_treemap` | 生成 PNG 树状图；`return_image=true` 直接返回图片 |
| `wiztree_export_csv` | 完整 CSV 导出，可用全部 WizTree CLI 选项 |
| `wiztree_gui_state` | 读**用户当前开着的** WizTree 窗口指向哪个目标 |
| `wiztree_open_gui` | 打开 WizTree 窗口让用户自己看 |
| `wiztree_import_csv` | 把 GUI 导出的 CSV 注册为快照（免重扫，且能继承管理员扫描） |
| `wiztree_list_scans` | 查看已缓存的快照 |
| `wiztree_clear_cache` | 清理缓存 |

> 所有查询类工具都接受 `path` / `refresh` / `max_age_minutes` / `admin` / `filter` / `filter_exclude` / `timeout_seconds`，**找不到有效快照时会自己扫**，所以不必先调 `wiztree_scan`。

---

## 7. 本服务专属注意事项

### 7.1 性能：不加管理员权限，整盘扫描会很慢

WizTree 读 NTFS 的 **MFT** 是秒级完成全盘扫描的前提，而读 MFT **需要管理员权限**。

- **未提权**（默认，也是本机现状 `Elevated process: no`）：退化为递归目录遍历，**整盘（如 `C:\`）可能数分钟**；单个文件夹仍然很快（约 50 万文件 5 秒）。
- **`admin: true`**：WizTree 会以提权重启，走 MFT，秒级——但会**弹 UAC 提示，需要人工确认**。

因此**优先扫描具体文件夹，而不是整盘**。`wiztree_info` 会告诉你当前处于哪种状态。

### 7.2 用户说"这里 / 这个盘 / 我现在开着的"

先调 `wiztree_gui_state`，读出 WizTree 窗口当前指向的目标，再拿该路径去查询——不需要用户手打路径。

注意其**能力边界**：WizTree 的文件列表是 owner-drawn（`TVirtualDrawTree`），从进程外只能看到**它指向哪里**，**读不到屏幕上的行**。

### 7.3 与 GUI 协作的两种方式

- **重扫（零操作）**：服务器自己扫描从 GUI 读到的目标，代价是一次扫描的时间。
- **CSV 交接（瞬时）**：在 GUI 里 `File > Export` 导出，再 `wiztree_import_csv`。不重扫，而且**如果 GUI 是以管理员身份运行的，就继承了那次 MFT 扫描**——这是处理整盘时最快的路径。CSV 文件不会被修改或删除。

命令行导出与已打开的 WizTree 窗口**可以并存**（全局互斥量不阻止并发实例）。

### 7.4 其他

- 快照默认留在 `%LOCALAPPDATA%\wiztree-mcp`，最多 12 个（`WIZTREE_MCP_MAX_SCANS` 可调），`wiztree_clear_cache` 可清理。
- `toolCallTimeoutMs` 设成 `900000`（15 分钟）：默认 60 秒对整盘扫描会被中途掐断。

---

## 8. 更新与卸载

**更新**：项目目录里 `git pull` 后无需重新构建（单文件 Python，直接加载）；若 `requirements.txt` 有变化，重新 pip install：

```powershell
& '<MCP 根目录>\wiztree-mcp\.venv\Scripts\python.exe' -m pip install -r '<MCP 根目录>\wiztree-mcp\requirements.txt'
```

**卸载**：

```powershell
# 1) 从 cordis.patch.yml 删掉 id: mcp-wiztree 条目（保存后热加载，子进程随之退出）
# 2) 需要时清理缓存与虚拟环境
Remove-Item "$env:LOCALAPPDATA\wiztree-mcp" -Recurse -Force
Remove-Item '<MCP 根目录>\wiztree-mcp\.venv' -Recurse -Force
```

> 索引：[README.md](README.md)
