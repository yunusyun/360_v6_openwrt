---
kind: downstream-contract
scope: github-actions-platform
status: draft
origin: generated
source_paths: [.github/workflows/build-openwrt.yml, .github/workflows/update-checker.yml]
symbols: [actions/checkout, actions/cache, actions/upload-artifact, softprops/action-gh-release, GitRML/delete-workflow-runs, dev-drprasad/delete-older-releases, peter-evans/repository-dispatch, GITHUB_TOKEN, ACTIONS_TRIGGER_PAT]
upstream: [integrations/index.md, integrations/topology-and-mapping.md]
downstream: []
---

# 下游出向接口契约：`GitHub Actions 平台服务`（runner / Artifact / Release / 缓存 / 派发 API）

GitHub 平台既是本仓工作流的宿主，也是被 `with:` 参数调用的下游服务集合。本页汇总六个 marketplace Action 的出向参数字典与两个令牌的使用边界；它们的远端实现（artifact 存储、release 服务、API 限流）均为仓外黑盒。

## 1. 封装定位与连接契约

| 契约项 | 精确值 / 规则 | 源码锚点（`路径:行号`） |
|---|---|---|
| 宿主形态 | `jobs.build`（`ubuntu-20.04`）与 `jobs.check`（`ubuntu-latest`）两个独立 job | `.github/workflows/build-openwrt.yml:38`、`.github/workflows/update-checker.yml:25` |
| 默认令牌 | `secrets.GITHUB_TOKEN`（Release 上传与旧 Release 删除使用） | `.github/workflows/build-openwrt.yml:171`、`build-openwrt.yml:189-190` |
| 巡检派发令牌 | `secrets.ACTIONS_TRIGGER_PAT`（repository_dispatch 需要 PAT） | `.github/workflows/update-checker.yml:51` |
| 时区约定 | `TZ: Asia/Shanghai`（仅构建工作流，runner 时间戳与产物 `FILE_DATE` 均此时区） | `.github/workflows/build-openwrt.yml:34`、`build-openwrt.yml:53` |

## 2. 出向 Action 调用参数字典（Outbound `with:` Schemas）

### 2.1 `actions/checkout@main`（`.github/workflows/build-openwrt.yml:41-42`）

| 参数键 | 值 | 说明 |
|---|---|---|
| （无 `with:`） | — | 默认 checkout 当前仓 `$GITHUB_WORKSPACE`，深度 1 |

### 2.2 `actions/cache@v2`（`.github/workflows/update-checker.yml:35-41`）

| 参数键 | 值 | 说明 | 组装行号 |
|---|---|---|---|
| `path` | `.commitHash` | 缓存承载文件（哈希内容物） | `.github/workflows/update-checker.yml:39` |
| `key` | `HEAD-${{ steps.getHash.outputs.commitHash }}` | 以上游新哈希为键实现「首次必未命中」幂等 | `.github/workflows/update-checker.yml:40` |

返回消费：`steps.cacheHash.outputs.cache-hit`（`'true'`/`'false'`）控制 Save 与 Trigger 两步的 `if` 守卫（`update-checker.yml:43`、`update-checker.yml:48`）。

### 2.3 `actions/upload-artifact@main`（两个上传步骤）

| 调用点 | `name` 值 | `path` 值 | 守卫条件 | 锚点 |
|---|---|---|---|---|
| bin 目录上传 | `OpenWrt_bin${{ env.DEVICE_NAME }}${{ env.FILE_DATE }}` | `openwrt/bin` | `UPLOAD_BIN_DIR == 'true'`（默认 false，恒不执行） | `.github/workflows/build-openwrt.yml:116-121` |
| 固件目录上传 | `OpenWrt_firmware${{ env.DEVICE_NAME }}${{ env.FILE_DATE }}` | `${{ env.FIRMWARE }}` | `steps.organize.outputs.status == 'success'`（依赖 `UPLOAD_FIRMWARE: true`） | `.github/workflows/build-openwrt.yml:132-137` |

### 2.4 `softprops/action-gh-release@v1`（`.github/workflows/build-openwrt.yml:167-175`）

| 参数键 | 值 | 说明 | 组装行号 |
|---|---|---|---|
| `tag_name` | `${{ steps.tag.outputs.release_tag }}`（格式 `%Y.%m.%d-%H%M`） | 动态生成 tag | `.github/workflows/build-openwrt.yml:173`（生成于 `build-openwrt.yml:161`） |
| `body_path` | `release.txt` | 下载链接清单（cowtransfer/wetransfer 行拼接，`build-openwrt.yml:162-164`） | `.github/workflows/build-openwrt.yml:174` |
| `files` | `${{ env.FIRMWARE }}/*` | 固件目录全部文件 | `.github/workflows/build-openwrt.yml:175` |
| env `GITHUB_TOKEN` | `secrets.GITHUB_TOKEN` | Release 写权限 | `.github/workflows/build-openwrt.yml:171` |

### 2.5 `GitRML/delete-workflow-runs@main`（两个工作流各一处）

| 调用点 | `retain_days` | `keep_minimum_runs` | 锚点 |
|---|---|---|---|
| 构建工作流清理 | `1` | `3` | `.github/workflows/build-openwrt.yml:177-181` |
| 巡检工作流清理 | `1` | `1` | `.github/workflows/update-checker.yml:54-58` |

注意：清理步骤无 `if` 守卫（始终执行，含构建失败场景——其前一步 `Remove old Releases` 才有 `UPLOAD_RELEASE` 守卫）。

### 2.6 `dev-drprasad/delete-older-releases@v0.1.0`（`.github/workflows/build-openwrt.yml:183-190`）

| 参数键 | 值 | 说明 |
|---|---|---|
| `keep_latest` | `3` | 保留最近 3 个 Release |
| `delete_tags` | `true` | 连同 tag 一并删除 |
| env `GITHUB_TOKEN` | `secrets.GITHUB_TOKEN` | `.github/workflows/build-openwrt.yml:189-190` |
| 守卫 | `UPLOAD_RELEASE == 'true' && !cancelled()`（默认 false，恒不执行） | `.github/workflows/build-openwrt.yml:185` |

### 2.7 `peter-evans/repository-dispatch@v1`（`.github/workflows/update-checker.yml:47-52`）

| 参数键 | 值 | 说明 |
|---|---|---|
| `token` | `secrets.ACTIONS_TRIGGER_PAT` | 派发凭据（`.github/workflows/update-checker.yml:51`） |
| `event-type` | `Source Code Update` | 接收端 `repository_dispatch`（`.github/workflows/build-openwrt.yml:14`） |

## 3. 下游返回消费与失败语义

| Action | 返回通道 | 本仓消费方式 |
|---|---|---|
| cache | `cache-hit` 输出 | `if` 守卫短路 Save/Trigger（`update-checker.yml:43`、`update-checker.yml:48`） |
| upload-artifact / gh-release / 清理类 | 无显式输出消费 | 失败即 job 失败；产物确认靠 Actions 页 Artifacts/Releases 面板 |
| repository-dispatch | 任务创建结果 | 无消费；触发失败只体现在巡检 run 日志 |

## 4. 共变耦合与修改清单（Co-Change Playbook）

- **令牌最小化**：`GITHUB_TOKEN` 仅出现在 Release 两处（`build-openwrt.yml:171`、`build-openwrt.yml:190`）；`ACTIONS_TRIGGER_PAT` 仅一处（`update-checker.yml:51`）。新增 GitHub API 调用应优先复用 `GITHUB_TOKEN` 并保持引用点登记在本页。
- **上传开关联动**：`UPLOAD_*` 五开关（`build-openwrt.yml:29-33`）与第 2 节各 `if` 守卫一一对应；改默认值时同步 [配置手册](../../contracts/configuration.md) 与 [openwrt-build-pipeline](../../architecture/openwrt-build-pipeline.md) 任务注册表。
- **保序参数**：`keep_minimum_runs`/`keep_latest`/`retain_days`（`build-openwrt.yml:180-181`、`build-openwrt.yml:187-188`、`update-checker.yml:56-57`）决定历史留存窗口，调整时须意识到仓库维护与排障所需的日志证据随删除一并消失（排障依赖见 [排障 Runbook](../../operations/runbook-and-observability.md)）。
- **Action 版本钉扎**：checkout/artifact 用浮动 `@main`，cache 用 `@v2`，gh-release 用 `@v1`，清理类用 `@main/@v0.1.0`（本页各表）；上游 Action 破坏性更新会直接改变本页 `with:` 契约的有效性，升级时逐表核对。
