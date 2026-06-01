---
name: pg-listen-notify-realtime
description: PostgreSQL LISTEN/NOTIFY 实现多租户实时事件推送，替代轮询和 Redis pub/sub
tags: [postgresql, realtime, pubsub, multi-tenant, websocket]
---

## 要点

PostgreSQL 内置的 LISTEN/NOTIFY 机制是实时事件推送的轻量方案，特别适合已使用 PostgreSQL 的项目：

- **零额外基础设施**：不需要 Redis、RabbitMQ 等额外中间件
- **事务性**：NOTIFY 仅在事务提交后才触发，天然与业务数据一致
- **多租户友好**：通过不同 channel 名称（如 `tenant_123_notifications`）隔离
- **跨进程广播**：多 Worker 实例自动同步，无需额外消息代理

### 适用场景
- 实时通知推送（新消息、告警）
- AI 任务完成回调
- 数据分析结果实时刷新
- 避开轮询的 HTTP 开销

> 注意：LISTEN/NOTIFY 不持久化消息。如无订阅者在线，事件丢失。适合最新状态推送，不适合可靠消息队列场景。

## 架构设计

```
┌─────────┐   WebSocket    ┌──────────────┐   LISTEN/NOTIFY   ┌──────────────┐
│  React   │ ◄────────────  │  FastAPI     │ ◄──────────────  │  PostgreSQL  │
│  Client  │                │  WebSocket   │                  │  (Publisher) │
└─────────┘                │  Server      │                  └──────────────┘
                           │              │                       ▲
                           │  (Listener)  │                       │
                           └──────────────┘     INSERT + NOTIFY   │
                                                    │              │
                                               ┌────┴──────┐
                                               │  AI Job   │
                                               │  Worker   │
                                               └───────────┘
```

## 代码示例

### 1. FastAPI WebSocket 服务端（监听端）

```python
import json
import asyncio
from fastapi import FastAPI, WebSocket
from sqlalchemy import text
from sqlalchemy.ext.asyncio import create_async_engine, AsyncSession

# 租户连接管理器
tenant_connections: dict[str, set[WebSocket]] = {}

app = FastAPI()

@app.websocket("/ws/notifications")
async def websocket_endpoint(websocket: WebSocket):
    await websocket.accept()
    tenant_id = websocket.query_params.get("tenant_id")
    if not tenant_id:
        await websocket.close(code=1008, reason="Missing tenant_id")
        return

    # 注册连接
    if tenant_id not in tenant_connections:
        tenant_connections[tenant_id] = set()
    tenant_connections[tenant_id].add(websocket)

    try:
        # 保持连接，等待客户端断开
        while True:
            await websocket.receive_text()
    except Exception:
        pass
    finally:
        tenant_connections[tenant_id].discard(websocket)
```

### 2. 后台通知发布端（事务内 NOTIFY）

```python
@app.post("/api/notifications")
async def send_notification(tenant_id: str, message: str, db: AsyncSession):
    # 1. 写入数据库
    await db.execute(
        text("""
            INSERT INTO notifications (tenant_id, message, created_at)
            VALUES (:tenant_id, :message, NOW())
        """),
        {"tenant_id": tenant_id, "message": message},
    )

    # 2. 发送 PG 通知（事务提交后触发）
    payload = json.dumps({
        "tenant_id": tenant_id,
        "type": "notification",
        "data": {"message": message},
    })
    await db.execute(
        text("SELECT pg_notify('notifications', :payload)"),
        {"payload": payload},
    )

    await db.commit()
    return {"ok": True}
```

### 3. AI 任务完成回调

```python
async def process_ai_feature(tenant_id: str, job_id: str, db: AsyncSession):
    # ... 执行 AI 推理 ...

    # 保存结果
    await db.execute(
        text("""
            UPDATE ai_jobs
            SET result = :result, status = 'completed'
            WHERE id = :job_id AND tenant_id = :tenant_id
        """),
        {"result": json.dumps(result), "job_id": job_id, "tenant_id": tenant_id},
    )

    # 实时通知租户
    payload = json.dumps({
        "tenant_id": tenant_id,
        "type": "ai_job_complete",
        "data": {"job_id": job_id, "result": result},
    })
    await db.execute(
        text("SELECT pg_notify('notifications', :payload)"),
        {"payload": payload},
    )

    await db.commit()
```

### 4. React 客户端

```typescript
// src/hooks/useNotifications.ts
import { useEffect, useRef } from "react";

export function useNotifications(
  tenantId: string,
  onMessage: (data: any) => void
) {
  const wsRef = useRef<WebSocket | null>(null);

  useEffect(() => {
    const protocol = window.location.protocol === "https:" ? "wss:" : "ws:";
    const wsUrl = `${protocol}//${window.location.host}/ws/notifications?tenant_id=${tenantId}`;

    const ws = new WebSocket(wsUrl);

    ws.onopen = () => console.log("✓ 实时连接已建立");
    ws.onmessage = (event) => {
      const message = JSON.parse(event.data);
      onMessage(message);
    };
    ws.onerror = () => console.error("WebSocket 错误");

    wsRef.current = ws;
    return () => ws.close();
  }, [tenantId, onMessage]);
}

// 使用示例
function NotificationCenter({ tenantId }: { tenantId: string }) {
  const [notifications, setNotifications] = useState<any[]>([]);

  useNotifications(tenantId, (message) => {
    setNotifications((prev) => [message, ...prev]);
  });

  return (
    <div className="space-y-2">
      {notifications.map((notif, i) => (
        <div key={i} className="p-3 bg-blue-50 rounded border-l-4 border-blue-400">
          {notif.data.message}
        </div>
      ))}
    </div>
  );
}
```

## 注意事项

### 1. 🚨 连接池冲突：LISTEN 连接必须独立

**错误的做法** — 在共享连接池连接上执行 LISTEN：

```python
# ❌ 错误：连接池的连接不能同时用于 LISTEN 和普通查询
async with engine.begin() as conn:
    await conn.execute(text("LISTEN notifications"))
```

**正确的做法** — 使用独占连接：

```python
# 创建独立的监听连接
listen_engine = create_async_engine(DATABASE_URL, pool_size=1, max_overflow=0)

async def listen_for_notifications():
    async with listen_engine.connect() as conn:
        await conn.execute(text("LISTEN notifications"))
        await conn.commit()  # LISTEN 需要提交才能生效

        while True:
            # 等待通知（PostgreSQL 驱动自动接收）
            await conn.execute(text("SELECT 1"))  # 触发通知接收
            # 检查已接收的通知
            notifications = conn.dialect._notifies  # 异步驱动特有关键
            ...
```

### 2. 连接生命周期

- LISTEN 连接必须保持长连接，断开后订阅丢失
- 使用连接池时要 `pool_size=1`，防止连接被复用
- 建议结合 `asyncpg` 或 `psycopg` 的 notification callback 机制

### 3. 消息大小限制

- `pg_notify` 的 payload 最大 **8000 bytes**
- 超长消息需要先写入表，NOTIFY 只发送 ID 或摘要

### 4. 可靠性权衡

| 特性 | LISTEN/NOTIFY | Redis pub/sub | RabbitMQ |
|------|--------------|---------------|----------|
| 持久化 | ❌ | ❌ | ✅ |
| 事务一致 | ✅ | ❌ | ❌ |
| 基础设施 | 无额外 | 需 Redis | 需 RabbitMQ |
| 消息大小 | 8KB 限制 | 512MB | 无限制 |
| 适用场景 | 实时推送 | 通用 pub/sub | 可靠消息 |

### 5. 多租户通道命名规范

```python
# 按租户独立通道
channel = f"notifications:{tenant_id}"

# 或全局通道 + 消息内 tenant_id
channel = "notifications"  # payload 中携带 tenant_id
```

推荐第二种方式（全局通道），减少连接数，便于扩展。
