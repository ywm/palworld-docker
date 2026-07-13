# 2026-07-12 上游同步记录

## 背景

从 `KagurazakaNyaa/palworld-docker` upstream 拉取了 5 个新提交（截至 2026-07-11），涉及 Palworld v1.0 适配。

## Upstream 提交摘要

| Commit | Message | 影响 |
|--------|---------|------|
| `fd22303` | align to 1.0 changes | Dockerfile, entrypoint, README, compose — 核心重构 |
| `2fe2129` | update actions version | GitHub Actions 版本升级 |
| `458e9ef` | update gitignore | 新增 `/mods/` |
| `f5f8632` | refine envs | 进一步环境变量整理和注释 |
| `589377c` | disable test build block | docker-compose build 注释修正 |

## 本地提交

```
c1ed907 feat: sync upstream v1.0 changes while preserving local customizations
```

## 各文件改动与取舍

### 1. Dockerfile

**Upstream 改动：**
- 移除 SteamCMD 6 次重试机制，改为两行直接执行
- 新增大量 v1.0 环境变量：`LOG_FORMAT`、`CROSSPLAY_PLATFORMS`、`WORKER_THREADS`、基地建筑参数、备份开关等
- `ENABLE_MULTITHREAD` 默认从 `true` 改为 `false`（v1.0 官方建议关闭旧版多线程）
- 环境变量按功能分组并加注释

**本地取舍：**
- **保留** 本地的 SteamCMD 6 次重试机制（`8749851` 引入），upstream 移除重试是其决策，但我们认为重试对构建可靠性有价值
- **采用** upstream 的所有 v1.0 新环境变量和分组注释
- **采用** `ENABLE_MULTITHREAD=false` 默认值

### 2. docker-entrypoint.sh

**Upstream 改动：**
- 引入 5 个 helper 函数：`escape_sed_replacement`、`escape_ini_string`、`set_quoted_setting`、`set_scalar_setting`、`set_bool_setting`、`set_crossplay_platforms`
- 使用 bash 数组 `server_opts` 替代字符串拼接，避免引号和空格问题
- 新增 `main()` 函数包裹和 `BASH_SOURCE` 守卫
- 新增所有 v1.0 INI 配置项的写入逻辑
- 使用 `exec` 启动 PalServer（更好的信号传递）
- 移除 `EpicApp=PalServer`（v1.0 不再需要）

**本地取舍：**
- **保留** 本地的 SteamCMD 6 次重试逻辑（在 `main()` 函数内）
- **采用** upstream 的所有 helper 函数、bash 数组、`main()` 结构
- **采用** `exec` 启动方式
- **采用** 所有新增 INI 配置项处理

### 3. docker-compose.yml

**Upstream 改动：**
- `build:` 注释格式微调
- 新增所有 v1.0 环境变量（Crossplay、Performance、Backups 分组）
- `ENABLE_MULTITHREAD` 默认改为 `false`
- 新增注释说明安全注意事项（RCON/RESTAPI 不要暴露到公网）

**本地取舍：**
- **全部采用** upstream 改动，与 Dockerfile 保持一致

### 4. .github/workflows/build.yml

**Upstream 改动：**
- 移除 `workflow_dispatch` 触发器
- 移除 maximize-build-space、Restart Docker、Check Free Space 步骤
- 镜像名改为 `kagurazakanyaa/palworld`
- 移除自定义 tag/latest 逻辑

**本地取舍：**
- **全部保留本地版本**，不动此文件。原因：
  - `maximize-build-space` 是本地优化（`8749851`），解决构建空间不足问题
  - `workflow_dispatch` 是手动触发所需
  - 镜像名 `ywmdocker/palworld` 是 fork 的核心用途
  - tag/latest 逻辑是本地 CI 需求

### 5. .github/workflows/update.yml

**Upstream 改动：**
- cron 从 `0 4,8 * * *` 改为 `0 12 * * *`（每天一次 vs 每天两次）
- `actions/checkout` v6 → v7

**本地取舍：**
- **采用** checkout v7 升级
- **保留** 本地 cron 频率（`0 4,8 * * *`），每天两次检查更新更适合我们的使用场景

### 6. .gitignore

**Upstream 改动：**
- 新增 `/mods/`

**本地取舍：**
- **全部采用**

### 7. README.md

**Upstream 改动：**
- 完全重构环境变量文档，按功能分组（Container lifecycle / Startup / Identity / Admin / Crossplay / Performance / Backups）
- 引用 v1.0 官方文档链接
- 新增 RCON 废弃说明和安全警告
- 新增所有 v1.0 变量的文档

**本地取舍：**
- **全部采用** upstream 的文档，结构更清晰、信息更完整

## 未动的文件

- `build.yml` — 保留本地定制（见上）
- `update.sh` — 未涉及
- 其他脚本和配置 — 未涉及

## 注意事项

- `ENABLE_MULTITHREAD` 默认值从 `true` 变为 `false`，如果现有部署依赖多线程启动参数，需要显式设置 `ENABLE_MULTITHREAD=true`
- upstream 移除了 `EpicApp=PalServer`，v1.0 不再需要此参数
- 新增的 INI 配置项（基地建筑、备份等）仅在首次创建 `PalWorldSettings.ini` 时写入，已有配置需要删除 ini 文件重启容器才能生效
