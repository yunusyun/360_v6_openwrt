---
kind: mapping
scope: inbound-to-outbound-mapping
status: draft
origin: generated
source_paths: [.github/workflows/build-openwrt.yml, .github/workflows/update-checker.yml, .wally/scripts/create_task.py, .wally/scripts/sop_context.py, .claude/scripts/token_report.py, .claude/hooks/_token_common.py, .wally/config.json, .config, diy-part1.sh]
symbols: [ssh, event-type, commitHash, cache-hit, DEVICE_NAME, FILE_DATE, FIRMWARE, inputs, Branch, execution.json, request_id]
upstream: [contracts/api-reference.md, integrations/index.md]
downstream: [integrations/downstream/coolsnowwolf-lede.md, integrations/downstream/openwrt-passwall-feed.md, integrations/downstream/github-actions-platform.md, integrations/downstream/file-transfer.md, integrations/downstream/ssh2actions-debug.md]
---

# 上下游参数组装与字段映射矩阵

专为「上游加字段/开关，要透传到下游动作」的需求设计：本仓无运行时 DTO 转换层，字段变形发生在三处静态管线——工作流触发输入 → 步骤守卫与出向 `with:` 参数、CLI 参数 → 任务工件字段、IDE hook 载荷 → 台账记录字段。逐链路给出精确键名与变形规则。

## 1. 上游入口 → 下游扇出编排顺序表（Fan-Out Sequence）

| 上游入口 | 编排主体（`路径:行号`） | 按序调用的下游动作链（含条件分支） | 短路 / 聚合规则 |
|---|---|---|---|
| `workflow_dispatch(ssh)` | `jobs.build`（`build-openwrt.yml:36`） | 1. checkout → 2. apt 依赖（`build-openwrt.yml:44-55`） → 3. clone lede（`:61`） → 4. diy-part1+feeds（`:64-75`） → 5. `.config`+diy-part2（`:77-83`） → [if ssh] ssh2actions（`:86`） → 6. download（`:92-99`） → 7. compile（`:101-110`） → [守卫链] artifact/transfer/release（`:116-175`） → 8. 清理（`:177-190`） | compile 失败即分发全短路；`ssh` 只插队调试不改变链 |
| `repository_dispatch` | 同上 | 同上（事件载荷 `action` 含 `ssh` 时同开调试，`build-openwrt.yml:87`） | 同上 |
| Update Checker `workflow_dispatch` | `jobs.check`（`update-checker.yml:23`） | 1. 浅 clone（`:32`） → 2. cache 比对（`:37-40`） → [未命中] 3. 存哈希（`:43-45`） → 4. repository-dispatch 派发（`:47-52`） → 5. 清理（`:54-58`） | `cache-hit == 'true'` 时 3/4 短路（每哈希至多派发一次） |
| `create_task.py argv` | `create`（`create_task.py:69`） | 1. 输入校验（`:71-104`） → 2. 模板展开（`:105-113`） → 3. `execution_config` 快照（`:26-53`→`:114-115`） → 4. Deploy/IT 注记（`:117-146`） → 5. 原子落盘（`:148-159`） | 任一校验失败即中止（fail-closed，无部分写入） |
| `sop_context.py --checkpoint` | `checkpoint_issue`（`sop_context.py:260-290`） | 1. `route` 装配（`:87-173`） → 2. 证据检查（`:268-284`） → 3. [有缺口] `require_confirmation`（`:325-345`） | 无缺口直接放行；缺口未确认阻断（exit 3） |
| IDE Stop/工具事件 | 采集器 main（各 `*_token_collect.py`） | 1. stdin 载荷解析 → 2. `find_repo`/`git_branch`/`find_task_dir`（`_token_common.py:210-291`） → 3. 转录解析（各客户端 scan_*） → 4. `sync_task_store`（`_token_common.py:639-666`） | 黑名单分支/无任务/锁超时均安全退出 |
| Commit 前报表 | `token_report.py main`（`token_report.py:430-484`） | 1. 定位概览（`:91-106`） → 2. `refresh_codex`（`:402-414`） → 3. `rollup_turns`（`:195-230`） → 4. `patch_overview`（`:417-427`） | 台账缺失降级渲染不中断 |

## 2. 核心场景字段级映射矩阵（Field-Level Mapping）

### 场景 A：构建触发输入 → 运行期变量 → 出向参数

| 上游输入字段（类型） | 转换/默认值逻辑 | 目标下游与动作 | 下游参数精确键路径（类型） | 组装锚点（`路径:行号`） |
|---|---|---|---|---|
| `inputs.ssh`（`str`，默认 `'false'`） | 字符串布尔判等（非真布尔） | ssh2actions 步骤 | 步骤 `if` 表达式（`github.event.inputs.ssh`） | `build-openwrt.yml:15-20` → `build-openwrt.yml:87` |
| `repository_dispatch` 载荷 `action`（`str`） | `contains(action, 'ssh')` 子串判定 | 同上 | 同上 | `build-openwrt.yml:14` → `build-openwrt.yml:87` |
| env `REPO_URL`/`REPO_BRANCH` | 直传无转换 | git clone | clone 位置参数与 `-b`（`str`） | `build-openwrt.yml:23-24` → `build-openwrt.yml:61` |
| `.config`（文件） | `mv` 整文件替换 + `make defconfig` 补全 | OpenWrt buildroot | `openwrt/.config`（文件路径） | `build-openwrt.yml:26` → `build-openwrt.yml:80`、`build-openwrt.yml:96` |
| `diy-part1.sh`（文件） | `chmod +x` 后在 `cd openwrt` 内执行，feed 声明追加进 `feeds.conf.default` | OpenWrt feeds 系统 | `src-git passwall <url>` 声明行 | `build-openwrt.yml:67` → `diy-part1.sh:18` |
| `.config:57` 目标行 | `grep '^CONFIG_TARGET.*DEVICE.*=y'` + sed 提取机型段 | Artifact 命名 | `name: OpenWrt_firmware${{ env.DEVICE_NAME }}${{ env.FILE_DATE }}`（`str`） | `build-openwrt.yml:108-109` → `build-openwrt.yml:136` |
| runner 时钟（`TZ=Asia/Shanghai`） | `date +"%Y%m%d%H%M"` 格式化 | 同上 | `env.FILE_DATE`（`str`，前缀 `_`） | `build-openwrt.yml:34` → `build-openwrt.yml:110` |
| `FIRMWARE=$PWD`（`str`） | `cd openwrt/bin/targets/*/*` 后取绝对路径 + `rm -rf packages` | upload-artifact / transfer / release | `path: ${{ env.FIRMWARE }}`、`files: ${{ env.FIRMWARE }}/*` | `build-openwrt.yml:127-129` → `build-openwrt.yml:137`、`build-openwrt.yml:144`、`build-openwrt.yml:175` |
| lede HEAD 哈希（巡检链） | `git rev-parse HEAD` → `::set-output` | actions/cache 键 / 派发幂等 | `key: HEAD-<hash>`（`str`） | `update-checker.yml:33` → `update-checker.yml:40` |
| 巡检缓存判定（`str`） | `cache-hit != 'true'` 守卫 | repository-dispatch | `event-type: Source Code Update`（固定字面量） | `update-checker.yml:43/48` → `update-checker.yml:52` |

### 场景 B：CLI 参数 → 任务工件字段

| 上游输入字段（类型） | 转换逻辑 | 目标工件字段路径（类型） | 组装锚点（`路径:行号`） |
|---|---|---|---|
| `create_task.py name`（`str`） | 正则校验 + 日期段拆分 | 概览 `# <name>` 标题、`Meta.Created`（`str`） | `create_task.py:71-73` → `create_task.py:111-113` |
| `--branch`（`str`） | 直接替换模板占位 `` `feature/xxx` `` | 概览 `Meta.Branch`（`str`） | `create_task.py:112` |
| `.wally/config.json` 的 `deploy`/`it` 段（`dict`） | 整体 `json.dumps` 快照 + 状态注记推导（`SKIPPED/PENDING/BLOCKED`） | 任务 `execution.json` 全文 + `04/05` 文件「Execution status」段 | `create_task.py:114-146` |
| `--repo-root`（`str`，默认 `.`） | `Path.resolve()` + 越界校验 | 脚本内读取基准（不落盘） | `create_task.py:70`、`create_task.py:77-81` |

### 场景 C：IDE hook 载荷 → 台账记录字段

| 上游载荷字段（类型） | 转换逻辑 | 目标记录键路径（类型） | 组装锚点（`路径:行号`） |
|---|---|---|---|
| `transcript_path`（`str`） | 可信根校验 + 路径展开 | `requests[].transcript_path` | `claude_token_collect.py:172-179` → `claude_token_collect.py:157` |
| `session_id`（`str`） | 直传（缺省回退 transcript.stem） | `requests[].session_id` | `claude_token_collect.py:190` |
| 转录行 `message.usage`（`dict`） | `normalize_usage` 键名归一（`inputTokens`/`cachedWriteTokens` 等方言折叠） | `requests[].usage` 七键 | `_token_common.py:294-360` |
| 转录行 `requestId`/`message.id`/`uuid`（`str`） | 三级兜底链 | `requests[].request_id`（合并主键） | `claude_token_collect.py:132-137` |
| 用户消息文本（`str`） | `<user_query>` 提取 / system-reminder 剥离，截 500 字符 | `requests[].prompt` | `stop_token_collect.py:168-179`、`claude_token_collect.py:123-125` |
| Payload 计数器（Cursor，`dict`） | 逐元素最大值合并（回传即全量） | `requests[].usage` | `cursor_token_collect.py:11-34` → `_token_common.py:363-393` |
| `PreToolUse.tool_input`（JSON `str`） | `tool_kind` 分类 + `file_path` 提取 | 门禁 deny 决策（不落盘） | `sop_context.py:369-391` |

### 场景 D：Token 台账 → 概览报表字段（回填映射）

| 台账键路径（类型） | 聚合/判空逻辑 | 概览报表列（类型） | 回填锚点（`路径:行号`） |
|---|---|---|---|
| `requests[].usage.total_tokens` | 回合分组求和（Cursor 单代取值） | `Tokens` 列合计 | `token_report.py:195-230` → `token_report.py:304-323` |
| `requests[].client` + `agent_kind/agent_type` | 标签推导（`Codex / subagent` 等） | `Client` 列 | `token_report.py:154-167` |
| `requests[].started_at` | 毫秒/秒自适应 + 本地时区格式化 | `Time` 列 | `token_report.py:124-134` |
| `store.collection`（`dict`） | 健康标签渲染（`ok/partial/missing`） | 表尾 `>` 引注行 | `token_report.py:368-399` |

## 3. 通用拼装模式与共变修改清单（Co-Change Checklist）

- **构建开关拼装模式**：`UPLOAD_*` env 布尔字面量（`build-openwrt.yml:29-33`）→ 步骤 `if` 字符串判等 → 出向 action `with:`；新增开关必须三段齐全并登记 [配置手册](../contracts/configuration.md) 第 2 节。
- **CLI→工件快照模式**：`.wally/config.json` → `execution.json` 一次性复制（`create_task.py:114-115`）；改配置不回溯既有任务。
- **字段透传标准改动链（Agent Checklist）**：
  1. 工作流输入：`on.workflow_dispatch.inputs` 新字段（`build-openwrt.yml:15-20`）；
  2. 消费守卫/变量：目标步骤 `if` 或 `GITHUB_ENV` 注入（参照 `build-openwrt.yml:108-110` 模式）；
  3. 出向参数：下游 action `with:` 键（同步对应 [downstream 契约页](downstream/coolsnowwolf-lede.md) 各表）；
  4. CLI 路径：argparse 定义 + 校验 + 工件字段写入三处（参照 `create_task.py:69-115`）；
  5. 台账路径：记录结构 + 归一化 + 报表列三处（参照 [token-ledger 模型](../business/domain-models/token-ledger.md) 第 5 节）。
- **映射表维护**：本页第 2 节四场景任一链路变更后同步本表对应行，并交叉核对 [WF-01](../business/firmware-build.md) 第 2 节判定表与 [api-reference](../contracts/api-reference.md) 入参表。

## 4. 跨链路字段一致性核对清单（Consistency Checklist）

- **双工作流四行同值**：`REPO_URL`/`REPO_BRANCH` 在 `build-openwrt.yml:23-24` 与 `update-checker.yml:15-16` 必须逐字一致（唯一无共享机制的重复配置对）。
- **运行期三变量闭环**：`DEVICE_NAME`（`build-openwrt.yml:108-109`）→ `FILE_DATE`（`build-openwrt.yml:110`）→ `FIRMWARE`（`build-openwrt.yml:129`）三者只经 `GITHUB_ENV` 传递，产物命名消费点共三处（`build-openwrt.yml:120`、`build-openwrt.yml:136`、`build-openwrt.yml:161`）须同步改。
- **口令双端一致**：`CONFIRM SOP ([0-9a-f]{12})` 产出端（`sop_context.py:352-354`）与消费端（`sop_context.py:348-366`）在同一文件内闭合；说明文本在 `AGENTS.md:52`。
- **发现字段四处同形**：概览 `Branch` 字段正则消费点为 `sop_context.py:19`、`create_task.py:102`、`_token_common.py:280`、`token_report.py:100`。
- **标记对两端同在**：产出端模板 `00-overview.md:51-53` 与消费端 `token_report.py:31-32` 必须同时演进。
