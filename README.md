# Docker 镜像同步工具

一个基于 GitHub Actions 的 Docker 镜像同步工具，自动将 Docker Hub 或其他镜像源上的镜像同步到阿里云容器镜像服务（ACR），并支持增量更新、多架构镜像、失效镜像记录等能力。

## 功能特性

- **自动同步**：在推送代码或手动触发时，自动拉取源镜像并推送到阿里云 ACR
- **增量更新**：通过 Digest 比对，仅同步有更新的镜像，节省带宽与构建时间
- **多架构支持**：支持 `--platform` 参数指定目标平台（如 `linux/arm64`），目标镜像名会自动加上平台前缀
- **重名镜像处理**：自动识别来自不同命名空间的同名镜像，避免互相覆盖
- **失效镜像记录**：拉取 / 打标签 / 推送失败的镜像会被记录到 `INVALID_IMAGES`，不影响整体流程
- **自动重试**：拉取与推送操作内置 3 次重试机制
- **版本记录**：通过 `version.txt` 记录每个镜像最新的 Digest，便于追踪变更
- **自动提交**：同步完成后自动将 `version.txt` 提交回仓库，提交信息会标注更新与失效的镜像

## 工作流程

1. **触发条件**
   - 推送到 `main` 分支（忽略 `version.txt` 与 `README.md` 的变更，避免循环触发）
   - 手动触发（`workflow_dispatch`）

2. **磁盘空间优化**
   - 使用 `easimon/maximize-build-space` 释放 GitHub Runner 磁盘空间，将 docker 数据目录挂载到最大化后的构建分区
   - 重启 docker 服务以确保配置生效

3. **镜像同步**（核心步骤）
   - 读取 `images.txt` 中的镜像列表
   - 第一遍扫描：识别来自不同命名空间的同名镜像，构建 `duplicate_images` 表
   - 第二遍扫描：对每个镜像
     - 调用 `docker manifest inspect` 获取源镜像 Digest
     - 与 `version.txt` 中的历史记录比对
     - 若一致则跳过；若不一致或无记录则执行 `pull → tag → push`
     - 任意步骤失败则记入 `INVALID_IMAGES` 并跳过该镜像
     - 成功则更新 `version_map`，并记入 `UPDATED_IMAGES`
   - 清理本地镜像以释放空间

4. **版本提交**
   - 仅当存在更新或 `version.txt` 不存在时，重写并提交 `version.txt`
   - 提交信息根据情况自动生成：
     - 仅有更新：`chore: update <images> [skip ci]`
     - 既有更新也有失效：`chore: update <images>; invalid <images> [skip ci]`
     - 仅版本变化：`chore: update image versions [skip ci]`
   - 失效镜像不会让整个 Workflow 失败（步骤末尾 `exit 0`）

## 前置准备

### 1. 阿里云容器镜像服务

在阿里云控制台开通容器镜像服务（ACR），并创建好命名空间。个人版实例即可使用。

### 2. GitHub Secrets

在仓库 **Settings → Secrets and variables → Actions** 中配置以下 Secrets：

| Secret 名称 | 说明 | 示例 |
| --- | --- | --- |
| `ALIYUN_REGISTRY` | 阿里云镜像仓库地址 | `registry.cn-hangzhou.aliyuncs.com` |
| `ALIYUN_NAME_SPACE` | 阿里云镜像仓库命名空间 | `mynamespace` |
| `ALIYUN_REGISTRY_USER` | 阿里云镜像仓库登录用户名 | `myuser` |
| `ALIYUN_REGISTRY_PASSWORD` | 阿里云镜像仓库登录密码（访问凭证） | `********` |

### 3. 仓库权限

Workflow 中已声明 `permissions: contents: write`，用于自动提交 `version.txt`，无需额外配置。

## 配置文件说明

### images.txt

每行一个镜像，支持注释、多架构指定，格式如下：

```text
# 简单镜像（自动补全为 docker.io/library/nginx:latest）
nginx:latest

# 指定命名空间的镜像
library/redis:7.0

# 完整路径的镜像
docker.io/library/python:3.11

# 指定平台（注意 --platform 参数与镜像名之间用空格分隔）
--platform linux/arm64 alpine:latest

# 注释行（以 # 开头）
# 这是一行注释，不会被处理
```

**行解析规则：**
- 以 `#` 开头的行视为注释，跳过
- 空行跳过
- 若行中包含 `--platform`，会提取出平台信息并从镜像名中剥离
- 镜像名中的 `@sha256:...` 后缀会被去除（用于处理带 Digest 的引用）

### version.txt

由 Workflow 自动维护，记录每个源镜像最新的 Digest，格式如下：

```text
nginx:latest=sha256:abc123...
library/redis:7.0=sha256:def456...
--platform linux/arm64 alpine:latest=sha256:ghi789...
```

> **注意**：请勿手动修改此文件，它会在每次同步完成且存在更新时被整体重写。

## 镜像命名规则

推送到阿里云的镜像名遵循以下规则：

```
<ALIYUN_REGISTRY>/<ALIYUN_NAME_SPACE>/[platform_prefix_][namespace_prefix_]<image_name:tag>
```

| 场景 | 源镜像 | 目标镜像名（不含仓库前缀） |
| --- | --- | --- |
| 普通镜像 | `nginx:latest` | `nginx:latest` |
| 多架构镜像 | `--platform linux/arm64 alpine:latest` | `linux_arm64_alpine:latest` |
| 重名镜像 | `library/nginx:latest`（与其它命名空间同名） | `library_nginx:latest` |
| 多架构 + 重名 | `--platform linux/arm64 library/nginx:latest` | `linux_arm64_library_nginx:latest` |

**重名判定逻辑**：当同一个 `image_name`（不含命名空间）在 `images.txt` 中出现多次且来自不同命名空间时，会自动添加命名空间前缀以避免覆盖。

## 失效镜像处理

当镜像出现以下情况时，会被标记为失效镜像并跳过：

- `docker pull` 失败（重试 3 次后仍失败）
- `docker tag` 失败
- `docker push` 失败（重试 3 次后仍失败）

失效镜像会：
1. 记录到 `INVALID_IMAGES` 环境变量
2. 在日志中汇总输出
3. 体现在 git 提交信息中，例如：`chore: update nginx:latest; invalid redis:7.0, mysql:8.0 [skip ci]`

> 失效镜像不会导致整个 Workflow 失败，便于下次运行时自动重试。

## 使用方法

1. **Fork 或克隆本仓库**

   ```bash
   git clone https://github.com/<your-username>/<your-repo>.git
   cd <your-repo>
   ```

2. **配置 GitHub Secrets**

   按上文表格在仓库 Settings 中添加 4 个 Secrets。

3. **编辑镜像列表**

   ```bash
   vim images.txt
   ```

4. **推送到 main 分支**

   ```bash
   git add images.txt
   git commit -m "chore: update images list"
   git push origin main
   ```

5. **手动触发（可选）**

   在仓库 **Actions** 页面选择 `Docker` workflow，点击 **Run workflow** 手动触发一次同步。

6. **查看结果**

   - 在 Actions 日志中查看同步详情（含失效镜像汇总）
   - 在阿里云 ACR 控制台查看推送结果
   - `version.txt` 会自动更新并提交回仓库

## 仓库结构

```text
.
├── .github
│   └── workflows
│       └── docker.yml        # GitHub Actions 工作流定义
├── images.txt                # 需要同步的镜像列表（用户维护）
├── version.txt               # 镜像 Digest 记录（自动维护）
└── README.md                 # 项目说明
```

## 关键实现细节

### Digest 获取

通过 `docker manifest inspect --verbose` 获取源镜像的 Manifest Descriptor，再使用 `jq` 按平台（os/architecture）筛选出对应的 Digest。若无法获取 Digest，会强制尝试同步一次，但不会更新 `version.txt`（避免误判为已同步）。

### 增量同步判定

- `source_digest == version_map[$map_key]` → 跳过同步
- 否则 → 执行同步，成功后更新 `version_map`
- 若 `source_digest` 为空，则同步但不写入记录

### 重试机制

```bash
retry_command <max_attempts> <delay_seconds> <command>
```

- 拉取镜像：3 次重试，间隔 5 秒
- 推送镜像：3 次重试，间隔 10 秒

### 磁盘空间优化

- `root-reserve-mb: 2048`：根分区保留 2GB
- `swap-size-mb: 128`：Swap 大小 128MB
- `remove-dotnet: true`：移除 .NET 运行时
- `remove-haskell: true`：移除 Haskell 工具链
- `build-mount-path: /var/lib/docker/`：将 docker 数据目录挂载到最大化后的构建分区

## 注意事项

- Workflow 设置了 `DOCKER_CLIENT_TIMEOUT=120` 和 `COMPOSE_HTTP_TIMEOUT=120`，避免大镜像拉取超时
- 提交信息包含 `[skip ci]`，避免 `version.txt` 的提交再次触发 Workflow
- `version.txt` 和 `README.md` 的变更已在 `paths-ignore` 中排除，不会触发同步
- 失效镜像不会导致 Workflow 失败，便于下次运行自动重试
- 阿里云 ACR 个人版实例对仓库数量有限制，请合理规划命名空间与镜像数量
- 推送到阿里云的镜像 Tag 与源镜像保持一致，可通过 `docker pull <ALIYUN_REGISTRY>/<ALIYUN_NAME_SPACE>/<image:tag>` 拉取使用

## 常见问题

### Q: 为什么某些镜像一直显示"无法获取源镜像指纹"？

A: 可能原因：
1. 源镜像已被删除或 Tag 失效
2. 源镜像仓库限流或网络异常
3. 镜像名格式不规范

此时 Workflow 会强制尝试同步一次，若拉取失败则标记为失效镜像。

### Q: 如何强制重新同步某个镜像？

A: 删除 `version.txt` 中对应行，或直接删除整个 `version.txt` 文件，下次运行时会重新同步所有镜像。

### Q: 如何添加新的镜像？

A: 直接在 `images.txt` 中新增一行镜像地址，提交到 `main` 分支即可。新镜像会自动被识别并同步。

### Q: 多架构镜像在阿里云上如何使用？

A: 推送后的镜像名带平台前缀，例如 `linux_arm64_nginx:latest`。使用时按平台拉取对应镜像即可。若需要统一入口，可在阿里云 ACR 中创建 Manifest List（暂未在本 Workflow 中实现）。

## License

MIT
