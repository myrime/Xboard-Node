# xboard-node (myrime fork)

Node backend for [Xboard](https://github.com/cedar2025/Xboard). Supports `sing-box` / `xray-core` dual kernels.

> **Disclaimer**: This project is for educational and learning purposes only.

---

## ⚠️ 本 Fork 说明

这是 [cedar2025/xboard-node](https://github.com/cedar2025/xboard-node) 的 fork，唯一目的是修复一个**上游 bug**（其余代码与上游一致）。

**问题**：面板下发配置变更（改黑白名单 / 路由规则）触发 `Reload()` 热重载时，sing-box 的 inbound 用 **NOP logger**（静默日志）重建，导致 `[uuid] inbound connection to` 这条**审计日志消失**。依赖该日志做「uuid → 用户」归属的下游 SOC 审计系统会因此采不到 uuid。

**修复**（`internal/service/service.go`）：`applyChanges()` 由热重载改为**完整重启**（先 `Stop()` 再 `Start()`，用真实 logger 重建 inbound），配置变更后 `[uuid] inbound connection to` 持续存在。

**本 fork 相对上游的差异**：

- 上述 `applyChanges()` 完整重启修复。
- `install.sh` 默认下载源指向本 fork（`myrime/xboard-node`），默认 release 版本 `dev`。
- CI 的 ghcr 镜像路径改为 `ghcr.io/myrime/xboard-node`。

**自动发布**：每次 push 到 `dev` 分支，CI 会自动编译并发布 `dev` tag 的 release（linux amd64 / arm64 二进制 + ghcr 镜像）。

---

## Features

- Protocols: V2Ray family, Trojan, Shadowsocks, Hysteria2, TUIC, AnyTLS
- Sync: WebSocket push + REST polling dual channel
- User controls: speed limit, device limit, alive-IP tracking, hot update
- Deploy modes: node mode, machine mode, standalone mode
- Multi-instance: single process binding multiple panels / nodes

## Install

### Installer (Linux systemd，推荐)

```bash
# Node mode
curl -fsSL https://raw.githubusercontent.com/myrime/Xboard-Node/dev/install.sh | \
  sudo bash -s -- --mode node --panel https://panel.example.com --token TOKEN --node-id 1

# Machine mode
curl -fsSL https://raw.githubusercontent.com/myrime/Xboard-Node/dev/install.sh | \
  sudo bash -s -- --mode machine --panel https://panel.example.com --token TOKEN --machine-id 1

# 从上游旧版升级到本 fork 修复版
curl -fsSL https://raw.githubusercontent.com/myrime/Xboard-Node/dev/install.sh | sudo bash -s -- upgrade
```

### Docker

```bash
docker run -d --restart=always --network=host \
  -e apiHost=https://panel.com -e apiKey=TOKEN -e nodeID=1 \
  ghcr.io/myrime/xboard-node:latest
```

### Docker Compose

> 本 fork 未同步上游的 `compose` 分支，如需 compose 部署请使用上游：
>
> ```bash
> git clone -b compose --depth 1 https://github.com/cedar2025/xboard-node.git
> ```

## xbctl

Run `xbctl` after installation for help. Common commands:

```bash
xbctl list                          # list all instances
xbctl status                        # running status
xbctl bind add-node --panel URL --token TOKEN --node-id 1
xbctl bind add-machine --panel URL --token TOKEN --machine-id 1
xbctl bind remove-node --panel URL --node-id 1
xbctl service restart
```

## Configuration

Legacy single-panel config is fully compatible. Appending bindings auto-migrates to `instances` format. See `config.yml.example`.

## Extensions

- Custom routes: [docs-custom-routes.md](docs-custom-routes.md)
- Custom outbounds: [docs-custom-outbounds.md](docs-custom-outbounds.md)
- DNS providers (ACME DNS-01): [docs-dns-providers.md](docs-dns-providers.md)

## License

MPL-2.0.
