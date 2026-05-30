---
name: docker-uds-ipc
description: 容器间 Unix Domain Socket IPC 优化：在同一宿主机上避免网络栈开销的实践方案
tags: [docker, container, networking, ipc, performance, uds, optimization]
---

## 概述

同一宿主机上的容器间通信默认走完整网络栈：`生产者 → 内核 TCP → ENI → VPC → LB → ENI → 内核 TCP → 消费者`，即使物理上只有几十厘米铜线距离。当吞吐量达到每秒数千次以上时，这种开销会主导成本和尾延迟。

**Unix Domain Socket (UDS)** 是解决此问题的经典方案：利用共享文件系统路径作为地址，数据直接在内核缓冲区传递，不走 TCP/IP 栈，单通道可达 50-55 Gbit/s。

## 问题本质

### awsvpc 模式下的 localhost 误区

在 ECS `awsvpc` 网络模式下，每个任务有独立 ENI 和独立网络命名空间。容器 A 的 `127.0.0.1:8080` 和容器 B 的 `127.0.0.1:8080` 是不同东西，互不可达。

### 默认方案的成本

| 方案 | 缺点 |
|------|------|
| `127.0.0.1` loopback | awsvpc 模式下不在同一网络命名空间 |
| `host` 网络模式 | 放弃端口隔离，所有容器共享端口空间 |
| 负载均衡器 | 额外 ENI 消耗、VPC 数据平面开销、TLS 终止/重加密、HTTP 帧开销 |

### UDS 优势

- 无 TCP/IP 栈：无头开销、无校验和、无拥塞控制
- 无网络：数据不出内核，不经过 iptables/VPC 路由
- 无端口冲突：地址是文件系统路径
- 性能：单通道 50-55 Gbit/s

## 部署拓扑

```
┌──────────────────────────────────────────┐
│  EC2 实例                                 │
│  ┌─────────┐  ┌─────────┐  ┌─────────┐  │
│  │ 容器 A   │  │ 容器 B   │  │ 容器 C   │  │
│  │(生产者)  │  │(生产者)  │  │(IPC 服务)│  │
│  └────┬────┘  └────┬────┘  └────┬────┘  │
│       │            │            │       │
│       └────────────┴────────────┘       │
│                  │ UDS socket            │
│                  ▼                       │
│         /var/run/ipc-server.sock         │
│         (共享卷 bind-mount)               │
└──────────────────────────────────────────┘
```

UDS socket 文件通过宿主机 bind-mount 卷在所有容器间共享。

## Wire Protocol 设计

### 帧格式

```
┌──────────────────────────────────────────────┐
│ 固定前导 (16 bytes)                           │
│ ┌──────┬────────┬─────┬──────┬─────────────┐ │
│ │ 帧长  │ Header │标志  │ 保留  │ Header CRC  │ │
│ │uint32│ 长度   │byte │ byte  │  uint32     │ │
│ │      │uint16  │     │      │(CRC32C)     │ │
│ └──────┴────────┴─────┴──────┴─────────────┘ │
│ 可变长度 Header (Protobuf)                    │
│ 负载 Payload (opaque bytes)                   │
└──────────────────────────────────────────────┘
```

设计原则：**先拒绝后分配** — 在分配 Payload 内存前先验证帧长，防止恶意/错误的大帧耗尽内存。

### Java (Netty) 编码示例

```java
public static void encode(Frame frame, ByteBuf output, CRC32C scratch) {
    byte[] headerBytes = frame.header().toByteArray();
    int payloadLength = frame.payload().length;
    long frameLength = headerBytes.length + payloadLength;

    if (frameLength > MAX_FRAME_BYTES) throw new IllegalArgumentException("frame too large");

    scratch.reset();
    scratch.update(headerBytes, 0, headerBytes.length);
    long headerCrc = scratch.getValue();

    output.writeInt((int) frameLength);       // 4 bytes
    output.writeShort(headerBytes.length);     // 2 bytes
    output.writeByte(0);                       // flags (1 byte)
    output.writeByte(0);                       // rsv   (1 byte)
    output.writeInt((int) headerCrc);          // 4 bytes — 总计 16 bytes
    output.writeBytes(headerBytes);
    if (payloadLength > 0) output.writeBytes(frame.payload());
}
```

### Netty 解码器优化

使用 `COMPOSITE_CUMULATOR` 代替默认的 `MERGE_CUMULATOR`，避免高吞吐下不必要的内存复制：

```java
// ByteToMessageDecoder 子类
public MyDecoder() {
    setCumulator(ByteToMessageDecoder.COMPOSITE_CUMULATOR);
}
```

## 三维背压机制

大多数系统只做一维的"允许 N 个请求并发"。这里使用三维信用控制：

| 维度 | 含义 | 防止问题 |
|------|------|----------|
| chunk-slot | 并发处理的 Chunk 数 | 内存耗尽 |
| raw-event | 聚合事件总数 | 索引/存储溢出 |
| payload-byte | 载荷字节总数 | 缓冲区溢满 |

```java
public synchronized boolean tryAcquire(int slots, long events, long payloadBytes) {
    if (availableChunkSlots < slots
            || availableRawEvents < events
            || availablePayloadBytes < payloadBytes) {
        return false;
    }
    availableChunkSlots -= slots;
    availableRawEvents -= events;
    availablePayloadBytes -= payloadBytes;
    return true;
}
```

生产者和服务端**各自独立执行同一套信用检查**，服务端是最终仲裁者，防御有缺陷或恶意的客户端。

信用恢复条件：ACCEPTED 确认、FAILED 确认（携带恢复量）、连接断开（释放所有占用）。

## 握手与协议版本协商

```java
private OptionalInt negotiateVersion(HelloFrame hello) {
    int producerMin = hello.getMinProtocolVersion();
    int producerMax = hello.getMaxProtocolVersion();
    if (producerMax < serverConfig.minProtocolVersion()) return OptionalInt.empty();
    if (producerMin > serverConfig.maxProtocolVersion()) return OptionalInt.empty();
    return OptionalInt.of(Math.min(producerMax, serverConfig.maxProtocolVersion()));
}
```

仅六行代码的版本协商，无需引入功能标志或可选能力层。

关键优化：`channelReadComplete` 时批量发送 ACCEPTED 确认，一个 ack 帧对应 N 个 DATA 帧，避免逐个确认的往返开销。

## 注意事项

- **UDS socket 文件通过共享卷 publish**：不同容器默认无共享文件系统，需通过宿主机 bind-mount 将 socket 文件暴露给所有参与者
- **Netty 事件循环不可阻塞**：不要在 Netty 的 event loop 线程执行阻塞操作，应使用专用线程池
- **重视 Credential 隔离**：UDS 仅提供文件权限控制 (`chmod 777`/`chown`)，不适合跨信任边界的通信
- **重试策略**：生产者连接 socket 文件需指数退避+随机抖动重试，处理服务端未就绪的情况
- **不反对云原生 LB**：数据面性能优化针对高吞吐场景（>1万 QPS），低吞吐场景保持使用 LB 更简单
- **仅适合同主机通信**：无法跨主机，跨主机仍需走 TCP/gRPC
