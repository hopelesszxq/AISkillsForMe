---
name: event-sourcing-pattern
description: Event Sourcing 事件溯源模式详解：事件存储、聚合重建、快照策略与 Spring Boot 整合实现
tags: [patterns, event-sourcing, architecture, java, spring-boot, axon]
---

## 概述

Event Sourcing（事件溯源）是一种**将状态变更以事件序列存储**的架构模式。不同于传统 CRUD 只保存最新状态，Event Sourcing 记录每个状态变更事件，当前状态由事件序列**重放（Replay）**得到。

### 核心特性

| 特性 | 说明 |
|------|------|
| **完整审计** | 所有变更都有记录，天然满足审计需求 |
| **时间旅行** | 可恢复到任意历史时间点的状态 |
| **事件驱动** | 事件可发布给其他消费者（CQRS 读模型、数据分析） |
| **最终一致性** | 写模型（事件存储）和读模型（投影）分离 |

### 适用场景

- 金融交易系统（需要完整审计日志）
- 订单/购物车系统（需要状态回溯）
- 协作编辑系统（如 Google Docs 操作日志）
- 合规要求严格的业务系统

## 架构概览

```
  Write Side (Command)          Read Side (Query)
  ====================          ===================
  Command → Command Handler     Query → Query Handler
                ↓                            ↑
          Aggregate (状态)              Projection (投影)
                ↓                            ↑
          Event Store ──── Pub/Sub ────> Read DB
          (事件存储)         (事件发布)    (关系型/NoSQL)
```

## 基础实现

### 1. 事件定义

```java
// 基础事件接口
public interface DomainEvent {
    String getAggregateId();
    long getVersion();
    Instant getOccurredAt();
}

// 账户相关事件
@Value
public class AccountOpenedEvent implements DomainEvent {
    String aggregateId;
    long version;
    Instant occurredAt;
    String accountHolder;
    Money initialDeposit;
}

@Value
public class MoneyDepositedEvent implements DomainEvent {
    String aggregateId;
    long version;
    Instant occurredAt;
    Money amount;
    String reference;
}

@Value
public class MoneyWithdrawnEvent implements DomainEvent {
    String aggregateId;
    long version;
    Instant occurredAt;
    Money amount;
    String reference;
}

@Value
public class AccountClosedEvent implements DomainEvent {
    String aggregateId;
    long version;
    Instant occurredAt;
    String reason;
}
```

### 2. 聚合根（Aggregate Root）

```java
public class BankAccount {
    private String id;
    private String accountHolder;
    private Money balance;
    private boolean active;
    private long version;

    // 用于事件重放的私有构造
    private BankAccount() {}

    // 工厂方法：开户 → 产生 AccountOpenedEvent
    public static BankAccount open(String id, String holder, Money initialDeposit) {
        if (initialDeposit.isNegativeOrZero()) {
            throw new IllegalArgumentException("开户金额必须为正数");
        }
        BankAccount account = new BankAccount();
        account.applyEvent(new AccountOpenedEvent(
            id, 1, Instant.now(), holder, initialDeposit));
        return account;
    }

    // 命令方法：存款
    public void deposit(Money amount, String reference) {
        if (!active) throw new IllegalStateException("账户已关闭");
        if (amount.isNegativeOrZero()) throw new IllegalArgumentException("存款金额必须为正数");
        applyEvent(new MoneyDepositedEvent(
            id, version + 1, Instant.now(), amount, reference));
    }

    // 命令方法：取款
    public void withdraw(Money amount, String reference) {
        if (!active) throw new IllegalStateException("账户已关闭");
        if (amount.isNegativeOrZero()) throw new IllegalArgumentException("取款金额必须为正数");
        if (balance.compareTo(amount) < 0) throw new InsufficientBalanceException();
        applyEvent(new MoneyWithdrawnEvent(
            id, version + 1, Instant.now(), amount, reference));
    }

    // 事件应用（同时更新状态）
    private void applyEvent(DomainEvent event) {
        if (event instanceof AccountOpenedEvent e) {
            this.id = e.getAggregateId();
            this.accountHolder = e.getAccountHolder();
            this.balance = e.getInitialDeposit();
            this.active = true;
            this.version = e.getVersion();
        } else if (event instanceof MoneyDepositedEvent e) {
            this.balance = this.balance.add(e.getAmount());
            this.version = e.getVersion();
        } else if (event instanceof MoneyWithdrawnEvent e) {
            this.balance = this.balance.subtract(e.getAmount());
            this.version = e.getVersion();
        } else if (event instanceof AccountClosedEvent e) {
            this.active = false;
            this.version = e.getVersion();
        }
    }

    // 批量重放事件（从事件存储加载）
    public static BankAccount replay(List<DomainEvent> events) {
        BankAccount account = new BankAccount();
        events.forEach(account::applyEvent);
        return account;
    }

    public List<DomainEvent> getUncommittedEvents() {
        // 返回未提交的事件列表（工作单元模式）
        return uncommittedEvents;
    }
}
```

### 3. 事件存储（Event Store）

```java
@Component
public class JdbcEventStore {

    private final JdbcTemplate jdbcTemplate;
    private final ObjectMapper objectMapper;

    public void saveEvents(String aggregateId,
            List<DomainEvent> events, long expectedVersion) {
        // 乐观锁：确保版本号连续
        events.forEach(event -> {
            jdbcTemplate.update("""
                INSERT INTO domain_events
                    (aggregate_id, version, event_type, event_data, occurred_at)
                VALUES (?, ?, ?, ?, ?)
                """,
                aggregateId,
                event.getVersion(),
                event.getClass().getTypeName(),
                toJson(event),
                event.getOccurredAt()
            );
        });
    }

    public List<DomainEvent> loadEvents(String aggregateId) {
        return jdbcTemplate.query("""
            SELECT * FROM domain_events
            WHERE aggregate_id = ?
            ORDER BY version ASC
            """,
            new EventRowMapper(),
            aggregateId
        );
    }

    private String toJson(DomainEvent event) {
        try {
            return objectMapper.writeValueAsString(event);
        } catch (Exception e) {
            throw new RuntimeException("事件序列化失败", e);
        }
    }
}
```

### 4. 事件表 DDL

```sql
CREATE TABLE domain_events (
    id            BIGSERIAL PRIMARY KEY,
    aggregate_id  VARCHAR(64)    NOT NULL,
    version       BIGINT         NOT NULL,
    event_type    VARCHAR(255)   NOT NULL,  -- 完全限定类名
    event_data    JSONB          NOT NULL,  -- 事件 JSON 载荷
    occurred_at   TIMESTAMPTZ    NOT NULL,
    metadata      JSONB,                     -- 可选：traceId, userId 等

    UNIQUE (aggregate_id, version)           -- 版本号唯一（乐观锁）
);

CREATE INDEX idx_events_aggregate ON domain_events (aggregate_id, version);
CREATE INDEX idx_events_type_time  ON domain_events (event_type, occurred_at);
```

## 快照策略

当事件数量达到数十万时，每次从零重放效率太低。快照周期性地保存聚合状态快照。

### 快照实现

```java
@Component
public class SnapshotManager {

    private static final int SNAPSHOT_INTERVAL = 100; // 每100个事件拍一次快照

    public Optional<Snapshot> loadSnapshot(String aggregateId) {
        return jdbcTemplate.query("""
            SELECT * FROM event_snapshots
            WHERE aggregate_id = ?
            ORDER BY version DESC LIMIT 1
            """,
            new SnapshotRowMapper(),
            aggregateId
        ).stream().findFirst();
    }

    public void saveSnapshot(BankAccount account) {
        if (account.getVersion() % SNAPSHOT_INTERVAL == 0) {
            jdbcTemplate.update("""
                INSERT INTO event_snapshots
                    (aggregate_id, version, snapshot_data, created_at)
                VALUES (?, ?, ?, NOW())
                ON CONFLICT (aggregate_id, version) DO NOTHING
                """,
                account.getId(),
                account.getVersion(),
                toSnapshotJson(account)
            );
        }
    }

    public BankAccount loadWithSnapshot(String aggregateId) {
        // 从快照 + 增量事件重建
        var snapshot = loadSnapshot(aggregateId);
        List<DomainEvent> events;
        long fromVersion;

        if (snapshot.isPresent()) {
            fromVersion = snapshot.get().getVersion();
            events = eventStore.loadEvents(aggregateId, fromVersion + 1);
            BankAccount account = fromSnapshot(snapshot.get());
            events.forEach(account::applyEvent);
            return account;
        } else {
            events = eventStore.loadEvents(aggregateId);
            return BankAccount.replay(events);
        }
    }
}
```

```sql
CREATE TABLE event_snapshots (
    aggregate_id  VARCHAR(64) NOT NULL,
    version       BIGINT      NOT NULL,
    snapshot_data JSONB       NOT NULL,
    created_at    TIMESTAMPTZ NOT NULL DEFAULT NOW(),

    UNIQUE (aggregate_id, version)
);
```

## Spring Boot 整合

### 事件发布与监听

```java
// 使用 Spring ApplicationEventPublisher 发布事件
@Component
public class EventPublisher {

    private final ApplicationEventPublisher publisher;

    public void publish(List<DomainEvent> events) {
        events.forEach(publisher::publishEvent);
    }
}

// 监听事件更新读模型
@Component
public class AccountProjection {

    @EventListener
    public void on(MoneyDepositedEvent event) {
        jdbcTemplate.update("""
            UPDATE account_summary
            SET balance = balance + ?, last_updated = ?
            WHERE account_id = ?
            """,
            event.getAmount().getValue(),
            event.getOccurredAt(),
            event.getAggregateId()
        );
    }

    @EventListener
    public void on(MoneyWithdrawnEvent event) {
        jdbcTemplate.update("""
            UPDATE account_summary
            SET balance = balance - ?, last_updated = ?
            WHERE account_id = ?
            """,
            event.getAmount().getValue(),
            event.getOccurredAt(),
            event.getAggregateId()
        );
    }
}
```

### 使用 Axon Framework（成熟框架方案）

```java
// Axon 聚合定义
@Aggregate
public class BankAccountAxon {

    @AggregateIdentifier
    private String id;
    private Money balance;

    @CommandHandler
    public BankAccountAxon(OpenAccountCommand cmd) {
        apply(new AccountOpenedEvent(
            cmd.getId(), cmd.getHolder(), cmd.getInitialDeposit()));
    }

    @CommandHandler
    public void handle(DepositMoneyCommand cmd) {
        apply(new MoneyDepositedEvent(id, cmd.getAmount()));
    }

    @EventSourcingHandler
    public void on(AccountOpenedEvent event) {
        this.id = event.getAggregateId();
        this.balance = event.getInitialDeposit();
    }

    @EventSourcingHandler
    public void on(MoneyDepositedEvent event) {
        this.balance = this.balance.add(event.getAmount());
    }
}
```

## 与 CQRS 的协同

Event Sourcing 和 CQRS 是**天然搭档**：

| | Event Sourcing + CQRS | 仅有 Event Sourcing |
|---|---|---|
| **写模型** | 事件存储 | 事件存储 |
| **读模型** | 独立投影数据库（SQL/ES/Redis） | 从事件重放（性能差） |
| **查询性能** | 优（独立优化读模型） | 差（每次都要重放） |
| **复杂度** | 较高 | 中等 |

## 注意事项

1. **事件版本管理**：事件 schema 会演进，需设计向后兼容的事件版本策略
   ```java
   // 版本化事件
   public class MoneyDepositedV2Event extends MoneyDepositedEvent {
       String currency;  // 新增字段，带默认值
       @JsonCreator
       public MoneyDepositedV2Event(@JsonProperty("aggregateId") ...,
               @JsonProperty("currency") String currency) {
           // 兼容旧版本：默认 USD
       }
   }
   ```

2. **存储膨胀**：事件无限增长，需要设置事件保留策略（如归档到冷存储）
3. **最终一致性**：读模型可能有延迟，需在 API 层面告知用户
4. **命令验证**：命令进入时需要验证当前状态（读模型可能有延迟，建议从事件存储做验证）
5. **重放性能**：对历史事件重放做优化 - 使用快照 + 并行投影重建
6. **事件不可变**：已存储的事件**永远不能修改**，错误通过补偿事件（compensating event）修正
7. **跨服务事件**：多个服务共享事件时，使用事件总线（Kafka/RabbitMQ）分发，避免直接耦合
8. **分布式追踪**：在事件 metadata 中携带 traceId，便于链路追踪
