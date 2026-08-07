---
name: "sync-image-to-aliyun"
description: "通过触发 syncimages 仓库的 GitHub Actions，将外部镜像同步（搬运）到阿里云 ACR（registry.cn-hangzhou.aliyuncs.com）。当用户要求同步/复制/搬运镜像到阿里云仓库、把 docker.io/mcr 等镜像转到阿里云、或提到 syncimages 仓库时调用。"
---

# 同步镜像到阿里云 ACR

通过触发 GitHub 仓库 `tiramission/syncimages` 的 GitHub Actions workflow，把外部镜像同步到阿里云容器镜像服务（ACR）。

## 仓库与认证信息

- **仓库**：`tiramission/syncimages`（默认分支 `all`，Public）
- **目标 Registry**：`registry.cn-hangzhou.aliyuncs.com`
- **目标命名空间/镜像**：`jaign-mirror/default`（单镜像 workflow 固定写死）
- **ACR 用户名**：`jaign`
- **认证 Secret**：仓库已配置 `ALIYUN_ACR_PASSWORD`，无需额外处理
- **前置工具**：`gh` CLI 已登录账号 `tiramission`

## 三种同步模式

### 模式 1：全架构同步（sync-image.yml）

把源镜像的所有架构（multi-arch）原样同步到 `jaign-mirror/default:[des_tag]`。底层用 skopeo copy --multi-arch all，带重试（最多 10 次，超时 360 分钟）。

**适用**：需要保留镜像全部架构（amd64/arm64/ppc64le/s390x 等）。

```powershell
gh workflow run sync-image.yml `
  --repo tiramission/syncimages `
  --ref all `
  -f runs_on="ubuntu-latest" `
  -f src_image="<源镜像>" `
  -f des_tag="<目标tag>"
```

**示例**：把 nginx:1.27 同步为 `jaign-mirror/default:nginx-1.27`

```powershell
gh workflow run sync-image.yml `
  --repo tiramission/syncimages `
  --ref all `
  -f runs_on="ubuntu-latest" `
  -f src_image="docker.io/library/nginx:1.27" `
  -f des_tag="nginx-1.27"
```

### 模式 2：过滤架构同步（sync-image-filter.yml）

只同步 `linux/amd64` 和 `linux/arm64` 两个平台，用 regctl index create 重建 manifest list。目标同样是 `jaign-mirror/default:[des_tag]`。

**适用**：源镜像含多余架构（如 windows/amd64），只想要 Linux 两个主架构；或需要减小同步体积。

```powershell
gh workflow run "Sync Image (Filter: linux/amd64, linux/arm64).yml" `
  --repo tiramission/syncimages `
  --ref all `
  -f runs_on="ubuntu-latest" `
  -f src_image="<源镜像>" `
  -f des_tag="<目标tag>"
```

> 注意：该 workflow 文件名含空格和括号，`gh workflow run` 时用完整文件名（带 .yml），或先用 `gh workflow list --repo tiramission/syncimages` 确认显示名。

### 模式 3：批量同步 devcontainer 镜像（devcontainer-images.yml）

用矩阵策略批量把 `mcr.microsoft.com/devcontainers/{repo}` 的全部镜像同步到 `registry.cn-hangzhou.aliyuncs.com/mcr-devcontainers`。无输入参数，手动触发即跑全部矩阵（anaconda/base/cpp/dotnet/go/java/javascript-node/jekyll/miniconda/php/python/ruby/rust/typescript-node/universal）。

**适用**：一次性刷新全部 devcontainer 官方镜像。

```powershell
gh workflow run devcontainer-images.yml `
  --repo tiramission/syncimages `
  --ref all `
  -f manual="manual"
```

## 参数说明（模式 1 & 2）

| 参数 | 必填 | 说明 |
|------|------|------|
| `runs_on` | 是 | 运行环境：`ubuntu-latest`(默认) / `windows-latest` / `macos-latest` / `self-hosted` |
| `src_image` | 是 | 源镜像完整引用，建议带 registry 前缀，如 `docker.io/library/nginx:1.27`、`mcr.microsoft.com/dotnet/runtime:8.0` |
| `des_tag` | 是 | 目标 tag，最终镜像为 `registry.cn-hangzhou.aliyuncs.com/jaign-mirror/default:[des_tag]` |

## Tag 命名规则

当用户未指定 `des_tag` 时，按以下规则生成（与镜像真实内容相关）：
- 用「镜像名-版本」格式，如 `nginx-1.27`、`redis-7.4`、`dotnet-runtime-8.0`
- 去掉 registry 和 library 前缀，保留 `名字:版本` 的语义
- 若源镜像用 `latest`，tag 保留 `latest` 或用 `名字-latest`

## 触发后查看运行状态

workflow 触发后，等几秒再查询 run（异步执行，不会立即有 run id）：

```powershell
# 查看最近 5 次运行
gh run list --repo tiramission/syncimages --limit 5

# 实时查看某个 run 的日志（用上一步拿到的 run id）
gh run watch <run_id> --repo tiramission/syncimages

# 查看某个 run 的详细日志
gh run view <run_id> --repo tiramission/syncimages --log
```

## 验证同步结果

同步完成后，可用 skopeo 或 docker 检查目标镜像：

```powershell
# 检查镜像是否存在及其架构（需本机有 skopeo 或通过 docker manifest）
docker manifest inspect registry.cn-hangzhou.aliyuncs.com/jaign-mirror/default:<tag>
```

## 注意事项

1. **目标镜像名固定**：单镜像 workflow 把目标写死为 `jaign-mirror/default`，无法通过参数改成别的命名空间/镜像名。如需其他目标，需修改仓库 workflow 文件。
2. **认证依赖**：workflow 用仓库 Secret `ALIYUN_ACR_PASSWORD` 登录 ACR，本地无需登录即可触发。
3. **异步执行**：`gh workflow run` 只负责触发，同步是否成功要看 run 状态。重要同步务必 `gh run watch` 确认。
4. **全架构模式体积大**：模式 1 同步全部架构，大镜像可能跑很久（超时上限 6 小时）。
5. **网络**：GitHub Actions runner 拉源镜像 + 推阿里云，源镜像越接近国内越快；docker.io 的镜像偶尔会超时，靠重试机制兜底。
6. **本地 docker build/push 规则不同**：若用户要求本地构建后推送，遵循 `jaign-mirror/build` 命名规则（见会话约定）；本 skill 仅处理「同步外部已有镜像」场景，目标是 `jaign-mirror/default`。

## 决策流程

1. 用户给一个外部镜像要同步到阿里云：
   - 默认用 **模式 1（全架构）**
   - 若用户明确说“只要 amd64 和 arm64”或“去掉 windows 架构”→ 用 **模式 2**
2. 用户要刷新 devcontainer 镜像集 → 用 **模式 3**
3. 用户没指定 `des_tag` → 按上面的 Tag 命名规则自动生成，并告知用户
4. 触发后主动 `gh run list` 拿 run id，并询问是否要 `gh run watch` 跟踪
