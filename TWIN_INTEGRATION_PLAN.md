# twin 协议集成进 FlClash — 实施计划

## 背景

twin 是一个基于 Hysteria 2 (apernet/quic-go) 的闭源加密翻墙协议，具有以下独特优化：

1. **自定义 BrutalSender** — 拥塞事件不缩窗，动态调整 ACK 频率
2. **自定义 Pacer (高 Burst)** — 一次 burst 发多包，而非标准令牌桶
3. **双通道 UDP (SideChannel)** — UDP 流量拆分到独立 QUIC 流减少 head-of-line blocking
4. **最大化 QUIC Flow Control 窗口** — 8-16 倍于默认值，确保高丢包链路上游不受限
5. **禁用 PMTU 发现** — 避免 ICMP 阻塞导致连接中断
6. **无端口跳跃（Port Hopping）** — 单端口固定连接，减少 QoS 特征

## 仓库结构

```
nowhere/
├── FlClash/                          ← 主仓库 (forked)
│   ├── core/
│   │   ├── Clash.Meta/               ← 子模块: condercx/Clash.Meta.git
│   │   │   └── adapter/outbound/twin.go  ← Twin 出站适配器 (新建)
│   │   └── twin/                      ← 子模块: condercx/twin-go.git
│   │       ├── config.go             默认/最大窗口配置
│   │       ├── quic.go               QUIC Config 工厂
│   │       ├── auth.go               密码认证协议
│   │       ├── client.go             客户端拨号 + SetConn()
│   │       ├── server.go             服务端监听
│   │       ├── portal_session.go     服务端连接管理
│   │       ├── portal_udpflow.go     UDP 流管理
│   │       ├── session.go            会话状态管理
│   │       ├── sidechannel.go        双通道选择策略
│   │       ├── sideframe.go          SideChannel 帧编码
│   │       ├── pacer.go              Brutal Pacer
│   │       ├── brutal.go             Brutal 拥塞控制
│   │       └── log.go                日志封装
│   └── lib/                           Flutter UI 层（无需改动 — proxyTypeMap 从 API 动态获取）
└── twin-server/                      独立服务端 (子模块: condercx/twin-server.git)
    ├── main.go                       命令行入口 (--listen, --password, --cert, --key)
    └── testclient/main.go            测试客户端
```

## 阶段完成状态

### ✅ P1: 逆向分析 nowhere-arm64
完成协议逆向，提取所有关键特征：
- BrutalSender 不缩窗逻辑
- 高 burst Pacer (30 packets vs 标准 10)
- 双通道 SideChannel
- QUIC Config 大窗口
- PMTU 禁用

### ✅ P2: twin-go 核心库
实现完整协议栈：QUIC 拨号/监听、密码认证、TCP 流代理、Brutal 拥塞控制、SideChannel 双通道。

### ✅ P3: twin-server 服务端 + 端到端测试
- twin-server 独立 Go module，支持 TLS 证书、密码认证
- testclient 通过 QUIC + auth → HTTP 代理测试通过
- 已验证从 testclient 成功请求 baidu.com

### ✅ P4: Clash.Meta 出站适配器

#### 改动文件

**core/twin/client.go**
- 将内联认证码替换为 `auth.go` 的 `WriteAuth` 函数
- 新增 `SetConn(conn *quic.Conn)` 方法，允许外部（Clash dialer）注入已建立的 QUIC 连接
- 原有 `Dial()` 自建连接行为不变

**core/Clash.Meta/constant/adapters.go**
- `AdapterType` 枚举加 `Twin` (line 53)
- `String()` 加 `case Twin: return "Twin"` (line 232-233)

**core/Clash.Meta/adapter/parser.go**
- 加 `case "twin":` 解析分支 → `outbound.NewTwin()` (line 93-99)

**core/Clash.Meta/adapter/outbound/twin.go** (新建)
- `Twin` 结构体（内嵌 `*Base`）
- `TwinOption` 配置字段：server, port, password, sni, skip-cert-verify, fingerprint, up, down, side-channel, side-strategy, 自定义 QUIC 窗口
- `NewTwin()` 构造函数
- `DialContext()` 通过 QUIC stream 转发 TCP
- `ListenPacketContext()` 暂返回 `ErrNotSupport`
- `ensureConn()` 懒加载：通过 Clash dialer 建立 QUIC 连接 + TLS + 密码认证
- 使用 `ca.GetTLSConfig` 支持指纹伪装、自定义证书

**core/Clash.Meta/go.mod**
- `require github.com/condercx/twin-go v0.0.0`
- `replace github.com/condercx/twin-go => ../twin`

#### 累计改动
```
4 files changed, 249 insertions(+), 1 deletion(-)
```

#### 编译验证
- `go build ./...` 全项目通过 ✅
- `go vet ./adapter/outbound/` 通过 ✅
- `go vet ./core/twin/` 通过 ✅

---

## Flutter UI 层

FlClash 的代理列表 (`proxyTypeMap`) 是从 Clash API `/proxies` 端点返回的 JSON 动态生成的，UI 没有硬编码协议类型列表。因此 **Flutter 层无需修改** — Clash.Meta 内核支持 `type: twin` 后，API 自动返回 Twin 类型的节点，UI 自动显示。

---

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

在 FlClash 配置中添加 twin 类型 proxy 即可使用。