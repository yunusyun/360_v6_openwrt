---
kind: workflow
scope: business-index
status: draft
origin: generated
source_paths: [.github/workflows/build-openwrt.yml, .github/workflows/update-checker.yml, .wally/scripts/create_task.py, .wally/scripts/sop_context.py, .claude/scripts/token_report.py, .claude/hooks/_token_common.py]
symbols: [jobs.build, jobs.check, create, route, sync_task_store, patch_overview]
upstream: [INDEX.md, overview/business-architecture.md]
downstream: [business/firmware-build.md, business/source-update-trigger.md, business/sop-task-lifecycle.md, business/token-report.md, business/domain-models/index.md, contracts/api-reference.md]
---

# 业务能力与核心流程总清单

全量归档本仓四个业务用例（`WF-01`–`WF-04`）：两个固件生产闭环（构建、巡检触发）与两个开发协作闭环（SOP 任务、用量核算）。每个流程的入口、下游扇出、状态实体与数据模型均在专页展开。

## 1. 全量业务流程路由总表（Complete Workflow Inventory）

| 流程 ID | 业务域 | 流程名称 | 触发入口（`路径:行号`） | 串联模块与外部下游链 | 变更的核心实体/状态 | 专页路径 |
|---|---|---|---|---|---|---|
| `WF-01` | 固件生产 | 360 V6 固件云端构建与分发 | `workflow_dispatch` / `repository_dispatch`（`.github/workflows/build-openwrt.yml:14-20`） | build job → lede → passwall feed → buildroot → Artifact / Release / transfer | Actions run、Artifact、`DEVICE_NAME`/`FIRMWARE` | [`firmware-build.md`](firmware-build.md) |
| `WF-02` | 固件生产 | 上游源码更新检测与联动触发 | `workflow_dispatch`（`.github/workflows/update-checker.yml:19`，cron 注释态） | check job → lede 浅克隆 → cache → dispatch 派发 → WF-01 | Actions 缓存 `.commitHash`、派发事件 | [`source-update-trigger.md`](source-update-trigger.md) |
| `WF-03` | 开发协作 | SOP 六阶段任务生命周期 | 开发请求（`AGENTS.md:8-12` 分类）→ `create_task.py`（`.wally/scripts/create_task.py:164`） | 建任务 → 六阶段（门禁 `sop_context.py:260-290`）→ 确认口令 → 报表 → 授权提交 | `.wally/plans/<task>/` 工件树、确认票据 | [`sop-task-lifecycle.md`](sop-task-lifecycle.md) |
| `WF-04` | 开发协作 | 模型用量采集与 Token 报表 | IDE hook 事件（`.claude/settings.json:3-13` 等） | 采集器 → 合并内核（`.claude/hooks/_token_common.py:639-666`）→ `token_report.py` 渲染 | `token.json`、claim ledger、概览报表区 | [`token-report.md`](token-report.md) |

## 2. 业务域 ↔ 接口组 ↔ 错误码 ↔ 配置键交叉速查

| 业务域 | 入向入口 | 调用的外部下游 | 专属错误/降级标识 | 核心控制配置键 |
|---|---|---|---|---|
| 固件生产 | 工作流触发器 3 个（见 [`api-reference`](../contracts/api-reference.md) 第 2 节） | [lede](../integrations/downstream/coolsnowwolf-lede.md)、[passwall](../integrations/downstream/openwrt-passwall-feed.md)、[GitHub 平台](../integrations/downstream/github-actions-platform.md)、[transfer](../integrations/downstream/file-transfer.md)、[ssh2actions](../integrations/downstream/ssh2actions-debug.md) | `make` 三级降级链（`build-openwrt.yml:106`）；残包清理（`build-openwrt.yml:98-99`）；`::warning` 外链注释（`build-openwrt.yml:145`）——全表见 [`error-codes`](../contracts/error-codes.md) | `REPO_URL`/`REPO_BRANCH`/`UPLOAD_*`/`TZ`（[`configuration`](../contracts/configuration.md) 第 2 节）+ 4 个 secrets |
| 开发协作 | CLI 3 个 + hook 事件组（见 `api-reference` 第 2 节） | wally-open-cli 服务（仅技能约定） | `exit 3` 待确认、`[ERROR]` 文本族、hook 静默降级族（`error-codes` 第 1–4 节） | `.wally/config.json` deploy/it 段（`readiness`/`gaps`）、采集时限常量（`configuration` 第 5–6 节） |

## 3. 流程间依赖与互斥关系

- `WF-02 → WF-01`：唯一流程间调用边（`repository_dispatch` 字面量桥，`update-checker.yml:52` → `build-openwrt.yml:14`）；`WF-01` 不回调 `WF-02`。
- `WF-03 ⊃ WF-04`：报表渲染是 `WF-03` 阶段 06 的内嵌步骤（`.wally/versions/3f84e7245abcc75d/commit.md:3`）；`WF-04` 采集链独立持续运行。
- `WF-01/WF-02` 与 `WF-03/WF-04` 零交集：CI 链不触碰 `.wally/plans/`，SOP 链不触碰 workflows——两条业务线的共变只通过「修改工作流须走 SOP 流程」这一开发纪律发生（`AGENTS.md:10`）。
- 数据模型归属：全部状态实体字典见 [`domain-models/index`](domain-models/index.md)（任务工件、用量台账两个聚合）。
