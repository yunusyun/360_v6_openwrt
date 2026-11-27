---
kind: component
scope: token-collectors
status: draft
origin: generated
source_paths: [.claude/hooks/_token_common.py, .claude/hooks/claude_token_collect.py, .codebuddy/hooks/stop_token_collect.py, .codex/hooks/codex_token_collect.py, .cursor/hooks/cursor_token_collect.py, .claude/scripts/token_report.py]
symbols: [sync_task_store, merge_store, foreign_ids, record_claims, normalize_usage, merge_cumulative_usage, install_deadline, file_lock, parse_transcript, scan_ide, scan_cli, rollup_turns, patch_overview, render]
upstream: [INDEX.md, overview/callchain-panorama.md]
downstream: [business/token-report.md, business/domain-models/token-ledger.md, integrations/index.md, contracts/configuration.md]
---

# `token-collectors`（IDE Hook 用量采集与台账底座组件）

本组件是四客户端共用的数据采集底座：`_token_common.py`（四处字节级相同副本，见 `.claude/hooks/_token_common.py:1-666`）提供锁、账本与合并内核；各客户端专属采集器解析自家会话转录；`token_report.py`（三处相同副本，见 `.claude/scripts/token_report.py:1-488`）在 Commit 准备期把台账渲染回任务概览。硬性 SLA：Hook 必须在宿主 3 秒超时内完成，全部失败非阻塞并记日志到 `/tmp/wally_debug/`（`.claude/hooks/_token_common.py:17-20`）。

## 1. 持久化与并发契约（数据底座层）

| 契约项 | 实现细节 | 锚点（`路径:行号`） |
|---|---|---|
| 任务台账写入 | `merge_store`：按 `request_id` 去重合并（新增/替换/自定义 usage 合并器），排序后原子替换写（临时文件 + `os.replace`） | `.claude/hooks/_token_common.py:466-541`、`write_json`（`_token_common.py:444-449`） |
| 互斥锁 | `file_lock`：对 `store:<path>` 的 SHA256 前 16 位命名的锁文件做定时非阻塞 `flock`，等待 `LOCK_WAIT_SEC=0.4` 超时即放弃（下次事件重试） | `.claude/hooks/_token_common.py:396-441`、`_token_common.py:84` |
| 自杀式 deadline | `install_deadline`：SIGALRM 到点 `os._exit(0)`（默认 `HOOK_DEADLINE_SEC=2.5`），防卡死 IDE | `.claude/hooks/_token_common.py:112-139`、`_token_common.py:81` |
| 非阻塞 stdin | `read_stdin_json`：select 轮询读管道，解出完整 JSON 即返回（宿主可能永不关管道） | `.claude/hooks/_token_common.py:153-193` |
| 防串账归属 | `sync_task_store` → `foreign_ids`：先查兄弟任务台账 + 仓外 claim ledger，剔除他人已认领的请求再合并 | `.claude/hooks/_token_common.py:639-666`、`_token_common.py:613-636` |
| 认领账本 | `record_claims`：写 `~/.wally/wally-init/token_claims.json`（env `WALLY_TOKEN_CLAIMS` 可覆盖），LRU 裁剪至 `MAX_CLAIMS=5000` | `.claude/hooks/_token_common.py:571-602`、`_token_common.py:66-69` |
| 分支黑名单 | `token_work_disabled`：`main/master/release*` 分支采集器立即 no-op，防止主线会话灌入历史任务 | `.claude/hooks/_token_common.py:223-234` |
| 任务定位 | `find_task_dir`：遍历 `.wally/plans/*/00-overview.md` 按分支匹配，取 mtime 最新者 | `.claude/hooks/_token_common.py:258-291` |

用量归一化不变量：`input_tokens = uncached + cache_read + cache_write` 且 `total = input + output`（`normalize_usage`，`.claude/hooks/_token_common.py:294-360`）；Cursor 的整轮回传计数器用逐元素取最大值合并而非求和（`merge_cumulative_usage`，`_token_common.py:363-393`）。

## 2. 四客户端采集器差异矩阵

| 采集器 | 挂载事件（配置锚点） | 转录源与解析函数 | 记录身份策略 |
|---|---|---|---|
| CodeBuddy（`.codebuddy/hooks/stop_token_collect.py`） | Stop（`.codebuddy/settings.json:3-13`） | IDE `index.json`（`scan_ide`，`stop_token_collect.py:236-278`）或 CLI jsonl（`scan_cli`，`stop_token_collect.py:319-370`）；可信根目录白名单见 `trusted_roots`（`stop_token_collect.py:69-82`），显式拒绝 Bugbot 目录（`stop_token_collect.py:91-93`） | IDE 请求 id / CLI assistant 消息 id；同会话连续同 prompt 的多请求在报表层聚成一回合（`_same_codebuddy_run`，`.claude/scripts/token_report.py:185-192`） |
| Claude Code（`.claude/hooks/claude_token_collect.py`） | Stop（`.claude/settings.json:3-13`） | JSONL 转录逐行解析（`parse_transcript`，`claude_token_collect.py:93-163`）；可信根 `~/.claude/projects` + `CLAUDE_CONFIG_DIR`（`claude_token_collect.py:38-45`） | `requestId` → `message.id` → `uuid` 兜底链（`claude_token_collect.py:132-137`） |
| Codex（`.codex/hooks/codex_token_collect.py`） | UserPromptSubmit/SubagentStart/Stop/SubagentStop/Interrupt/SessionEnd（`.codex/hooks.json:3-88`） | 运行态文件在 `~/.wally/wally-init/codex`（`state_dir`，`codex_token_collect.py:40-43`）；`token_usage_record` 是唯一记账源、`token_count` 仅诊断（`codex_token_collect.py:1-7`） | 会话绑定由 `registered_session` 解析父会话（`codex_token_collect.py:70-78`） |
| Cursor（`.cursor/hooks/cursor_token_collect.py`） | beforeSubmitPrompt/afterAgentResponse/stop（`.cursor/hooks.json:3-27`） | 事件载荷内嵌计数器（turn-cumulative，`cursor_token_collect.py:11-34`）；多根工作区候选序 `CURSOR_PROJECT_DIR → CLAUDE_PROJECT_DIR → workspace_roots → cwd`（`cursor_token_collect.py:128-142`） | `generation_id` 一代一记录，最大值合并 |

## 3. 报表渲染层（token_report.py）

渲染链：`main`（`.claude/scripts/token_report.py:430-484`）→ 定位分支概览（`find_overview`，`token_report.py:91-106`）→ 可选刷新 Codex 采集（`refresh_codex`，`token_report.py:402-414`）→ `rollup_turns` 按回合聚合（`token_report.py:195-230`：Cursor 单代不合、Codex 按 `root_turn_id` 求和、CodeBuddy 按连续同 prompt 求和）→ `render` 生成表体与会计口径说明（`token_report.py:278-365`）→ `patch_overview` 在 `TOKEN_USAGE_START/END` 标记对内原子替换（`token_report.py:417-427`，标记常量定义 `token_report.py:31-32`）。

## 4. 共变耦合与修改清单（Agent Co-Change Playbook）

- **改动 `_token_common.py` 任一副本**：四个目录（`.claude/.codebuddy/.codex/.cursor` 的 `hooks/`）的副本必须同步重拷（当前四处 MD5 一致：`5895cc4e654709a4ecc08ebebe24bdbe`）；该模块自述「stdlib-only、禁止反向 import 采集器」（`.claude/hooks/_token_common.py:12-15`），新增依赖会同时破坏四个客户端。
- **改 `token.json` 记录结构**：同步改 store 契约文档（`_token_common.py:22-45` 的 docstring）、[token-ledger 领域模型](../business/domain-models/token-ledger.md) 与 `token_report.py` 的列渲染（`token_report.py:304-323`）；旧记录兼容由 `load_store` 的防御性解析承担（`_token_common.py:456-463`）。
- **新增客户端采集器**：按差异矩阵复制四要素——hook 注册（各 settings/hooks.json）、可信根白名单、`sync_task_store` 调用（`.claude/hooks/_token_common.py:639-666`）、`token_report.py` 的 `client_label` 分支（`token_report.py:154-167`）；并更新 [agent-integration-layer](agent-integration-layer.md) 与 [配置手册](../contracts/configuration.md) 的环境变量表。
- **Hook 超时调参**：`HOOK_DEADLINE_SEC=2.5` / `STDIN_TIMEOUT_SEC=0.8` / `LOCK_WAIT_SEC=0.4`（`_token_common.py:81-84`）与各客户端 settings 的 `timeout: 3`（如 `.codebuddy/settings.json:9`）联动——宿主超时必须始终大于自杀 deadline，否则进程被 SIGKILL 在写盘中途。
