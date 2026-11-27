---
kind: panorama
scope: glossary
status: draft
origin: generated
source_paths: [.github/workflows/build-openwrt.yml, .wally/scripts/sop_context.py, .wally/scripts/create_task.py, .wally/install-state.json, .wally/versions/3f84e7245abcc75d/plans-template/00-overview.md, .claude/hooks/_token_common.py]
symbols: [route, create, sync_task_store, repository_dispatch, workflow_dispatch]
upstream: [INDEX.md]
downstream: [business/index.md, business/domain-models/index.md]
---

# 领域术语与代码符号精确映射表

本仓是「GitHub Actions 云端构建 360 V6 路由器 OpenWrt 固件」+「Wally SOP 开发协作工作区」双形态仓库：上层是 CI 构建流水线，下层是面向 AI Agent 的六阶段开发流程资产。本表消除需求话术与源码符号之间的命名鸿沟；术语对应的流程展开见[业务流程总表](../business/index.md)，术语承载的数据结构见[领域模型总览](../business/domain-models/index.md)，完整符号调用链见[主干调用链全景](callchain-panorama.md)。

## 1. 自然语言术语 ↔ 源码符号映射字典

| 需求/业务常见叫法（含别名/缩写） | 精确技术与业务定义 | 源码对应符号 / 字段 | 权威定义位置（`路径:行号`） |
|---|---|---|---|
| 云端构建固件 / Actions 构建 | 在 GitHub 托管 runner 上拉取 lede 源码并编译出路由器固件的单 job 工作流 | `jobs.build`（`.github/workflows/build-openwrt.yml:37`） | `.github/workflows/build-openwrt.yml:36-190` |
| 手动触发构建 | 打开 Actions 页点 Run workflow 触发（可带 `ssh` 调试开关） | `workflow_dispatch.inputs.ssh` | `.github/workflows/build-openwrt.yml:15-20` |
| 事件触发构建 / 联动构建 | 外部系统发 `repository_dispatch` 事件触发本仓构建 | `repository_dispatch` | `.github/workflows/build-openwrt.yml:14` |
| 更新检查器 / 上游巡检 | 浅克隆 lede 仓比对 HEAD 哈希，变化则派发构建的独立工作流 | `name: Update Checker` | `.github/workflows/update-checker.yml:12-58` |
| 源码更新事件 | update-checker 通过 repository-dispatch 发出的事件类型字面量 | `event-type: Source Code Update` | `.github/workflows/update-checker.yml:52` |
| 上游源码仓 / lede | OpenWrt 源码提供方 coolsnowwolf/lede master 分支 | `REPO_URL` / `REPO_BRANCH` | `.github/workflows/build-openwrt.yml:23-24` |
| Feed / 软件源 | OpenWrt 扩展包仓库声明，写入 `feeds.conf.default` | `src-git passwall` | `diy-part1.sh:18` |
| DIY 脚本 part1 / feeds 前置定制 | feeds update 之前执行的定制脚本 | `DIY_P1_SH` | `.github/workflows/build-openwrt.yml:27`、`diy-part1.sh:1-18` |
| DIY 脚本 part2 / 配置后定制 | `.config` 载入后、编译前执行的定制脚本 | `DIY_P2_SH` | `.github/workflows/build-openwrt.yml:28`、`diy-part2.sh:1-14` |
| 目标机型 / 360 V6 | 编译目标设备：高通 IPQ60xx 平台 360_v6 型号 | `CONFIG_TARGET_ipq60xx_generic_DEVICE_360_v6` | `.config:57` |
| 设备名（产物命名用） | 编译成功后从 `.config` 提取的设备标识，注入 `DEVICE_NAME` | `grep '^CONFIG_TARGET.*DEVICE.*=y'` | `.github/workflows/build-openwrt.yml:108-109` |
| 产物时间戳 | 编译产物的分钟级时间戳环境变量 | `FILE_DATE` | `.github/workflows/build-openwrt.yml:110` |
| SOP / 开发流程 | 本仓的六阶段（Clarify→Plan→Implement→Deploy→IT→Review+Commit）任务流程 | `steps: 6` | `.wally/install-state.json:5` |
| 任务 / Task | 绑定分支与规则快照的开发任务目录 | `.wally/plans/<YYYY-MM-DD_title>/` | `.wally/scripts/create_task.py:95` |
| 任务概览 / 总览文件 | 任务的进度与元数据唯一事实源（Branch、Phase 等发现字段） | `00-overview.md` | `.wally/versions/3f84e7245abcc75d/plans-template/00-overview.md:1-65` |
| 任务创建脚本 | 从版本模板生成全套任务工件并快照执行方法配置的 CLI | `create_task.py` (`create`) | `.wally/scripts/create_task.py:69-161` |
| 任务路由脚本 / 检查点脚本 | 按当前分支路由到已绑定任务并执行门禁的 CLI/Hook 双形态脚本 | `route` / `checkpoint_issue` | `.wally/scripts/sop_context.py:87-173`、`.wally/scripts/sop_context.py:260-290` |
| 绑定规则版本 / 不可变规则 | 任务绑定 `.wally/versions/<revision>/core`，禁止回退最新版 | `revision`（16 位十六进制） | `.wally/install-state.json:4`、`.wally/scripts/create_task.py:91` |
| 确认口令 / CONFIRM SOP | 用户在聊天中回复以放行门禁例外的精确口令格式 | `CONFIRM SOP ([0-9a-f]{12})` | `.wally/scripts/sop_context.py:352-354` |
| Token 台账 / 用量账本 | 每任务一份的模型调用量 JSON 存储 | `token.json`（`store` 契约） | `.claude/hooks/_token_common.py:22-45` |
| 认领账本 / claim ledger | 仓库外记录「请求 ID 归属哪个任务」的防串账文件 | `token_claims.json` | `.claude/hooks/_token_common.py:66-69` |
| 采集器 / Collector | 挂在各 IDE hook 事件上增量合并用量记录的脚本 | `sync_task_store` | `.claude/hooks/_token_common.py:639-666` |
| Token 报表脚本 | Commit 前把 `token.json` 渲染回任务概览的脚本 | `render` / `patch_overview` | `.claude/scripts/token_report.py:278-365`、`.claude/scripts/token_report.py:417-427` |
| Skills 三件套 | 仓管技能：clarify、sop-aware-code-review、wally-open-cli | `.agents/skills/*/SKILL.md` | `AGENTS.md:24` |
| 双骨架占位页 | wally-init 预留、等待 wally-codewiki 填充的空页 | `origin: generated` 骨架 | `.wally/docs/integrations/index.md:1-10` |

## 2. 易混概念严格辨析（Disambiguation）

| 概念 A | 概念 B | 关键差异（粒度 / 读写时机 / 存储位置 / 混用后果） | 对照源码锚点 |
|---|---|---|---|
| `workflow_dispatch`（手动触发） | `repository_dispatch`（事件触发） | 前者来自 Actions 页面人工点击，可带 `ssh` 输入；后者由 update-checker 的 PAT 派发，载荷 `github.event.action` 含 `ssh` 时也会开调试。混用会导致误判触发来源 | `.github/workflows/build-openwrt.yml:14-15` vs `.github/workflows/update-checker.yml:49-52` |
| 任务分支路由（`route`） | 检查点确认（`require_confirmation`） | 前者只做「当前分支 ↔ 任务」匹配与上下文装配；后者在 Implement/Deploy/IT 前做证据完整性拦截并等待 `CONFIRM SOP` 口令。把路由当门禁会漏拦截 | `.wally/scripts/sop_context.py:87` vs `.wally/scripts/sop_context.py:325-345` |
| `token.json`（仓内任务台账） | `token_claims.json`（仓外认领账本） | 前者在 `.wally/plans/<task>/`，会被 `.gitignore` 排除提交；后者在 `~/.wally/wally-init/`，跨分支存活。删错后者会导致换任务后请求串账 | `.claude/hooks/_token_common.py:63` vs `.claude/hooks/_token_common.py:67-68` |
| `install-state.json`（安装状态） | `.wiki-meta.json`（codewiki 元数据） | 前者是 wally-init 的安装账本（含 `managed_files` 哈希）；后者是 wally-codewiki 的页面路由与基线。两套账本独立演进 | `.wally/install-state.json:27-52` vs 本库 `.wiki-meta.json` |
| `diy-part1.sh`（feeds 定制） | `diy-part2.sh`（编译前定制） | part1 在 `scripts/feeds update` 之前运行，改的是 feed 源；part2 在 `.config` 载入后运行，改的是源码树/配置。把加 feed 的逻辑写进 part2 不会生效 | `diy-part1.sh:10` vs `diy-part2.sh:10` |
| 采集（collector hook） | 报表（token_report.py） | 采集是高频增量合并（锁超时即放弃，下次重试）；报表是 Commit 前的一次性渲染，会回写 `00-overview.md`。把报表当采集跑会反复重写概览 | `.claude/hooks/_token_common.py:466-541` vs `.claude/scripts/token_report.py:417-427` |

## 3. 代码命名与枚举惯例

- **工作流环境变量前缀约定**：构建行为开关统一 `UPLOAD_*` 布尔字面量（`UPLOAD_BIN_DIR` / `UPLOAD_FIRMWARE` / `UPLOAD_COWTRANSFER` / `UPLOAD_WETRANSFER` / `UPLOAD_RELEASE`，定义于 `.github/workflows/build-openwrt.yml:29-33`）。
- **任务目录名铁律**：`YYYY-MM-DD_title` 正则 `\d{4}-\d{2}-\d{2}_[\w.-]+`，禁止路径分隔符与结尾句点（`.wally/scripts/create_task.py:71`）。
- **阶段文件名字面量**：`00-overview.md` 与 `01-clarify.md` … `06-review-commit.md` 固定映射（`.wally/scripts/sop_context.py:22-23`）；概览发现字段为表格键 `Branch` / `分支`、`SOP Rules` / `SOP 规则`、`Phase:` / `步骤：`（`.wally/scripts/sop_context.py:19-21`）。
- **采集记录身份字段**：`request_id`、`session_id`、`branch`、`client`（`codebuddy` / `cursor` / `codex` / `claude`）、`source`（如 `ide_index` / `cli_jsonl` / `claude_stop`）（`.claude/hooks/_token_common.py:26-45`）。
- **阶段状态字面量**：Deploy/IT 执行状态三态 `SKIPPED` / `PENDING` / `BLOCKED`，由 `readiness` 与 command/guide 配置组合决定（`.wally/scripts/create_task.py:120`）。
