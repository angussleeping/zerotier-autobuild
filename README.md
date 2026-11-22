# zerotier-autobuild

自建 ZeroTier Docker 镜像仓库：跟随 ZeroTier 官方 GitHub Release，自动构建并同时推送到 GHCR 与 Docker Hub，供群晖 Container Manager 使用，并在本仓库发布 Release 与更新日志。Docker Hub 短名 `angussleeping/zerotier:latest` 为群晖侧推荐拉取地址，国内拉取更稳。

仓库地址：https://github.com/angussleeping/zerotier-autobuild

## 仓库结构

```text
.github/workflows/build.yml   # GitHub Actions：每周检查上游 Release，有新版本才构建、发 Release、更新 CHANGELOG
synology-compose.yaml         # 群晖部署 YAML 模板（带注释、已脱密，Release 正文直接引用）
CHANGELOG.md                  # 更新日志（由 Actions 在新版本时自动追加）
state/version.txt             # 已构建版本记录（由 Actions 自动回写）
README.md                     # 本说明
```

## 它是怎么工作的

1. 每周一 03:00 UTC（北京时间 11:00）自动运行一次，也可在 Actions 页面手动点 **Run workflow** 立即检查。
2. 读取上游最新版本：
   `https://api.github.com/repos/zerotier/ZeroTierOne/releases/latest`
3. 同时检查 GHCR 与 Docker Hub 是否已收录该版本：两端都有才跳过；任一端缺失（例如 Docker Hub 是新接入的）就重新构建并双端推送，存量版本也能补齐。未配置 Docker Hub 凭据时自动降级为只处理 GHCR，工作流不报错。
4. 下载该 tag 的官方 `Dockerfile.release` 和 `entrypoint.sh.release`。
5. 用 `--build-arg VERSION=<版本号>` 构建镜像。构建在 GitHub 服务器上完成，不在群晖本地编译，不受国内访问 `download.zerotier.com` 速度影响。
6. 推送镜像到 GHCR 与 Docker Hub，各打两个 tag：
   - `ghcr.io/angussleeping/zerotier:latest` / `ghcr.io/angussleeping/zerotier:<版本号>`
   - `angussleeping/zerotier:latest` / `angussleeping/zerotier:<版本号>`（Docker Hub，群晖推荐）
7. 把版本号回写到 `state/version.txt`，并向 `CHANGELOG.md` 追加该版本条目。
8. 在本仓库发布 GitHub Release，正文含版本号、发布日期、更新要点、双 registry 镜像地址、群晖升级步骤，以及带注释、已脱密的部署 YAML。

Release 与 CHANGELOG 都按版本号去重：已发布过的版本不会重复发布，也不会重复追加日志。邮件提醒由 WorkBuddy 自动化任务负责，不在此 workflow 中。

> 当前上游最新稳定版为 **1.16.2**（发布于 2026-05-28）。GHCR 中已有 `latest` 和 `1.16.2` 两个标签；Docker Hub 双推需在下面「Docker Hub 前置配置」完成后触发一次构建。

## 在群晖上使用

### 0. Docker Hub 前置配置（一次性）

双推依赖两个 GitHub Actions Secrets，在仓库 **Settings → Secrets and variables → Actions → New repository secret** 添加：

| Secret 名 | 值 |
|---|---|
| `DOCKERHUB_USERNAME` | Docker Hub 用户名，例如 `angussleeping` |
| `DOCKERHUB_TOKEN` | Docker Hub Access Token（在 hub.docker.com → Account Settings → Security 生成，用途选 Read/Write） |

另需在 Docker Hub 网页上先手动创建一个名为 `zerotier` 的 **public** 仓库（namespace 与 `DOCKERHUB_USERNAME` 一致）。Docker Hub 不会自动建仓库，缺失时 workflow 会给出警告并只推 GHCR。

配置完成后，到 Actions 页面手动点 **Run workflow** 触发一次，即可把已有的 1.16.2 补推到 Docker Hub。未配置这两项 secrets 时，工作流照常只推 GHCR，不影响原有链路。

### 1. 首次拉取镜像

SSH 登录群晖后执行（Docker Hub 短名，国内更稳）：

```bash
docker pull angussleeping/zerotier:latest
```

GHCR 地址同样可用，作为备用：`docker pull ghcr.io/angussleeping/zerotier:latest`。

### 2. Container Manager 项目 YAML

完整模板见仓库根目录的 `synology-compose.yaml`（同样会嵌入每个 Release 正文）。该文件带逐行中文注释，且**已完成脱密**：网络 ID、节点 ID、身份数据目录等敏感值均以占位符表示，请替换为你自己的实际值。

```yaml
# 群晖 DS918+ / Container Manager 项目部署模板
# 已脱密：请将 <你的网络ID> 与 <你的身份数据目录> 替换为自己的实际值
services:
  zerotier:
    # 自建镜像，跟随上游版本；Docker Hub 短名，国内拉取更稳
    # 同一镜像也会同步推送到 ghcr.io/angussleeping/zerotier，可按需替换
    # 固定 latest，升级时在 Container Manager 手动重新构建（容器不会自动换镜像）
    image: angussleeping/zerotier:latest
    # 容器名，验证命令 docker exec -it zerotier ... 会用到
    container_name: zerotier
    # 开机自启；异常退出后自动重启
    restart: always
    # ZeroTier 需直连宿主机网络，必须用 host 模式
    network_mode: host
    # 映射 TUN 虚拟网卡，是建立隧道的必需设备
    devices:
      - /dev/net/tun
    # 授予网络管理能力，用于创建虚拟网卡与路由
    cap_add:
      - NET_ADMIN
    environment:
      # 时区
      - TZ=Asia/Shanghai
      # 要加入的 ZeroTier 网络 ID（已脱密，替换为自己的）
      - ZEROTIER_JOIN_NETWORKS=<你的网络ID>
    volumes:
      # 身份数据目录必须持久化：否则节点 ID 变化，需在 Central 重新授权
      # 冒号左侧替换为 NAS 上持久化存放身份数据的实际路径
      - <你的身份数据目录>:/var/lib/zerotier-one
      # 同步宿主机时区（可选）
      - /etc/localtime:/etc/localtime:ro
```

说明：

- `ZEROTIER_JOIN_NETWORKS` 是官方 entrypoint 支持的参数；容器重建后会自动重新加入网络。
- 身份数据目录务必持久化。保留该目录后节点身份不变，不需要在 Central 重新授权。
- 若 `listnetworks` 出现 `ACCESS_DENIED`，执行 `docker restart zerotier` 清除缓存。

### 3. 升级到新版本

GitHub Actions 推送新镜像后，群晖容器**不会自动更新**。收到更新通知或发现新 Release 后，手动操作：

1. Container Manager → 项目 → 选择 `zerotier` → 操作 → **重新构建**。
2. 或 SSH：`docker pull angussleeping/zerotier:latest` 后，在项目里重新构建。
3. 验证：
   ```bash
   docker exec -it zerotier zerotier-cli -v
   docker exec -it zerotier zerotier-cli listnetworks
   docker exec -it zerotier zerotier-cli peers
   ```

不建议用 Watchtower 无人值守自动更新关键网络容器——手动确认更稳妥。

## 变更追踪

- `CHANGELOG.md`：每次有新版本构建时自动追加条目，记录版本号、发布日期、更新要点与镜像标签。
- **Releases 页面**：https://github.com/angussleeping/zerotier-autobuild/releases ——每个版本一个 Release，正文含更新要点与脱密部署 YAML，便于随时查看。
- **Actions 页面**：https://github.com/angussleeping/zerotier-autobuild/actions ——可查看每周检查结果与手动触发。

## 邮件更新通知

提醒邮件由 **WorkBuddy 自动化任务**发送（每周一 12:00 北京时间，紧接镜像构建之后），收件邮箱固定为 `angus-sleeping@hotmail.com`。GitHub 仓库本身不配置任何 SMTP 凭据，也不负责发信。

WorkBuddy 任务的工作流程：

1. 查询上游最新 Release 版本；
2. 查询 GHCR 中 `angussleeping/zerotier` 的标签，确认新镜像是否已构建完成（群晖侧实际拉取走 Docker Hub，两处由 Actions 同步推送）；
3. 与本地通知记录 `state/notified-version.txt` 比较；
4. 只有出现新版本时才发邮件，正文含版本号、更新要点、手动升级步骤与验证命令；
5. 发信后更新通知记录，避免重复提醒。

如需核对，可在 WorkBuddy 的自动化任务列表中查看名为「ZeroTier 版本更新邮件提醒」的任务状态。

## 注意事项

- 该镜像基于 ZeroTier 官方 Debian 包构建，不是 Docker Official Image，但版本严格跟随官方 Release，构建文件也直接取自官方仓库对应 tag。
- 1.16 起核心/服务采用 MPL 许可证，默认构建不含 controller；作为客户端节点使用没有授权问题。
- 若你在同一容器里跑本地 controller 或 ztncui，则不能升到 1.16+，必须停在 1.14.2。
- 镜像压缩后约 80–90 MB，比 Alpine 社区镜像大，但更接近官方发行包。
