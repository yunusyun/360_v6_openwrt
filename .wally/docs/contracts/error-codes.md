---
kind: error-dict
scope: error-codes
status: draft
origin: generated
source_paths: [.wally/scripts/create_task.py, .wally/scripts/sop_context.py, .claude/hooks/_token_common.py, .claude/scripts/token_report.py, .claude/hooks/claude_token_collect.py, .codebuddy/hooks/stop_token_collect.py, .github/workflows/build-openwrt.yml]
symbols: [ValueError, parser.exit, LockTimeout, RuntimeError, install_deadline, exit 3, [ERROR], [SKIP], [OK]]
upstream: [contracts/api-reference.md, integrations/index.md]
downstream: [operations/runbook-and-observability.md]
---

# 错误码、异常传播与降级标识字典

本仓无集中错误码枚举：错误面由三个 CLI 的 `ValueError` 文本、Hook 进程的静默降级、工作流 shell 失败语义构成。本页只登记源码字面存在的错误类型、退出码与日志通道，严禁虚构编号。

## 1. 异常捕获与退出码映射规则

| 边界位置（`路径:行号`） | 捕获的异常类型 | 对外输出 / 退出码 | 是否阻断 |
|---|---|---|---|
| `create_task.py` main（`.wally/scripts/create_task.py:172-173`） | `OSError / ValueError / KeyError` | `[ERROR] <msg>` stderr，`exit 1` | 阻断（不落盘） |
| `sop_context.py` 交互态（`.wally/scripts/sop_context.py:410-419`） | `OSError / ValueError / SubprocessError` | `SOP context warning: <msg>`，`exit 1` | 阻断上下文输出 |
| `sop_context.py` checkpoint 态（`sop_context.py:417-419`） | 有未决确认 | 确认提示文本，**`exit 3`（等待用户确认）** | 阻断该阶段 |
| `sop_context.py` hook 态兜底（`sop_context.py:466-472`） | 任何异常 | 打印 `{}`，`exit 0` | **不阻断**（hook 失败不升级为工具禁令） |
| `token_report.py` main（`.claude/scripts/token_report.py:473-484`） | `RuntimeError`（标记对缺失/不唯一） | 未捕获 traceback，`exit 非 0` | 阻断报表 |
| 采集器 hook 兜底（`stop_token_collect.py:428-431`、`claude_token_collect.py:198-202`） | 任何异常 | `log()` 写 `/tmp/wally_debug/`，`exit 0` | 不阻断 IDE |
| 采集器 deadline（`.claude/hooks/_token_common.py:126-131`） | SIGALRM 到点 | `os._exit(0)` 自杀 | 放弃本次（下次事件重试） |
| 工作流步骤（`.github/workflows/build-openwrt.yml:106`） | `make` 非零 | 链式降级重试，最终失败 → job failure | 编译失败即终止分发 |

## 2. CLI 错误全表（`ValueError` / `KeyError` 字面文本）

| 错误文本（前缀 `[ERROR] `） | 产出位置 | 精确触发条件 | 排障处置 |
|---|---|---|---|
| `task name must be YYYY-MM-DD_title without path separators` | `.wally/scripts/create_task.py:72` | 名字不匹配正则或以 `.` 结尾 | 改目录名重跑 |
| `unconverted data remains` / 日期无效 | `create_task.py:73`（`datetime.strptime`） | 日期段非法 | 修正日期 |
| `supply the actual non-default task branch` | `create_task.py:75` | 分支为 `main/master/HEAD` 或含 `` ` ``/`|`/换行 | 换任务分支 |
| `execution configuration escapes repository` | `create_task.py:30` | `.wally/config.json` 解析后逃逸仓库根 | 修复软链/路径 |
| `invalid execution configuration schema` | `create_task.py:40` | `schema != 1` 或非 dict | 修正 config.json |
| `invalid execution method: deploy/it` | `create_task.py:45` | 方法四键非全 str | 修正字段类型 |
| `invalid execution readiness` | `create_task.py:47` | readiness 非法枚举 | 用 `pending/ready/null` |
| `invalid execution gaps` | `create_task.py:49-50` | gaps 非 str 列表 | 修正列表 |
| `ready execution method requires a command or guide` | `create_task.py:52` | `ready` 但空方法 | 补方法或改 pending |
| `finish pending project translations before creating a new task` | `create_task.py:85` | `translation_sources` 非空 | 先完成翻译迁移 |
| `upgrade SOP initialization before creating six-phase tasks` | `create_task.py:87` | `schema != 2` 或 `steps != 6` | 升级 wally-init |
| `invalid SOP revision` | `create_task.py:90` | revision 非 16 位十六进制 | 修复 install-state |
| `missing bound SOP rule: <f>` | `create_task.py:94` | 绑定 core 文件缺失 | 恢复版本目录 |
| `task already exists: <name>` | `create_task.py:97` | 同名任务已存在 | 换名或续用 |
| `task appeared while preparing: <name>` | `create_task.py:158` | staging 期间目录被抢占 | 重跑创建 |
| `SOP path escapes repository` | `.wally/scripts/sop_context.py:76` | 路径越界仓库根 | 修正引用 |
| `Missing SOP file: <p>` | `sop_context.py:83` | 绑定文件不存在 | 恢复/修正概览 |
| `Cannot route SOP without an actual Git branch` | `sop_context.py:95` | detached HEAD 或 git 失败 | 先修复 Git 状态 |
| `Multiple SOP tasks match branch <b>` | `sop_context.py:107` | 多任务绑定同分支 | 人工消歧 |
| `No SOP task for branch <b> ...` | `sop_context.py:108-114` | 分支无任务 | 走建任务流程（含提示文本） |
| `Historical task is missing its compat/v1 policy snapshot` | `sop_context.py:121` | legacy 任务无快照 | 找回快照，禁止隐式迁移 |
| `Invalid bound SOP Rules path; do not use the latest rules as a fallback` | `sop_context.py:132` | Rules 字段非版本路径 | 修正概览字段 |
| `Missing current phase section in task overview` | `sop_context.py:51` | 概览缺 Current phase 段 | 补段落 |
| `Missing current phase file link in task overview` | `sop_context.py:58` | Current phase 段内缺少阶段文件的相对链接 | 补链接 |
| `Current phase number and file link disagree` | `sop_context.py:141-142` | 编号与文件名前缀不一致 | 对齐编号/链接 |
| `Missing or invalid current phase in task overview` | `sop_context.py:135` | Phase 行不匹配 `0[1-6]` | 修正 Phase 行 |
| `invalid or missing Core export markers` / `empty Core export` | `sop_context.py:31-35` | 规则文件导出标记对损坏 | 修复规则文件标记 |
| `Cannot find the .wally directory` | `.claude/scripts/token_report.py:446` | 向上找不到 `.wally` | 在仓库内运行 |
| `Cannot find 00-overview.md for branch` | `token_report.py:454` | 分支无绑定任务 | 先建任务 |
| `<path> missing Token marker` / `Token marker is not unique` | `token_report.py:423-425`（`RuntimeError`） | 概览缺标记对/重复 | 修复模板标记对 |
| `invalid token store` / `invalid collection status` | `token_report.py:462-466` | 台账结构非法 | 按警告降级渲染（不退出） |

## 3. Hook 降级与静默失败标识表（`degraded` 类字面量）

| 标识 / 日志通道 | 产出位置（`路径:行号`） | 触发条件 | 自动兜底行为 |
|---|---|---|---|
| `skip merge: lock timeout <path>` | `.claude/hooks/_token_common.py:540` | 台账锁 0.4s 未获取 | 放弃本次合并，下次事件重试 |
| `skip claims: lock timeout` / `claim ledger write failed` | `_token_common.py:599-602` | 认领账本写失败 | 跳过认领（可能重复剔除由幂等键兜底） |
| `deadline: abort after 2.5s` | `_token_common.py:128` | SIGALRM 到点 | `os._exit(0)`，IDE 无感 |
| `skip: no .wally near cwd` / `no task for branch` / `token collect disabled on branch` | `.codebuddy/hooks/stop_token_collect.py:383-393` | 无仓/无任务/黑名单分支 | 静默退出 |
| `ignore untrusted transcript_path` | `stop_token_collect.py:117` | 转录在可信根外（如 Bugbot） | 回退按 session_id 检索 |
| `claude_token_collect: unhandled error` | `.claude/hooks/claude_token_collect.py:201` | 采集器未捕获异常 | 记日志 exit 0 |
| `Codex backfill is incomplete (or timed out)` | `.claude/scripts/token_report.py:411-413` | refresh_codex 失败/超时 | 报表带警告继续渲染 |
| `Usage ledger missing or unreadable; do not assume zero usage` | `token_report.py:469` | token.json 缺失/损坏 | 渲染空表 + 警告行 |
| `No usage records collected; this does not mean complete zero usage` | `token_report.py:397` | 台账空 | 同上 |
| 工作流 `::warning file=<域名>::<url>` | `.github/workflows/build-openwrt.yml:145` | 外链上传产出链接 | 链接进 Actions 注释流 |

## 4. 确认流提示文案（非错误，但阻断执行）

| 文案关键片段 | 产出位置 | 语义 |
|---|---|---|
| `SOP user confirmation required: <reason>` | `.wally/scripts/sop_context.py:341` | 阶段证据缺口，等待用户 |
| `CONFIRM SOP <12hex token>` | `sop_context.py:342-343` | 用户须在聊天精确回复的口令 |
| `SOP confirmation is unknown or stale` | `sop_context.py:366` | 口令过期/指纹不符，需重新确认 |
| `SOP checkpoint ready (prerequisites satisfied...)` | `sop_context.py:418` | 放行提示（非通过证明） |

## 5. 共变耦合与修改清单（Co-Change Playbook）

- 新增 `ValueError` 文案：登记本页第 2 节同表结构（产出位置、触发条件、处置），并核对 [api-reference](api-reference.md) 对应 CLI 错误分支行。
- 改 hook 兜底策略：`sop_context.py:466-472` 的「吞异常输出 `{}`」是刻意设计（docstring：`A workflow hook failure must not become a blanket tool prohibition`，`sop_context.py:468-470`），改成 fail-closed 前必须评估所有客户端阻断面。
- 工作流错误语义：步骤守卫链（`build-openwrt.yml:118`、`build-openwrt.yml:134`）变更时同步 [WF-01](../business/firmware-build.md) 第 2 节判定表；`make` 三级降级链（`build-openwrt.yml:106`）属编译容错策略，排障入口见 [排障 Runbook](../operations/runbook-and-observability.md)。
