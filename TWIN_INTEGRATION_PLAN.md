# twin 协议集成进 FlClash — 实施计划

## 背景

twin 是一个基于 Hysteria 2 (apernet/quic-go) 的闭源加密翻墙协议，具有以下独特优化：
1. **自定义 BrutalSender** — 拥塞事件不缩窗，动态调整 ACK 频率
2. **自定义 Pacer (高 Burst)** — 一次 burst 多发包，而非标准令牌桶 10 包限制
3. **双通道 UDP (SideChannel)** — UDP 流量拆分到独立 QUIC 流减少 head-of-line blocking
4. **最大化 QUIC Flow Control 窗口** — 8-16 倍于默认值，确保高丢包链路油不受限
5. **禁用 PMTU 发现** — 避免 ICMP 阻塞导致连接中断
6. **无端口跳跃（Port Hopping）** — 单端口固定连接，减少 QoS 特征

## 仓库结构

```
nowhere/
├── FlClash/                        ← 主仓库 (forked)
│   ├── core/
│   │   ├── Clash.Meta/               ← 子模块 condercx/Clash.Meta.git (FlClash 分支)
│   │   │   └── adapter/outbound/twin.go  ← Twin 出站适配器
│   │   └── twin/                      ← 子模块 condercx/twin-go.git
│   │       ├── config.go              默认/最大窗口配置
│   │       ├── quic.go               QUIC Config 工厂
│   │       ├── auth.go               密码认证 + 带宽协商
│   │       ├── client.go             客户端拨号 + BrutalSender + OpenStream 超时保护
│   │       ├── server.go             服务端监听
│   │       ├── portal_session.go     服务端连接管理 + TCP/UDP 带超时代理
│   │       ├── protocol.go           UDP 数据报序列化 + 分片重组
│   │       ├── sidechannel.go        双通道选择策略
│   │       ├── sideframe.go          SideChannel 帧编解码
│   │       ├── pacer.go              Brutal Pacer (高 burst, 30 packets)
│   │       ├── brutal.go             Brutal 拥塞控制实现
│   │       ├── session.go            会话状态管理
│   │       └── log.go                日志封装
│   └── lib/                          Flutter UI 层（无需改动 — proxyTypeMap 从 API 动态获取）
└── twin-server/                      独立服务端（同目录 condercx/twin-server.git）
    ├── main.go                       命令行入口 (--listen, --password, --cert, --key, --conf, --version)
    ├── install_server.sh             一键安装脚本
    ├── README.md
    └── testclient/main.go            测试客户端
```

## 版本里程碑

### v0.0.1 — 初始实现 (完成 ✅)
- 完整的 QUIC 拨号/监听、密码认证、TCP/UDP 流代理
- BrutalSender / Pacer 代码存在但**未挂载**到 QUIC 连接上
- 无带宽协商

### v0.0.2 — BrutalSender 挂载 (完成 ✅)
- **auth.go 扩展**：带宽协商 — 客户端发送 sendBPS/recvBPS，服务端回复
- **client.go 重构**：auth 后挂载 BrutalSender 到 QUIC 连接
- **portal_session.go 重构**：服务端 auth 后挂载 BrutalSender
- 测试验证：本地 testclient 通过 BrutalSender 成功连接

### v0.0.3 — 断线重连 + UDP Datagram 中继 (完成 ✅)
- **Bug 修复**：adapter ensureConn 健康检查 (`IsClosed()`) + 断线自动重连
- **Bug 修复**：DialContext 错误后重置 client 状态，允许下一次重新拨号
- **New**：`protocol.go` — udpMessage 序列化、分片/重组，服务端 udpRelay
- **New**：`client.go` — `NewUDPSession()` QUIC datagram 式 UDP 回话
- **New**：`config.go` / `quic.go` — `KeepAlivePeriod`、`MaxIdleTimeout`、`DisablePMTU` 可配置
- 在 VPS 多线程测试中：多线程高速下载后上传卡死（stream.Write 流控阻塞）

### v0.0.4 — 多线程卡死修复 + 超时保护 (当前版本)
- **Bug 修复**：client.go `OpenStream()` → `OpenStreamSync()` 带 context 超时
- **Bug 修复**：client.go `writeHeader()` — 协议头写入带 `SetWriteDeadline(10s)`，避免 stream.Write 永久阻塞
- **Bug 修复**：portal_session.go `net.Dial("tcp")` → `net.DialTimeout("tcp", ..., 10s)`
- **Bug 修复**：portal_session.go TCP/UDP 转发 `io.Copy` → 32KB buf 读写循环，带 `SetReadDeadline`/`SetWriteDeadline(300s)`
- **Bug 修复**：adapter/twin.go DialContext 错误时立即 `cleanupLocked()`，下次调用 `ensureConn` 会重建完整连接
- 本地/远端端到端测试通过，多线程测速后不会 timeout

## CI/CD

### FlClash（GitHub Actions）
- `.github/workflows/build.yaml` — 打 v* tag 触发
- 平台：Android (3 ABIs)、Windows amd64、Linux amd64/arm64、macOS arm64
- 构建产物：APK + mihomo 内核二进制（含 twin 出站）

### twin-server（GitHub Actions）
- `.github/workflows/build.yaml` — 打 v* tag 触发
- 平台：Linux amd64/arm64
- 构建产物：`twin-server-linux-{arch}` 二进制
- 安装脚本：`install_server.sh`（支持 `--remove`）

## 配置示例

```yaml
proxies:
  - name: "twin-example"
    type: twin
    server: your-server.com
    port: 8443
    password: your-password
    sni: your-server.com
    skip-cert-verify: true
    udp: true
    up: "100 Mbps"
    down: "200 Mbps"
    side-channel: true
    side-strategy: auto
```

## 开发与测试

服务端启动：
```
cd twin-server
go run . -listen :8443 -password test -cert cert.pem -key key.pem
```

测试客户端：
```
go run ./testclient -server 127.0.0.1:8443 -password test -target https://www.baidu.com
```
