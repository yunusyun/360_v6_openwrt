---
kind: component
scope: agent-integration-layer
status: draft
origin: generated
source_paths: [.agents/skills/clarify/SKILL.md, .agents/skills/sop-aware-code-review/SKILL.md, .agents/skills/wally-open-cli/SKILL.md, .github/copilot-instructions.md, .agent/rules/wally-sop.md, .windsurf/rules/wally-sop.md, .cursor/rules/wally-sop.mdc, AGENTS.md, .claude/settings.json, .codebuddy/settings.json, .codex/hooks.json, .cursor/hooks.json]
symbols: [clarify, sop-aware-code-review, wally-open-cli, wally-init:entry, wally_sop:stop, wally:sop_context]
upstream: [INDEX.md, overview/system-architecture.md]
downstream: [architecture/sop-engine-and-routing.md, architecture/token-collectors.md, contracts/api-reference.md, contracts/configuration.md]
---

# `agent-integration-layer`（多客户端 AI Agent 接入层组件）

本组件把同一套 SOP 规则与采集器分发到七个 AI 客户端（`codex/cursor/codebuddy/claude/windsurf/copilot/antigravity`，登记于 `.wally/install-state.json:16-24`）：技能与规则以 `.agents/s` 为唯一正本、其余客户端全部软链引用；hook 经各家 settings 注册到 `sop_context.py --hook` 与 token 采集器。核心不变量：**正本唯一，副本零漂移**——任何规则改动只应发生在 `.agents/skills/` 与 `AGENTS.md`。

## 1. 技能与规则分发拓扑（正本 ↔ 软链）

| 正本（唯一权威） | 消费方软链（引用同一目标） | 链接形态证据 |
|---|---|---|
| `.agents/skills/clarify/SKILL.md` | `.claude/skills/clarify`、`.codebuddy/skills/clarify`、`.cursor/skills/clarify`、`.windsurf/skills/clarify`、`.agent/skills/clarify`、`.github/skills/clarify` | 六处软链均指向 `../../.agents/skills/clarify`（如 `.claude/skills/` 目录列表所示） |
| `.agents/skills/sop-aware-code-review/SKILL.md`（附 `references/sop.md`） | 同上六客户端 `sop-aware-code-review` 软链 | 同上；子参考文件随目录软链整体可见 |
| `.agents/skills/wally-open-cli/SKILL.md` | 同上六客户端 `wally-open-cli` 软链 | 同上 |
| `AGENTS.md`（协作入口正本） | `CLAUDE.md`、`CODEBUDDY.md` 为仓库根软链 | `CLAUDE.md -> AGENTS.md`、`CODEBUDDY.md -> AGENTS.md`（git 跟踪的符号链接） |
| `AGENTS.md` 内嵌的 SOP 入门段 | `.github/copilot-instructions.md` 为逐字同源副本（`wally-init:entry` 标记对包裹） | `.github/copilot-instructions.md:1-51` |
| 规则摘要 `.agent/rules/wally-sop.md` 与 `.windsurf/rules/wally-sop.md` | 两文件 MD5 相同（`367e92ffcae45acbed64692979185f25`），为指向 AGENTS.md 行为的浓缩副本；`.cursor/rules/wally-sop.mdc` 为 Cursor mdc 格式等价物（`alwaysApply: true`） | `.cursor/rules/wally-sop.mdc:1-12` |

三个 Skills 的职责边界：`clarify` 是进入阶段 01 的产品经理澄清技能（强制场景见 `.agents/skills/clarify/SKILL.md:19-28`）；`sop-aware-code-review` 是只读审查技能（`.agents/skills/sop-aware-code-review/SKILL.md:8`）；`wally-open-cli` 是把任务委托给开源 Wally 托管服务的客户端技能，明确区分 `@tencent/wally-open-cli`（开源包）与内部 `wally-cli`（禁装，`.agents/skills/wally-open-cli/SKILL.md:8-10`）。

## 2. Hook 注册矩阵（事件 → 脚本桥接）

| 客户端配置文件 | 注册的事件与脚本 | 超时 |
|---|---|---|
| `.claude/settings.json:3-36` | Stop → `claude_token_collect.py`（标记 `# wally:token_collect:Stop`）；UserPromptSubmit 与 PreToolUse → `sop_context.py --hook --client claude` | 3s / 5s / 5s |
| `.codebuddy/settings.json:2-36` | Stop → `stop_token_collect.py`（标记 `# wally_sop:stop`）；UserPromptSubmit 与 PreToolUse → `sop_context.py --hook --client codebuddy` | 3s / 5s / 5s |
| `.codex/hooks.json:3-88` | UserPromptSubmit（token 采集 + SOP 上下文双 hook）、SubagentStart、Stop、SubagentStop、Interrupt、SessionEnd → `codex_token_collect.py`；PreToolUse → `sop_context.py --hook --client codex` | 3s / 5s |
| `.cursor/hooks.json:3-40` | beforeSubmitPrompt（token + SOP 双 hook）、afterAgentResponse、stop → `cursor_token_collect.py`；sessionStart、preToolUse → `sop_context.py --hook --client cursor`（`failClosed: false`） | 3s / 5s |
| （无 hook 客户端） | Windsurf、Copilot、Antigravity 仅消费规则与技能软链，未注册 hook | — |

仓库根定位是所有 hook 命令的前置步骤：各配置统一用内联 Python 探测「从 cwd 向上及同级目录中含 `.wally/scripts/sop_context.py` 的目录」（如 `.claude/settings.json:18`），Codex/Claude 变体则用 `git rev-parse --show-toplevel`（`.codex/hooks.json:8`）或 `CLAUDE_PROJECT_DIR`（`.claude/settings.json:8`）。

## 3. Hook 输出协议矩阵

| 事件 | 输出形态 | 关键行为 |
|---|---|---|
| PreToolUse（sop_context） | CodeBuddy/Codex/Claude：`{"hookSpecificOutput": {permissionDecision: "deny", permissionDecisionReason: ...}}`；Cursor：`{"permission": "deny", ...}`（`.wally/scripts/sop_context.py:446-461`） | 仅编辑类工具且路径越界时 deny；其他放行（`check_tool`，`sop_context.py:369-391`） |
| UserPromptSubmit / sessionStart | `{"additional_context": <scope.md + 上下文摘要>}`；Cursor 变体 `{"additional_context": text}` | 注入 SOP 分类与任务路由（`sop_context.py:435-442`）；未绑定任务时输出「informational, not a blocker」 |
| UserPromptSubmit（确认接收） | `receipt` 文本追加 | `accept_confirmation` 匹配 `CONFIRM SOP <token>` 后把 pending 转 approved（`sop_context.py:348-366`） |
| Stop / afterAgentResponse（token） | 无输出（静默采集） | 失败仅写 `/tmp/wally_debug/stop_token_collect.log`（`.claude/hooks/_token_common.py:62`） |

## 4. 共变耦合与修改清单（Agent Co-Change Playbook）

- **修改技能正文**：只改 `.agents/skills/<name>/SKILL.md` 正本即可（软链自动生效）；但 `wally-init` 的 `install-state.json` `managed_files` 记录了三个 SKILL.md 的哈希（`.wally/install-state.json:43-46`），改动会造成安装账本漂移，须在 wally-init 升级侧对账。
- **修改 `AGENTS.md` 的 SOP 入门段**：`wally-init:entry:start/end` 标记对内的内容必须与 `.github/copilot-instructions.md` 逐字同步（两文件都完整包含该段），否则 Copilot 与其他客户端行为分叉。
- **新增客户端接入**：按「规则副本/软链 + skills 软链 + settings hook 注册」三件套扩展（参照 `.windsurf/` 的无 hook 最小形态），并把客户端标识加进 `sop_context.py --client` 枚举（`.wally/scripts/sop_context.py:399`）与 [token-collectors](token-collectors.md) 的差异矩阵。
- **Hook 命令调整**：改任一 settings 的命令串时核对「仓库根探测片段」与脚本相对路径的一致性（四客户端当前四份近似但非逐字相同的探测串，属既有实现事实）；同时更新 [配置手册](../contracts/configuration.md) 的项目目录环境变量表（`CODEBUDDY_PROJECT_DIR` 等，`.claude/scripts/token_report.py:439-441`）。
