---
kind: operations
scope: pitfalls-and-adr
status: draft
origin: generated
source_paths: [.github/workflows/build-openwrt.yml, .github/workflows/update-checker.yml, diy-part1.sh, diy-part2.sh, .wally/install-state.json, .wally/scripts/sop_context.py, .wally/scripts/create_task.py, .claude/hooks/_token_common.py, .wally/versions/3f84e7245abcc75d/core/knowledge.md, .gitignore]
symbols: [managed_files, revision, repository_dispatch, set-output, git.io, ubuntu-20.04, workflow_dispatch, wally-init]
upstream: [INDEX.md, overview/system-architecture.md, architecture/module-dep-matrix.md]
downstream: []
---

# 关键架构决策（ADR）、修改高危雷区与存量死路径警示

本页汇总决定本仓结构的硬性设计权衡、Agent 易误踩的隐式耦合雷区、以及「源码里搜得到但当前不生效」的存量死代码。改代码前必读本页。

## 1. 核心架构决策与不可违背的工程约束（Key ADRs）

| 编号 | 决策主题 | 核心机制与源码锚点 | 修改时必须遵守的硬约束 |
|---|---|---|---|
| `ADR-01` | 固件构建完全托管于 GitHub Actions（不在本地构建） | 构建链唯一入口 `workflow_dispatch`/`repository_dispatch`（`build-openwrt.yml:14-20`）；`.wally/config.json:9-13` 明确「无本地部署入口」缺口 | 不得引入本地构建路径冒充部署方法；客户方补 `.wally/config.json` 才解锁 Deploy 阶段 |
| `ADR-02` | SOP 规则版本不可变快照绑定 | 任务概览 `SOP Rules` 指向 `.wally/versions/3f84e7245abcc75d/core`，校验禁回退最新版（`sop_context.py:130-132`）；`create_task.py:91-94` 建任务时锁定 | 禁止原地修改 `versions/` 内容；规则演进走 wally-init 升级（新 revision 目录），历史任务按旧契约执行（`sop_context.py:116-129` compat 路径） |
| `ADR-03` | 任务-分支文本绑定（无数据库） | 分支匹配靠概览 Meta 表 `Branch` 字段正则（`sop_context.py:19`、`create_task.py:98-104`） | 改字段必须四处同步正则（`_token_common.py:280`、`token_report.py:100`）；不得引入「同分支多任务」布局 |
| `ADR-04` | Hook 永不阻塞宿主 IDE | 采集器 2.5s 自杀 deadline + 锁超时放弃 + 异常吞掉（`_token_common.py:81`、`_token_common.py:126-131`、`sop_context.py:466-472`） | 新增 hook 逻辑必须保持「失败即静默、下次事件补偿」语义；宿主 `timeout: 3` 恒大于自杀时限 |
| `ADR-05` | 认领账本置于仓库外 | `~/.wally/wally-init/token_claims.json`（`_token_common.py:68`）跨分支存活，防切换分支后串账 | 不得把 claim 状态移进仓内可提交文件；`WALLY_TOKEN_CLAIMS` 是唯一合法覆盖途径 |
| `ADR-06` | 正本唯一的多客户端分发 | 技能/入口规则正本在 `.agents/skills/` 与 `AGENTS.md`，六客户端软链引用；`_token_common.py`/`token_report.py` 多副本靠「拷贝一致」纪律 | 改副本必须全量同步重拷（MD5 一致性验证）；不得让单客户端副本私有分叉；SOP 工件忽略面以 `.gitignore:1-10` 为准 |

## 2. 修改高危雷区与隐式耦合表（Danger Zones & Hidden Couplings）

| 高危源码位置（`路径:行号`） | 涉及符号 | 隐蔽陷阱机制 | 安全修改标准动作 |
|---|---|---|---|
| `.wally/install-state.json:27-52` | `managed_files` 哈希表 | 两个 codewiki 骨架页（`overview/system-architecture.md`、`integrations/index.md`）也在哈希账本里；codewiki 填充内容后账本哈希必然失配 | 属预期漂移：wally-init 升级时对账；不要为保哈希而保留占位骨架 |
| `.github/workflows/build-openwrt.yml:23-24` + `update-checker.yml:15-16` | `REPO_URL`/`REPO_BRANCH` 双份 | 两工作流逐字重复无共享机制；只改一处即「巡检 A 构建 B」 | 永远四行一起改，并核对 [coolsnowwolf-lede 契约](../integrations/downstream/coolsnowwolf-lede.md) |
| `build-openwrt.yml:107` 等六处 `::set-output` | 步骤输出协议 | 使用 GitHub 已废弃的 `::set-output` 语法（107/110/130/146/155/161/165 行）；平台彻底下线该语法时六个守卫链全部失灵 | 迁移到 `echo "name=value" >> $GITHUB_ENV` / `echo "{outputs}">>$GITHUB_OUTPUT` 时逐处核对消费点（`build-openwrt.yml:118`、`134`、`141`、`150`、`169`、`173-175`） |
| `build-openwrt.yml:38` | `runs-on: ubuntu-20.04` | 旧版 runner 镜像面临平台退役；退役后构建 job 直接无法调度 | 换 runner 须同步复核 apt 清理行（`build-openwrt.yml:48`）与 `git.io/depends-ubuntu-2004` 依赖清单（`build-openwrt.yml:50`） |
| `build-openwrt.yml:50`、`build-openwrt.yml:143` | `git.io` 短链 | `git.io/depends-ubuntu-2004` 与 `git.io/file-transfer` 依赖短链服务存活；失效则环境初始化/外链工具获取失败 | 替换为上游直链（P3TERX 仓 depends 文件 / Mikubill/transfer release）并同步 [file-transfer 契约](../integrations/downstream/file-transfer.md) |
| `build-openwrt.yml:163-164` | `release.txt` 链接行 | `[ $UPLOAD_COWTRANSFER = true ]` 只判开关不判 url 非空——上传失败也写入空链接 Markdown | 发布前人工核对 release.txt 或加 `&& [ -n "$url" ]` 类守卫 |
| `diy-part2.sh` 执行位置 | DIY 注入点 | part1/part2 时机不可互换（feeds 前 vs 配置后）；把 feed 追加写进 part2 不会生效 | feed 类改动只进 `diy-part1.sh`；源码树/配置类才进 part2 |
| `.claude/scripts/token_report.py:420-426` | 概览标记对替换 | 正则替换要求 `TOKEN_USAGE_START/END` 恰一对；模板改版丢标记则所有任务渲染失败 | 模板改动必须保留标记对（`.wally/versions/3f84e7245abcc75d/plans-template/00-overview.md:51-53`） |
| `.wally/scripts/sop_context.py:19-21` | 三个发现正则 | 正则被 create_task、_token_common、token_report 以同形复用；改名漏一处即路由/防串账/报表三重失明 | 四处同步并回归四个 CLI 冒烟（[dev-and-test](dev-and-test.md) 第 1 节） |

## 3. 存量死路径与历史兼容保留代码警示（Legacy / Unmounted Code Traps）

| 存量代码位置（`路径:行号`） | 符号 / 配置名 | 当前真实状态 | 当前真正生效的入口 |
|---|---|---|---|
| `.github/workflows/update-checker.yml:20-21` | `schedule: cron 0 */18 * * *` | 注释态：定时巡检未启用 | 生效触发仅 `workflow_dispatch`（`update-checker.yml:19`） |
| `diy-part1.sh:13-14` | helloworld 取消注释 sed | 注释态存量（保留两种 helloworld 接法示例） | 生效的 feed 追加仅 passwall 行（`diy-part1.sh:18`） |
| `diy-part1.sh:17` | helloworld `src-git` 追加 echo | 注释态存量 | 同上 |
| `diy-part2.sh:14` | 默认 IP 修改 sed | 全文件唯一行为行即注释：part2 当前无实际生效定制 | part2 仍被工作流执行（`build-openwrt.yml:81-83`），但为 no-op 脚本 |
| `.github/workflows/build-openwrt.yml:116-121` | `Upload bin directory` 步骤 | 存在且注册，但守卫 `UPLOAD_BIN_DIR == 'true'` 默认 false 恒跳过 | 默认生效分发仅 `Upload firmware directory`（`build-openwrt.yml:132-137`） |
| `.github/workflows/build-openwrt.yml:139-155`、`build-openwrt.yml:157-175`、`build-openwrt.yml:183-190` | cowtransfer/wetransfer/release/旧件清理 | 步骤齐备但开关默认 false，恒跳过 | 同上 |
| `.wally/docs/overview/system-architecture.md`、`.wally/docs/integrations/index.md` | wally-init 双骨架占位页 | 本次 codewiki 填充前的占位（`origin: generated` 骨架声明） | 现已由本知识库正式页面取代（本页与 [integrations/index](../integrations/index.md)） |
| `.wally/docs/apis/`、`sqls/`、`unittest/`、`integration_test/`、`devops/` | SOP 规范外部页 | wally-init 生成的模板态权威页（多为待填骨架），不属 codewiki 管辖 | codewiki 通过 [INDEX](../INDEX.md) 链接不复制（`.wally/versions/3f84e7245abcc75d/core/knowledge.md:11`） |

## 4. 共变耦合与修改清单（Agent Co-Change Playbook）

- 解注释任何「存量开关」（第 3 节表）：先确认配套开关/secret 已就绪（如启用 release 需理解 `GITHUB_TOKEN` 权限，`build-openwrt.yml:171`），再同步 [配置手册](../contracts/configuration.md) 与 [WF-01](../business/firmware-build.md) 判定表。
- 迁移 `::set-output`：属全链路协议改动，六个产出点与全部消费守卫一次迁移，逐处登记本页第 2 节表。
- 触碰 `install-state.json`：只读（wally-init 的账本）；发现哈希失配先判定是「预期演进」还是「意外篡改」再行动。
- 本页雷区表新增条目时：保持「位置 → 符号 → 陷阱机制 → 标准动作」四列结构，并与 [module-dep-matrix](../architecture/module-dep-matrix.md) 的高扇入底座表交叉引用。
