# 更新日志

记录自建 ZeroTier 镜像的版本变更。新版本由 GitHub Actions 在每周检查上游 Release 时发现、构建并自动追加条目。

字段说明：

- 版本号与发布日期来自上游 `zerotier/ZeroTierOne` 的 Release；
- 更新要点为上游发布说明原文；
- 镜像标签为同时推送到 GHCR 与 Docker Hub 的结果（Docker Hub 条目在凭据配置后才会写入）；
- 群晖升级步骤为手动操作提示（容器不会自动换镜像）。

---

## 1.16.2 — 2026-05-28

上游发布：https://github.com/zerotier/ZeroTierOne/releases/tag/1.16.2

更新要点：

- 修复 Windows ARM x64 安装包 `zttap300.inf` 的换行符问题。
- 中央控制器（CV1）内部更新，包含 schema 变更与多架构 Docker 构建流程。
- `ZT_MAX_NETWORK_SPECIALISTS` 从 256 提升到 512，网络配置字典容量相应扩大，以满足大型网络需求。
- 修复 Windows 下退出一个网络再加入另一个网络时，间歇性出现数分钟延迟的问题。
- 全代码库编译器警告清理，覆盖 GCC 14、Clang 18 与 Clang 21。

镜像标签：

- `ghcr.io/angussleeping/zerotier:latest`
- `ghcr.io/angussleeping/zerotier:1.16.2`

群晖升级：Container Manager → 项目 → zerotier → 操作 → 重新构建。群晖容器不会自动更新镜像，需手动执行。
