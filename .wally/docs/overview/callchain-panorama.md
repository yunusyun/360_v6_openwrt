---
kind: panorama
scope: callchain-panorama
status: draft
origin: generated
source_paths: [.github/workflows/build-openwrt.yml, .github/workflows/update-checker.yml, .wally/scripts/create_task.py, .wally/scripts/sop_context.py, .claude/hooks/_token_common.py, .claude/scripts/token_report.py, .claude/hooks/claude_token_collect.py, .codebuddy/hooks/stop_token_collect.py]
symbols: [jobs.build, jobs.check, create, route, sync_task_store, parse_transcript, scan_ide, rollup_turns, patch_overview, check_tool]
upstream: [INDEX.md]
downstream: [integrations/topology-and-mapping.md, contracts/error-codes.md, business/firmware-build.md, business/sop-task-lifecycle.md, business/token-report.md]
---

# 主干调用链全景

提炼全仓四条主干执行链（固件构建链、巡检派发链、SOP 门禁链、采集报表链），以符号级条件调用树与跨进程桥接表供快速追踪执行流。本仓无线程/消息队列：跨接全部发生在「事件字符串、stdin 管道、子进程、软链目录」四种静态可定位机制上。

## 1. 主干链路索引

| 链路 ID | 链路名称 | 模式 | 起点符号（`路径:行号`） | 终点落点（产物 / 状态） | 对应业务页 |
|---|---|---|---|---|---|
| `CC-01` | 固件构建分发链 | CI 异步批处理 | `jobs.build`（`.github/workflows/build-openwrt.yml:37`） | Actions Artifact / Release / 外链 | [`business/firmware-build.md`](../business/firmware-build.md) |
| `CC-02` | 巡检派发链 | CI 异步批处理 | `jobs.check`（`.github/workflows/update-checker.yml:24`） | repository_dispatch 事件 → CC-01 | [`business/source-update-trigger.md`](../business/source-update-trigger.md) |
| `CC-03` | SOP 路由门禁链 | 本地同步 CLI | `route`（`.wally/scripts/sop_context.py:87`） | 上下文文档包 / 确认票据 | [`business/sop-task-lifecycle.md`](../business/sop-task-lifecycle.md) |
| `CC-04` | 采集报表链 | hook 异步增量 + 同步报表 | `sync_task_store`（`.claude/hooks/_token_common.py:639`） | `token.json` → 概览报表区 | [`business/token-report.md`](../business/token-report.md) |

---

## 2. 符号级条件调用树（Hop-by-Hop Symbolic Traces）

### `CC-01`：固件构建分发链

```text
workflow 触发 -> build-openwrt.yml:36 jobs.build
  ├─ [checkout+init] build-openwrt.yml:41-55 actions/checkout; apt 装依赖(git.io/depends-ubuntu-2004)
  ├─ [guard: 无]     build-openwrt.yml:57-62 clone lede(master) -> /workdir/openwrt; ln -sf 软链桥
  ├─ [transform]     build-openwrt.yml:64-83 feeds.conf 守卫 -> diy-part1 追加 passwall -> feeds update/install
  │                   -> mv .config -> 执行 diy-part2
  ├─ [if ssh]        build-openwrt.yml:85-90 ssh2actions(tmate + Telegram secrets)
  ├─ [download]      build-openwrt.yml:92-99 make defconfig; download -j8; 删 <1KB 残包
  ├─ [compile]       build-openwrt.yml:101-110 make -j(nproc)||-j1||-j1 V=s
  │    └─ fail 三连 -> job 失败, 分发链路全部短路(守卫 status=='success')
  │    └─ success -> DEVICE_NAME/FILE_DATE 注入 GITHUB_ENV
  ├─ [organize]      build-openwrt.yml:123-130 [UPLOAD_FIRMWARE] FIRMWARE=$PWD; rm -rf packages
  └─ [distribute]    build-openwrt.yml:116-175 artifact(bin)/artifact(firmware)/transfer(cow/wet)/release
```

### `CC-02`：巡检派发链

```text
workflow_dispatch -> update-checker.yml:23 jobs.check
  ├─ [getHash]   update-checker.yml:29-33 clone --depth 1 lede; rev-parse HEAD -> commitHash 输出
  ├─ [compare]   update-checker.yml:35-41 actions/cache(key=HEAD-<hash>)
  │    └─ cache-hit=='true' -> 后两步短路(幂等终止)
  ├─ [save]      update-checker.yml:42-45 [未命中] tee .commitHash
  └─ [trigger]   update-checker.yml:47-52 [未命中] repository-dispatch(event-type="Source Code Update", PAT)
       └─ 接收端 build-openwrt.yml:14 -> CC-01
```

### `CC-03`：SOP 路由门禁链

```text
CLI/hook -> sop_context.py:394 main (建任务入口 create_task.py:69 create)
  ├─ [route]     sop_context.py:87-173 git branch -> 概览 Branch 匹配(唯一性)
  │    ├─ 零/多命中 -> ValueError 提示或消歧要求(链路终止)
  │    ├─ legacy -> compat/v1 快照装载(sop_context.py:116-129)
  │    └─ versioned -> 校验 Rules 路径 -> 装配 workflow/gates/interaction/profile/阶段文件
  ├─ [checkpoint] sop_context.py:260-290 implement: 01/02 完成度; deploy/it: execution.json 方法配置
  │    └─ 缺口 -> require_confirmation(sop_context.py:325-345) 写 pending 票据, exit 3
  │              -> 用户口令 "CONFIRM SOP <12hex>" -> accept_confirmation 转 approved(重跑放行)
  └─ [hook 拦截]  sop_context.py:369-391 PreToolUse 编辑类 + 任务外路径 -> deny 决策
```

### `CC-04`：采集报表链

```text
IDE Stop 事件 -> <client>_token_collect.py main
  ├─ [guard]    install_deadline(2.5s); 黑名单分支/无任务即退出(_token_common.py:223-291)
  ├─ [parse]    claude: parse_transcript(claude_token_collect.py:93-163)
  │             codebuddy: scan_ide/scan_cli(stop_token_collect.py:236-278/319-370)
  ├─ [merge]    _token_common.py:639-666 sync_task_store
  │    ├─ foreign_ids 剔除他人认领请求(防串账)
  │    └─ merge_store flock+request_id 去重+原子替换(防双计)
  └─ [render]   token_report.py:430-484 main(Commit 前, 人工触发)
       ├─ rollup_turns 回合聚合(token_report.py:195-230)
       └─ patch_overview 标记对内替换(token_report.py:417-427)
```

---

## 3. 跨进程 / 子进程 / 事件桥接表（Static-Jump Bridges）

静态搜索在事件字符串、stdin、子进程、软链处会断链；下表显式连接投递点与消费入口：

| 投递方符号（`路径:行号`） | 桥接机制 | 实际接收执行入口（`路径:行号`） | 载荷结构 |
|---|---|---|---|
| `peter-evans/repository-dispatch`（`update-checker.yml:49-52`） | GitHub `repository_dispatch` 事件（字面量 `Source Code Update`） | `on.repository_dispatch`（`build-openwrt.yml:14`） | 事件头 event-type；载荷 `github.event.action` |
| 各 settings hook 命令（`.claude/settings.json:8`、`.codebuddy/settings.json:8` 等） | stdin 管道 + 子进程 | `read_stdin_json`（`_token_common.py:153-193`） | hook 事件 JSON（`session_id`/`transcript_path`/`tool_input` 等） |
| `refresh_codex`（`.claude/scripts/token_report.py:406-409`） | `sys.executable` 子进程 | `codex_token_collect.py --refresh-task <dir>`（`.codex/hooks/codex_token_collect.py`） | argv + 10s 超时 |
| `ln -sf /workdir/openwrt`（`build-openwrt.yml:62`） | 文件系统软链 | 后续所有 `cd openwrt` 步骤（`build-openwrt.yml:68/72/75/82/95/104`） | 目录引用 |
| make 子进程（`build-openwrt.yml:97/106`） | shell 子进程链 + 管道退出码 | OpenWrt buildroot（仓外黑盒） | argv + 退出码 |
| hook 注册命令内联 Python（`.claude/settings.json:18`） | 运行期目录探测 | `.wally/scripts/sop_context.py`（`--repo` 参数接收） | 仓库根路径字符串 |
