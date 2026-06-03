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
│   │   ├── Clash.Meta/               ← 子模块 condercx/Clash.Meta.git
│   │   │   └── adapter/outbound/twin.go  ← Twin 出站适配器
│   │   └── twin/                      ← 子模块 condercx/twin-go.git
│   │       ├── config.go              默认/最大窗口配置
│   │       ├── quic.go               QUIC Config 工厂
│   │       ├── auth.go               密码认证 + 带宽协商
│   │       ├── client.go             客户端拨号 + BrutalSender 挂载
│   │       ├── server.go             服务端监听
│   │       ├── portal_session.go     服务端连接管理 + BrutalSender 挂载
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

### v0.0.2 — 性能基础 (完成 ✅)
- **auth.go 扩展**：带宽协商 — 客户端发送 sendBPS/recvBPS，服务端回复
- **client.go 重构**：auth 后挂载 BrutalSender 到 QUIC 连接（client.go:40-56）
- **portal_session.go 重构**：服务端 auth 后挂载 BrutalSender（portal_session.go:48-52）
- 测试验证：testclient 通过 BrutalSender 成功连接，日志确认 `BPS=100000000`
- 预期：在垃圾 IPv6 线路（hax）上从几十 Kbps 提升到 200+ Mbps

### v0.0.3 — UDP Datagram + SideChannel 深度优化 (计划中)
- UDP 转发改为 QUIC Datagram（而非 Stream），降低延迟
- SideChannel 双通道数据面实际生效
- QUIC Config 参数暴露为配置选项

## CI/CD

### FlClash（GitHub Actions）
- `.github/workflows/build.yaml` — 打 v* tag 触发
- 平台：Android (3 ABIs)、Windows amd64、Linux amd64/arm64、macOS arm64
- 构建产物：APK + Clash.Meta 内核二进制（含 twin 出站）

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
