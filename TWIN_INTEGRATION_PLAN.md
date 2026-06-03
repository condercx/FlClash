# twin 协议集成进 FlClash - 实施计划

## 背景

twin 是一个基于 Hysteria 2 的加密翻墙协议，具有以下独特优化：
1. **自定义 BrutalSender** - 拥塞事件不缩窗，动态调整 ACK 频率
2. **自定义 Pacer (高 Burst)** - 一次 burst 多发包，而非标准令牌桶 10 包限制
3. **双通道 (SideChannel)** - TCP 流量通过 FlowMux 拆分到独立 QUIC 流
4. **XPlus 载荷混淆** - QUIC datagram 层进行 salt+SHA256 XOR，DPI 无法识别协议指纹
5. **最大化 QUIC Flow Control 窗口** - 8-16 倍于默认值，确保高丢包链路油不受限
6. **禁用 PMTU 发现** - 避免 ICMP 阻塞导致连接中断
7. **无端口跳跃（Port Hopping）** - 单端口固定连接，减少 QoS 特征

## 版本里程碑

### v0.0.1 - 初始实现 [完成]
### v0.0.2 - BrutalSender 挂载 [完成]
### v0.0.3 - 断线重连 + UDP Datagram 中继 [完成]
### v0.0.4 - 多线程卡死修复 [完成]
### v0.0.5 - 回归修复 + SideChannel 基础 [完成]
### v0.0.6 - SideChannel 数据面 + XPlus 混淆 + 流控 [当前版本]

### v0.0.6 改动
- **New**: obfs/packet_conn.go / obfs/obfs.go - XPlus 载荷混淆
- **New**: flow_mux.go - 基于 SideFrame 的 TCP 流多路复用器
- **Refactor**: sideframe.go - 加入 FlowID uint32 字段
- **Refactor**: sidechannel.go - 返回 ChannelType，split 轮询，auto 按丢包率切换
- **Refactor**: client.go - DialTCP 优先走 sideMux OpenFlow
- **Refactor**: portal_session.go - FlowMux 分发 + goroutine 池
- **Refactor**: server.go - 包裹 ObfsPacketConn
- **Refactor**: adapter/outbound/twin.go - 改用 NewUDPSession
- **Config**: MaxIncomingStreams 硬上限 1024

## 遗留问题
- 多线程高速测速结束后连接短暂不可用（数秒恢复）

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
