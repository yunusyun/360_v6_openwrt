---
kind: component
scope: architecture-index
status: draft
origin: generated
source_paths: [.github/workflows, .wally/scripts, .wally/core, .wally/versions, .claude/hooks, .codebuddy/hooks, .codex/hooks, .cursor/hooks, .agents/skills, .config, diy-part1.sh, diy-part2.sh]
symbols: [jobs.build, jobs.check, create, route, sync_task_store, clarify, sop-aware-code-review, wally-open-cli]
upstream: [INDEX.md, overview/system-architecture.md]
downstream: [architecture/module-dep-matrix.md, architecture/openwrt-build-pipeline.md, architecture/update-checker.md, architecture/sop-engine-and-routing.md, architecture/token-collectors.md, architecture/agent-integration-layer.md]
---

# 组件技术架构总表

按运行时框架模型（Archetype）汇总全仓五个技术组件，并将不单独成篇的 Tier 3 辅助资产归档登记，确保全仓源码目录 100% 纳入路由索引。组件判定依据：仓库双形态——CI 构建流水线（无进程驻留）+ 本地 SOP 工具链（CLI/Hook 进程）。

## 1. 全量核心组件矩阵（Tier 1 & Tier 2）

| 组件标识 | 框架模型（Archetype） | 源码路径 | 核心入口符号（`路径:行号`） | 技术能力与业务边界 | 直接上游 | 直接下游 | 详情文档 |
|---|---|---|---|---|---|---|---|
| `openwrt-build-pipeline` | `Async/Cron Worker`（CI 批处理） | `.github/workflows/build-openwrt.yml` + `diy-part*.sh` + `.config` | `jobs.build`（`build-openwrt.yml:37`） | 事件驱动固件构建：取源、定制注入、三级降级编译、产物分发 | Actions 触发器（手动/派发） | lede、feeds、GitHub 平台、transfer、tmate | [`openwrt-build-pipeline.md`](openwrt-build-pipeline.md) |
| `update-checker` | `Async/Cron Worker`（巡检） | `.github/workflows/update-checker.yml` | `jobs.check`（`update-checker.yml:24`） | 上游哈希巡检 + 缓存幂等 + 派发构建 | 手动触发（cron 注释态） | lede、GitHub 缓存/派发 API | [`update-checker.md`](update-checker.md) |
| `sop-engine-and-routing` | `UI & CLI`（本地 CLI/Hook 双形态） | `.wally/scripts/` + `.wally/core/` + `.wally/versions/` + `.wally/config.json` | `create`（`create_task.py:69`）、`route`（`sop_context.py:87`） | 任务脚手架、分支路由、阶段门禁、确认口令、规则快照绑定 | 用户/IDE hook | 任务工件、确认状态 | [`sop-engine-and-routing.md`](sop-engine-and-routing.md) |
| `token-collectors` | `Data & Adapter`（采集与台账底座） | 四客户端 `hooks/` + `scripts/token_report.py` | `sync_task_store`（`_token_common.py:639`） | 用量采集、防串账防双计、原子台账、报表渲染 | IDE hook 事件 | token.json、claim ledger、概览 | [`token-collectors.md`](token-collectors.md) |
| `agent-integration-layer` | `UI & CLI`（多客户端接入层） | `.agents/skills/` + 各 settings/hooks.json + 规则副本 | Hook 注册矩阵（`.claude/settings.json:3-36` 等） | 技能/规则正本分发、hook 桥接、输出协议适配 | 五客户端宿主 | 上两组件 | [`agent-integration-layer.md`](agent-integration-layer.md) |

## 2. 轻量辅助模块归档表（Tier 3 Utilities — 零遗漏兜底）

| 源码文件路径 | 核心导出函数 / 类 / 常量 | 调用方模块 | 具体职责说明 |
|---|---|---|---|
| `diy-part1.sh` | feed 追加语句（`diy-part1.sh:18`） | 构建工作流步骤 4 | feeds 前置定制唯一注入点（注释态 helloworld 示例两行） |
| `diy-part2.sh` | （当前无生效语句，唯一行 `diy-part2.sh:14` 为注释） | 构建工作流步骤 6 | 配置后定制占位脚本 |
| `.config` | `CONFIG_TARGET_*` 目标段（`.config:41-64`）等 1046 条 | `make defconfig`（`build-openwrt.yml:96`） | 构建目标与包选择声明（数据资产） |
| `.wally/core/entry.md` | SOP 路由说明（`.wally/core/entry.md:1-12`） | `route`（`sop_context.py:88`） | 路由必读文档 |
| `.wally/core/scope.md` | 请求分类规则（`.wally/core/scope.md:3-5`） | hook 注入上下文（`sop_context.py:437-439`） | 开发/咨询分流判据 |
| `.wally/review.md` | 审查清单与知识一致性段（`.wally/review.md:35-37`） | 审查技能、阶段 06 | CR 门禁 |
| `.wally/rules/README.md` | 规则位声明（`.wally/rules/README.md:8`） | Implement 阶段装载 | 当前声明「无额外项目规则」 |
| `.wally/profiles/project.md` | Profile 快照（`.wally/profiles/project.md:7-8`） | `route` 装载（`sop_context.py:144-151`） | 项目策略与技术文档路径绑定 |
| `.wally/install-state.json` | `revision`/`clients`/`managed_files`（`.wally/install-state.json:4-52`） | `create`（`create_task.py:83-90`） | 安装账本（只读元数据） |
| `.wally/versions/3f84e7245abcc75d/**` | 规则模板全家（`core/*`、`plans-template/*`、`commit.md`、`review.md`、`profiles/generic/project.md`） | `create_task.py:105`、`sop_context.py:144-151` | 不可变规则版本包 |
| `AGENTS.md` / `CLAUDE.md` / `CODEBUDDY.md` | 协作入口（`AGENTS.md:1-57`，后两者为软链） | 全客户端 | SOP 入口正本 |
| `.github/copilot-instructions.md` | 入门段副本（`.github/copilot-instructions.md:1-51`） | Copilot | 与 AGENTS 内嵌段同步 |
| `.agent/rules/wally-sop.md`、`.windsurf/rules/wally-sop.md` | 规则摘要副本（MD5 `367e92ff...` 一致） | Windsurf/Agent | 指回 AGENTS.md |
| `.cursor/rules/wally-sop.mdc` | mdc 格式规则（`.cursor/rules/wally-sop.mdc:1-12`，`alwaysApply: true`） | Cursor | 同上 |
| `README.md`、`LICENSE`、`.gitignore` | 仓库身份与忽略规则（`README.md:9`、`LICENSE:1-2`、`.gitignore:1-10`） | — | 模板出处声明、MIT 许可、SOP 工件忽略面 |
| `.codex/hooks/codex_token_collect.py`（614 行） | `state_dir`/`registered_session` 等主体（`codex_token_collect.py:37-78`） | Codex hook 事件 | Codex 采集器（细节见 [token-collectors](token-collectors.md) 第 2 节矩阵） |

## 3. 组件 ↔ 源码目录覆盖对账表（G2 兜底）

| 顶层源码目录 | 归档组件页 | 归档方式 |
|---|---|---|
| `.github/workflows/` | openwrt-build-pipeline / update-checker | 组件页 + Tier 3 |
| `.github/`（copilot-instructions、skills 软链） | agent-integration-layer | 组件页 + Tier 3 |
| `.wally/scripts/` | sop-engine-and-routing / token-collectors | 组件页 |
| `.wally/core/`、`.wally/rules/`、`.wally/profiles/`、`.wally/versions/`、`.wally/config.json`、`.wally/install-state.json`、`.wally/review.md` | sop-engine-and-routing + Tier 3 归档 | 组件页 + 本页第 2 节 |
| `.wally/docs/` | 本知识库自身（SOP 外部页见 [INDEX](../INDEX.md)） | 自描述 |
| `.claude/`、`.codebuddy/`、`.codex/`、`.cursor/` | token-collectors + agent-integration-layer | 组件页 |
| `.agents/` | agent-integration-layer | 组件页 |
| `.agent/`、`.windsurf/` | agent-integration-layer + Tier 3 | 组件页 + 本页第 2 节 |
| 根级文件（`AGENTS.md`、`README.md`、`LICENSE`、`.gitignore`、`.config`、`diy-part*.sh`、软链） | 本页第 2 节 + openwrt-build-pipeline | Tier 3 归档 |

未映射源码目录 = 0；目录级依赖关系与高扇入底座分析见 [module-dep-matrix](module-dep-matrix.md)。
