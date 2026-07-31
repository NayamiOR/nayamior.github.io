---
author: Nayami
pubDatetime: 2026-07-26T12:00:00.000Z
modDatetime: 2026-07-26T12:00:00.000Z
title: 从零写一个 IoT 边缘网关：LEPG 的设计与实现
tags:
  - go
  - iot
  - modbus
  - networking
description: 用 Go 从零构建一个轻量级 IoT 边缘网关——自定义 TLV 协议、Modbus 采集、断点续传、北向数据输出。11K 行代码的设计决策与踩坑记录。
draft: true
---

## 为什么写 LEPG

2025 年底，我在做智能家居改造——树莓派接了几个 Modbus 温湿度传感器，想把数据传到云端用手机看。市面上的方案让我很头疼：

- **云厂商 IoT 套件**：年费贵，协议锁定，树莓派上 agent 吃 200MB 内存
- **自建方案**：需要自己搭 frp 穿透 + MQTT Broker + 协议转换脚本，折腾一周还跑不稳
- **工业网关**：动辄几千块，而且是为 PLC/SCADA 设计的，对个人开发者极不友好

我就想：能不能写一个**单二进制、10 分钟部署、树莓派能跑**的网关？于是 LEPG（Lightweight Edge Piercing Gateway）诞生了。

写完第一版后，我发现这个需求并不小众——很多小型制造车间、农业大棚、实验室课题组都面临同样的问题：**设备不多（几十到几百台），但缺一个简单可靠的中间层把数据从设备端送到云端**。

目前 LEPG 约 11K 行 Go 代码，覆盖了从设备采集到北向数据输出的完整链路。这篇文章聊聊架构设计和几个有意思的技术决策。

## 整体架构

LEPG 的链路分四层：

```
┌──────────────┐     ┌─────────────────┐     ┌──────────────┐     ┌──────────┐
│  现场设备层    │ ──► │  边缘网关（客户端） │ ──► │  中继端（服务端） │ ──► │  用户侧   │
│  Modbus/PLC   │     │  树莓派/小主机     │     │  云服务器/Docker  │     │  MQTT/HTTP│
└──────────────┘     └─────────────────┘     └──────────────┘     └──────────┘
```

**上行链路**：设备 → 协议解析 → 本地缓存 → TCP 隧道穿透 → 服务端入库 → MQTT 输出

**下行链路**：用户下发指令 → 服务端路由 → TCP 隧道 → 边缘端写设备

核心设计原则是 **边缘端有脑**——不只是透传，而是在本地做协议解析、数据缓存、断点续传，减轻服务端压力。

## 三个有意思的设计

### 1. 自定义 TLV 协议：为什么不用 MQTT 直连？

最初我想让边缘端直接连服务端的 MQTT Broker，但很快发现三个问题：

- **MQTT 是 Pub/Sub 模型**，不适合"边缘端上报 → 服务端确认"的**请求-确认**模式。我需要 UploadAck 机制：服务端入库成功后回一个 ACK，边缘端收到 ACK 才删本地缓存。用 MQTT 做这个需要额外的 correlation ID 和 response topic，徒增复杂度。
- **多路复用**：一个边缘端可能同时有多个数据流（实时读数、批量历史、心跳），MQTT 需要多个 topic，而自定义协议可以在一根 TCP 连接上按消息类型路由。
- **轻量化**：MQTT 包头在 CONNECT/SUBACK 阶段有不少开销，自定义 TLV 协议最小帧只有 16 字节头，适合低带宽场景。

最终协议格式：

```
[MAGIC:2B][VERSION:1B][FLAGS:1B][TYPE:1B][MSGID:2B][PAYLOAD_LEN:2B][TIMESTAMP:4B][PAYLOAD:N][CRC16:2B]
```

- `MAGIC = 0x4E59`（"NY"，我的 initials——写协议时的小乐趣）
- CRC16 校验，防止比特翻转
- `MSGID` 自增 + `UploadAck` 做可靠传输

Go 实现很直接：

```go
type Msg struct {
    Magic      uint16
    Version    uint8
    Flags      uint8
    Type       uint8
    MsgID      uint16
    PayloadLen uint16
    Timestamp  uint32
    Payload    []byte
    Checksum   uint16
}

// 消息类型
const (
    MsgTypeHandshake    uint8 = 1  // 握手
    MsgTypeHandshakeAck uint8 = 2  // 握手确认
    MsgTypeUpload       uint8 = 3  // 数据上传
    MsgTypeUploadAck    uint8 = 4  // 上传确认（关键！）
    MsgTypeHeartbeat    uint8 = 5  // 心跳
    MsgTypeHeartbeatAck uint8 = 6
)
```

`UploadAck` 是整个链路可靠性的基石。边缘端在收到服务端 `UploadAck` 之前不会删除本地 SQLite 缓存——如果超时，触发重传；连续超时，触发重连。

### 2. Modbus 运行时：goroutine 驱动的轮询 + 写调度

Modbus 是半双工协议——同一时间只能有一个请求在线上。这意味着并发的"读"和"写"不能同时发。我的方案是**一根 goroutine 管轮询，一根 goroutine 管写，共享一个 channel 做同步**：

```go
type ModbusRuntime struct {
    cfg     *DeviceConfig
    client  modbus.Client
    handler connectableHandler
    writeCh chan writeCmd     // 写命令队列
    stopCh  chan struct{}
    state   atomic.Int32      // online / offline / degraded
}

func (rt *ModbusRuntime) Start(ctx context.Context, readingCh chan<- model.Reading) error {
    rt.handler.Connect()
    rt.state.Store(stateOnline)

    rt.wg.Add(2)
    go rt.pollLoop(ctx, readingCh)   // 按 poll_interval 轮询寄存器
    go rt.writeLoop(ctx)             // 消费 writeCh 下发写命令
    return nil
}
```

`pollLoop` 内部是一个 `time.Ticker` 循环，每次轮询完所有点位后等待下一个 tick。`writeLoop` 在有外部写请求时，等当前轮询周期结束、拿到"线"的使用权后再发写命令。

这让 Modbus 的半双工约束变成了**结构化的 goroutine 协作**，而不是到处加 mutex。用户侧的 `Write()` 方法是同步阻塞的——调用方不需要知道内部有 goroutine 和 channel：

```go
func (rt *ModbusRuntime) Write(pointName string, value float64) error {
    if rt.state.Load() != stateOnline {
        return fmt.Errorf("device offline")
    }
    // 校验 pointName 存在且有 rw 权限
    // ...
    cmd := writeCmd{pointName, value, make(chan error, 1)}
    rt.writeCh <- cmd
    return <-cmd.resultCh  // 阻塞等待写结果
}
```

### 3. Provider Chain：一个轻量的配置优先级系统

配置来源有四处：**默认值 < 环境变量 < 配置文件 < 命令行参数**。我不想引入 Viper 的全部依赖（太重），但也不想手写 if-else 地狱。于是写了一个极简的 Provider Chain：

```go
type IProvider interface {
    Get(key string) (any, bool)
    Set(key string, value any)
}

type ProviderChain struct {
    providers []IProvider  // 从低到高优先级
}

func (pc *ProviderChain) Get(key string) (any, bool) {
    var value any
    found := false
    for _, p := range pc.providers {
        if v, ok := p.Get(key); ok {
            value = v
            found = true
            // 不 break——后面的 provider 优先级更高，继续覆盖
        }
    }
    return value, found
}
```

然后四个 Provider 各实现 `IProvider`：`DefaultProvider`（硬编码 map）、`EnvProvider`（`os.Getenv`）、`FileProvider`（TOML 解析）、`FlagProvider`（命令行参数）。链式查找，后面的自动覆盖前面的。总共不到 200 行 Go，完全够用。

## 当前状态与下一步

**已完成（R1+R2）**：
- Modbus TCP 采集（FC1-4，float32/uint32/int32，4 种字节序）
- TLV 协议 + TCP 隧道 + UploadAck 可靠传输
- SQLite 断点缓存 + PostgreSQL 持久化
- MQTT Broker 内建 + TB Gateway Sinker → ThingsBoard 对接
- 反向控制链路：`Write()` → TCP 隧道 → 边缘端 → Modbus 写寄存器

**接下来（R3）**：
- Modbus 写操作（FC5/6/16）
- 设备上下线 MQTT 通知
- HTTP 健康检查端点

代码在 [github.com/nayami/lepg](https://github.com/nayami/lepg)，欢迎 issue 和 PR。

---

写 LEPG 最大的收获不是代码量，而是理解了"工业现场"和"互联网后端"之间的差异——工业协议是半双工的，设备会掉线，数据有字节序，实时性不是越快越好而是越稳定越好。如果你也在做 IoT 相关的项目，欢迎交流。
