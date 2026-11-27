---
kind: operations
scope: boundary
status: draft
origin: generated
source_paths: [.github/workflows/build-openwrt.yml, .github/workflows/update-checker.yml, .agents/skills/wally-open-cli/SKILL.md, .wally/install-state.json, README.md, LICENSE]
symbols: [REPO_URL, git.io, make, scripts/feeds, wally-open-cli, api_base_url, managed_files, commitHash]
upstream: [INDEX.md]
downstream: []
---

# 认知边界与外部黑盒声明（Agent Anti-Hallucination Boundary）

界定本套 `.wally/docs/` 核验的源码快照、仓外不可见黑盒与静态分析盲区，防止 Agent 脑补仓外实现或把弱命中当字面证据。

## 1. 核验基准快照与生成物状态

- **Git 分支与提交**：`chore/wally-init-20261127-c48491` @ `2ccf86c05d8da4b11ab8afd8621d939e17577ba5`（工作树干净：无未提交/未跟踪源码变更）。
- **生成模式与覆盖目录**：`full` · 已完整扫描并建档的顶层源码目录：`.agent`、`.agents`、`.claude`、`.codebuddy`、`.codex`、`.cursor`、`.github`（含 `workflows` 与 `copilot-instructions.md`、`skills` 软链）、`.wally`（`core`/`rules`/`profiles`/`scripts`/`versions`/`config.json`/`install-state.json`/`review.md`/`docs`）、`.windsurf` 及根级文件（`AGENTS.md`、`README.md`、`LICENSE`、`.gitignore`、`.config`、`diy-part1.sh`、`diy-part2.sh`、`CLAUDE.md`/`CODEBUDDY.md` 软链）。未映射源码目录为 0。
- **生成物可信度声明**：全部 codewiki 管理页为 `status: draft`、`origin: generated`（本库当前无 `origin: human`/`status: verified` 人工页）；Agent 改动核心逻辑前必须按页内 `路径:行号` 回读源码字面核验。`.wally/docs/` 下 `apis/`、`sqls/`、`unittest/`、`integration_test/`、`devops/` 为 SOP 外部权威页（多为 wally-init 待填模板），只链接不复制、不强加生成页格式。

## 2. 仓库外黑盒系统清单（Out-of-Repo Black Boxes）

| 外部黑盒目标 | 本仓边界文件（`路径:行号`） | 本仓已核实的契约范围 | 不可见的远端黑盒范围 |
|---|---|---|---|
| coolsnowwolf/lede 源码仓 | `build-openwrt.yml:23-24`、`update-checker.yml:15-16` | clone 参数、分支、HEAD 哈希读取 | lede 仓内部构建系统、包版本演进、force-push 行为、可用性 |
| OpenWrt buildroot（`make`/`scripts/feeds`） | `build-openwrt.yml:72-110` | 命令行与退出码语义、`bin/targets` 输出布局 | defconfig 补全规则、feeds 拉取内部协议、编译器行为 |
| GitHub Actions 平台 | `build-openwrt.yml:37-190`、`update-checker.yml:23-58` | 各 `with:` 参数字典（见 [github-actions-platform 契约](../integrations/downstream/github-actions-platform.md)）、runner 镜像选择 | 平台限流、缓存淘汰策略、`::set-output` 语法退役时间表、runner 镜像退役计划 |
| Mikubill/transfer 与 git.io 短链 | `build-openwrt.yml:143`、`build-openwrt.yml:152` | 子命令参数、日志/URL 提取方式 | 传输服务端配额、短链服务存活状态 |
| tmate / Telegram Bot | `build-openwrt.yml:86-90` | 触发条件与 env 注入 | 会话服务实现、通知投递 |
| wally-open-cli 托管服务 | `.agents/skills/wally-open-cli/SKILL.md:20`（`api_base_url` 优先级与 `/ok` 探活） | CLI 参数面、任务名派生规则（技能文档字面） | 服务端实现、任务编排内部、`@tencent/wally-open-cli` 包行为 |
| wally-init 安装器 | `.wally/install-state.json:1-56`（账本是被安装产物） | `revision`/`clients`/`managed_files` 哈希现状 | 安装器本体代码、升级迁移流程 |
| 各 IDE 宿主（CodeBuddy/Claude/Codex/Cursor 等） | 各 settings/hooks.json 注册（[agent-integration-layer](../architecture/agent-integration-layer.md)） | hook 事件名、载荷字段、超时行为 | 宿主内部调度、未来 schema 变更（Cursor 计数字段「非文档化但实测稳定」，`.cursor/hooks/cursor_token_collect.py:31-34`） |

## 3. 动态分发与运行期注入边界

| 动态机制位置（`路径:行号`） | 机制说明 | Agent 注意事项 |
|---|---|---|
| `build-openwrt.yml:14` + `update-checker.yml:49-52` | `repository_dispatch` 事件字符串桥接（`Source Code Update`） | 跨文件字面量契约；静态搜索单文件看不到两端 |
| `build-openwrt.yml:62` | `ln -sf /workdir/openwrt $GITHUB_WORKSPACE/openwrt` 软链桥 | 构建目录跨挂载点；工作流步骤全部依赖该软链可达 |
| `GITHUB_ENV` 注入（`build-openwrt.yml:109-110`、`build-openwrt.yml:129`） | `DEVICE_NAME`/`FILE_DATE`/`FIRMWARE` 运行期生成 | 仓内静态读不到这些值；命名逻辑见产物步骤 |
| `_token_common.py:66-68`、`codex_token_collect.py:42-43` | 仓外账本/状态目录路径（env 可覆盖） | 主目录内容不在 Git 仓内，排障需用户机器实况 |
| 各 settings.json 的内联 Python 仓库根探测（如 `.claude/settings.json:18`） | 运行期向上/同级搜索 `.wally/scripts/sop_context.py` | 多仓嵌套目录时行为取决于 cwd，测试需模拟目录布局 |
| `.wally/scripts/sop_context.py:98-104` | 任务路由依赖运行期 Git 分支与概览文件内容 | 概览是数据不是代码：改概览即改路由行为 |

## 4. 静态分析盲区与诚实声明

- **本知识库未运行过任何构建/采集**：`.config` 的 1046 条配置项只核验了目标段与关键行；OpenWrt 包选择语义、编译可行性未验证（无 CI 执行记录入档）。`README.md:9` 自述本仓为「GitHub Actions 构建 OpenWrt 模板」、`LICENSE:1-2` 为 P3TERX MIT 许可——仓身份与许可以这两处为准。
- **未读全文的仓外依赖**：lede / passwall / marketplace Action / transfer 工具的远端文档未fetch，全部结论限于本仓 YAML/shell 字面。
- **`.wally/plans/` 无实例**：任务工件结构取自模板与创建代码推导（`.wally/versions/3f84e7245abcc75d/plans-template/00-overview.md`），尚无真实任务目录可对照。
- **`codex_token_collect.py` 全文 614 行只精读了前 79 行**：其余部分（sqlite 读取、线程发现细节）按 docstring 与调用方契约建档；下结论前须 `Read` 原文核验。
- 报告语言：本知识库正文为中文（用户提问语言）；SOP 外部页与仓内英文工件保持原样，属预期语言混排边界。
