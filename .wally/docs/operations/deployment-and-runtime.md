---
kind: operations
scope: deployment-and-runtime
status: draft
origin: generated
source_paths: [.github/workflows/build-openwrt.yml, .github/workflows/update-checker.yml, .wally/config.json, .gitignore, .claude/hooks/_token_common.py]
symbols: [runs-on, /workdir, GITHUB_WORKSPACE, bin/targets, Artifacts, releases, token_claims.json]
upstream: [INDEX.md, contracts/configuration.md]
downstream: [operations/runbook-and-observability.md, integrations/downstream/github-actions-platform.md]
---

# 部署产物、进程拓扑与运行期目录契约

本仓无自托管服务部署：唯一「部署」是 GitHub Actions 托管 runner 上的构建产物分发（Artifacts / 可选 Release），「运行时」分两段——构建 job 的 runner 磁盘布局与本地开发机的 SOP 工具运行期目录。`.wally/config.json:9-13` 明确记录三条部署缺口（无本地部署入口、无目标环境、无刷机回滚），属客户方待补配置。

## 1. 构建资产清单

| 构建文件（`路径`） | 构建产物 | 关键约束 |
|---|---|---|
| `.github/workflows/build-openwrt.yml` | 固件 Artifact `OpenWrt_firmware<DEVICE_NAME><FILE_DATE>`（默认开）；可选 bin 目录 Artifact、Release、外链 | runner `ubuntu-20.04`（`build-openwrt.yml:38`）；产物目录 `openwrt/bin/targets/*/*` 去 packages（`build-openwrt.yml:127-129`） |
| `.github/workflows/update-checker.yml` | 无固件产物；Actions 缓存条目 `.commitHash` + 派发事件 | runner `ubuntu-latest`（`update-checker.yml:25`） |
| `.config` + `diy-part1.sh` + `diy-part2.sh` | （构建输入资产，非独立产物） | 目标 `360_v6`（`.config:57`）；feed 追加点（`diy-part1.sh:18`） |

## 2. 构建期 runner 磁盘目录树契约（Build-Time Disk Layout）

> 编写工作流步骤/排障前必须对齐的运行期目录层次（全部位于 GitHub 托管 runner，job 结束即回收）：

```text
$GITHUB_WORKSPACE/              # 本仓 checkout 根（actions/checkout, build-openwrt.yml:41-42）
├── openwrt -> /workdir/openwrt # 软链桥（ln -sf, build-openwrt.yml:62）
├── diy-part1.sh                # chmod +x 后执行（build-openwrt.yml:67）
├── diy-part2.sh                # chmod +x 后执行（build-openwrt.yml:81）
├── .config                     # mv 进源码树后原位消失（build-openwrt.yml:80）
├── cowtransfer.log             # 可选通道日志（build-openwrt.yml:144）
├── wetransfer.log              # 可选通道日志（build-openwrt.yml:153）
├── release.txt                 # 可选发布正文（build-openwrt.yml:162）
└── DEVICE_NAME                 # 机型提取临时文件（build-openwrt.yml:108）
/workdir/                       # 独立构建根（sudo mkdir + chown, build-openwrt.yml:54-55）
└── openwrt/                    # lede 全量克隆 + feeds + .config + 编译树
    └── bin/targets/ipq60xx/generic/   # 固件输出目录（cd 通配进入, build-openwrt.yml:127）
```

巡检 job 布局：`.commitHash` 写在 job 工作目录并进 Actions 缓存（`update-checker.yml:39`、`update-checker.yml:45`）。

## 3. 本地开发机运行期目录契约（SOP 工具态）

| 目录 / 文件 | 产生者 | 生命周期 | 锚点 |
|---|---|---|---|
| `.wally/plans/<task>/`（含 `token.json`） | `create_task.py` / 采集器 | 跟随任务；`token.json` 永不提交 | `.gitignore:3-4` |
| `.wally/local/`（含 `sop-confirmations.json`） | `sop_context.py` 确认流 | 会话级，gitignore | `.gitignore:2`、`.wally/scripts/sop_context.py:295` |
| `~/.wally/wally-init/token_claims.json` | 采集器认领 | 跨仓/跨分支持久（LRU 5000） | `.claude/hooks/_token_common.py:68` |
| `~/.wally/wally-init/codex/*.json` | Codex 采集器运行态 | 跨会话（hash 命名 per-session） | `.codex/hooks/codex_token_collect.py:46-48` |
| `/tmp/wally_debug/` | 全部 hook 失败日志 | 机器重启即失 | `.claude/hooks/_token_common.py:62` |
| `/tmp/wally_debug/token-<digest>.lock` | 采集互斥锁 | 短暂 | `_token_common.py:404-406` |

## 4. 进程、子进程与外部网络边界

| 边界类型 | 方向 / 介质 | 调用方（`路径:行号`） | 安全与超时约束 |
|---|---|---|---|
| 出向 git | HTTPS clone lede（全量/浅层） | `build-openwrt.yml:61`、`update-checker.yml:32` | 无凭据（公开仓）；无显式超时 |
| 出向 HTTP | `git.io` 短链（依赖安装与 transfer 工具） | `build-openwrt.yml:50`、`build-openwrt.yml:143` | `curl -fsSL` fail-fast 语义 |
| 出向 API | Release / workflow 清理 / dispatch（GITHUB_TOKEN / PAT） | `build-openwrt.yml:168`、`build-openwrt.yml:178`、`update-checker.yml:49` | 令牌仅 env 注入不落盘 |
| 出向通知 | Telegram（ssh 调试通道） | `build-openwrt.yml:89-90` | secret 注入 |
| 入向 SSH | tmate 会话（人工调试） | `build-openwrt.yml:86` | 条件触发（`build-openwrt.yml:87`） |
| 本地子进程 | `git`（采集器定位分支） | `_token_common.py:238-254` | argv 列表无 shell、超时 1s |
| 本地子进程 | `git rev-parse`（报表/路由） | `sop_context.py:90-92`、`token_report.py:65-78` | 超时 3s |
| 本地子进程 | Codex 采集器回填 | `token_report.py:406-409` | `sys.executable` + 超时 10s |
| 子进程纪律 | 采集器脚本 | `_token_common.py:243`（`NOCA` 注释：argv 列表、无 shell） | `subprocess.check_output` 直调 |

## 5. 共变耦合与修改清单（Agent Co-Change Playbook）

- 调整 runner 或构建根：`ubuntu-20.04`（`build-openwrt.yml:38`）与 `/workdir` 软链桥（`build-openwrt.yml:54-62`）是隐式耦合对——换 runner 镜像须复核 apt 源清理行（`build-openwrt.yml:48`）与 `git.io/depends-ubuntu-2004` 依赖清单名（`build-openwrt.yml:50`）。
- 改产物目录结构：`bin/targets/*/*` 通配进入（`build-openwrt.yml:127`）依赖 OpenWrt 输出布局（仓外黑盒）；只可改 `rm -rf packages` 与 `FIRMWARE=$PWD` 注入（`build-openwrt.yml:128-129`）。
- 改本地运行期目录：认领账本/运行态路径变更须同步 [配置手册](../contracts/configuration.md) 第 5 节环境变量表与 [token-ledger 模型](../business/domain-models/token-ledger.md) 第 3 节矩阵。
- 补齐部署方法：客户方填写 `.wally/config.json` deploy/it 段后，本页第 1 节的缺口描述与 [sop-task-artifacts 模型](../business/domain-models/sop-task-artifacts.md) 2.4 表的「本仓实值」列需同步更新。
