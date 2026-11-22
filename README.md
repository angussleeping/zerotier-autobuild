# zerotier-autobuild

自建 ZeroTier Docker 镜像仓库：跟随 ZeroTier 官方 GitHub Release，自动构建并推送到 GHCR，供群晖 Container Manager 使用。

仓库地址：https://github.com/angussleeping/zerotier-autobuild

## 仓库结构

```text
.github/workflows/build.yml   # GitHub Actions：每周检查上游 Release，有新版本才构建
state/version.txt             # 已构建版本记录（由 Actions 自动回写）
README.md                     # 本说明
```

## 它是怎么工作的

1. 每周一 03:00 UTC（北京时间 11:00）自动运行一次，也可在 Actions 页面手动点 **Run workflow**。
2. 读取上游最新版本：
   `https://api.github.com/repos/zerotier/ZeroTierOne/releases/latest`
3. 用匿名 token 查询 GHCR 的 `tags/list`。如果该版本镜像已存在，直接跳过构建；不存在才继续。
4. 下载该 tag 的官方 `Dockerfile.release` 和 `entrypoint.sh.release`。
5. 用 `--build-arg VERSION=<版本号>` 构建镜像。构建在 GitHub 服务器上完成，不在群晖本地编译，不受国内访问 `download.zerotier.com` 速度影响。
6. 推送镜像到 GHCR，同时打两个 tag：
   - `ghcr.io/angussleeping/zerotier:latest`
   - `ghcr.io/angussleeping/zerotier:<版本号>`，例如 `ghcr.io/angussleeping/zerotier:1.16.2`
7. 把版本号回写到 `state/version.txt`，便于人工核对。

> 当前上游最新稳定版为 **1.16.2**（发布于 2026-05-28）。首次构建已完成，GHCR 中已有 `latest` 和 `1.16.2` 两个标签。

## 在群晖上使用

### 1. 首次拉取镜像

SSH 登录群晖后执行：

```bash
docker pull ghcr.io/angussleeping/zerotier:latest
```

如果 GHCR 在国内拉取慢或失败，可以把 workflow 中的 registry 改为 Docker Hub，并把 tags 改成 `你的DockerHub用户名/zerotier:latest`；你已在 Container Manager 配好 `1panel + daocloud` 镜像源，拉 Docker Hub 会更顺。

### 2. Container Manager 项目 YAML

在 Container Manager → 项目 → 新建/编辑项目，使用以下配置（注意缩进）：

```yaml
services:
  zerotier:
    image: ghcr.io/angussleeping/zerotier:latest
    container_name: zerotier
    restart: always
    network_mode: host
    devices:
      - /dev/net/tun
    cap_add:
      - NET_ADMIN
    environment:
      - TZ=Asia/Shanghai
      - ZEROTIER_JOIN_NETWORKS=zt7nnf2uv4
    volumes:
      - /volume1/docker/zerotier:/var/lib/zerotier-one
      - /etc/localtime:/etc/localtime:ro
```

说明：

- 保留 `/volume1/docker/zerotier` 卷，节点身份 `ADD677CB40` 不会变，不需要重新在 Central 授权。
- `ZEROTIER_JOIN_NETWORKS` 是官方 entrypoint 支持的参数；容器重建后会自动重新加入网络。
- 若 `listnetworks` 出现 `ACCESS_DENIED`，执行 `docker restart zerotier` 清除缓存。

### 3. 升级到新版本

GitHub Actions 推送新镜像后，群晖容器**不会自动更新**。请手动操作：

1. 收到更新邮件后，Container Manager → 项目 → 选择 `zerotier` → 操作 → **重新构建**。
2. 或 SSH：`docker pull ghcr.io/angussleeping/zerotier:latest` 后，在项目里重新构建。
3. 验证：
   ```bash
   docker exec -it zerotier zerotier-cli -v
   docker exec -it zerotier zerotier-cli listnetworks
   docker exec -it zerotier zerotier-cli peers
   ```

不建议用 Watchtower 无人值守自动更新关键网络容器——你之前刚经历过 Container Manager 崩溃，手动确认更稳妥。

## 邮件更新通知

WorkBuddy 中已配置每周一 12:00（北京时间，紧接镜像构建之后）的自动化任务，它会：

1. 查询上游最新 Release 版本；
2. 查询 GHCR 中 `angussleeping/zerotier` 的标签，确认新镜像是否已构建完成；
3. 与本地通知记录 `state/notified-version.txt` 比较；
4. 只有出现新版本时才发邮件到 **angus-sleeping@hotmail.com**，内容包括版本号、更新要点、群晖拉取命令和 Container Manager 重新构建步骤；
5. 发信后更新通知记录，避免重复提醒；若镜像尚未构建完成，则邮件说明构建中并给出 Actions 页面链接。

如果收不到邮件，请检查：

1. WorkBuddy 的 **Agent Mail** 是否已开通（当前尚未开通，需要先在「更多 → 我的邮箱」开通）；
2. 自动化任务是否处于 ACTIVE 状态；
3. 通知邮箱是否为 `angus-sleeping@hotmail.com`。

## 注意事项

- 该镜像基于 ZeroTier 官方 Debian 包构建，不是 Docker Official Image，但版本严格跟随官方 Release，构建文件也直接取自官方仓库对应 tag。
- 1.16 起核心/服务采用 MPL 许可证，默认构建不含 controller；作为客户端节点使用没有授权问题。
- 若你在同一容器里跑本地 controller 或 ztncui，则不能升到 1.16+，必须停在 1.14.2。
- 镜像压缩后约 80–90 MB，比 Alpine 社区镜像大，但更接近官方发行包。
