---
kind: operations
scope: runbook-and-observability
status: draft
origin: generated
source_paths: [.github/workflows/build-openwrt.yml, .github/workflows/update-checker.yml, .claude/hooks/_token_common.py, .claude/scripts/token_report.py, .wally/scripts/sop_context.py, .wally/review.md]
symbols: [df -hT, make V=s, DEVICE_NAME, cowtransfer.log, wetransfer.log, /tmp/wally_debug, token.json, collection, GITHUB_ENV]
upstream: [contracts/error-codes.md]
downstream: []
---

# 可观测性信号与故障定位决策树（Agent Runbook）

本仓无自建监控：可观测信号分三域——GitHub Actions run 页（构建事实）、`/tmp/wally_debug/` 与任务台账（SOP/采集事实）、`::warning` 注释流（外链分发事实）。本页给出常见症状到源码根因的确定性排查路径，全部锚点可在 Actions 日志与本地文件中直接核验。

## 1. 诊断信号读取入口表

| 信号类别 | 位置 | 核心诊断字段 | 源码锚点（`路径:行号`） |
|---|---|---|---|
| 构建 run 状态 | Actions 页 Build OpenWrt run | 步骤级成功/失败；`Compile the firmware` 的 `::set-output status`（`build-openwrt.yml:107`） | `.github/workflows/build-openwrt.yml:101-110` |
| 磁盘水位 | run 日志 `df -hT`（两处固定打印） | 挂载点容量 | `build-openwrt.yml:60`、`build-openwrt.yml:112-114`（`!cancelled()` 兜底打印） |
| 产物命名与机型 | Artifact 名 `OpenWrt_firmware<DEVICE_NAME><FILE_DATE>` | `DEVICE_NAME` 是否为空（多机型拼串） | `build-openwrt.yml:108-110`、`build-openwrt.yml:136` |
| 外链下载地址 | run 日志 `::warning file=<域名>::<url>` 行与 `cowtransfer.log`/`wetransfer.log` | URL 是否为空串 | `build-openwrt.yml:145-146`、`build-openwrt.yml:154-155` |
| 巡检哈希与派发 | Actions 页 Update Checker run | `commitHash` 步骤输出、`cache-hit` 结果、Trigger build 步骤状态 | `.github/workflows/update-checker.yml:33`、`update-checker.yml:37-52` |
| 采集健康 | 任务 `token.json` 的 `collection.codex.sessions`（sessions/sources/guardian_discovery 状态） | `ok/partial/missing/unsupported/pending` 标签 | `.claude/scripts/token_report.py:380-394` |
| 用量快照 | 概览 `TOKEN_USAGE_START/END` 区表格 | 回合数、总量、`—` 列语义 | `.claude/scripts/token_report.py:346-365` |
| Hook 失败细节 | `/tmp/wally_debug/stop_token_collect.log`（含 `deadline`、`skip ...` 行） | 时间戳 + 事件类别 | `.claude/hooks/_token_common.py:62-103` |
| SOP 门禁状态 | `.wally/local/sop-confirmations.json` 的 `pending/approved` | scope、fingerprint、reason、token | `.wally/scripts/sop_context.py:293-308` |
| 固件镜像类型 | 产物文件名（`...-squashfs-sysupgrade.bin` 等）与 `.config` rootfs 段 | `CONFIG_TARGET_ROOTFS_SQUASHFS` 等 | `.config:173-176` |

## 2. 症状 → 根因定位与修复决策树

### 症状 A：构建 run 失败在 `Compile the firmware`

1. 读日志中 `make -j$(nproc) || make -j1 || make -j1 V=s` 的最后一段（`build-openwrt.yml:106`）——`V=s` 输出即单线程详日志，逐包定位失败目标。
2. 查前一步 `Download package` 日志的 `find dl -size -1024c` 输出（`build-openwrt.yml:98-99`）：残包已被删，直接重跑 run 常可自愈。
3. 查 `Check space usage` 的 `df -hT`（`build-openwrt.yml:112-114`）：磁盘打满时考虑精简 `.config` 包选择（`.config` 包段）。
4. 仍失败：用 `workflow_dispatch` 的 `ssh` 输入开调试会话人工复现（`build-openwrt.yml:85-90`），或临时改 DIY 脚本插入 `make menuconfig` 保存诊断信息。

### 症状 B：构建成功但拿不到固件 Artifact

1. 确认 `Upload firmware directory` 步骤执行：守卫 `steps.organize.outputs.status == 'success'`（`build-openwrt.yml:134`）依赖 `UPLOAD_FIRMWARE: true`（`build-openwrt.yml:30`）。
2. 查 `Organize files` 日志 `FIRMWARE=$PWD`（`build-openwrt.yml:129`）：`cd openwrt/bin/targets/*/*` 失败说明编译无产物（回症状 A）。
3. 查 Artifact 名是否含 `DEVICE_NAME`：空机型说明 `grep '^CONFIG_TARGET.*DEVICE.*=y'` 无命中（`build-openwrt.yml:108-109`），核对 `.config:57` 目标段未破坏。

### 症状 C：update-checker 派发了但没触发构建

1. 查巡检 run 的 `Trigger build` 步骤是否成功（`update-checker.yml:47-52`）：失败多为 `ACTIONS_TRIGGER_PAT` 失效/无权限——在 GitHub secret 面轮换后手动重跑巡检。
2. 查 Build OpenWrt 工作流是否被禁用（`repository_dispatch` 接收端 `build-openwrt.yml:14` 需工作流处于启用态）。
3. 查 `cache-hit`（`update-checker.yml:37-41`）：命中即「已派发过」，属幂等短路而非故障。

### 症状 D：固件里缺 passwall 等预期包

1. 查 `Load custom feeds` 步骤日志确认 `diy-part1.sh:18` 追加行执行（`build-openwrt.yml:64-69`）。
2. 查 `Update feeds`/`Install feeds` 步骤失败（`build-openwrt.yml:71-75`）——feed 拉取失败即 job 失败，直接看日志。
3. feed 安装成功但包未选：`make defconfig` 会静默丢弃 `.config` 中悬空的 `CONFIG_PACKAGE_*` 选择（`build-openwrt.yml:96`），核对 `.config` 包选择段与 feed 名拼写一致。

### 症状 E：外链（cowtransfer/wetransfer）链接无效或为空

1. 查对应步骤日志 `transfer` 输出与 `cowtransfer.log`/`wetransfer.log`（`build-openwrt.yml:144`、`build-openwrt.yml:153`）。
2. 链接为空但开关开着：`grep https` 无命中（`build-openwrt.yml:145-146`），多为远端配额/网络问题；`release.txt` 会写入空链接行（`build-openwrt.yml:163-164` 只判开关不判 url），发布前人工核对。
3. `curl -fsSL git.io/file-transfer` 失败：短链服务不可用，见 [高危雷区](pitfalls-and-adr.md) 第 2 节的替换指引。

### 症状 F：Token 报表数值可疑（为零/偏小/重复）

1. 查概览报表区行尾 `> Snapshot: ...` 与 collection 状态行（`token_report.py:362`、`token_report.py:368-394`）：`missing/partial` 标签说明采集不完整，不可当零用量。
2. 查 `/tmp/wally_debug/stop_token_collect.log` 的 `skip merge: lock timeout` / `deadline: abort` 行（`_token_common.py:128`、`_token_common.py:540`）——锁竞争放弃的记录由后续事件补齐，重跑报表即可。
3. 查任务 `token.json` 记录数与报表「Records: N turns (model requests M)」对照（`token_report.py:328-331`）；差异大说明回合聚合口径（`token_report.py:195-230`）与预期不符，核对 [WF-04](../business/token-report.md) 第 3 节口径表。
4. 确认当前分支不在 `main/master/release*` 黑名单（`_token_common.py:223-234`），黑名单分支采集器静默 no-op。

### 症状 G：SOP 检查点一直等待确认

1. 读 `sop_context.py --checkpoint <x>` 输出的 reason 与 `CONFIRM SOP <token>` 口令（`sop_context.py:341-345`）。
2. 用户必须在聊天里逐字回复口令；确认后重跑 checkpoint（`accept_confirmation` 需经 UserPromptSubmit hook 或输入流接收，`sop_context.py:348-366`）。
3. 口令报 stale/unknown（`sop_context.py:366`）：证据指纹已变（`sop_context.py:287-289`），须重新跑 checkpoint 取新口令。

## 3. 证据留存与清理窗口约束

- 构建 run 只保留 3 条（`build-openwrt.yml:180-181`）、巡检 run 只保留 1 条（`update-checker.yml:56-57`）、Release 只留最近 3 个（`build-openwrt.yml:187-188`，开关默认关）——**排障取证要趁早**，删除后仅剩本地下载副本。
- `/tmp/wally_debug/` 与 runner 临时目录随机器回收（`_token_common.py:62`、`build-openwrt.yml` 全部 runner 路径）；长期证据只有仓内可提交文件与 Artifacts。
- 审计复核依据：`.wally/review.md:35-37` 要求 Review 阶段核对 `.wally/docs/INDEX.md`、codewiki 页与 `.wiki-meta.json` 映射的源码锚点一致性——知识库漂移是可观测性的一部分。

## 4. 共变耦合与修改清单（Agent Co-Change Playbook）

- 新增诊断信号：登记本页第 1 节表（位置 + 字段 + 锚点）；若是 workflow `::warning`/输出注入，同步产生它的步骤页（[WF-01](../business/firmware-build.md)）。
- 新增症状树：保持「先读信号 → 再对守卫链 → 最后给修复动作」结构，引用守卫须带 `path:line`。
- 清理策略调整（retain/keep 参数）：本页第 3 节与 [github-actions-platform 契约](../integrations/downstream/github-actions-platform.md) 2.5/2.6 节同步。
