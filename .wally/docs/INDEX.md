# 360_v6_openwrt · Agent 知识库总路由表（INDEX）

> **Agent 读取规程（必读）**：
> 1. **禁止一次性全量读取 `.wally/docs/`**。先查下方「任务意图路由表」或「源码路径反向索引」，只打开命中的 1–3 篇目标文档（可用 `Read(limit=15)` 先看顶部 YAML 头确认 `source_paths`、`symbols` 与 `status`）。
> 2. **弱命中与草稿核验纪律**：标记 `status: draft` 的文档为源码提取草稿；从文档拿到 `路径:行号`（`符号名`）、上下游契约与「共变耦合清单」后，执行修改前必须调用 `Read(path, offset, limit)` 复核源码字面实现（近邻词线索不等于原词覆盖）。
> 3. **人工精修保护**：标记 `origin: human` 或 `status: verified` 的页面为人工确认权威结论，后续重跑知识库严禁覆盖。当前基准提交：`chore/wally-init-20261127-c48491` @ `2ccf86c05d8da4b11ab8afd8621d939e17577ba5`（本库尚无人工页）。

本仓是以 GitHub Actions 云端编译 360 V6（高通 IPQ60xx）路由器 OpenWrt 固件的模板仓（`README.md:9`，MIT 许可 `LICENSE:1-2`），叠加 wally-init 安装的六阶段 SOP 开发协作工作区（`AGENTS.md:1-4`）。核心输入是 `.config` + 两个 DIY 脚本 + passwall feed 声明，核心输出是 Actions Artifact 固件与规范化的 AI 开发流程工件。

---

## 1. 任务意图最短路由表（Agent Intent Router）

| Agent 当前待办任务类型 | 按序必读文档（1 → 2 → 3） |
|---|---|
| 改固件构建流程 / 换机型 / 换源 / 调编译 | [固件构建流程](business/firmware-build.md) → [构建流水线组件](architecture/openwrt-build-pipeline.md) → [配置手册](contracts/configuration.md) |
| 追加/修改软件包（feed / 包选择） | [构建流水线组件](architecture/openwrt-build-pipeline.md) → [passwall feed 契约](integrations/downstream/openwrt-passwall-feed.md) → [配置手册](contracts/configuration.md) |
| 改产物分发（Artifact / Release / 外链 / 清理策略） | [固件构建流程](business/firmware-build.md) → [GitHub 平台契约](integrations/downstream/github-actions-platform.md) → [file-transfer 契约](integrations/downstream/file-transfer.md) |
| 恢复/调整自动触发（巡检、cron、派发） | [巡检触发流程](business/source-update-trigger.md) → [update-checker 组件](architecture/update-checker.md) → [lede 契约](integrations/downstream/coolsnowwolf-lede.md) |
| 新建/推进/排障 SOP 开发任务 | [SOP 任务流程](business/sop-task-lifecycle.md) → [SOP 引擎组件](architecture/sop-engine-and-routing.md) → [错误码字典](contracts/error-codes.md) |
| 修改 token 采集/报表逻辑 | [用量报表流程](business/token-report.md) → [采集器组件](architecture/token-collectors.md) → [台账领域模型](business/domain-models/token-ledger.md) |
| 新增 IDE 客户端接入 / 改 hook 注册 | [接入层组件](architecture/agent-integration-layer.md) → [模块依赖矩阵](architecture/module-dep-matrix.md) → [配置手册](contracts/configuration.md) |
| 定位报错 / 异常 / 降级原因 | [错误码字典](contracts/error-codes.md) → [主干调用链全景](overview/callchain-panorama.md) → [排障 Runbook](operations/runbook-and-observability.md) |
| 跨模块重构 / 评估联动修改 | [模块依赖矩阵](architecture/module-dep-matrix.md) → [高危雷区与 ADR](operations/pitfalls-and-adr.md) → [构建与验证手册](operations/dev-and-test.md) |
| 改字段透传 / 上游输入到下游参数 | [字段映射矩阵](integrations/topology-and-mapping.md) → 对应 [downstream 契约页](integrations/index.md) → [入向接口参考](contracts/api-reference.md) |

---

## 2. 源码目录/核心文件 → 文档反向索引（Source-to-Doc Reverse Index）

| 仓库源码路径（前缀或关键文件） | 架构模型 / 角色 | 首选关联文档 | 关联上下游/业务文档 |
|---|---|---|---|
| `.github/workflows/build-openwrt.yml` | `Async/Cron Worker`（CI 构建） | [`architecture/openwrt-build-pipeline.md`](architecture/openwrt-build-pipeline.md) | [`business/firmware-build.md`](business/firmware-build.md)、[`integrations/downstream/github-actions-platform.md`](integrations/downstream/github-actions-platform.md) |
| `.github/workflows/update-checker.yml` | `Async/Cron Worker`（巡检派发） | [`architecture/update-checker.md`](architecture/update-checker.md) | [`business/source-update-trigger.md`](business/source-update-trigger.md) |
| `diy-part1.sh` / `diy-part2.sh` / `.config` | 构建输入资产 | [`architecture/openwrt-build-pipeline.md`](architecture/openwrt-build-pipeline.md) | [`integrations/downstream/openwrt-passwall-feed.md`](integrations/downstream/openwrt-passwall-feed.md) |
| `.wally/scripts/create_task.py` | `UI & CLI`（任务脚手架） | [`architecture/sop-engine-and-routing.md`](architecture/sop-engine-and-routing.md) | [`business/sop-task-lifecycle.md`](business/sop-task-lifecycle.md)、[`business/domain-models/sop-task-artifacts.md`](business/domain-models/sop-task-artifacts.md) |
| `.wally/scripts/sop_context.py` | `UI & CLI`（路由/门禁） | [`architecture/sop-engine-and-routing.md`](architecture/sop-engine-and-routing.md) | [`contracts/api-reference.md`](contracts/api-reference.md)、[`contracts/error-codes.md`](contracts/error-codes.md) |
| `.wally/core/` / `.wally/versions/` / `.wally/config.json` / `.wally/install-state.json` | SOP 规则与配置 | [`architecture/sop-engine-and-routing.md`](architecture/sop-engine-and-routing.md) | [`business/domain-models/sop-task-artifacts.md`](business/domain-models/sop-task-artifacts.md) |
| `.claude/hooks/`、`.codebuddy/hooks/`、`.codex/hooks/`、`.cursor/hooks/`（`_token_common.py`、`*_token_collect.py`） | `Data & Adapter`（采集底座） | [`architecture/token-collectors.md`](architecture/token-collectors.md) | [`business/token-report.md`](business/token-report.md)、[`business/domain-models/token-ledger.md`](business/domain-models/token-ledger.md) |
| `.claude/.codebuddy/.codex/.cursor` 的 `scripts/token_report.py` | `Data & Adapter`（报表渲染） | [`architecture/token-collectors.md`](architecture/token-collectors.md) | [`business/token-report.md`](business/token-report.md) |
| `.agents/skills/*/SKILL.md` | 技能正本（接入层） | [`architecture/agent-integration-layer.md`](architecture/agent-integration-layer.md) | [`integrations/index.md`](integrations/index.md) |
| 各客户端 `settings.json` / `hooks.json` / 规则副本（`.agent`、`.windsurf`、`.cursor/rules`、`.github/copilot-instructions.md`） | 多客户端接入配置 | [`architecture/agent-integration-layer.md`](architecture/agent-integration-layer.md) | [`architecture/module-dep-matrix.md`](architecture/module-dep-matrix.md) |
| `AGENTS.md`（及 `CLAUDE.md`/`CODEBUDDY.md` 软链） | 协作入口正本 | [`architecture/agent-integration-layer.md`](architecture/agent-integration-layer.md) | [`business/sop-task-lifecycle.md`](business/sop-task-lifecycle.md) |
| `.wally/review.md` / `.wally/rules/` / `.wally/profiles/` | SOP 规则外围 | [`architecture/index.md`](architecture/index.md)（Tier 3 归档表） | [`operations/pitfalls-and-adr.md`](operations/pitfalls-and-adr.md) |

---

## 3. 模块技术能力清单（Component Registry）

| 组件标识 | 框架模型 | 核心源码路径 | 核心入口符号（`路径:行号`） | 上游调用方 | 下游依赖 | 文档 |
|---|---|---|---|---|---|---|
| `openwrt-build-pipeline` | `Worker` | `.github/workflows/build-openwrt.yml` + DIY + `.config` | `jobs.build`（`build-openwrt.yml:37`） | Actions 触发器 | lede、feeds、GitHub、transfer、tmate | [查看](architecture/openwrt-build-pipeline.md) |
| `update-checker` | `Worker` | `.github/workflows/update-checker.yml` | `jobs.check`（`update-checker.yml:24`） | 手动触发 | lede、GitHub 缓存/派发 | [查看](architecture/update-checker.md) |
| `sop-engine-and-routing` | `UI-CLI` | `.wally/scripts/` + `.wally/core/` + `.wally/versions/` | `create`（`create_task.py:69`）、`route`（`sop_context.py:87`） | 用户/IDE hook | 任务工件、确认状态 | [查看](architecture/sop-engine-and-routing.md) |
| `token-collectors` | `DAO` | 四客户端 `hooks/` + `scripts/token_report.py` | `sync_task_store`（`_token_common.py:639`） | IDE hook 事件 | 台账、认领账本、概览 | [查看](architecture/token-collectors.md) |
| `agent-integration-layer` | `UI-CLI` | `.agents/skills/` + 各 settings + 规则副本 | Hook 注册矩阵（`.claude/settings.json:3`） | 五客户端宿主 | 上两组件 | [查看](architecture/agent-integration-layer.md) |

---

## 4. 业务流程与下游依赖速查

- **全景认知**：[系统技术架构](overview/system-architecture.md) · [业务架构全景](overview/business-architecture.md) · [主干调用链全景](overview/callchain-panorama.md) · [领域术语与符号映射](overview/glossary.md)
- **上下游与契约**：[上下游集成总览](integrations/index.md) · [参数组装映射](integrations/topology-and-mapping.md) · 下游契约五篇：[lede](integrations/downstream/coolsnowwolf-lede.md) / [passwall feed](integrations/downstream/openwrt-passwall-feed.md) / [GitHub 平台](integrations/downstream/github-actions-platform.md) / [file-transfer](integrations/downstream/file-transfer.md) / [ssh2actions 调试](integrations/downstream/ssh2actions-debug.md)
- **业务与数据**：[业务流程总表](business/index.md)（WF-01 构建 · WF-02 巡检 · WF-03 SOP 任务 · WF-04 用量报表）· [领域模型总览](business/domain-models/index.md)（[任务工件](business/domain-models/sop-task-artifacts.md) · [用量台账](business/domain-models/token-ledger.md)）
- **契约字典**：[入向接口参考](contracts/api-reference.md) · [错误码字典](contracts/error-codes.md) · [配置手册](contracts/configuration.md)
- **验证、雷区与边界**：[构建与验证手册](operations/dev-and-test.md) · [部署与运行时](operations/deployment-and-runtime.md) · [排障 Runbook](operations/runbook-and-observability.md) · [高危雷区与 ADR](operations/pitfalls-and-adr.md) · [认知边界声明](operations/boundary.md)

---

## 5. SOP 外部权威页（wally-init 生成，非 codewiki 管辖）

以下页面由 wally-init 安装、属 SOP 执行规范外部权威页（当前多为待项目填充的模板态）；codewiki 只链接不复制，更新走其自身维护流程（`.wally/versions/3f84e7245abcc75d/core/knowledge.md:11`）：

- [API 规范索引](apis/index.md) 与 [API 契约模板](apis/_template.md)（本仓实际入向接口见 [contracts/api-reference.md](contracts/api-reference.md)）
- [数据库与 SQL 规范](sqls/README.md)（本仓无数据库，事实记录见 [领域模型总览](business/domain-models/index.md)）
- [单元测试规范](unittest/unittest.md)（本仓无测试套件的实况见 [构建与验证手册](operations/dev-and-test.md)）
- [集成测试规范](integration_test/integration_test.md)
- [开发环境](devops/env.md) 与 [测试环境部署](devops/test-env-deploy.md)（本仓环境配置实况见 [配置手册](contracts/configuration.md)）
