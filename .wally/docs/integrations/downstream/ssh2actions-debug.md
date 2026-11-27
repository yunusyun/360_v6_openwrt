---
kind: downstream-contract
scope: ssh2actions-debug
status: draft
origin: generated
source_paths: [.github/workflows/build-openwrt.yml]
symbols: [P3TERX/ssh2actions, TELEGRAM_CHAT_ID, TELEGRAM_BOT_TOKEN, inputs.ssh, github.event.action]
upstream: [integrations/index.md, integrations/topology-and-mapping.md]
downstream: []
---

# 下游出向接口契约：`ssh2actions / tmate`（构建期 SSH 调试通道）

`P3TERX/ssh2actions@v1.0.0` 是构建流水线内嵌的可选交互调试步骤：满足触发条件时在 runner 上开一个 tmate SSH 会话并（可选）经 Telegram Bot 推送连接信息，用于人工进入半成品环境排障。它是「上游输入 → 条件判定 → 外部服务」的最小完整闭环，远端 tmate 服务与 Telegram API 均为仓外黑盒。

## 1. 封装定位与连接契约

| 契约项 | 精确值 / 规则 | 源码锚点（`路径:行号`） |
|---|---|---|
| Action 引用 | `uses: P3TERX/ssh2actions@v1.0.0` | `.github/workflows/build-openwrt.yml:86` |
| 触发条件 A | `github.event.inputs.ssh == 'true' && github.event.inputs.ssh != 'false'`（手动触发时勾选 ssh 输入） | `.github/workflows/build-openwrt.yml:87` |
| 触发条件 B | `contains(github.event.action, 'ssh')`（repository_dispatch 事件载荷 action 含 `ssh` 字样） | `.github/workflows/build-openwrt.yml:87` |
| 执行位置 | `Load custom configuration` 之后、`Download package` 之前（源码树与 `.config` 已就位，可用于手工干预下载/编译） | `.github/workflows/build-openwrt.yml:85-90` |
| 通知凭据 | env `TELEGRAM_CHAT_ID: ${{ secrets.TELEGRAM_CHAT_ID }}`、`TELEGRAM_BOT_TOKEN: ${{ secrets.TELEGRAM_BOT_TOKEN }}` | `.github/workflows/build-openwrt.yml:88-90` |
| 步骤间桥接 | 步骤 id `compile` 之前；后续步骤不依赖本步骤输出 | `.github/workflows/build-openwrt.yml:92` |

## 2. 本仓上游调用方反查表（Who Calls This Wrapper）

| 调用入口 | 触发形态 | 调用点锚点（`路径:行号`） |
|---|---|---|
| Actions 页 Run workflow 勾选 `SSH connection to Actions` | `workflow_dispatch` 输入 `ssh`（default `'false'`） | `.github/workflows/build-openwrt.yml:15-20` |
| `repository_dispatch` 载荷带 `action` 含 `ssh` | 事件字段 `github.event.action` | `.github/workflows/build-openwrt.yml:14`、`build-openwrt.yml:87` |

## 3. 出向请求与返回消费契约

### 3.1 发往 Telegram 的通知字典（env 注入，由 Action 内部消费）

| env 键 | 值来源 | 用途 | 注入行号 |
|---|---|---|---|
| `TELEGRAM_CHAT_ID` | secret 同名 | 接收 tmate 连接信息的会话 ID | `.github/workflows/build-openwrt.yml:89` |
| `TELEGRAM_BOT_TOKEN` | secret 同名 | 推送凭据 | `.github/workflows/build-openwrt.yml:90` |

secret 未配置时 Action 自行降级（不在日志打印连接串到公开渠道；本仓代码不读取任何返回值——连接信息供人工使用）。

### 3.2 返回消费

无程序化消费：本步骤没有 `id`、没有 `::set-output`、没有 `GITHUB_ENV` 写入；`Download package`（`.github/workflows/build-openwrt.yml:92-99`）无条件随后执行（调试会话中断/退出后 workflow 继续）。

## 4. 下游错误码拦截与翻译

无拦截：条件不满足时整步跳过；Action 内部失败（tmate 服务不可达、Telegram 未配置）由 Action 自身决定是否阻断，本仓未加 `continue-on-error`（`.github/workflows/build-openwrt.yml:85-90` 无该字段），即该步骤失败会按默认语义令 job 失败。

## 5. 共变耦合与修改清单（Co-Change Playbook）

- **改触发字面量**：`'ssh'` 判定串出现在两处逻辑（输入比对与 action 包含判断，`.github/workflows/build-openwrt.yml:87`）；改输入名（`build-openwrt.yml:17-20`）时必须同步该行，否则调试入口静默失灵。
- **移动步骤位置**：调试会话的价值取决于所在阶段——当前位于 `.config` 装载后（`build-openwrt.yml:85`）：可手改配置再放行编译；移到编译后则只能检查产物。移动时同步 [openwrt-build-pipeline](../../architecture/openwrt-build-pipeline.md) 步骤树与本页「执行位置」行。
- **凭据管理**：两个 Telegram secret 只在 `build-openwrt.yml:88-90` 引用，值本身在 GitHub secret 配置面（[配置手册](../../contracts/configuration.md) 记录引用点与用途，不记录值）。
- **安全边界**：tmate 会话等于把 runner 交给操作者，仓库同步了 checkout 与源码但不含生产密钥；会话结束 runner 即回收。排障使用记录只存在于 Actions 日志（[排障 Runbook](../../operations/runbook-and-observability.md) 症状 B 的入口之一）。
