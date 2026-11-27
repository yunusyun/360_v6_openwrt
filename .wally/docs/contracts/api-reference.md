---
kind: api-contract
scope: inbound-api
status: draft
origin: generated
source_paths: [.github/workflows/build-openwrt.yml, .github/workflows/update-checker.yml, .wally/scripts/create_task.py, .wally/scripts/sop_context.py, .claude/scripts/token_report.py]
symbols: [workflow_dispatch, repository_dispatch, main, create, route, require_confirmation, argparse]
upstream: [INDEX.md]
downstream: [integrations/topology-and-mapping.md, contracts/error-codes.md, architecture/sop-engine-and-routing.md]
---

# 入向接口契约全集（Workflow 触发 / CLI / IDE Hook Reference）

本仓无 HTTP 服务：入向接口由三族构成——GitHub Actions 工作流触发器（2 个）、本地 Python CLI（3 个入口脚本）、IDE Hook 事件（由四家客户端 settings 注入载荷）。本页为全量清单，共 6 组入口，无省略。

## 1. 公共协议与鉴权约定

| 维度 | 精确实现规则 | 源码锚点（`路径:行号`） |
|---|---|---|
| 工作流鉴权 | GitHub 仓库权限（Actions 页运行权 / PAT 派发权）；无仓内鉴权代码 | `.github/workflows/build-openwrt.yml:13-20` |
| CLI 鉴权 | 无：脚本不做身份校验，依赖文件系统权限；`sop_context.py` 自述「workflow check, not a sandbox」（docstring） | `.wally/scripts/sop_context.py:1-6` |
| Hook 载荷形态 | stdin JSON（`read_stdin_json` 非阻塞解析，0.8s 超时） | `.claude/hooks/_token_common.py:153-193` |
| CLI 失败输出约定 | `[ERROR] <message>`（stderr，exit 1）；checkpoint 待确认 exit 3 | `.wally/scripts/create_task.py:172-173`、`.wally/scripts/sop_context.py:414-419` |

## 2. 全量入向接口路由总表

| 分组 | 协议 / 触发 | 精确入口 | 鉴权/前置 | 执行模式 | Handler 锚点 | 下游落点 |
|---|---|---|---|---|---|---|
| 工作流 | `workflow_dispatch` | Build OpenWrt 手动运行（输入 `ssh`） | 仓库 Actions 权限 | 异步单 job | `.github/workflows/build-openwrt.yml:15-20` | [WF-01](../business/firmware-build.md) |
| 工作流 | `repository_dispatch` | Build OpenWrt 事件 `Source Code Update` | `ACTIONS_TRIGGER_PAT` 派发 | 异步单 job | `.github/workflows/build-openwrt.yml:14` | WF-01 |
| 工作流 | `workflow_dispatch` | Update Checker 手动运行（无输入；cron 注释态） | 仓库 Actions 权限 | 异步单 job | `.github/workflows/update-checker.yml:19` | [WF-02](../business/source-update-trigger.md) |
| CLI | `python3` argv | `create_task.py <name> --branch <b> [--repo-root .]` | 文件系统 | 同步 | `.wally/scripts/create_task.py:164-173` | 任务目录落盘 |
| CLI | `python3` argv | `sop_context.py [--repo .] [--hook] [--checkpoint x] [--client c] [--event e]` | 绑定任务路由 | 同步 / Hook | `.wally/scripts/sop_context.py:394-401` | 上下文/确认状态 |
| CLI | `python3` argv | `token_report.py [repo] [--branch b] [--overview p]`（三客户端目录等价副本） | 分支可路由 | 同步 | `.claude/scripts/token_report.py:430-435` | 概览报表区 |
| Hook | stdin JSON | Stop / UserPromptSubmit / PreToolUse / Subagent* / Interrupt / SessionEnd 等（四客户端注册矩阵） | 客户端注入 | ≤3s | 各 settings 注册（[agent-integration-layer](../architecture/agent-integration-layer.md) 第 2 节） | 采集/门禁 |

## 3. 逐接口入参/出参精确契约

### 3.1 `workflow_dispatch` — Build OpenWrt（`.github/workflows/build-openwrt.yml:15-20`）

- **入参**：

| 位置 | 字段 | 类型 | 必填 | 默认 / 约束 | 含义与去向 |
|---|---|---|---|---|---|
| inputs | `ssh` | `str` | 否 | `'false'` | `'true'` 时第 7 步开 tmate 调试（`build-openwrt.yml:87`） |

- **出参**：无返回体；结果为 Actions run 状态 + Artifact `OpenWrt_firmware*`（`build-openwrt.yml:136`）。
- **错误分支**：任一步非零退出 → job failure（无自定义错误体）。

### 3.2 `repository_dispatch` — Build OpenWrt（`.github/workflows/build-openwrt.yml:14`）

- **入参**：事件头 `event-type: Source Code Update`（发送方 `.github/workflows/update-checker.yml:52`）；载荷 `github.event.action` 含 `ssh` 时启用调试步骤（`build-openwrt.yml:87`）。
- **出参/错误**：同 3.1。

### 3.3 `workflow_dispatch` — Update Checker（`.github/workflows/update-checker.yml:18-21`）

- **入参**：无。
- **出参**：步骤输出 `steps.getHash.outputs.commitHash`（run 内部消费于缓存键，`update-checker.yml:40`）。
- **错误分支**：clone 失败 / 派发失败 → job failure。

### 3.4 `create_task.py` CLI（`.wally/scripts/create_task.py:164-173`）

- **入参**：

| 位置 | 字段 | 类型 | 必填 | 默认 / 约束 | 含义 |
|---|---|---|---|---|---|
| argv | `name` | `str` | 是 | `\d{4}-\d{2}-\d{2}_[\w.-]+` | 任务目录名 |
| argv | `--branch` | `str` | 是 | 非默认分支、无特殊字符 | 绑定分支 |
| argv | `--repo-root` | `str` | 否 | `.` | 仓库根 |

- **出参**：stdout `Created <task-path> (SOP 2.0 / <revision>)`（`create_task.py:160`）。
- **错误分支**：全部校验失败路径见 [错误码字典](error-codes.md) 第 2 节（`[ERROR] <ValueError 文本>`，exit 1）。

### 3.5 `sop_context.py` CLI / Hook（`.wally/scripts/sop_context.py:394-401`）

- **入参**：

| 位置 | 字段 | 类型 | 必填 | 默认 / 约束 | 含义 |
|---|---|---|---|---|---|
| argv | `--repo` | `str` | 否 | `.` | 仓库根 |
| argv | `--hook` | flag | 否 | 关 | stdin JSON 载荷模式 |
| argv | `--checkpoint` | enum | 否 | — | `implement / deploy / it` |
| argv | `--client` | enum | 否 | `codex` | `codex / codebuddy / cursor / claude` |
| argv | `--event` | `str` | 否 | `PreToolUse` | hook 事件名 |
| stdin | hook 载荷 | JSON | hook 模式必填 | 须为 dict | 工具调用/提示载荷 |

- **出参**：交互态打印绑定文档包（`context_text`，`sop_context.py:176-198`）；checkpoint 态打印放行/确认提示；hook 态打印客户端协议 JSON（`sop_context.py:446-461`）。
- **错误分支**：无分支路由失败 → `[ERROR]`/warning 文本；hook 态异常吞掉输出 `{}`（`sop_context.py:466-472`）。

### 3.6 `token_report.py` CLI（`.claude/scripts/token_report.py:430-484`，三副本等价）

- **入参**：

| 位置 | 字段 | 类型 | 必填 | 默认 / 约束 | 含义 |
|---|---|---|---|---|---|
| argv | `repo` | `str` | 否 | 探测 `CODEBUDDY/CLAUDE/CURSOR_PROJECT_DIR` → cwd | 起始查找目录 |
| argv | `--branch` | `str` | 否 | `git branch --show-current` | 概览定位 |
| argv | `--overview` | `path` | 否 | 自动按分支匹配 | 直指概览文件 |

- **出参**：stdout `[OK] Token rollup: <N> turns ...`（`token_report.py:479-483`）；副作用为概览标记对内替换。
- **错误分支**：找不到 `.wally` / 概览 / 标记对缺失 → `[ERROR]`/`RuntimeError`，见 [错误码字典](error-codes.md)。

### 3.7 IDE Hook 事件入向（四客户端注册矩阵）

| 客户端 | 事件 → 处理脚本（载荷要点） | 注册锚点 |
|---|---|---|
| CodeBuddy | Stop → `stop_token_collect.py`（stdin：`session_id`/`transcript_path`/`cwd`） | `.codebuddy/settings.json:2-13` |
| Claude | Stop → `claude_token_collect.py`（stdin：`session_id`/`transcript_path`/`cwd`） | `.claude/settings.json:3-13` |
| Codex | UserPromptSubmit/SubagentStart/Stop/SubagentStop/Interrupt/SessionEnd → `codex_token_collect.py` | `.codex/hooks.json:3-77` |
| Cursor | beforeSubmitPrompt/afterAgentResponse/stop → `cursor_token_collect.py`（载荷含 `workspace_roots`、`generation_id`、计数器） | `.cursor/hooks.json:3-27` |
| 四客户端 SOP 桥 | prompt/工具事件 → `sop_context.py --hook`（PreToolUse 载荷含 `tool_name`/`tool_input`） | 各 settings 同文件 |

## 4. 新增接口的共变清单（Co-Change Playbook）

- 新增工作流输入：`on.workflow_dispatch.inputs`（`build-openwrt.yml:15-20`）+ 消费步骤 `if` 守卫（如 `build-openwrt.yml:87`）成对；同步本页 3.1 表与 [WF-01](../business/firmware-build.md) 判定表。
- 新增 CLI 参数：argparse 定义（各脚本 `main`）与 [sop-engine-and-routing 组件](../architecture/sop-engine-and-routing.md) 第 1 节职责表同步；错误文本入 [错误码字典](error-codes.md)。
- 新增 hook 事件：客户端 settings 注册 + 采集器/路由器事件分支 + [agent-integration-layer](../architecture/agent-integration-layer.md) 矩阵三处联动。
