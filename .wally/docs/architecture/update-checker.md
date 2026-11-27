---
kind: component
scope: update-checker
status: draft
origin: generated
source_paths: [.github/workflows/update-checker.yml]
symbols: [jobs.check, getHash, cacheHash, repository-dispatch, Source Code Update]
upstream: [INDEX.md, overview/callchain-panorama.md]
downstream: [integrations/downstream/coolsnowwolf-lede.md, integrations/downstream/github-actions-platform.md, architecture/openwrt-build-pipeline.md]
---

# `update-checker`（上游源码更新巡检 Worker 组件）

本组件是第二条 CI 工作流 `Update Checker`：浅克隆 lede 源码仓读取 HEAD 提交哈希，用 GitHub Actions 缓存判断「是否上次已经见过该哈希」，未见过的才消耗 `ACTIONS_TRIGGER_PAT` 令牌向本仓派发 `repository_dispatch` 构建事件。它是纯只读巡检 Worker，除 Actions 缓存 `.commitHash` 与运行日志外不产生任何仓内产物。

## 1. 后台任务类型全集（Job Registry）

| 任务标识（Kind） | 触发源 | 调度配置（代码实情） | 执行主体（`路径:行号`） | 最终状态/产物落点 |
|---|---|---|---|---|
| `check`（哈希巡检 + 派发） | `workflow_dispatch` 手动（`.github/workflows/update-checker.yml:19`） | `schedule` cron `0 */18 * * *` 处于注释态（`update-checker.yml:20-21`），当前无定时触发 | `jobs.check`（`update-checker.yml:24-58`） | Actions 缓存 `.commitHash` + 派发事件 |
| `check`（历史上定时态） | （已停用） | 注释保留的每 18 小时巡检计划 | 同上 | 同上 |

## 2. 执行流与哈希比对调用树

```text
workflow "Update Checker" -> .github/workflows/update-checker.yml:23 jobs.check (ubuntu-latest)
  ├─ [1. getHash]   update-checker.yml:29-33  git clone --depth 1 $REPO_URL -b $REPO_BRANCH .
  │                  └─ ::set-output name=commitHash::$(git rev-parse HEAD)   (update-checker.yml:33)
  ├─ [2. cacheHit]  update-checker.yml:35-41  actions/cache@v2: path=.commitHash, key=HEAD-<commitHash>
  │                  └─ cache-hit == 'true' => 该哈希已巡检过，后续步骤全部短路
  ├─ [3. save]      update-checker.yml:42-45  [if 未命中] echo <hash> | tee .commitHash（写入缓存路径）
  ├─ [4. trigger]   update-checker.yml:47-52  [if 未命中] peter-evans/repository-dispatch@v1
  │                  └─ token: ACTIONS_TRIGGER_PAT, event-type: "Source Code Update"  (update-checker.yml:51-52)
  └─ [5. cleanup]   update-checker.yml:54-58  GitRML/delete-workflow-runs@main (retain_days=1, keep_minimum_runs=1)
```

哈希缓存语义是本组件的核心不变量：缓存键 `HEAD-${{ steps.getHash.outputs.commitHash }}`（`update-checker.yml:40`）以「新哈希」为键——新哈希首次出现必然未命中，写入后同哈希的后续运行全部命中并跳过派发；这使派发动作天然幂等（每个上游提交至多触发一次构建）。该幂等不依赖任何仓内文件，缓存淘汰后同哈希可能重复派发一次构建，属可接受的重复构建成本。

## 3. 跨仓桥接契约（派发 → 构建）

| 桥接项 | 精确值 | 锚点（`路径:行号`） |
|---|---|---|
| 派发令牌 | secret `ACTIONS_TRIGGER_PAT`（必须是具有 `repository_dispatch` 权限的 PAT，不能用默认 `GITHUB_TOKEN` 之外随意令牌） | `.github/workflows/update-checker.yml:51` |
| 事件类型字面量 | `Source Code Update` | `.github/workflows/update-checker.yml:52` |
| 接收方 | `Build OpenWrt` 工作流的 `repository_dispatch` 触发器 | `.github/workflows/build-openwrt.yml:14` |
| 源变量同步 | `REPO_URL`/`REPO_BRANCH` 与构建工作流逐字相同（`https://github.com/coolsnowwolf/lede` / `master`） | `.github/workflows/update-checker.yml:15-16` vs `.github/workflows/build-openwrt.yml:23-24` |
| runner 差异 | 本工作流 `ubuntu-latest`，构建工作流 `ubuntu-20.04`（两仓内不一致，属既有事实） | `.github/workflows/update-checker.yml:25` vs `.github/workflows/build-openwrt.yml:38` |

## 4. 共变耦合与修改清单（Agent Co-Change Playbook）

- **恢复定时巡检**：取消注释 `schedule` 段（`update-checker.yml:20-21`）即可恢复每 18 小时巡检；无需改其他文件，派发幂等由缓存键保证。恢复后同步 [配置手册](../contracts/configuration.md) 与 [业务流程：source-update-trigger](../business/source-update-trigger.md)。
- **换巡检对象**：改 `REPO_URL`/`REPO_BRANCH`（`update-checker.yml:15-16`）时必须同步构建工作流同名变量（`build-openwrt.yml:23-24`），否则出现「巡检 A 仓、构建 B 源」的分裂；两个变量对共见 [下游契约：coolsnowwolf-lede](../integrations/downstream/coolsnowwolf-lede.md)。
- **改事件类型**：`event-type` 字面量（`update-checker.yml:52`）与接收端（`build-openwrt.yml:14` 的 `repository_dispatch` 无类型过滤）配合——接收端当前不过滤 action 类型，若未来加 `if: github.event.action == ...` 守卫，两处字面量必须同步改。
- **PAT 轮换**：`ACTIONS_TRIGGER_PAT` 只存在于 GitHub secret 配置面（代码仅引用，`update-checker.yml:51`）；轮换不涉及仓内改动，但派发失败排障入口见 [排障 Runbook](../operations/runbook-and-observability.md)。
