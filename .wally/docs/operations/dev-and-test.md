---
kind: operations
scope: dev-and-test
status: draft
origin: generated
source_paths: [.github/workflows/build-openwrt.yml, .wally/scripts/create_task.py, .wally/scripts/sop_context.py, .claude/scripts/token_report.py, .wally/rules/README.md, .gitignore]
symbols: [workflow_dispatch, python3, stdlib, pytest, unittest]
upstream: [INDEX.md]
downstream: [operations/runbook-and-observability.md, contracts/configuration.md]
---

# 构建启动与单测验证手册（Agent Verification Playbook）

本仓无本地可运行服务、无依赖清单、无自动化测试套件（全仓 `Glob` 无 `tests/`、`Makefile`、`pyproject.toml`/`package.json` 等构建定义）：验证闭环是「CLI 干跑冒烟 + GitHub Actions 手动工作流冒烟」。Python 脚本全部 stdlib-only（`.claude/hooks/_token_common.py:12-15`），只需系统 `python3`。所有命令均核实自仓库源码，未配置项如实标注。

## 1. 环境前置与可执行命令速查

| 动作 | 精确执行命令 | 配置来源（`路径:行号`） | 备注 / 前置 |
|---|---|---|---|
| 验证 SOP 路由 | `python3 .wally/scripts/sop_context.py` | `.wally/scripts/sop_context.py:396` | 当前分支无任务时输出「No development task is bound…informational」（预期，非错误） |
| 验证建任务准入（可用性检查） | `python3 .wally/scripts/create_task.py 2026-01-01_smoke --branch feature/smoke`（会在 `.wally/plans/` 真实落盘，验证后删除目录） | `.wally/scripts/create_task.py:164-169` | 前置 `install-state.json` schema 2 / steps 6（`create_task.py:86-87`） |
| 阶段门禁检查 | `python3 .wally/scripts/sop_context.py --checkpoint implement` | `.wally/scripts/sop_context.py:398` | exit 0/1/3 语义见 [错误码字典](../contracts/error-codes.md) 第 1 节 |
| Token 报表干跑 | `python3 .claude/scripts/token_report.py` | `.claude/scripts/token_report.py:432` | 无任务分支时 `[ERROR] Cannot find 00-overview.md`（预期行为） |
| 固件构建冒烟（唯一集成验证） | GitHub Actions 页手动 `workflow_dispatch` 运行 Build OpenWrt | `.github/workflows/build-openwrt.yml:15-20` | runner `ubuntu-20.04`（`build-openwrt.yml:38`）；产物核对见 [部署与运行时](deployment-and-runtime.md) |
| 巡检工作流冒烟 | GitHub Actions 页手动运行 Update Checker | `.github/workflows/update-checker.yml:19` | 需已配置 `ACTIONS_TRIGGER_PAT` 才会真正派发 |
| 静态检查 / Lint | 未配置 | — | 全仓无 lint/typecheck 配置文件（如实记录） |
| 单元测试 | 未配置 | — | 本仓无 `tests/` 目录与测试框架（如实记录；单测规范位 `.wally/docs/unittest/unittest.md` 为 SOP 外部权威页） |

## 2. 源码模块 → 验证方式映射表（Module-to-Verification Map）

> 本仓无单测可跑；「必跑回归」退化为「改后最小验证动作」：

| 被修改的源码文件 / 目录 | 最小验证动作 | 验证要点 | 精确命令 |
|---|---|---|---|
| `.github/workflows/build-openwrt.yml` | Actions 手动冒烟 | run 成功 + Artifact `OpenWrt_firmware*` 出现（命名含 `DEVICE_NAME`/`FILE_DATE`） | Actions 页 Run workflow |
| `.github/workflows/update-checker.yml` | Actions 手动冒烟 | getHash 步骤输出哈希；新哈希时 Trigger build 步骤成功 | Actions 页 Run workflow |
| `diy-part1.sh` / `diy-part2.sh` / `.config` | Actions 构建冒烟 | feeds 步骤日志含 passwall；编译步骤无缺包告警 | Actions 页 Run workflow |
| `.wally/scripts/create_task.py` | 建任务冒烟（第 1 节命令） | 概览/阶段文件/execution.json 生成齐全 | `python3 .wally/scripts/create_task.py <name> --branch <b>` |
| `.wally/scripts/sop_context.py` | 路由 + checkpoint 双冒烟 | 交互态打印文档包；hook 态输出合法 JSON | `python3 .wally/scripts/sop_context.py` / `--hook --client codebuddy --event UserPromptSubmit` |
| `.wally/rules/README.md` | 只读核对（无额外项目规则的声明面） | 当前唯一规则文件即声明「无额外项目规则」（`.wally/rules/README.md:8`） | `python3 -c "print(open('.wally/rules/README.md').read()[:200])"` |
| 采集器任一副本（`.claude/.codebuddy/.codex/.cursor` hooks/scripts） | 四副本一致性核对 + hook JSON 输出 | `_token_common.py` 四处 MD5 相同；`token_report.py` 三处相同 | `md5 .*/hooks/_token_common.py .*/scripts/token_report.py` |
| `.agents/skills/*/SKILL.md`、`AGENTS.md` | 软链连通性 + 副本同步核对 | 六客户端软链指向正本；copilot 副本与 AGENTS 内嵌段一致 | `ls -la .claude/skills/ .agent/skills/` |
| `.wally/versions/3f84e7245abcc75d/**` | 禁止直接改（规则版本不变量） | 改动会令 `install-state.json` `managed_files` 哈希漂移 | 见 [高危雷区](pitfalls-and-adr.md) |

## 3. 测试环境隔离与编写规范

本仓尚无自建单测；未来新增测试的硬约定（取自 SOP 规则与现有代码防护机制）：

1. **不得直接改绑定规则目录**：`.wally/versions/3f84e7245abcc75d/` 是任务契约的不可变快照源（`.wally/scripts/create_task.py:91-94`），对其验证只能读。
2. **采集器测试不得触碰真实账本**：以 `WALLY_TOKEN_CLAIMS` 指向临时文件隔离认领账本（`.claude/hooks/_token_common.py:66-68`）；台账测试用临时目录构造 `token.json`（原子写契约见 [token-ledger](../business/domain-models/token-ledger.md) 第 3 节）。
3. **hook 时限即测试时限**：任何针对采集器的用例须在 2.5s 自杀 deadline 内完成（`.claude/hooks/_token_common.py:81`），阻塞用例会触发 `os._exit(0)` 假通过。
4. **提交禁令**：测试产生的 `token.json`、`sop-confirmations.json`、`/tmp/wally_debug/` 日志、Actions 日志均不提交（`.gitignore:2-4`、`.wally/versions/3f84e7245abcc75d/commit.md:6`）。
5. **新测试受 SOP 管辖**：新增测试文件属开发改动，须走 [WF-03 任务流程](../business/sop-task-lifecycle.md) 后提交；SOP 单测规范外部权威页位于 `.wally/docs/unittest/unittest.md`（本知识库不复制其正文）。

## 4. 共变耦合与修改清单（Agent Co-Change Playbook）

- 新增本地验证命令：登记本页第 1 节表（含配置来源锚点）；若属 CI 化改动（新 workflow / 新步骤），同步 [WF-01](../business/firmware-build.md) 或 [WF-02](../business/source-update-trigger.md)。
- 引入测试框架：本页第 1 节「未配置」两行如实更新；同时 `.wally/docs/unittest/unittest.md`（SOP 外部权威页）应由其维护流程填充，codewiki 不代写。
- Python 版本约束：脚本兼容 Python 3.6+（`sop_context.py:89` 注释：PIPE + universal_newlines 兼容 3.6；`create_task.py:11-17` 为 3.9 前兼容写法）——新增语法特性前先核对该约束。
