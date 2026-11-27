---
kind: workflow
scope: WF-04-token-report
status: draft
origin: generated
source_paths: [.claude/hooks/_token_common.py, .claude/hooks/claude_token_collect.py, .codebuddy/hooks/stop_token_collect.py, .codex/hooks/codex_token_collect.py, .cursor/hooks/cursor_token_collect.py, .claude/scripts/token_report.py, .codebuddy/settings.json, .codex/hooks.json, .cursor/hooks.json, .claude/settings.json]
symbols: [sync_task_store, parse_transcript, scan_ide, scan_cli, rollup_turns, render, patch_overview, refresh_codex, token_work_disabled, find_task_dir]
upstream: [business/index.md, business/sop-task-lifecycle.md]
downstream: [business/domain-models/token-ledger.md, architecture/token-collectors.md, contracts/configuration.md]
---

# `WF-04` · 模型用量采集与 Token 报表生成流程

两条子链路：采集链在每次 IDE 会话事件（Stop / UserPromptSubmit 等）上被 hook 拉起，增量把「已完成请求的用量记录」合并进当前任务 `token.json`（防串账、防双计、3 秒内必返回）；报表链在 Commit 准备期由人工运行 `token_report.py`，把台账按「用户回合」聚合渲染回 `00-overview.md` 的标记对内。采集是高频增量进程，报表是一次性渲染进程，二者绝不混用。

## 1. 符号级端到端执行调用树

```text
采集链 (IDE hook 事件驱动):
  Stop/UserPromptSubmit -> 各 settings.json 注册的 hook 命令 -> <client>_token_collect.py
    ├─ [guard] install_deadline(2.5s) 自杀式限时 (_token_common.py:112-139)
    ├─ [locate] find_repo -> git_branch -> token_work_disabled(main/master/release* 即 no-op)
    │          (_token_common.py:210-234) -> find_task_dir(按分支匹配概览, mtime 最新)
    │          (_token_common.py:258-291)
    ├─ [parse] 客户端转录解析:
    │    ├─ claude: parse_transcript JSONL 逐行 (claude_token_collect.py:93-163)
    │    ├─ codebuddy: scan_ide(index.json) / scan_cli(jsonl) (stop_token_collect.py:236-278/319-370)
    │    ├─ codex: 运行态文件 + token_usage_record 记账源 (codex_token_collect.py:1-78)
    │    └─ cursor: 事件载荷内嵌 turn-cumulative 计数 (cursor_token_collect.py:11-34)
    └─ [merge] sync_task_store (_token_common.py:639-666)
         ├─ foreign_ids: 剔除兄弟任务台账/claim ledger 已认领请求 (防串账)
         ├─ merge_store: flock + request_id 去重 + 原子替换写 (防双计/防撕裂)
         └─ record_claims: 认领新请求 -> ~/.wally/wally-init/token_claims.json

报表链 (Commit 准备期人工执行):
  python3 .claude/scripts/token_report.py -> main (token_report.py:430-484)
    ├─ find_overview 按分支定位任务概览 (token_report.py:91-106)
    ├─ token_work_disabled 分支黑名单即退出 (token_report.py:58-62)
    ├─ refresh_codex 可选回填 Codex 日志 (token_report.py:402-414)
    ├─ rollup_turns 回合聚合 (token_report.py:195-230)
    │    ├─ cursor: 一个 generation_id 一行, 不再求和
    │    ├─ codex: 同 thread+root_turn_id 的模型调用求和
    │    └─ codebuddy: 同会话连续同 prompt 的请求求和
    └─ patch_overview 在 TOKEN_USAGE_START/END 标记对内替换 (token_report.py:417-427)
```

## 2. 分阶段规则与分支判定表

| 阶段 | 负责函数（`路径:行号`） | 精确条件 / 阈值 | 命中行为 | 副作用 |
|---|---|---|---|---|
| 采集准入 | `token_work_disabled`（`.claude/hooks/_token_common.py:223-234`） | 分支 ∈ `main/master` 或 `release*` 前缀 | 采集器立即 no-op | 无（防主线会话灌入任务） |
| 任务定位 | `find_task_dir`（`_token_common.py:258-291`） | 概览 Branch 字段与当前分支相等 | 取 mtime 最新的命中目录；零命中跳过 | 无 |
| 转录可信根 | `trusted_roots`（`.claude/hooks/claude_token_collect.py:38-45`、`.codebuddy/hooks/stop_token_collect.py:69-82`） | 转录路径在白名单根内；CodeBuddy 显式拒绝 `.bugbot`/`codebuddy-config-dir` | 不可信即忽略 | 无 |
| 零用量过滤 | `normalize_usage` 消费侧（`claude_token_collect.py:148`、`stop_token_collect.py:261`） | `total_tokens <= 0` | 丢弃该记录 | 无 |
| 锁超时放弃 | `file_lock`（`_token_common.py:396-441`） | `LOCK_WAIT_SEC=0.4` 内未获锁 | 跳过本次写（下次事件重试）；`sync_task_store` 返回 skipped | 无 |
| 防串账 | `foreign_ids`（`_token_common.py:613-636`） | 请求 ID 已被其他任务台账/claim ledger 认领 | 从本次合并集中剔除 | claim 不更新 |
| 双计防护 | `merge_cumulative_usage`（`_token_common.py:363-393`，仅 Cursor 传入） | 回传计数器逐元素取最大 | 汇聚终值且保持 `input = uncached + read + write` 恒等式 | 台账收敛 |
| 报表准入 | `main`（`.claude/scripts/token_report.py:448-451`） | 分支黑名单同上 | 打印 `[SKIP]` 退出 0 | 无 |
| 标记对唯一性 | `patch_overview`（`token_report.py:420-426`） | `TOKEN_USAGE_START/END` 标记必须恰出现一对 | 缺失/不唯一抛 `RuntimeError` | 概览不被破坏 |
| 台账缺失 | `main`（`token_report.py:468-469`） | `token.json` 不可读 | 追加警告「不可假定为零用量」并渲染空表 | 概览含警示行 |

## 3. 数据形态与不变量（无状态机）

- 回合定义：一行 = 一个用户回合；Cursor 一个 `generation_id`、Codex 一组 `root_turn_id`、CodeBuddy 一段连续同 prompt（`rollup_turns`，`token_report.py:195-230`）。
- 会计恒等式：`Total input = Uncached + Cache read + Cache write`、`Tokens = Total input + Output`、推理 token 属输出不重复计（渲染说明段，`token_report.py:354-361`）。
- Credit 缺省：Codex 与 Cursor 无 credit 概念，列渲染 `—` 且合计不估算成本（`token_report.py:320-321`、`token_report.py:296-297`）。
- 快照语义：报表只含渲染时已持久化的记录，之后发生的请求不计入（`token_report.py:362`）。

## 4. 幂等、并发与故障补偿表

| 场景 | 机制（`路径:行号`） | 动作 |
|---|---|---|
| 同请求重复采集 | `merge_store` 按 `request_id` 键控（`_token_common.py:492-516`） | 已存在则按合并器更新（默认替换、Cursor 取最大） |
| 两客户端同时写同任务 | `file_lock` + 原子 `os.replace`（`_token_common.py:396-449`） | 后写者合并前者结果；抢锁失败方安全放弃 |
| Hook 进程卡死 | SIGALRM `os._exit(0)`（`_token_common.py:126-131`） | 2.5s 自杀，不阻塞 IDE；增量语义保证下次补齐 |
| 管道永不关闭 | `read_stdin_json` select 超时 0.8s（`_token_common.py:153-193`） | 解出完整 JSON 即返回，未解出按空载荷处理 |
| 报表标记对损坏 | `pattern.subn` 校验（`token_report.py:424-426`） | 抛错退出，概览保持原状 |
| Codex 回填超时 | `refresh_codex` 10s 超时（`token_report.py:408-413`） | 带警告继续渲染已保存用量 |

## 5. 共变耦合与修改清单（Agent Co-Change Playbook）

1. 台账结构字段增删：同步 store 契约 docstring（`_token_common.py:22-45`）、[token-ledger 领域模型](domain-models/token-ledger.md)、报表列渲染（`token_report.py:304-323`）与四客户端采集器写入点。
2. Hook 事件注册调整：改任一 settings/hooks.json（`.claude/settings.json:3-36`、`.codebuddy/settings.json:2-36`、`.codex/hooks.json:3-88`、`.cursor/hooks.json:3-40`）须同步 [agent-integration-layer](../architecture/agent-integration-layer.md) Hook 注册矩阵与 [token-collectors 组件](../architecture/token-collectors.md) 差异矩阵；宿主 `timeout: 3` 必须大于 `HOOK_DEADLINE_SEC=2.5`。
3. 报表入口换客户端目录：`token_report.py` 三副本 MD5 一致（`a639a9a43bdb211573527324ee00868a`），修改任一副本须三处同步重拷（`.claude/.codebuddy/.codex` 的 `scripts/`）。
4. 环境变量（`WALLY_TOKEN_CLAIMS`、`CLAUDE_CONFIG_DIR`、`CODEBUDDY_EXTENSION_DATA`、`WALLY_CODEX_STATE_DIR` 等）增删：同步 [配置手册](../contracts/configuration.md) 环境变量全集表。
5. 本流程属于 SOP 阶段 06 的一环：执行时机与提交纪律见 [WF-03](sop-task-lifecycle.md) 与 `.wally/versions/3f84e7245abcc75d/commit.md:6`（禁止提交原始 `token.json`）。
