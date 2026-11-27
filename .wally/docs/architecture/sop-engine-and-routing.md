---
kind: component
scope: sop-engine-and-routing
status: draft
origin: generated
source_paths: [.wally/scripts/create_task.py, .wally/scripts/sop_context.py, .wally/core/entry.md, .wally/core/scope.md, .wally/config.json, .wally/install-state.json, .wally/versions/3f84e7245abcc75d/core/workflow.md, .wally/versions/3f84e7245abcc75d/core/knowledge.md, .wally/versions/3f84e7245abcc75d/plans-template/00-overview.md, .wally/rules/README.md, .wally/profiles/project.md, .wally/review.md]
symbols: [create, execution_config, route, checkpoint_issue, require_confirmation, accept_confirmation, check_tool, method_configured, plain_document]
upstream: [INDEX.md, overview/callchain-panorama.md]
downstream: [business/sop-task-lifecycle.md, business/domain-models/sop-task-artifacts.md, contracts/api-reference.md, contracts/error-codes.md, contracts/configuration.md]
---

# `sop-engine-and-routing`（SOP 规则引擎与任务路由 CLI 组件）

本组件是仓库的第二条业务主线：由两个 stdlib-only Python CLI（`create_task.py` 建任务、`sop_context.py` 路由/门禁）加上不可变规则版本树（`.wally/versions/3f84e7245abcc75d/`）构成的六阶段开发流程引擎。核心设计不变量：任务通过「概览文件里的 Branch 字段 ↔ 当前 Git 分支」文本匹配绑定，规则通过 16 位修订号快照绑定，二者都不依赖数据库。

## 1. 双 CLI 入口与职责分界

| CLI 命令 | 职责 | 核心函数（`路径:行号`） | 幂等性 |
|---|---|---|---|
| `python3 .wally/scripts/create_task.py <name> --branch <b>` | 从版本模板生成任务目录（模板展开 + 执行方法快照 + Deploy/IT 状态注记） | `create`（`.wally/scripts/create_task.py:69-161`）、`execution_config`（`.wally/scripts/create_task.py:26-53`） | 同名任务已存在即报错（`create_task.py:96-97`）；目录经临时 staging 原子改名落盘（`create_task.py:148-159`） |
| `python3 .wally/scripts/sop_context.py`（交互态） | 装配当前分支绑定任务的完整上下文文档包 | `route`（`.wally/scripts/sop_context.py:87-173`）、`context_text`（`.wally/scripts/sop_context.py:176-198`） | 纯只读 |
| `python3 .wally/scripts/sop_context.py --checkpoint implement\|deploy\|it` | 阶段前置门禁：证据不齐时落盘 pending 确认并要求用户回复口令 | `checkpoint_issue`（`.wally/scripts/sop_context.py:260-290`）、`require_confirmation`（`.wally/scripts/sop_context.py:325-345`） | 确认状态带指纹去重（`sop_context.py:287-289`） |
| `python3 .wally/scripts/sop_context.py --hook --client <c> --event <e>` | IDE Hook 形态：PreToolUse 拦截越界编辑、UserPromptSubmit 接收确认口令 | `check_tool`（`.wally/scripts/sop_context.py:369-391`）、`accept_confirmation`（`.wally/scripts/sop_context.py:348-366`） | 无确认时不重试（`sop_context.py:343-345`） |

任务创建前置校验链（`create_task.py:69-109`）：任务名正则 `\d{4}-\d{2}-\d{2}_[\w.-]+`（`create_task.py:71`）→ 分支名禁含 `` `|`` 换行且不得为 `main/master/HEAD`（`create_task.py:74`）→ `install-state.json` 的 `schema==2`、`steps==6`、`revision` 为 16 位十六进制（`create_task.py:86-90`）→ 同分支不得已属于其他任务（`create_task.py:98-104`，只读概览的 Branch 发现字段）→ 模板与项目 Profile 必须存在（`create_task.py:105-109`）。

## 2. 路由与门禁执行调用树

```text
sop_context.py main (sop_context.py:394-462)
  ├─ [route] route (sop_context.py:87-173)
  │    ├─ git branch --show-current (sop_context.py:90-93)  无分支 => ValueError 直接失败
  │    ├─ 遍历 .wally/plans/*/00-overview.md 按 Branch 字段精确匹配 (sop_context.py:97-104)
  │    │    ├─ 多任务命中 => ValueError "Multiple SOP tasks match" (sop_context.py:106-107)
  │    │    └─ 零命中 => ValueError 提示先问用户是否开任务分支 (sop_context.py:108-114)
  │    ├─ [legacy] 无 SOP Rules 字段 => 读 .wally/compat/v1 快照 (sop_context.py:116-129)
  │    └─ [versioned] 校验 Rules 路径形如 .wally/versions/<16hex>/core (sop_context.py:130-132)
  │         ├─ 校验 Phase 编号与当前阶段文件名一致 (sop_context.py:133-143)
  │         ├─ 装载 workflow/gates/interaction/profile/阶段文件 (sop_context.py:144-151)
  │         ├─ [role=03] 追加 .wally/rules/ 全量规则文件 (sop_context.py:152-165)
  │         └─ [role=01] 追加 clarify.md (sop_context.py:170-171)
  ├─ [checkpoint] checkpoint_issue (sop_context.py:260-290)
  │    ├─ implement: 01/02 未勾选或仍为空模板 => gap (sop_context.py:268-274)
  │    └─ deploy/it: execution.json 的方法未配置 => gap (sop_context.py:276-284)
  ├─ [confirm] require_confirmation (sop_context.py:325-345)
  │    └─ 写 .wally/local/sop-confirmations.json (pending + 12hex token) 并返回提示
  └─ [hook] check_tool (sop_context.py:369-391)
       ├─ 仅拦编辑类工具 (sop_context.py:372) 且路径不在任务目录内 (sop_context.py:387)
       ├─ plain_document 白名单: 根级 README/LICENSE 等 + .wally/docs 被动文档 (sop_context.py:210-256)
       └─ 命中 => 返回 deny 决策 (sop_context.py:449-460)
```

退出码契约：交互态 `--checkpoint` 有未决确认时 `exit 3`（`sop_context.py:417-419`）；Hook 态任何异常都吞掉并输出 `{}`（`sop_context.py:466-472`）——Hook 失败绝不升级为全局工具禁令。

## 3. 规则版本树与配置快照机制

| 机制 | 实现细节 | 锚点（`路径:行号`） |
|---|---|---|
| 不可变规则包 | `.wally/versions/3f84e7245abcc75d/`：`core/workflow.md`、`core/gates.md`、`core/interaction.md`、`core/clarify.md`、`core/knowledge.md` + `plans-template/*` + `profiles/generic/project.md` + `commit.md` + `review.md`；任务概览的 `SOP Rules` 字段指向其 `core` 目录 | `.wally/scripts/create_task.py:91-94`、绑定校验 `.wally/scripts/sop_context.py:130-132` |
| 路由文档包 | `route` 装配 `.wally/core/entry.md`（入口）+ 概览 + 绑定 `core/workflow.md`/`core/gates.md`/`core/interaction.md` + `.wally/profiles/project.md` + 当前阶段文件；hook 态注入 `.wally/core/scope.md` 分类规则 | `.wally/scripts/sop_context.py:144-151`、`sop_context.py:437-439` |
| 模板展开 | `{{STEPS}}`→`6`、`{{RULE_ROOT}}`→`.wally/versions/<rev>`；概览标题/分支/创建日期三处替换 | `.wally/scripts/create_task.py:56-66`、`create_task.py:111-113` |
| 执行方法快照 | 任务创建时把 `.wally/config.json` 整体复制为任务内 `execution.json`；`readiness: pending` + `gaps` => Deploy/IT 标 `BLOCKED`，未配置 => `SKIPPED`（决策时间记真实时间） | `.wally/scripts/create_task.py:114-146`、`method_configured`（`create_task.py:20-23`） |
| 安装账本 | `.wally/install-state.json` 记录 `revision`、`clients`、`managed_files` 哈希与 `language_basis` | `.wally/install-state.json:1-56` |
| 项目规则位 | `.wally/rules/` 当前仅有 README（声明无额外项目规则） | `.wally/rules/README.md:1-9` |

## 4. 共变耦合与修改清单（Agent Co-Change Playbook）

- **新增/修改绑定规则文件**：只改 `.wally/versions/3f84e7245abcc75d/` 内文件后，`install-state.json` 的 `managed_files` 哈希会失配（`.wally/install-state.json:27-52`）——wally-init 升级流程负责核对；日常 Agent 修改前先跑 `python3 .wally/scripts/sop_context.py` 确认路由仍正常。
- **改概览发现字段正则**：`BRANCH`/`RULES`/`PHASE` 三个正则（`.wally/scripts/sop_context.py:19-21`）被 `create_task.py:102`、`_token_common.py:280`、`token_report.py:100` 处复用同形模式——改字段名必须四处同步，否则路由、防串账、报表同时失明。
- **改确认口令格式**：`accept_confirmation` 的 `CONFIRM SOP ([0-9a-f]{12})`（`.wally/scripts/sop_context.py:352-354`）与 AGENTS.md 说明（`AGENTS.md:52`）必须同步，且 Hook 输入端见 [agent-integration-layer](agent-integration-layer.md)。
- **新增 CLI 参数**：`create_task.py` 的 argparse（`create_task.py:164-169`）与 `sop_context.py` 的 argparse（`sop_context.py:394-401`）互不共享；文档侧同步 [contracts/api-reference.md](../contracts/api-reference.md) 的 CLI 命令表与错误分支 [contracts/error-codes.md](../contracts/error-codes.md)。
- **任务工件结构变更**：阶段文件名映射 `FILENAMES`（`sop_context.py:22-23`）与模板目录 `plans-template/`（`.wally/versions/3f84e7245abcc75d/plans-template/00-overview.md:25-30`）联动；改动后旧任务概览将无法路由，须评估 compat 兼容路径（`sop_context.py:116-129`）。
