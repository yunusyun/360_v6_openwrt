---
kind: workflow
scope: WF-02-source-update-trigger
status: draft
origin: generated
source_paths: [.github/workflows/update-checker.yml, .github/workflows/build-openwrt.yml]
symbols: [Get Commit Hash, Compare Commit Hash, Save New Commit Hash, Trigger build, ACTIONS_TRIGGER_PAT]
upstream: [business/index.md, contracts/api-reference.md]
downstream: [integrations/downstream/coolsnowwolf-lede.md, integrations/downstream/github-actions-platform.md, business/firmware-build.md]
---

# `WF-02` · 上游源码更新检测与联动构建触发流程

巡检工作流手动触发（定时 cron 处于注释态）后浅克隆 lede 读取 HEAD 哈希，与 Actions 缓存中已见过的哈希集比对：新哈希首次出现时写缓存并通过 `repository_dispatch` 派发 `Source Code Update` 事件，触发 [WF-01 固件构建](firmware-build.md)。价值闭环：上游每个新提交至多自动触发一次构建。

## 1. 符号级端到端执行调用树

```text
Trigger: workflow_dispatch -> .github/workflows/update-checker.yml:23 jobs.check (ubuntu-latest)
  ├─ [Step 1 取哈希]  update-checker.yml:29-33  git clone --depth 1 lede master .
  │    └─ ::set-output name=commitHash::$(git rev-parse HEAD)
  ├─ [Step 2 比缓存]  update-checker.yml:35-41  actions/cache@v2(path=.commitHash, key=HEAD-<hash>)
  │    ├─ cache-hit == 'true'  => 哈希已巡检过, Step 3/4 短路
  │    └─ cache-hit != 'true'  => 新哈希, 继续
  ├─ [Step 3 存哈希]  update-checker.yml:42-45  [if 未命中] echo <hash> | tee .commitHash (写缓存路径)
  ├─ [Step 4 派发]    update-checker.yml:47-52  [if 未命中] peter-evans/repository-dispatch@v1
  │    └─ token=ACTIONS_TRIGGER_PAT, event-type="Source Code Update"
  │         └─ 接收端: .github/workflows/build-openwrt.yml:14 repository_dispatch -> WF-01
  └─ [Step 5 清理]    update-checker.yml:54-58  delete-workflow-runs(retain_days=1, keep_minimum_runs=1)
```

## 2. 分阶段业务规则与分支判定表

| 阶段 | 负责步骤（`路径:行号`） | 精确布尔条件 / 阈值 | 命中分支行为 | 状态或数据副作用 |
|---|---|---|---|---|
| 定时入口 | `on:`（`update-checker.yml:18-21`） | `workflow_dispatch` 生效；`schedule: cron 0 */18 * * *` 注释态 | 当前仅手动触发 | 无 |
| 哈希获取 | `Get Commit Hash`（`update-checker.yml:32-33`） | `git clone --depth 1` 必须成功 | 输出 `commitHash` 步骤变量 | 巡检 job 工作目录 |
| 命中判定 | `Compare Commit Hash`（`update-checker.yml:37-40`） | 缓存键 `HEAD-<hash>` 是否命中 | 命中 → 后续两步 `if: != 'true'` 全跳过 | Actions 缓存读写 |
| 派发守卫 | `Save New Commit Hash` / `Trigger build`（`update-checker.yml:43`、`update-checker.yml:48`） | `steps.cacheHash.outputs.cache-hit != 'true'` | 新哈希才写缓存 + 派发 | `.commitHash` 缓存条目 + 远端事件 |
| 派发凭据 | `Trigger build`（`update-checker.yml:51`） | secret `ACTIONS_TRIGGER_PAT` 有效 | 无效/过期 → 派发步骤失败、job 失败 | 无构建触发 |
| 日志留存 | `Delete workflow runs`（`update-checker.yml:54-58`） | 无守卫（始终执行） | 仅保留 1 条巡检 run | 巡检证据窗口极窄 |

## 3. 幂等与补偿语义（本流程无状态机）

- **派发幂等**：以「新哈希为缓存键」（`update-checker.yml:40`）保证同哈希只派发一次；缓存被平台淘汰后可能重复派发一次（接受重复构建成本，产物靠 `FILE_DATE` 区分，`build-openwrt.yml:110`）。
- **无仓内状态**：`.commitHash` 只活在 Actions 缓存域（`update-checker.yml:39`），仓内零写入；流程失败自愈方式是手动重跑巡检工作流。
- **跨工作流弱一致**：`REPO_URL`/`REPO_BRANCH` 在两份工作流逐字重复（`update-checker.yml:15-16` vs `build-openwrt.yml:23-24`），靠人工纪律保持同步（共变清单第 1 条）。

## 4. 边界 / 故障场景与处置表

| 场景 | 检测条件（`路径:行号`） | 处置动作 |
|---|---|---|
| lede 仓 force-push 改写历史 | 旧哈希缓存键永不复命中（`update-checker.yml:40`） | 每次巡检都视为新哈希 → 重复派发构建（预期行为，无告警） |
| PAT 失效 | 派发步骤 HTTP 失败（`update-checker.yml:49-52`） | job 失败并留 run 日志；修复 = 在 GitHub secret 面轮换 PAT 后手动重跑 |
| 接收端工作流禁用 | `repository_dispatch` 无接收（`build-openwrt.yml:14`） | 派发成功但无构建发生；检查 Actions 页工作流开关 |
| 缓存服务异常 | `actions/cache@v2`（`update-checker.yml:37`） | 未命中分支多派发一次构建；命中分支漏派发（下次巡检补偿） |
| 巡检 run 已被清理 | `keep_minimum_runs=1`（`update-checker.yml:57`） | 排障需及时取证，历史巡检日志仅存 1 条 |

## 5. 共变耦合与修改清单（Agent Co-Change Playbook）

1. 恢复/调整定时巡检：改 `update-checker.yml:20-21` 的 cron 注释；无需其他联动（幂等由缓存键承担）。
2. 换巡检源：`REPO_URL`/`REPO_BRANCH`（`update-checker.yml:15-16`）必须与 `build-openwrt.yml:23-24` 同步修改，同步核对 [coolsnowwolf-lede 契约](../integrations/downstream/coolsnowwolf-lede.md) 第 1 节。
3. 改事件类型字面量：`event-type: Source Code Update`（`update-checker.yml:52`）与接收端触发器（`build-openwrt.yml:14`）成对；接收端当前不过滤 action 值，新增过滤时两处同改。
4. 令牌轮换：`ACTIONS_TRIGGER_PAT` 引用点仅 `update-checker.yml:51`；值在 GitHub secret 面（引用登记见 [配置手册](../contracts/configuration.md)）。
5. 架构级行为差异（runner 版本、清理参数）见 [update-checker 组件页](../architecture/update-checker.md) 与 [github-actions-platform 契约](../integrations/downstream/github-actions-platform.md) 第 2.5/2.7 节。
