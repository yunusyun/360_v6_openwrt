---
kind: downstream-contract
scope: coolsnowwolf-lede
status: draft
origin: generated
source_paths: [.github/workflows/build-openwrt.yml, .github/workflows/update-checker.yml]
symbols: [REPO_URL, REPO_BRANCH, git clone, git rev-parse, actions/checkout]
upstream: [integrations/index.md, integrations/topology-and-mapping.md]
downstream: []
---

# 下游出向接口契约：`coolsnowwolf/lede`（OpenWrt 源码仓）

lede 是构建流水线的唯一上游源码依赖：两个工作流各自以不同深度克隆同一 URL/分支（变量逐字一致），构建工作流取全量历史编译固件，巡检工作流取浅层历史读取 HEAD 哈希。本仓对它的全部调用就是两条 `git clone` 与一条 `git rev-parse`。

## 1. 封装定位与连接契约

| 契约项 | 精确值 / 规则 | 源码锚点（`路径:行号`） |
|---|---|---|
| 下游地址 | `https://github.com/coolsnowwolf/lede` | `.github/workflows/build-openwrt.yml:23`、`.github/workflows/update-checker.yml:15` |
| 分支 | `master` | `.github/workflows/build-openwrt.yml:24`、`.github/workflows/update-checker.yml:16` |
| 连接方式 A（构建） | `git clone $REPO_URL -b $REPO_BRANCH openwrt`（全量克隆到 runner 的 `/workdir/openwrt`，随后 `ln -sf` 挂到工作区） | `.github/workflows/build-openwrt.yml:61-62` |
| 连接方式 B（巡检） | `git clone --depth 1 $REPO_URL -b $REPO_BRANCH .`（浅克隆到巡检 job 工作目录） | `.github/workflows/update-checker.yml:32` |
| 超时 / 重试 | 无显式配置（两条 clone 均无 timeout/retry 守卫，失败即整个 job 失败） | 同上两处 |
| 认证 | 公开仓无凭据；GitHub Actions runner 自带网络出口 | — |

## 2. 本仓上游调用方反查表（Who Calls This Wrapper）

| 调用点 | 所属工作流 / 步骤 | 用途 | 调用点锚点（`路径:行号`） |
|---|---|---|---|
| `git clone ... openwrt` | Build OpenWrt / Clone source code | 取得可编译源码树 | `.github/workflows/build-openwrt.yml:61` |
| `git rev-parse HEAD` | Update Checker / Get Commit Hash | 读上游最新提交哈希作变更指纹 | `.github/workflows/update-checker.yml:33` |

## 3. 出向请求与返回消费契约

### 3.1 构建工作流出向请求（clone 参数字典）

| 请求参数键 | 值 | 来源与说明 | 组装行号 |
|---|---|---|---|
| 位置参数 `<url>` | `$REPO_URL` | 环境变量注入 | `.github/workflows/build-openwrt.yml:61` |
| `-b <branch>` | `$REPO_BRANCH` | 环境变量注入 | `.github/workflows/build-openwrt.yml:61` |
| 目标目录 `openwrt` | 固定字面量 | 相对 `/workdir`（步骤 `working-directory: /workdir`，`build-openwrt.yml:58`） | `.github/workflows/build-openwrt.yml:61` |

返回消费：克隆成功无显式返回值消费；后续步骤把 `feeds.conf.default`、`.config`、DIY 脚本全部投放到该目录（`build-openwrt.yml:66-83`）再编译。克隆产物目录经 `ln -sf /workdir/openwrt $GITHUB_WORKSPACE/openwrt`（`build-openwrt.yml:62`）形成软链桥，后续步骤以 `cd openwrt` 相对引用。

### 3.2 巡检工作流出向请求与返回解包

| 请求参数键 | 值 | 组装行号 |
|---|---|---|
| `--depth` | `1`（浅克隆） | `.github/workflows/update-checker.yml:32` |
| 位置参数 `<url>` / `-b <branch>` | `$REPO_URL` / `$REPO_BRANCH` | `.github/workflows/update-checker.yml:32` |

| 返回提取键 | 消费方式 | 解包行号 |
|---|---|---|
| `git rev-parse HEAD` stdout | 经 `::set-output name=commitHash::` 写入步骤输出 `steps.getHash.outputs.commitHash` | `.github/workflows/update-checker.yml:33` |
| `steps.getHash.outputs.commitHash` | 拼进缓存键 `HEAD-<hash>`（`update-checker.yml:40`）与 `.commitHash` 文件内容（`update-checker.yml:45`） | `.github/workflows/update-checker.yml:40`、`update-checker.yml:45` |

## 4. 下游错误码拦截与翻译

无本仓侧错误翻译层：`git clone` 非零退出直接判定 job 失败（Actions 默认 `fail-fast`），日志进入 Actions run 页；排障路径见 [排障 Runbook](../../operations/runbook-and-observability.md)。

## 5. 共变耦合与修改清单（Co-Change Playbook）

- **换源 / 换分支**：必须同时改两份工作流的 `REPO_URL`/`REPO_BRANCH` 四行（`build-openwrt.yml:23-24` + `update-checker.yml:15-16`）；只改其一会出现巡检与构建指向不同源。改动后同步本页与 [openwrt-build-pipeline](../../architecture/openwrt-build-pipeline.md)、[update-checker](../../architecture/update-checker.md) 两组件页。
- **克隆深度调整**：构建工作流不可改浅克隆（编译需要 tags/历史供 feeds 与版本号推导），巡检工作流保持 `--depth 1` 即可满足哈希读取。
- **上游不可用**：属仓外黑盒（lede 仓的可用性、分支保护、force-push 历史），本仓无重试机制；force-push 会使旧哈希缓存键永久未命中（派发一次新构建），属预期行为。
