# twin 协议集成进 FlClash — 实施计划

## 背景

twin 是一个基于 Hysteria 2 (apernet/quic-go) 的闭源加密翻墙协议，具有以下独特优化：

1. **自定义 BrutalSender** — 拥塞事件不缩窗，动态调整 ACK 频率
2. **自定义 Pacer (高 Burst)** — 一次 burst 发多包，而非标准令牌桶
3. **双通道 UDP (SideChannel)** — UDP 流量拆分到两条 QUIC 流
4. **极大化的 QUIC 流控窗口** — Initial/Max 窗口值拉满
5. **禁用 PMTU 发现** — 避免 IPv6 ICMP 黑洞
6. **预分配 Buffer 池** — 减少 GC 抖动

实测在 HAX 垃圾 IPv6 环境下达到 200+ Mbps (同等条件下 Hysteria2 仅 60 Mbps)。

---

## 架构决策：独立 module vs 写进 Clash.Meta

**推荐方案：核心逻辑写成独立 Go module，Clash.Meta 只改 2 行。**

```
FlClash/
├── core/
│   ├── Clash.Meta/                 只改 2 行
│   │   ├── constant/adapters.go    +1 行：twin 类型常量
│   │   └── adapter/parser.go       +1 行："twin" case
│   ├── twin/                    新目录：独立 Go module
│   │   ├── go.mod
│   │   ├── brutal.go               BrutalSender
│   │   ├── pacer.go                自定义 Pacer
│   │   ├── sideframe.go            SideFrame 编解码
│   │   ├── sidechannel.go          双通道 UDP
│   │   ├── udpflow.go              UDP 流封装
│   │   ├── auth.go                 认证协议
│   │   ├── session.go              Session 管理
│   │   ├── config.go               配置
│   │   └── quic.go                 QUIC 配置工厂
│   ├── lib.go                      调用 twin.NewClient (或不改)
│   └── go.mod                      + replace 指向本地 twin/
├── lib/                            Flutter UI 改动 (主仓库)
└── plugins/                        不动
```

### 为什么推荐独立 module

| 方案 | Clash.Meta 修改量 | 上游同步代价 | 可维护性 |
|------|-------------------|-------------|---------|
| 写进 Clash.Meta 内部 | ~500+ 行新代码 + 常量 + parser | 高 (每次上游更新要解决冲突) | 低 |
| 独立 module + 2 行 patch | 2 行 | 极低 (改 2 行, 几秒的事) | 高 |

---

## 第 1 阶段：核心模块 (独立 module)

### 1.1 BrutalSender

```go
// 核心行为：
// - 拥塞窗口 = 目标带宽 * RTT (不因丢包缩减)
// - OnCongestionEvent：记录丢包事件，不缩窗口
// - updateAckRate：丢包率高 -> 降低 ACK 频率
// - OnPacketAcked：窗口不变或渐增
type BrutalSender struct {
    targetBPS         uint64
    congestionWindow  uint64
    lossRate          float64
    ackRate           int  // 每个 N 个包发一次 ACK
}
```

### 1.2 自定义 Pacer

```go
// 核心差异：burst 大小是标准的 3 倍
// 标准：maxBurstSize = 2 * maxDatagramSize
// twin：maxBurstSize = 6 * maxDatagramSize
type Pacer struct {
    bandwidth     func() uint64   // 动态带宽估计
    maxBurstSize  uint64          // 高 burst
    lastSendTime  time.Time
}
```

### 1.3 SideFrame 编解码

```
帧结构 (推测)：
[0-1]   Magic       (2 bytes, 固定值)
[2]     FrameType   (1 byte)
[3-4]   PayloadLen  (2 bytes, big-endian)
[5-6]   TargetLen   (2 bytes)
[7-...] Payload     (variable)
[...]   TargetAddr  (variable)

FrameType:
  0x01 = DATA       - UDP 数据
  0x02 = OPEN_FLOW  - 打开 UDP 流
  0x03 = CLOSE_FLOW - 关闭 UDP 流
  0x04 = SIDE_DATA  - 副通道数据
  0x05 = HEARTBEAT
```

### 1.4 双通道 UDP

```
UDP 数据路径：

客户端应用
    -> (SOCKS5)
Vector.sendUDP()
    +-- 主通道 (datagramLoop) -> QUIC Stream A
    +-- useSidePath=true -> 副通道 (sideLoop) -> QUIC Stream B
                                              |
                                        sidePeer (对端 peer)

策略：
- 小包 (< 200 bytes) -> 主通道 (低延迟)
- 大包 (>= 200 bytes) 且丢包率 > 5% -> 走副通道
- 丢包率 > 15% -> 双通道冗余发送
```

### 1.5 QUIC 配置

```go
func NewtwinQUICConfig() *quic.Config {
    return &quic.Config{
        InitialStreamReceiveWindow:     8 * 1024 * 1024,    // 8 MB
        MaxStreamReceiveWindow:          64 * 1024 * 1024,   // 64 MB
        InitialConnectionReceiveWindow:  16 * 1024 * 1024,   // 16 MB
        MaxConnectionReceiveWindow:      256 * 1024 * 1024,  // 256 MB
        MaxIdleTimeout:                  120 * time.Second,
        KeepAlivePeriod:                 15 * time.Second,
        EnableDatagrams:                 true,
        DisablePathMTUDiscovery:         true,
    }
}
```

所有窗口值为 Hysteria2 默认值的 8-16 倍，确保高丢包链路不会被 flow control 瓶颈卡住。

---

## 第 2 阶段：Outbound 适配器

### 2.1 配置结构

```go
type twinOption struct {
    BasicOption
    Name             string   `proxy:"name"`
    Server           string   `proxy:"server"`
    Port             int      `proxy:"port,omitempty"`
    Password         string   `proxy:"password,omitempty"`
    SNI              string   `proxy:"sni,omitempty"`
    SkipCertVerify   bool     `proxy:"skip-cert-verify,omitempty"`
    Fingerprint      string   `proxy:"fingerprint,omitempty"`
    Certificate      string   `proxy:"certificate,omitempty"`
    PrivateKey       string   `proxy:"private-key,omitempty"`
    Up               string   `proxy:"up,omitempty"`
    Down             string   `proxy:"down,omitempty"`

    // 双通道
    SideChannel      bool     `proxy:"side-channel,omitempty"`
    SideStrategy     string   `proxy:"side-strategy,omitempty"`

    // QUIC 窗口覆盖
    InitialStreamReceiveWindow     uint64 `proxy:"initial-stream-receive-window,omitempty"`
    MaxStreamReceiveWindow         uint64 `proxy:"max-stream-receive-window,omitempty"`
    InitialConnectionReceiveWindow uint64 `proxy:"initial-connection-receive-window,omitempty"`
    MaxConnectionReceiveWindow     uint64 `proxy:"max-connection-receive-window,omitempty"`
}
```

### 2.2 Clash.Meta 需要改的 2 个文件

**constant/adapters.go** - 加类型常量：
```go
const (
    // ... existing ...
    twin AdapterType = "twin"
)
```

**adapter/parser.go** - 加 switch case：
```go
case "twin":
    nwOption := &outbound.twinOption{BasicOption: basicOption}
    if err := decoder.Decode(rawConfig, nwOption); err != nil {
        return nil, err
    }
    proxy, err = outbound.Newtwin(*nwOption)
```

### 2.3 连接流程

```
Newtwin(option)
  +-- NewtwinQUICConfig()
  +-- NewBrutalSender(up, down)
  +-- NewPacer(bandwidth)
  +-- dial QUIC 连接
  +-- auth stream 认证
  +-- 启动 sideLoop goroutine
  +-- 返回 *twin 实例 (实现 C.ProxyAdapter)
```

---

## 第 3 阶段：Flutter UI 层

| 文件 | 修改内容 |
|------|---------|
| lib/common/protocol.dart | 注册 twin 协议 |
| lib/common/proxy.dart | 增加 twinNode 类 |
| lib/pages/proxy/editor.dart | 增加编辑表单 (复用 Hysteria2 布局 + side-channel 开关) |
| lib/models/proxy.dart | 增加配置模型 |
| lib/enum/proxy_type.dart | 增加 twin 枚举值 |

---

## 第 4 阶段：测试验证

### 4.1 功能测试
- TCP CONNECT
- UDP ASSOCIATE (单通道)
- UDP 双通道负载分配
- 断线重连

### 4.2 性能对比 (HAX IPv6 环境)

| 方案 | 预期吞吐 |
|------|---------|
| mihomo 内置 Hysteria2 | ~60 Mbps (基准) |
| twin (单通道，无 side channel) | ~120 Mbps |
| twin (完整双通道) | ~200+ Mbps |

---

## 实施路线图

| 阶段 | 内容 | 预估时间 |
|------|------|---------|
| P1 | 核心模块：brutal sender + pacer + sideframe 编解码 | 5-7 天 |
| P2 | QUIC 配置 + 认证 + session 管理 | 2-3 天 |
| P3 | 双通道 side channel | 3-4 天 |
| P4 | Outbound 适配器 + 2 行 Clash.Meta 修改 | 1-2 天 |
| P5 | Flutter UI | 2-3 天 |
| P6 | 端到端调试优化 | 3-5 天 |
| **总计** | | **16-24 天** |

**MVP 路线 (先出能用的版本)**：P1 -> P2 -> P4 -> P5 (P3 延后)，约 **8-10 天**，预计可达 120+ Mbps。

---


---

## 服务端 (twin-server)

### 架构

twin 与服务端是独立的 Go 可执行文件，基于同一个 `twin-go` 核心库。

```
twin-go (github.com/condercx/twin-go)       ← 共享协议库
├── config.go, brutal.go, pacer.go, quic.go
├── sideframe.go, sidechannel.go
├── auth.go           ← 认证协议（双方共享）
├── client.go         ← 客户端拨号
└── server.go         ← 服务端监听 (Portal)

twin-server (github.com/condercx/twin-server)  ← 服务端可执行文件
├── go.mod → require github.com/condercx/twin-go
├── cmd/twin-server/main.go
└── config.yaml (默认配置模板)

FlClash core/twin/  (submodule: condercx/twin-go)  ← 客户端侧共享库
```

### 服务端 (Portal) 工作流程

```
1. UDP 监听端口
2. 接受 QUIC 连接 (TLS 1.3)
3. 每个连接创建 portalSession
   ├── authenticateConnection — QUIC Stream 认证
   ├── handleStream — TCP 代理（dial 目标 + 双向复制）
   ├── handleUDPRequest — UDP 代理（portalUDPFlow）
   │     ├── readLoop → sendResponse
   │     └── sendSideResponse (双通道)
   ├── datagramLoop — QUIC DATAGRAM 帧处理
   └── sideLoop — 副通道管理 (sidePeer)
4. BrutalSender (下行带宽)
```

### 认证协议

```
客户端 QUIC 连接建立后，打开 unidirectional stream：
  1. 发送: [password length:2][password bytes][nonce length:2][nonce bytes]
  2. 服务端验证后回复 auth result
  3. 认证通过后 Session 进入 Ready 状态
```

### 配置格式 (YAML / URL)

```
# 命令行启动
twin-server --listen 0.0.0.0:443 --password secret --cert cert.pem --key key.pem

# 或配置文件
server:
  listen: 0.0.0.0:443
  password: secret
  tls:
    cert: cert.pem
    key: key.pem
  bandwidth:
    up: 200M
    down: 1000M
```

### 需要开发的服务端模块

| 文件 | 对应 nowhere | 说明 |
|------|-------------|------|
| `twin-go/server.go` | `portal.Portal` | 核心服务端逻辑 |
| `twin-go/auth.go` | `portal.authenticateConnection` | 认证处理 |
| `twin-go/portal_session.go` | `portal.portalSession` | 连接生命周期 |
| `twin-go/portal_udpflow.go` | `portal.portalUDPFlow` | UDP 流管理 |
| `twin-go/sidepeer.go` | `portal.sidePeer` | 副通道对端 |
| `twin-server/main.go` | — | 入口，加载配置，启动 Portal |

### 本地联调方案

```
本机 127.0.0.1
┌─────────────┐      QUIC (UDP)       ┌────────────────┐
│ twin-server  │◄──────────────────────►│ FlClash / twin  │
│ :8443        │                        │ client          │
│ password=x   │                        │ server=:8443    │
└─────────────┘                        │ password=x      │
                                       └────────────────┘
```

开发流程：
1. 本地启动 `twin-server` 监听 127.0.0.1:8443
2. FlClash 配置 twin 类型 outbound，指向该地址
3. tcpdump/Wireshark 抓 QUIC 包调试
4. 验证通过后部署到 HAX IPv6 实测
## 文件变更总览

```
core/twin/  (submodule: condercx/twin-go.git) 独立 Go module
+-- go.mod
+-- brutal.go
+-- pacer.go
+-- sideframe.go
+-- sidechannel.go
+-- udpflow.go
+-- auth.go
+-- session.go
+-- config.go
+-- quic.go

core/Clash.Meta/                        改 2 行
+-- constant/adapters.go                +1 行
+-- adapter/parser.go                   +1 行

core/
+-- adapter/outbound/twin.go         新：薄适配层
+-- go.mod                              + require 和 replace

lib/                                    Flutter 层
+-- common/protocol.dart
+-- common/proxy.dart
+-- models/proxy.dart
+-- enum/proxy_type.go
+-- pages/proxy/editor.dart


