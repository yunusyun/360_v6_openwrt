---
kind: downstream-contract
scope: openwrt-passwall-feed
status: draft
origin: generated
source_paths: [diy-part1.sh, .github/workflows/build-openwrt.yml]
symbols: [src-git passwall, FEEDS_CONF, feeds.conf.default, scripts/feeds]
upstream: [integrations/index.md, integrations/topology-and-mapping.md]
downstream: []
---

# 下游出向接口契约：`openwrt-passwall`（扩展 Feed 源）

passwall 是本仓在默认 feeds 之外唯一追加的第三方软件源：`diy-part1.sh` 把 `src-git` 声明行追加进 lede 的 `feeds.conf.default`，由 OpenWrt 的 `scripts/feeds update -a / install -a` 在后续步骤拉取安装。本仓不直接调用该下游的任何 API——契约只有一行 feed 声明与两个消费命令。

## 1. 封装定位与连接契约

| 契约项 | 精确值 / 规则 | 源码锚点（`路径:行号`） |
|---|---|---|
| 本仓业务封装文件 | `diy-part1.sh` | feed 声明的唯一组装点 |
| 出向声明字面量 | `src-git passwall https://github.com/xiaorouji/openwrt-passwall` | `diy-part1.sh:18` |
| 注入目标 | lede 源码树内 `feeds.conf.default`（`>>` 追加，不覆盖默认行） | `diy-part1.sh:18` |
| 执行时机 | feeds update 之前（步骤 `Load custom feeds` 内 `cd openwrt` 后执行） | `.github/workflows/build-openwrt.yml:64-69` |
| 官方 feeds 位覆盖机制 | `FEEDS_CONF=feeds.conf.default`：若本仓根存在同名文件则 `mv` 覆盖 lede 默认文件（当前仓内不存在该文件，`[ -e ]` 守卫恒假） | `.github/workflows/build-openwrt.yml:25`、`build-openwrt.yml:66` |
| 拉取与安装命令 | `./scripts/feeds update -a` → `./scripts/feeds install -a` | `.github/workflows/build-openwrt.yml:72`、`build-openwrt.yml:75` |

## 2. 本仓上游调用方反查表（Who Calls This Wrapper）

| 调用点 | 所属步骤 | 用途 | 调用点锚点（`路径:行号`） |
|---|---|---|---|
| `$GITHUB_WORKSPACE/$DIY_P1_SH` | Build OpenWrt / Load custom feeds | 注入 passwall feed 声明 | `.github/workflows/build-openwrt.yml:69` |
| `./scripts/feeds update -a` | Build OpenWrt / Update feeds | 按 feeds.conf.default 拉取全部 feed（含 passwall） | `.github/workflows/build-openwrt.yml:72` |
| `./scripts/feeds install -a` | Build OpenWrt / Install feeds | 把 feed 包符号链接进 package 目录 | `.github/workflows/build-openwrt.yml:75` |

## 3. 出向请求与返回消费契约

### 3.1 追加进 `feeds.conf.default` 的声明字典

| 声明字段 | 值 | 说明 | 组装行号 |
|---|---|---|---|
| 协议关键字 | `src-git` | OpenWrt feed 协议字面量（git 源） | `diy-part1.sh:18` |
| feed 名 | `passwall` | 后续 `feeds install` 与 `.config` 引用的包名前缀 | `diy-part1.sh:18` |
| 源地址 | `https://github.com/xiaorouji/openwrt-passwall` | 未钉分支/提交（默认分支浮动） | `diy-part1.sh:18` |

### 3.2 返回消费

`scripts/feeds update -a` 的拉取结果不进任何显式返回通道；安装是否成功只能从该步骤日志与最终编译产物包列表推断。`.config` 中的 passwall 相关 `CONFIG_PACKAGE_*` 选项（分布于 `.config` 包选择段）在 feeds install 之后由 `make defconfig`（`.github/workflows/build-openwrt.yml:96`）解析——feed 缺失时对应包会被静默丢弃而非报错。

### 3.3 相邻存量（未生效）声明

| 存量行 | 状态 | 锚点 |
|---|---|---|
| `# sed -i 's/^#\(.*helloworld\)/\1/' feeds.conf.default` | 注释态：取消注释 helloworld 默认 feed | `diy-part1.sh:14` |
| `#echo 'src-git helloworld https://github.com/fw876/helloworld' >>feeds.conf.default` | 注释态：追加 helloworld feed 的备选写法 | `diy-part1.sh:17` |

## 4. 下游错误码拦截与翻译

无拦截层：feed 拉取失败发生在 `scripts/feeds update -a`（`.github/workflows/build-openwrt.yml:72`），非零退出即 job 失败；「feed 存在但包未选中」不报错，仅在固件中缺包（隐性失败，排障见 [排障 Runbook](../../operations/runbook-and-observability.md) 症状 D）。

## 5. 共变耦合与修改清单（Co-Change Playbook）

- **追加新 feed**：按 `diy-part1.sh:18` 同形追加 `src-git <name> <url>`（可加 `^<branch/tag>` 钉版本）；改完同步 `.config` 的 `CONFIG_PACKAGE_*` 选择与本页第 3.1 节表。
- **改用官方 feeds 位覆盖**：在仓根新增 `feeds.conf.default` 文件即可让 `build-openwrt.yml:66` 的 `mv` 分支生效，此时默认 feeds 完全以本仓文件为准（与追加式二选一，避免双写冲突）。
- **删除 passwall**：删 `diy-part1.sh:18` 一行即可，但需检查 `.config` 是否仍有 passwall 包选中（`make defconfig` 会丢弃悬空选择）；相关联动见 [openwrt-build-pipeline](../../architecture/openwrt-build-pipeline.md) 第 3 节输入契约表。
- **钉版本纪律**：feed 源未钉引用意味着构建不可重现（上游任意提交都可能进入固件）；若需可重现构建，把声明改为 `src-git passwall <url>^<commit>` 并更新本页契约表。
