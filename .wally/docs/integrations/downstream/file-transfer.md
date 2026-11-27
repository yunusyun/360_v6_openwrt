---
kind: downstream-contract
scope: file-transfer
status: draft
origin: generated
source_paths: [.github/workflows/build-openwrt.yml]
symbols: [transfer cow, transfer wet, cowtransfer.log, wetransfer.log, UPLOAD_COWTRANSFER, UPLOAD_WETRANSFER, git.io/file-transfer]
upstream: [integrations/index.md, integrations/topology-and-mapping.md]
downstream: []
---

# 下游出向接口契约：`Cowtransfer / WeTransfer`（固件外链分发服务）

两条可选的产物分发通道：构建成功后用 Mikubill/transfer 工具（经 `git.io/file-transfer` 安装脚本拉取）把固件目录上传到 Cowtransfer 或 WeTransfer，把下载链接注入 GitHub Actions warning 注释、步骤输出与 `release.txt`。两通道默认关闭（`UPLOAD_COWTRANSFER: false`、`UPLOAD_WETRANSFER: false`，`.github/workflows/build-openwrt.yml:31-32`），远端服务与安装脚本均为仓外黑盒。

## 1. 封装定位与连接契约

| 契约项 | 精确值 / 规则 | 源码锚点（`路径:行号`） |
|---|---|---|
| 工具获取方式 | `curl -fsSL git.io/file-transfer \| sh`（两次出现，运行时下载可执行文件 `./transfer`） | `.github/workflows/build-openwrt.yml:143`、`build-openwrt.yml:152` |
| 上传对象 | `${FIRMWARE}`（`openwrt/bin/targets/*/*` 去 packages 后的固件目录，`build-openwrt.yml:127-129`） | `.github/workflows/build-openwrt.yml:144`、`build-openwrt.yml:153` |
| 执行守卫 | `steps.organize.outputs.status == 'success'` 且对应 `UPLOAD_* == 'true'` 且 `!cancelled()` | `.github/workflows/build-openwrt.yml:141`、`build-openwrt.yml:150` |
| 链接回写通道 A | `::warning file=<域名>::<url>`（Actions 注释流） | `.github/workflows/build-openwrt.yml:145`、`build-openwrt.yml:154` |
| 链接回写通道 B | `::set-output name=url::<url>`（供 release 步骤拼接 `release.txt`） | `.github/workflows/build-openwrt.yml:146`、`build-openwrt.yml:155` |
| 日志留痕 | `cowtransfer.log` / `wetransfer.log`（tee 落盘于 runner 工作目录） | `.github/workflows/build-openwrt.yml:144`、`build-openwrt.yml:153` |

## 2. 本仓上游调用方反查表（Who Calls This Wrapper）

| 调用步骤 | 消费其输出的位置 | 调用点锚点（`路径:行号`） |
|---|---|---|
| `Upload firmware to cowtransfer` | `Generate release tag` 的 `[ $UPLOAD_COWTRANSFER = true ] && echo "🔗 [Cowtransfer](${{ steps.cowtransfer.outputs.url }})" >> release.txt` | `.github/workflows/build-openwrt.yml:139-146`、消费于 `build-openwrt.yml:163` |
| `Upload firmware to WeTransfer` | `Generate release tag` 的 `[ $UPLOAD_WETRANSFER = true ] && echo "🔗 [WeTransfer](${{ steps.wetransfer.outputs.url }})" >> release.txt` | `.github/workflows/build-openwrt.yml:148-155`、消费于 `build-openwrt.yml:164` |

## 3. 出向请求与返回消费契约

### 3.1 Cowtransfer 请求参数字典

| 参数键 | 值 | 说明 | 组装行号 |
|---|---|---|---|
| 子命令 | `cow` | Cowtransfer 通道 | `.github/workflows/build-openwrt.yml:144` |
| `--block` | `2621440` | 分块大小（字节，约 2.5MB） | `.github/workflows/build-openwrt.yml:144` |
| `-s` | （开关） | 静默/单文件模式标志 | `.github/workflows/build-openwrt.yml:144` |
| `-p` | `64` | 并发分块数 | `.github/workflows/build-openwrt.yml:144` |
| `--no-progress` | （开关） | 关闭进度输出（日志体积控制） | `.github/workflows/build-openwrt.yml:144` |
| 位置参数 | `${FIRMWARE}` | 上传目标目录 | `.github/workflows/build-openwrt.yml:144` |

### 3.2 WeTransfer 请求参数字典

| 参数键 | 值 | 说明 | 组装行号 |
|---|---|---|---|
| 子命令 | `wet` | WeTransfer 通道 | `.github/workflows/build-openwrt.yml:153` |
| `-s` | （开关） | 单文件模式标志 | `.github/workflows/build-openwrt.yml:153` |
| `-p` | `16` | 并发数（低于 cowtransfer 的 64） | `.github/workflows/build-openwrt.yml:153` |
| `--no-progress` | （开关） | 关闭进度输出 | `.github/workflows/build-openwrt.yml:153` |
| 位置参数 | `${FIRMWARE}` | 上传目标目录 | `.github/workflows/build-openwrt.yml:153` |

### 3.3 返回解包规则

| 返回内容 | 提取方式 | 消费去向 | 解包行号 |
|---|---|---|---|
| stdout 中的下载 URL | `grep https` 过滤日志行 | 注入 `::warning` 注释与 `::set-output name=url` | `.github/workflows/build-openwrt.yml:145-146`、`build-openwrt.yml:154-155` |
| URL 文本列 | `cut -f3 -d" "`（按空格切第 3 列） | 步骤输出 `url` 的最终值 | `.github/workflows/build-openwrt.yml:146`、`build-openwrt.yml:155` |

## 4. 下游错误码拦截与翻译

无结构化错误拦截：上传失败（网络/配额/服务不可用）不阻断 job——步骤输出 `url` 为空时 `release.txt` 仍会写入空链接 Markdown 行（`[ $UPLOAD_COWTRANSFER = true ]` 只判开关、不判 url 非空，`.github/workflows/build-openwrt.yml:163`）；坏链排障入口见 [排障 Runbook](../../operations/runbook-and-observability.md) 症状 E。

## 5. 共变耦合与修改清单（Co-Change Playbook）

- **开启通道**：把 `UPLOAD_COWTRANSFER`/`UPLOAD_WETRANSFER` 置 `true`（`build-openwrt.yml:31-32`）即可，无需其他改动；若要进 Release 正文需同时开 `UPLOAD_RELEASE`（`build-openwrt.yml:33`），否则链接只存在于 Actions 注释流。
- **调上传参数**：`--block`/`-p` 直接影响远端接受率与 runner 内存（`.github/workflows/build-openwrt.yml:144`）；调整后同步本页 3.1/3.2 表与 [配置手册](../../contracts/configuration.md)。
- **换安装源**：`git.io` 短链服务已属历史资产（见 [高危雷区](../../operations/pitfalls-and-adr.md)）；替换安装地址时须同步 `build-openwrt.yml:143` 与 `build-openwrt.yml:152` 两处。
- **新增外链通道**：按第 1 节六要素复制步骤块（工具获取、transfer 调用、warning 注释、set-output、日志 tee、release.txt 拼接），并新增 `UPLOAD_<NAME>` 开关注入 env 块（`build-openwrt.yml:29-33`），联动 [openwrt-build-pipeline](../../architecture/openwrt-build-pipeline.md) 分发步骤树。
