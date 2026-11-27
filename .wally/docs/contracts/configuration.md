---
kind: config-dict
scope: configuration
status: draft
origin: generated
source_paths: [.github/workflows/build-openwrt.yml, .github/workflows/update-checker.yml, .wally/config.json, .wally/install-state.json, .claude/hooks/_token_common.py, .claude/scripts/token_report.py, .claude/hooks/claude_token_collect.py, .codebuddy/hooks/stop_token_collect.py, .codex/hooks/codex_token_collect.py, .cursor/hooks/cursor_token_collect.py, .agents/skills/wally-open-cli/SKILL.md]
symbols: [REPO_URL, REPO_BRANCH, UPLOAD_*, TZ, FEEDS_CONF, CONFIG_FILE, DIY_P1_SH, DIY_P2_SH, GITHUB_TOKEN, ACTIONS_TRIGGER_PAT, TELEGRAM_CHAT_ID, TELEGRAM_BOT_TOKEN, WALLY_TOKEN_CLAIMS, WALLY_CODEX_STATE_DIR, deploy, it, readiness, gaps]
upstream: [INDEX.md, overview/system-architecture.md]
downstream: [operations/deployment-and-runtime.md, architecture/token-collectors.md]
---

# 配置项、环境变量与动态开关全集

本仓三层配置：工作流 env 块（构建行为开关）、GitHub secrets（凭据）、进程环境变量（采集器/报表运行期）。另有 JSON 配置面 `.wally/config.json`（SOP 执行方法）与 `.wally/install-state.json`（安装账本，只读元数据）。覆盖优先级：单次 CLI 显式参数 > 进程环境变量 > 仓库文件配置（workflow env / JSON）> 代码默认值。

## 1. 配置覆盖优先级链

`token_report.py` 的仓库定位是优先级链的实例：`args.repo` > `CODEBUDDY_PROJECT_DIR` > `CLAUDE_PROJECT_DIR` > `CURSOR_PROJECT_DIR` > `os.getcwd()`（`.claude/scripts/token_report.py:437-443`）；Codex 状态目录 `WALLY_CODEX_STATE_DIR` > `~/.wally/wally-init/codex`（`.codex/hooks/codex_token_collect.py:42-43`）；认领账本 `WALLY_TOKEN_CLAIMS` > `~/.wally/wally-init/token_claims.json`（`.claude/hooks/_token_common.py:66-68`）。

## 2. 工作流环境变量全集（`.github/workflows/build-openwrt.yml:22-34`）

| 环境变量名 | 类型 | 代码默认值 | 必填 | 控制行为与边界影响 | 读取锚点（`路径:行号`） |
|---|---|---|---|---|---|
| `REPO_URL` | `str` | `https://github.com/coolsnowwolf/lede` | 是 | 源码仓地址（与 update-checker 同值约束） | `build-openwrt.yml:23`、`update-checker.yml:15` |
| `REPO_BRANCH` | `str` | `master` | 是 | 源码分支（同上） | `build-openwrt.yml:24`、`update-checker.yml:16` |
| `FEEDS_CONF` | `str` | `feeds.conf.default` | 否 | 存在同名文件时覆盖官方 feeds 位 | `build-openwrt.yml:25`、`build-openwrt.yml:66` |
| `CONFIG_FILE` | `str` | `.config` | 是 | 构建配置注入文件 | `build-openwrt.yml:26`、`build-openwrt.yml:80` |
| `DIY_P1_SH` | `str` | `diy-part1.sh` | 是 | feeds 前置定制脚本 | `build-openwrt.yml:27`、`build-openwrt.yml:67-69` |
| `DIY_P2_SH` | `str` | `diy-part2.sh` | 是 | 配置后定制脚本 | `build-openwrt.yml:28`、`build-openwrt.yml:81-83` |
| `UPLOAD_BIN_DIR` | `bool` 字面量 | `false` | 否 | 上传整个 bin 目录 artifact | `build-openwrt.yml:29`、`build-openwrt.yml:118` |
| `UPLOAD_FIRMWARE` | `bool` 字面量 | `true` | 否 | 上传固件目录 artifact（唯一默认开） | `build-openwrt.yml:30`、`build-openwrt.yml:125` |
| `UPLOAD_COWTRANSFER` | `bool` 字面量 | `false` | 否 | Cowtransfer 外链上传 | `build-openwrt.yml:31`、`build-openwrt.yml:141` |
| `UPLOAD_WETRANSFER` | `bool` 字面量 | `false` | 否 | WeTransfer 外链上传 | `build-openwrt.yml:32`、`build-openwrt.yml:150` |
| `UPLOAD_RELEASE` | `bool` 字面量 | `false` | 否 | 发布 GitHub Release（联动旧件清理） | `build-openwrt.yml:33`、`build-openwrt.yml:159` |
| `TZ` | `str` | `Asia/Shanghai` | 否 | runner 时区（影响 `FILE_DATE` 与 tag 名） | `build-openwrt.yml:34`、`build-openwrt.yml:53`、`build-openwrt.yml:110` |
| `DEBIAN_FRONTEND` | `str` | `noninteractive`（步骤级） | 否 | apt 非交互 | `build-openwrt.yml:46` |

## 3. GitHub Secrets 凭据表（值均在平台配置面，仓内只有引用点）

| Secret 名 | 用途 | 引用锚点（`路径:行号`） | 缺省行为 |
|---|---|---|---|
| `GITHUB_TOKEN` | Release 创建与旧 Release 删除 | `.github/workflows/build-openwrt.yml:171`、`build-openwrt.yml:190` | 平台自动注入（仅开 Release 时需要） |
| `ACTIONS_TRIGGER_PAT` | update-checker 派发 repository_dispatch | `.github/workflows/update-checker.yml:51` | 未配置/过期 → 派发步骤失败、job 失败 |
| `TELEGRAM_CHAT_ID` | ssh 调试会话通知接收者 | `.github/workflows/build-openwrt.yml:89` | Action 自行降级（不推送） |
| `TELEGRAM_BOT_TOKEN` | ssh 调试通知凭据 | `.github/workflows/build-openwrt.yml:90` | 同上 |

## 4. SOP 执行方法配置（`.wally/config.json`，schema 1）

| 配置域 | 键 | 类型 | 本仓实值 | 允许值与生效语义 | 校验锚点 |
|---|---|---|---|---|---|
| `deploy` | `command/guide/verify/rollback` | `str` | 全空 | 至少一项非空 + `readiness` 非 pending 才是「已配置」；全空 → 新任务 Deploy 自动 `SKIPPED` | `.wally/scripts/create_task.py:20-23` |
| `deploy` | `readiness` | `str` | `pending` | `pending` → `BLOCKED`（须补 gaps）；`ready` 须有方法 | `create_task.py:46-52` |
| `deploy` | `gaps` | `list[str]` | 3 条（无本地部署入口/无目标环境/无刷机回滚） | 待补信息清单，进任务注记 | `.wally/config.json:9-13`、`create_task.py:132-136` |
| `it` | `command/guide/success_criteria` | `str` | 全空 | 同 deploy 语义 | `.wally/config.json:15-24` |
| `it` | `readiness` / `gaps` | — | `pending` / 2 条（无 IT 入口/无成功判据） | 同上 | `.wally/config.json:18-24` |

## 5. 进程环境变量全集（100% 扫描 `getenv` / `os.environ`）

| 环境变量名 | 类型 | 默认值 | 必填 | 控制行为 | 读取锚点（`路径:行号`） |
|---|---|---|---|---|---|
| `WALLY_TOKEN_CLAIMS` | `path` | `~/.wally/wally-init/token_claims.json` | 否 | 认领账本路径覆盖 | `.claude/hooks/_token_common.py:67-68`、`_token_common.py:551-553` |
| `CLAUDE_CONFIG_DIR` | `path` | — | 否 | Claude 转录可信根追加 | `.claude/hooks/claude_token_collect.py:42-45` |
| `CODEBUDDY_EXTENSION_DATA` | `path` | — | 否 | CodeBuddy IDE 会话可信根追加 | `.codebuddy/hooks/stop_token_collect.py:79-82` |
| `CODEBUDDY_PROJECT_DIR` | `path` | — | 否 | 报表/采集仓库定位（多客户端探测链第一优先） | `.claude/scripts/token_report.py:439`、`stop_token_collect.py:377` |
| `CLAUDE_PROJECT_DIR` | `path` | — | 否 | 同上（第二优先；Claude hook 也用它定位脚本） | `token_report.py:440`、`.claude/settings.json:8` |
| `CURSOR_PROJECT_DIR` | `path` | — | 否 | 同上（第三优先；Cursor hook 命令内注入，亦参与 `.cursor/hooks/cursor_token_collect.py:128-142` 的多根候选序） | `token_report.py:441`、`.cursor/hooks.json:6` |
| `WALLY_CODEX_STATE_DIR` | `path` | `~/.wally/wally-init/codex` | 否 | Codex 采集运行态目录 | `.codex/hooks/codex_token_collect.py:42-43` |
| `CODEX_HOME` | `path` | `~/.codex` | 否 | Codex 会话/线程发现根 | `codex_token_collect.py:299` |
| `WALLY_API_BASE_URL` | `url` | — | 否 | wally-open-cli 服务地址（参数 > env > `~/.wally/config.json` > CLI 默认，技能约定） | `.agents/skills/wally-open-cli/SKILL.md:20` |

## 6. 采集器运行时限常量（代码级调参面）

| 常量 | 值 | 作用 | 定义锚点 |
|---|---|---|---|
| `HOOK_DEADLINE_SEC` | `2.5` | hook 进程自杀时限（宿主 3s 内） | `.claude/hooks/_token_common.py:81` |
| `STDIN_TIMEOUT_SEC` | `0.8` | stdin 非阻塞读取窗口 | `_token_common.py:82` |
| `GIT_TIMEOUT_SEC` | `1.0` | git 子进程超时 | `_token_common.py:83` |
| `LOCK_WAIT_SEC` | `0.4` | 台账/账本锁等待 | `_token_common.py:84` |
| `MAX_CLAIMS` | `5000` | 认领账本 LRU 上限 | `_token_common.py:69` |

## 7. 共变耦合与修改清单（Co-Change Playbook）

- 改 `UPLOAD_*` 默认值：同步 [WF-01](../business/firmware-build.md) 判定表、[github-actions-platform 契约](../integrations/downstream/github-actions-platform.md) 各守卫行、[file-transfer 契约](../integrations/downstream/file-transfer.md) 开关行。
- 改 `REPO_URL`/`REPO_BRANCH`：双工作流四行联动（第 2 节表），并同步 [coolsnowwolf-lede 契约](../integrations/downstream/coolsnowwolf-lede.md)。
- 改 `.wally/config.json` 执行方法：只影响此后新建任务（快照语义，`create_task.py:114-115`）；校验规则改动须同步 [sop-task-artifacts 模型](../business/domain-models/sop-task-artifacts.md) 2.4 字段表。
- 新增进程环境变量：必须同时登记本页第 5 节与采集器所属页（[token-collectors](../architecture/token-collectors.md)）；技能级变量（`WALLY_API_BASE_URL`）只依据技能文档字面，服务端为仓外黑盒（[认知边界](../operations/boundary.md)）。
- `install-state.json` 属只读安装账本：`language`/`clients`/`managed_files` 变更由 wally-init 升级流程负责（[高危雷区](../operations/pitfalls-and-adr.md) 第 2 节）。
