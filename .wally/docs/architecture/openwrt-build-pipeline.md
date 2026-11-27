---
kind: component
scope: openwrt-build-pipeline
status: draft
origin: generated
source_paths: [.github/workflows/build-openwrt.yml, diy-part1.sh, diy-part2.sh, .config]
symbols: [jobs.build, workflow_dispatch, repository_dispatch, DIY_P1_SH, DIY_P2_SH, CONFIG_FILE, DEVICE_NAME, FILE_DATE, FIRMWARE]
upstream: [INDEX.md, overview/callchain-panorama.md]
downstream: [integrations/downstream/coolsnowwolf-lede.md, integrations/downstream/openwrt-passwall-feed.md, integrations/downstream/github-actions-platform.md, integrations/downstream/file-transfer.md, integrations/downstream/ssh2actions-debug.md]
---

# `openwrt-build-pipeline`（CI 异步构建流水线组件）

本组件是仓库的核心「业务」：单 job GitHub Actions 工作流 `Build OpenWrt`，在 `ubuntu-20.04` 托管 runner 上克隆 lede 源码、装载本仓定制（feeds + `.config` + DIY 脚本）、编译 360 V6 固件并按开关分发产物。它是典型的无守护进程、事件驱动的一次性批处理 Worker：状态只存在于 GitHub Actions 运行期（`GITHUB_ENV`），进程结束即消失。

## 1. 后台任务类型全集（Job Registry）

| 任务标识（Kind） | 触发源（事件 / 入口） | 控制开关（环境变量） | 执行主体（`路径:行号`） | 最终状态/产物落点 |
|---|---|---|---|---|
| `build`（固件构建） | `workflow_dispatch`（`.github/workflows/build-openwrt.yml:15-20`） | `REPO_URL`、`REPO_BRANCH`、`CONFIG_FILE`、`DIY_P1_SH`、`DIY_P2_SH`、`UPLOAD_*`（`build-openwrt.yml:22-34`） | `jobs.build`（`build-openwrt.yml:37-190`） | Actions Artifact / Release / 外链 |
| `build`（联动构建） | `repository_dispatch`（`build-openwrt.yml:14`，事件来自 `update-checker.yml:47-52`） | 同上 | 同上 | 同上 |
| `ssh-debug`（可选调试会话） | `inputs.ssh == 'true'` 或事件载荷含 `ssh`（`build-openwrt.yml:87`） | `TELEGRAM_CHAT_ID` / `TELEGRAM_BOT_TOKEN` secrets | `P3TERX/ssh2actions@v1.0.0`（`build-openwrt.yml:86-90`） | tmate 临时会话（不产固件） |

## 2. 执行流与阶段控制调用树（Step-Level Call Tree）

```text
workflow "Build OpenWrt" -> .github/workflows/build-openwrt.yml:36 jobs.build (ubuntu-20.04)
  ├─ [1. checkout]      build-openwrt.yml:41-42  actions/checkout@main
  ├─ [2. init-env]      build-openwrt.yml:44-55  apt 清理 + 装 depends(git.io/depends-ubuntu-2004) + TZ + /workdir
  ├─ [3. clone-src]     build-openwrt.yml:57-62  git clone $REPO_URL -b $REPO_BRANCH openwrt; ln -sf -> $GITHUB_WORKSPACE/openwrt
  ├─ [4. diy-part1]     build-openwrt.yml:64-69  [ -e $FEEDS_CONF ] && mv -> openwrt/feeds.conf.default; 执行 $DIY_P1_SH
  ├─ [5. feeds]         build-openwrt.yml:71-75  ./scripts/feeds update -a; install -a
  ├─ [6. load-config]   build-openwrt.yml:77-83  mv $CONFIG_FILE openwrt/.config; 执行 $DIY_P2_SH
  ├─ [7. ssh-debug]     build-openwrt.yml:85-90  [if ssh 开启] ssh2actions (Telegram 通知)
  ├─ [8. download]      build-openwrt.yml:92-99  make defconfig; make download -j8; find dl -size -1024c 删残包
  ├─ [9. compile]       build-openwrt.yml:101-110 make -j$(nproc) || make -j1 || make -j1 V=s; 提取 DEVICE_NAME/FILE_DATE
  ├─ [10. organize]     build-openwrt.yml:123-130 [if UPLOAD_FIRMWARE] cd bin/targets/*/*; rm -rf packages; FIRMWARE=$PWD
  ├─ [11. upload]       build-openwrt.yml:116-137 [UPLOAD_BIN_DIR] artifact(bin); [UPLOAD_FIRMWARE] artifact(firmware)
  ├─ [12. share]        build-openwrt.yml:139-155 [UPLOAD_COWTRANSFER] transfer cow; [UPLOAD_WETRANSFER] transfer wet
  ├─ [13. release]      build-openwrt.yml:157-175 [UPLOAD_RELEASE] 生成 tag + release.txt + softprops/action-gh-release
  └─ [14. cleanup]      build-openwrt.yml:177-190 delete-workflow-runs(retain 1d/3 条) + delete-older-releases(keep 3)
```

关键失败短路：`make -j$(nproc) || make -j1 || make -j1 V=s`（`build-openwrt.yml:106`）三级降级——并行编译失败先单线程重试，再失败用 `V=s` 冗长日志定位；步骤 8–13 均带 `!cancelled()` 或前置 step 输出守卫（如 `steps.compile.outputs.status == 'success'`，`build-openwrt.yml:118`），编译失败时产物分发步骤整体跳过。

## 3. 构建输入契约（.config 与 DIY 注入点）

| 输入资产 | 精确内容 | 注入时机与锚点 | 消费方 |
|---|---|---|---|
| `.config` | 1046 条 `CONFIG_*`（共 7232 行）；目标三元组 `CONFIG_TARGET_ipq60xx=y`（`.config:41`）、`CONFIG_TARGET_ipq60xx_generic=y`（`.config:53`）、`CONFIG_TARGET_ipq60xx_generic_DEVICE_360_v6=y`（`.config:57`）；`CONFIG_TARGET_PROFILE="DEVICE_360_v6"`（`.config:63`）；架构包 `aarch64_cortex-a53`（`.config:64`） | 步骤 6 `mv .config openwrt/.config`（`build-openwrt.yml:80`），随后 `make defconfig` 补全（`build-openwrt.yml:96`） | OpenWrt buildroot（仓外黑盒） |
| `diy-part1.sh` | 追加 passwall feed：`echo 'src-git passwall https://github.com/xiaorouji/openwrt-passwall' >>feeds.conf.default`（`diy-part1.sh:18`）；helloworld 相关两行均为注释态存量（`diy-part1.sh:13-17`） | 步骤 4，feeds update 之前（`build-openwrt.yml:67-69`） | `./scripts/feeds update -a`（`build-openwrt.yml:72`） |
| `diy-part2.sh` | 仅含注释态默认 IP 修改 sed（`diy-part2.sh:14`），当前无实际生效行为 | 步骤 6，`.config` 载入后（`build-openwrt.yml:81-83`） | 后续 `make download/compile` |
| `FEEDS_CONF` | 环境变量指向 `feeds.conf.default`（`build-openwrt.yml:25`）；本仓根目录当前不存在该文件，步骤 4 的 `[ -e $FEEDS_CONF ]` 判空跳过（`build-openwrt.yml:66`） | 步骤 4 守卫 | OpenWrt feeds 系统 |

运行期状态变量（`GITHUB_ENV`）：`DEVICE_NAME=_<机型>`（`build-openwrt.yml:108-109`，由 `grep '^CONFIG_TARGET.*DEVICE.*=y'` 提取）、`FILE_DATE=_YYYYmmddHHMM`（`build-openwrt.yml:110`）、`FIRMWARE=<bin/targets/*/* 绝对路径>`（`build-openwrt.yml:129`）——三者仅拼进 Artifact 命名与上传路径（`build-openwrt.yml:120`、`build-openwrt.yml:136-137`），不落盘。

## 4. 共变耦合与修改清单（Agent Co-Change Playbook）

- **换目标机型 / 换源**：改 `.config` 目标段（`.config:41-64`）或 `REPO_URL`/`REPO_BRANCH`（`build-openwrt.yml:23-24`）→ 核对 [下游契约：coolsnowwolf-lede](../integrations/downstream/coolsnowwolf-lede.md) 与 update-checker 的同名变量（`.github/workflows/update-checker.yml:15-16`，两处必须同步）→ 跑一次 `workflow_dispatch` 冒烟并检查 `DEVICE_NAME` 是否为空（多机型会拼出下划线连接串）。
- **新增 feed**：只允许写在 `diy-part1.sh`（feeds update 前执行，`build-openwrt.yml:64-69`）；写进 `diy-part2.sh` 不会生效。改后同步 [下游契约：openwrt-passwall-feed](../integrations/downstream/openwrt-passwall-feed.md)。
- **新增产物分发通道**：按步骤 12 的模板复制 `curl -fsSL git.io/file-transfer | sh` + `./transfer <mode>` 模式（`build-openwrt.yml:143-144`），新增 `UPLOAD_<NAME>` 开关注入 env 块（`build-openwrt.yml:29-33`），并把链接写进 `release.txt` 拼装段（`build-openwrt.yml:163-164`）；同步更新 [配置手册](../contracts/configuration.md) 与 [file-transfer 契约](../integrations/downstream/file-transfer.md)。
- **上传行为默认值调整**：`UPLOAD_FIRMWARE: true` 是唯一默认开启的分发开关（`build-openwrt.yml:30`）；改开关后必须核对 [github-actions-platform 契约](../integrations/downstream/github-actions-platform.md) 中的删除保序策略（`build-openwrt.yml:177-190` 清理步骤不受 UPLOAD 开关守卫，始终执行）。
