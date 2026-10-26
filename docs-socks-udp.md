# Socks UDP Shared（socks 节点同端口 UDP）

## 背景

sing-box 原生的 socks5 入站，`UDP ASSOCIATE` 会为每个客户端单独监听一个
**随机高位端口**（写在 `SagerNet/sing` 库的握手实现里，无法通过配置修改）。
生产上若防火墙/安全组只放行了节点端口本身，UDP 就不通；放行全部端口又扩大暴露面。

Xray 的 socks 实现没有这个问题：它用**一个共享 UDP socket 绑定在节点监听端口上**，
按客户端源地址分发。本 fork 的 `myrime/sing-box` 在 `v1.14.0-alpha.2-xboard.2`
引入了同款模型，通过 `udp_shared` 开关启用。

## 启用

**默认已启用（`v1.0.0-udp.2` 起）**——升级或新装节点无需任何配置改动。
只作用于 socks 入站，其他协议（vless/trojan/shadowsocks 等）不受影响。

如需回退旧的随机端口行为，`config.yml` 显式设为 `false`：

```yaml
kernel:
  type: "singbox"
  socks_udp_shared: false   # 仅 socks 节点需要；省略该行 = 默认开启
```

生成的 sing-box 入站配置等效于：

```json
{
  "type": "socks",
  "listen": "::",
  "listen_port": 1080,
  "users": [...],
  "udp_shared": true
}
```

启用后防火墙只需放行 `节点端口/udp`（与 TCP 同端口），无需放行随机端口段。

## 工作原理

1. listener 同时在监听端口上绑定 TCP + UDP；
2. 客户端 `UDP ASSOCIATE` 时（自实现握手），服务端登记
   `客户端源 IP → 用户`，回复 `BND.PORT = 节点端口`，并按 RFC 1928
   保持该 TCP 连接直至客户端关闭；
3. 到达共享 UDP socket 的包按源 IP 查表认人，剥掉 socks5 UDP 头
   （RSV/FRAG/ATYP/DST），经 `udpnat2` 按源地址维护 NAT 会话并路由
   （审计日志格式与 TCP 一致：`[user] inbound packet connection to <dest>`）；
4. 回包向客户端源地址写回，并重新加上 socks5 UDP 响应头。

## 与默认（独占 socket）模式的差异

| 项目 | 旧行为（socks_udp_shared: false） | udp_shared（默认） |
|------|----------|-----------|
| UDP 监听端口 | 每客户端随机高位端口 | 节点端口（与 TCP 相同） |
| 防火墙要求 | 放行随机端口段 | 只放行节点端口 |
| UDP 认人精度 | 精确（每个 associate 独立认证连接） | 按 TCP associate 的源 IP 认人（NAT 后同 IP 多用户归最近一次 associate） |
| 用户删除后的 UDP | 随 socket 立即失效 | 登记表随 `UpdateUsers` 清空，活跃客户端重新 associate 后恢复 |
| FRAG 分片 | 同样不支持 | 同样不支持（丢弃） |

> 注意：`UpdateUsers` 热更新用户时共享 UDP listener 不重建，TCP 会话不受影响。

## 回滚

```bash
xbctl service stop
# 编辑 config.yml 将 socks_udp_shared 设为 false（或补上这一行）
xbctl service start
```

或降级二进制：`install.sh ... --version dev`。
