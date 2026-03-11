# Docker Images Pusher（优化版）

> 🙏 **致谢**：本项目基于 **[技术爬爬虾](https://github.com/tech-shrimp/me)** 的 [docker_image_pusher](https://github.com/tech-shrimp/docker_image_pusher) 项目进行重写优化。原项目提供了非常优秀的思路和实现方案，感谢原作者的开源贡献！
>
> 原作者 B站、抖音、Youtube 全网同名 **技术爬爬虾**，原项目视频教程：https://www.bilibili.com/video/BV1Zn4y19743/

---

使用 Github Action 将国外的 Docker 镜像转存到阿里云私有仓库，供国内服务器使用，免费易用

- 支持 DockerHub, gcr.io, k8s.io, ghcr.io 等任意仓库
- 支持最大 40GB 的大型镜像
- 使用阿里云的官方线路，速度快

## 相较于原版的优化

本项目在原版基础上进行了以下重写与优化：

### 🧹 深度空间管理
- **三阶段流水线设计**：将工作流拆分为 `环境清理` → `最大化构建空间` → `处理镜像` 三个独立阶段，各阶段按依赖顺序执行，确保每一步都在最干净的环境下运行
- **深度清理 Runner 环境**：在任务启动前主动清理诊断日志（`_diag`）、工作目录、系统缓存（`.cache`、`.npm`、`.m2`）及临时文件，防止历史运行残留占用空间
- **最大化构建空间**：使用 [easimon/maximize-build-space](https://github.com/easimon/maximize-build-space) Action，移除 .NET、Haskell、Android、CodeQL、Python、Node、Java、Ruby、Go、PHP、Rust 等预装环境，并配置 4GB swap，最大限度释放磁盘空间
- **逐镜像即时清理**：每处理完一个镜像立即执行 `docker rmi` 删除源镜像和目标镜像，避免磁盘被大量镜像层占满
- **周期性深度清理**：每处理 3 个镜像自动执行 `docker system prune` 和 `docker builder prune`，回收构建缓存和悬空层
- **最终清理保障**：无论任务成功或失败，始终执行最终清理步骤（`if: always()`），确保不会残留任何数据影响后续运行

### 📊 更友好的进度提示
- **分隔线与计数器**：每个镜像处理时显示 `[当前/总数]` 格式的进度信息，清晰直观
- **Emoji 状态标识**：使用 ✅ ❌ 🧹 🚀 🐳 📥 等 Emoji 快速区分成功、失败和清理操作
- **最终统计报告**：处理结束后输出汇总信息（成功数、失败数、总计），一目了然
- **磁盘状态监控**：在清理前后分别输出磁盘使用情况（`df -h`），便于排查空间问题

### 🛡️ 稳定性增强
- **GitHub 空间超限问题解决**：原版在频繁运行后容易因 Runner 日志和缓存累积导致磁盘空间不足，优化版通过主动清理彻底解决此问题
- **失败隔离**：单个镜像拉取或推送失败不影响其他镜像的处理，最终仅在全部失败时标记工作流失败
- **健壮的错误处理**：所有清理命令均添加 `2>/dev/null || true` 容错，避免因清理操作本身报错而中断流程

## 使用方式

### 配置阿里云

登录阿里云容器镜像服务：https://cr.console.aliyun.com/

启用个人实例，创建一个命名空间（**ALIYUN_NAME_SPACE**）

![命名空间](/doc/命名空间.png)

访问凭证 → 获取环境变量：
- 用户名（**ALIYUN_REGISTRY_USER**）
- 密码（**ALIYUN_REGISTRY_PASSWORD**）
- 仓库地址（**ALIYUN_REGISTRY**）

![用户名密码](/doc/用户名密码.png)

### Fork 本项目

Fork 本项目到您自己的 GitHub 账号下。

#### 启用 Action

进入您自己的项目，点击 **Action**，启用 Github Action 功能。

#### 配置环境变量

进入 **Settings** → **Secrets and variables** → **Actions** → **New Repository secret**

![配置环境变量](doc/配置环境变量.png)

将以下 **四个值** 配置为仓库密钥（Repository secrets）：

| Secret 名称 | 说明 |
|---|---|
| `ALIYUN_REGISTRY` | 阿里云仓库地址，如 `registry.cn-hangzhou.aliyuncs.com` |
| `ALIYUN_NAME_SPACE` | 阿里云命名空间 |
| `ALIYUN_REGISTRY_USER` | 阿里云登录用户名 |
| `ALIYUN_REGISTRY_PASSWORD` | 阿里云登录密码 |

### 添加镜像

打开 `images.txt` 文件，添加你想要的镜像：

- 可以加 tag，也可以不加（默认 latest）
- 可添加 `--platform=xxxxx` 参数指定镜像架构
- 可使用 `k8s.gcr.io/kube-state-metrics/kube-state-metrics` 格式指定私库
- 可使用 `#` 开头作为注释

![images](doc/images.png)

文件提交后，自动进入 Github Action 构建。

### 使用镜像

回到阿里云，镜像仓库，点击任意镜像，可查看镜像状态（可以改成公开，拉取镜像免登录）。

![开始使用](doc/开始使用.png)

在国内服务器 pull 镜像，例如：

```bash
docker pull registry.cn-hangzhou.aliyuncs.com/shrimp-images/alpine
```

其中：
- `registry.cn-hangzhou.aliyuncs.com` 即 **ALIYUN_REGISTRY**（阿里云仓库地址）
- `shrimp-images` 即 **ALIYUN_NAME_SPACE**（阿里云命名空间）
- `alpine` 即阿里云中显示的镜像名

### 多架构

需要在 `images.txt` 中用 `--platform=xxxxx` 手动指定镜像架构。
指定后的架构会以前缀的形式放在镜像名字前面。

![多架构](doc/多架构.png)

### 镜像重名

程序自动判断是否存在名称相同但属于不同命名空间的情况。
如果存在，会把命名空间作为前缀加在镜像名称前。

例如：
```
xhofe/alist
xiaoyaliu/alist
```

![镜像重名](doc/镜像重名.png)

### 定时执行

本项目默认配置为 **每天北京时间凌晨 2:00 自动执行**（UTC 18:00）。

如需修改定时计划，编辑 `/.github/workflows/docker.yaml` 文件中的 `schedule` 部分：

```yaml
on:
  schedule:
    - cron: '0 18 * * *'  # UTC 时区，对应北京时间凌晨2点
```

> ⚠️ 注意：cron 表达式使用 **UTC 时区**，北京时间 = UTC + 8 小时。

同时支持手动触发（`workflow_dispatch`），可在 Actions 页面随时点击 **Run workflow** 手动执行。

![定时执行](doc/定时执行.png)

## 工作流程架构

```
┌─────────────┐     ┌──────────────────┐     ┌──────────────────┐
│  🧹 环境清理  │ ──▶ │ 📦 最大化构建空间   │ ──▶ │  🚀 处理Docker镜像 │
│   cleanup    │     │ maximize-space   │     │  process-images  │
└─────────────┘     └──────────────────┘     └──────────────────┘
  · 清理诊断日志       · 移除预装环境            · 登录阿里云
  · 清理工作目录       · 释放磁盘空间            · 逐个拉取/推送镜像
  · 清理系统缓存       · 配置 swap              · 逐镜像即时清理
  · 清理 Docker       · 保留最小根分区           · 周期性深度清理
  · 释放内存缓存                               · 最终统计与清理
```

## 许可证

本项目遵循原项目的开源许可协议。

## 致谢

再次感谢 **[技术爬爬虾](https://github.com/tech-shrimp/me)** 提供的原始项目和创意，本项目在其基础上针对 GitHub Actions Runner 的空间限制问题进行了深度优化，使其在高频率、大批量镜像同步场景下更加稳定可靠。
