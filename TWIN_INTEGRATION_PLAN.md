# twin 协议集成进 FlClash — 实施计划

## 背景

twin 是一个基于 Hysteria 2 的加密翻墙协议，具有以下独特优化：
1. **自定义 BrutalSender** — 拥塞事件不缩窗，动态调整 ACK 频率
2. **自定义 Pacer (高 Burst)** — 一次 burst 多发包，而非标准令牌桶 10 包限制
3. **双通道 (SideChannel)** — TCP 流量通过 FlowMux 拆分到独立 QUIC 流
4. **XPlus 载荷混淆** — QUIC datagram 层进行 salt+SHA256 XOR，DPI 无法识别协议指纹
5. **最大化 QUIC Flow Control 窗口** — 8-16 倍于默认值，确保高丢包链路油不受限
6. **禁用 PMTU 发现** — 避免 ICMP 阻塞导致连接中断
7. **无端口跳跃（Port Hopping）** — 单端口固定连接，减少 QoS 特征

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
│   │       ├── auth.go               密码认证 + 带宽协商 + 混淆密钥派生
│   │       ├── client.go             客户端拨号 + BrutalSender + FlowMux SideChannel
│   │       ├── server.go             服务端监听（ObfsPacketConn 包裹）
│   │       ├── portal_session.go     服务端连接管理 + TCP/UDP 代理 + FlowMux 分发 + goroutine 池
│   │       ├── protocol.go           UDP 数据报序列化 + 分片重组
│   │       ├── sidechannel.go        双通道选择策略
│   │       ├── sideframe.go          SideChannel 帧编解码 (FlowID)
│   │       ├── flow_mux.go           FlowMux — 单流多路复用（OpenFlow / FrameData / FrameCloseFlow）
│   │       ├── pacer.go              Brutal Pacer (高 burst, 30 packets)
│   │       ├── brutal.go             Brutal 拥塞控制实现
│   │       ├── obfs/obfs.go          Obfuscator interface
│   │       ├── obfs/packet_conn.go   ObfsPacketConn — XPlus 混淆的 net.PacketConn wrapper
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

### v0.0.1 — 初始实现 (完成 ?)
- 完整的 QUIC 拨号/监听、密码认证、TCP/UDP 流代理
- BrutalSender / Pacer 代码存在但未挂载到 QUIC 连接上
- 无带宽协商

### v0.0.2 — BrutalSender 挂载 (完成 ?)
- 带宽协商 — 客户端发送 sendBPS/recvBPS，服务端回复
- auth 后挂载 BrutalSender 到 QUIC 连接

### v0.0.3 — 断线重连 + UDP Datagram 中继 (完成 ?)
- ensureConn 健康检查 + 断线自动重连
- udpMessage 序列化、分片/重组，服务端 udpRelay
- QUIC datagram 式 UDP 会话
- KeepAlivePeriod、MaxIdleTimeout、DisablePMTU 可配置

### v0.0.4 — 多线程卡死修复 (完成 ?)
- OpenStreamSync 带 10s 超时
- SetWriteDeadline 协议头保护
- net.DialTimeout

### v0.0.5 — 回归修复 + SideChannel 基础 (完成 ?)
- 恢复 io.Copy，替换手动 buf 循环
- TCP idle timeout 改用 context.WithTimeout
- 延迟回到与 v0.0.3 一致
- type 0x03/0x04 SideChannel 流建立

### v0.0.6 — SideChannel 数据面 + XPlus 混淆 + 流控 (当前版本)
- **New**: `obfs/packet_conn.go` / `obfs/obfs.go` — XPlus 载荷混淆，salt + SHA256(key+salt) XOR payload
- **New**: `flow_mux.go` — 基于 SideFrame 的 TCP 流多路复用器（OpenFlow / FrameData / FrameCloseFlow）
- **Refactor**: `sideframe.go` — 加入 FlowID uint32 字段，帧头从 7 字节扩展到 11 字节
- **Refactor**: `sidechannel.go` — 返回 ChannelType，split 策略轮询，auto 按丢包率切换
- **Refactor**: `client.go` — DialTCP 优先走 sideMux OpenFlow，atomic.Pointer 避免竞态
- **Refactor**: `portal_session.go` — type 0x03/0x04 创建 FlowMux 并启动 sideMuxAcceptLoop
- **Refactor**: `portal_session.go` — handleFlowTCP 使用 goroutine 池（maxConcurrentTCP=256）
- **Refactor**: `server.go` — 包裹 ObfsPacketConn
- **Refactor**: `adapter/outbound/twin.go` — 移除 DialUDPStream，ListenPacketContext 改用 NewUDPSession
- **Config**: `MaxIncomingStreams` 硬上限 1024

## 遗留问题
- 多线程高速测速结束后连接短暂不可用（数秒恢复）
- 原因：BrutalSender 高速发包结束时 ACK 处理滞后
- 后续：服务端流控保护已加 goroutine 池缓解，但仍需优雅 drain

## CI/CD

### FlClash（GitHub Actions）
- `.github/workflows/build.yaml` — 打 v* tag 触发
- 平台：Android (3 ABIs)、Windows amd64、Linux amd64/arm64、macOS arm64
- 构建产物：APK + mihomo 内核二进制（含 twin 出站）

### twin-server（GitHub Actions）
- `.github/workflows/build.yaml` — 打 v* tag 触发
- 平台：Linux amd64/arm64
- 构建产物：twin-server-linux-{arch} 二进制
- 安装脚本：install_server.sh（支持 --remove）

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
