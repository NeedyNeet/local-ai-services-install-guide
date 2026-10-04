# 项目规范与提效手段

> **本文档基本与 DSH 无关，是通用内容。** 只要你在 Windows 上写代码就用得上。
> 仅第 6 节（提效手段）提到的四套服务与本仓库其余文档相关，其中浏览器自动化部分依赖宿主注入的工具；
> 其余小节（路径规范、git 配置、规范分层、目录结构、常见坑、决策表）与任何 AI 工具、任何宿主都无关。

面向**第一次自己做项目**的场景：按"什么时候做什么"排序，不追求一次性上齐所有规范。
每条都尽量给出本机的真实命令与文件位置。

- 配套的可复制起点文件：[`<Dev 根目录>\_templates\project-skeleton\`](../_templates/project-skeleton/)（骨架）与 [`AGENTS.md`](../_templates/AGENTS.md)（轻量模板）
- 本机服务怎么用：见 [common-guide.md](common-guide.md) 与各服务文档

---

## 1. 项目放在哪（路径规范）

| 类型 | 位置 |
|---|---|
| 要推到 GitHub 的项目 | `<Dev 根目录>\Github\<项目名>` |
| 纯本地实验、不打算推 | `<Dev 根目录>\Projects\<项目名>` |
| 给 AI 用的 MCP 服务器 | `<MCP 根目录>\<项目名>` |
| 包管理器缓存 | `<Dev 根目录>\Cache\`（npm / pip / gradle / nuget…） |

每个项目**一个独立目录 + 一个独立 git 仓库**，不要在一个仓库里塞多个不相关的项目。

---

## 2. git 基础配置（一次性，已在本机设好）

```bash
git config --global init.defaultBranch main   # 新仓库默认 main（本机系统级已设，保持即可）
git config --global core.autocrlf input       # 提交时统一 LF，跨平台不产生"整个文件都改了"的假 diff
git config --global core.quotepath false      # 中文文件名在 git status 里正常显示（不再显示 \xxx 转义）
```

本机实测状态（`user.name` / `user.email` 是你的提交身份，会写在每条提交记录里、公开可见，所以这里不列具体值）：

| 项 | 值 | 来源 |
|---|---|---|
| `user.name` / `user.email` | 已配置（提交身份，公开可见） | 全局 |
| `init.defaultBranch` | `main` | 系统级 |
| `core.autocrlf` | `input` | 全局（覆盖系统级的 `true`） |
| `core.quotepath` | `false` | 全局 |
| `credential.helper` | `manager` | 系统级（Git Credential Manager，推送时记住凭据） |

检查自己的配置：

```bash
git config --global user.name
git config --global user.email
```

> **关于提交邮箱的隐私**：`user.email` 会出现在每条提交记录里，公开仓库上任何人都能读到。
> 如果不想暴露真实邮箱，可以在 GitHub 设置里开启 "Keep my email addresses private"，
> 然后把 `user.email` 改成 GitHub 提供的 `noreply` 地址（形如 `<ID>+<用户名>@users.noreply.github.com`）。

已实测：新建仓库默认分支为 `main`；CRLF 文件提交进索引后自动转为 LF。

**行尾问题的根治办法**是在项目里放 `.gitattributes`（比各机器各自的 `core.autocrlf` 可靠，团队共享），骨架已包含。

### 2.1 推送认证

本机 `credential.helper=manager` 已可用，HTTPS 推送时凭据会被记住。若想更省事，可选：

```powershell
winget install GitHub.cli
gh auth login
```

---

## 3. 项目规范：按"必须 / 推荐 / 可选"分三层

**不要一次性上齐**——规范是为了省事，不是为了填满时间。

### 3.1 必做（第一天就弄）

| 项 | 为什么 | 骨架是否已含 |
|---|---|---|
| `README.md` | 说明：解决什么问题、怎么跑、怎么用。半年后最大的受益者是你自己 | ✅ |
| `.gitignore` | 不写就会把 `node_modules`/`.venv`/`dist`/`.env` 提交进仓库，事后清理很痛 | ✅（通用 + 类型片段） |
| `.gitattributes` | Windows 上防止"整个文件都被改动"的假 diff | ✅ |
| `AGENTS.md` | 让 AI 每次都能快速理解项目上下文与本机约定 | ✅ |
| 依赖锁定文件入库 | `package-lock.json` / `bun.lock` / `uv.lock` / `requirements.txt`，保证环境可复现 | 由包管理器生成 |

### 3.2 推荐（跑通后再加）

- **一条命令能跑起来**：把常用操作收进 `package.json` 的 `scripts`、`justfile`、`Makefile` 或一个 `*.ps1`。参考本机两个真实例子：`<MCP 根目录>\wiztree-mcp`（`just`）、`<MCP 根目录>\beyondcompare-mcp`（`bun run build`）。
- **提交信息写清"为什么"**：格式 `类型: 做了什么`，类型用 `feat` / `fix` / `docs` / `refactor` / `chore`。骨架里的 `.gitmessage` 是模板，启用：
  ```bash
  git config commit.template .gitmessage
  ```
- **`.env` 不入库，但提交 `.env.example`**：前者是真实值（含密钥），后者是"需要哪些变量"的文档。骨架已含 `.env.example` 且 `.gitignore` 已排除 `.env`。
- **`.editorconfig`**：让不同编辑器自动统一缩进/编码/换行。骨架已含。
- **类型检查 / 格式化工具**：TypeScript 用 `tsc --noEmit`；Python 用 `ruff`、`mypy`。**先加检查，再加测试**——检查发现的错误比测试更多也更早。

### 3.3 可选（有真实需求再加）

测试框架（vitest / pytest）、CI（GitHub Actions）、CHANGELOG 自动化、版本号管理、容器化。

> 判断标准：**如果一件事你还没被它坑过，先别做**。比如单项目阶段不需要 monorepo，没有协作者时不需要复杂的分支策略。

### 3.4 明确不要做的三件事

1. ❌ 提交 `node_modules`、`.venv`、`dist`、数据库文件、大二进制（GitHub 单文件上限 100 MB，超了很麻烦）
2. ❌ 把密钥/令牌写进代码。**一旦推上 GitHub 就当作已泄露**——必须立刻轮换密钥，光删提交没用（历史里还在，且可能已被抓取）
3. ❌ 为了"规范"而搭建自己都不理解的复杂工程结构（monorepo、多分支流、自研 CI）

---

## 4. 目录结构：按项目类型选

| 类型 | 结构 |
|---|---|
| Node / TypeScript 库或 CLI | `src/` · `tests/` · `package.json` · `tsconfig.json` |
| Python 项目 | `src/<包名>/` · `tests/` · `pyproject.toml`（**不要提交 `.venv`**） |
| 极简单文件脚本 | 直接 `main.py` / `main.js` + `README.md` + `.gitignore`，不必套 `src/` |
| 前端 / 网页 | `src/` · `public/` · `index.html` · 构建配置 |

**不要自创结构**——按语言社区约定，别人（和以后的你）才能立刻看懂。

---

## 5. 新建项目的标准流程

```powershell
# 一条命令：建目录 + 复制骨架 + git init(main) + 首次提交
<Dev 根目录>\_templates\project-skeleton\new-project.ps1 -Name my-tool -Type python
```

`-Type` 可选 `none`（默认）/ `python` / `node-ts` / `web`，会额外复制对应的项目定义文件并把该类型的忽略规则**追加**进 `.gitignore`。

脚本会打印"接下来做什么"，核心是三件事：

1. 改 `README.md` 第一段（这个项目解决什么问题）
2. 改 `AGENTS.md` 顶部的项目目的 / 技术栈 / 常用命令
3. 在 `AGENTS.md` 末尾的「项目约定」补本项目规范

**手动流程**（不想用脚本时）：

```powershell
cd <Dev 根目录>\Github
mkdir my-tool; cd my-tool
Copy-Item '<本仓库>\..\_templates\project-skeleton\base\*' . -Recurse
# 如需要，再把 <类型>\.gitignore.append 的内容【追加】到 .gitignore 末尾
git init -b main
git add -A; git commit -m "chore: 初始化项目骨架"
```

### 5.1 推到 GitHub

在 GitHub 网页新建**空仓库**（不要勾选自动生成 README/.gitignore，否则和本地冲突），然后：

```bash
git remote add origin https://github.com/<用户名>/my-tool.git
git push -u origin main
```

---

## 6. 提效手段：本机已有的四套服务

这些是**已经装好的提效工具**，新手最容易忘了用：

| 场景 | 用什么 | 说明 |
|---|---|---|
| 找文件、找代码 | `mcp__everything__everything_search` | 全盘检索**几十毫秒**，比资源管理器搜索和递归遍历快几个数量级。"我记得写过但不知道在哪个文件"直接搜 |
| 磁盘快满了 | `mcp__wiztree__*` | 一眼看出哪个目录吃空间。典型：`node_modules` 堆积、重复的 `.venv`、构建缓存 |
| 改代码前后对比、备份 | `mcp__bcompare__*` | 改前存基线；大改前用 `dryRun` 预览备份/同步方案 |
| 看网页效果、调试前端 | `browser_*` | 在已登录的浏览器里打开页面、点击、截图，不需要你手动复现 |
| 清理空间 | `wiztree_duplicates` | 找重复文件（本机曾用它发现微信缓存里的重复音频副本） |

具体调用方式与注意事项见 [common-guide.md](common-guide.md) 和各服务文档。

### 6.1 两个收益极高、成本极低的习惯

1. **让 AI 先读 `AGENTS.md`**。项目根目录有它，AI 就会按本机约定行动（优先用 Everything 而不是慢速遍历、用 `dryRun` 预览同步等），不需要你每次重复交代。
2. **改配置 / 改代码前先备份**。本机这一路操作的惯例：改 `cordis.patch.yml` 前留 `.bak-*`，修复 `report.ts` 前留 `.bak-before-ltonly-fix`。成本≈0，救回来的时候价值极大。

### 6.2 其他日常习惯

- **小步提交**：一次提交只做一件事，"修 bug"和"顺手重构"分开。回滚时才知道该回滚哪一步。
- **改动前先 `git status`/`git diff`**：确认自己清楚当前状态，再动手。
- **定期清理构建产物与缓存**：用 wiztree 找 `node_modules`、`.venv`、`__pycache__`、`dist`，确认无用后删（都能用锁文件重建）。
- **写完 README 再写代码**（哪怕只有三行）：逼自己说清"这个项目到底要做什么"，能避免做一半发现方向不对。

---

## 7. 常见坑速查

| 现象 | 原因 / 处理 |
|---|---|
| 提交后 GitHub 显示"整个文件都改了" | 行尾不一致。加 `.gitattributes`（`* text=auto eol=lf`）并确认 `core.autocrlf` 设置 |
| 中文文件名显示成 `\346\226\207...` | `git config --global core.quotepath false` |
| 仓库里混进了 `node_modules` | `.gitignore` 写晚了。删缓存后重建：`git rm -r --cached node_modules`，确认 `.gitignore` 生效再提交 |
| 推送要反复输密码 | 用 `credential.helper=manager`（本机已启用）或 `gh auth login` |
| 误提交了密钥 | **立即轮换密钥**（不是删提交就完事），再考虑清理历史 |
| `git push` 报分支不存在 | 本地是 `master` 而远程是 `main`（或反之）。`git branch -M main` 后重推 |
| AI 在我的项目里用慢速方式找文件 | 项目根目录缺 `AGENTS.md`。复制模板过去 |

---

## 8. 什么时候该加什么：一张决策表

| 你现在的情况 | 该做的事 |
|---|---|
| 刚建好项目 | README 三段式 + `.gitignore` + `.gitattributes` + `AGENTS.md` + 首次提交 |
| 能跑起来了 | 把常用命令收进一个入口（scripts / justfile）；加格式化工具 |
| 开始改来改去怕改坏 | `git commit` 更频繁；学会 `git diff` / `git restore`；大改前用 bcompare 存基线 |
| 出现重复代码/结构混乱 | 再考虑抽模块、加 `src/` 分层——**不要提前设计** |
| 要和别人协作 | 加 CI（至少跑检查与测试）、约定提交信息格式、把 `.gitattributes` 确认好 |
| 要发布给别人用 | 写清安装步骤、版本号、LICENSE；考虑打包 |
