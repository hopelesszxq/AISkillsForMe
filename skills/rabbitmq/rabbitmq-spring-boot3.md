---
name: rabbitmq-spring-boot3
description: Spring Boot 3.x RabbitMQ 深度配置：重试、事务、消费者并发调优与测试
tags: [rabbitmq, spring-boot, retry, consumer, testing]
---

## Spring Boot 3.x RabbitMQ 深度配置

Spring Boot 3.x（3.2+）对 RabbitMQ 自动配置做了多项改进，包括虚拟线程支持、新的 Retry 模板、以及更灵活的消费者容器配置。

## 一、基础配置与连接工厂

```yaml
spring:
  rabbitmq:
    host: 192.168.1.100
    port: 5672
    username: admin
    password: ${RABBITMQ_PASSWORD}
    virtual-host: /order-service        # 按服务隔离 VHost
    # 连接池（CachingConnectionFactory 默认）
    cache:
      channel:
        size: 25                        # 连接保活的 channel 数
        checkout-timeout: 5000          # 获取 channel 超时
    # 连接重试（连接失败时重试）
    connection-timeout: 5000
    template:
      retry:
        enabled: true
        initial-interval: 1000ms
        max-attempts: 3
        multiplier: 2.0
    # 启用 Publisher Confirm（确保消息发送到 Exchange）
    publisher-confirm-type: CORRELATED  # NONE / SIMPLE / CORRELATED
    # 启用 Return Callback（Exchange→Queue 失败时回调）
    publisher-returns: true
```

```java
// 自定义连接工厂（需要 SSL/TLS 时）
@Bean
public CachingConnectionFactory connectionFactory() {
    CachingConnectionFactory factory = new CachingConnectionFactory();
    factory.setHost("rabbitmq.example.com");
    factory.setPort(5671);
    factory.setUsername("admin");
    factory.setPassword(password);

    // SSL 配置
    SslContextBuilder builder = SslContextBuilder.forClient()
        .trustManager(new File("/etc/ssl/certs/ca-cert.pem"))
        .keyManager(new File("/etc/ssl/certs/client-cert.pem"),
                    new File("/etc/ssl/certs/client-key.pem"));
    factory.getRabbitConnectionFactory().setSslContext(builder.build());

    // 虚拟线程支持（Spring Boot 3.2+）
    factory.setExecutor(Executors.newVirtualThreadPerTaskExecutor());

    return factory;
}
```

## 二、消息发送最佳实践

### 2.1 RabbitTemplate 配置

```java
@Configuration
@RequiredArgsConstructor
public class RabbitMqProducersConfig {

    private final RabbitTemplate rabbitTemplate;

    @PostConstruct
    public void init() {
        // 确认回调
        rabbitTemplate.setConfirmCallback((correlationData, ack, cause) -> {
            if (ack) {
                log.info("消息确认成功: {}", correlationData.getId());
            } else {
                log.error("消息确认失败: {}, cause={}", correlationData.getId(), cause);
                // 将消息写入失败表，定时重试
                saveToRetryTable(correlationData);
            }
        });

        // 消息无法路由的回调
        rabbitTemplate.setReturnsCallback(returned -> {
            log.warn("消息路由失败: exchange={}, routingKey={}, replyCode={}, replyText={}",
                returned.getExchange(), returned.getRoutingKey(),
                returned.getReplyCode(), returned.getReplyText());
        });

        rabbitTemplate.setMandatory(true);
    }

    // 发送消息
    public void sendOrderEvent(OrderEvent event) {
        CorrelationData cd = new CorrelationData(UUID.randomUUID().toString());
        cd.getFuture().addCallback(
            result -> log.debug("消息发送结果: {}", result.isAck()),
            ex -> log.error("消息发送异常", ex)
        );

        rabbitTemplate.convertAndSend(
            "order.exchange",
            "order.created",
            event,
            message -> {
                message.getMessageProperties().setMessageId(cd.getId());
                message.getMessageProperties().setContentType("application/json");
                message.getMessageProperties().setDeliveryMode(MessageDeliveryMode.PERSISTENT);
                return message;
            },
            cd
        );
    }
}
```

### 2.2 批量发送优化

```yaml
spring:
  rabbitmq:
    template:
      batching:
        enabled: true               # 启用批量发送
        batch-size: 100             # 每批最大消息数
        buffer-limit: 32768         # 缓冲字节数上限
        timeout: 10000              # 缓冲时间上限（毫秒）
```

```java
// 使用 BatchingRabbitTemplate
@Bean
public BatchingRabbitTemplate batchingRabbitTemplate(
        ConnectionFactory connectionFactory,
        @Value("${spring.rabbitmq.template.batching.batch-size:100}") int batchSize) {
    BatchingStrategy strategy = new SimpleBatchingStrategy(
        batchSize,                // 每批最大消息数
        32768,                    // 缓冲字节数
        10000                     // 超时(毫秒)
    );
    return new BatchingRabbitTemplate(connectionFactory, strategy);
}
```

## 三、消费者配置与并发调优

### 3.1 手动 ACK + 重试

```yaml
spring:
  rabbitmq:
    listener:
      simple:
        # 并发消费者数
        concurrency: 5                     # 最小消费者数
        max-concurrency: 10                # 最大消费者数（自动扩容）
        prefetch: 10                       # 每次获取的消息数（关键性能参数）
        # ACK 模式
        acknowledge-mode: MANUAL           # NONE(自动) / AUTO / MANUAL(手动)
        # 消费重试
        retry:
          enabled: true
          max-attempts: 3                  # 最大重试次数（含首次）
          initial-interval: 2000ms
          multiplier: 2.0
          max-interval: 30000ms
        # 消费者异常处理
        default-requeue-rejected: false    # 重试耗尽后不入队（走 DLQ）
        missing-queues-fatal: false        # 队列不存在时不阻塞应用启动
      # 并发消费者容器设置
      direct:
        consumers-per-queue: 5
```

### 3.2 Prefetch 调优指南

| Prefetch 值 | 适用场景 | 说明 |
|-------------|---------|------|
| 1 | 处理时间长的任务（>1s） | 每个消费者一次处理 1 个，负载均衡最均匀 |
| 10~30 | 通用微服务 | 平衡吞吐与公平性（推荐默认值） |
| 100~300 | 批量处理/IO密集型 | 高吞吐场景，但要注意内存溢出 |
| 0 | 无限制（不推荐） | 全部消息推送，消费者内存水位不可控 |

```java
// 3.3 手动 ACK 消费者
@RabbitListener(queues = "order.queue")
public void handleOrder(OrderEvent event, Channel channel,
                        @Header(AmqpHeaders.DELIVERY_TAG) long tag) {
    try {
        processOrder(event);
        // 处理成功 → 确认消息
        channel.basicAck(tag, false);
    } catch (RetryableException e) {
        // 可重试异常 → 重新入队
        channel.basicNack(tag, false, true);
    } catch (FatalException e) {
        // 不可重试异常 → 丢弃/发往 DLQ
        channel.basicNack(tag, false, false);
        saveDeadLetter(event, e);
    }
}
```

### 3.4 消费者声明式重试（推荐）

```java
@Configuration
public class RabbitMqRetryConfig {

    // 重试耗尽后，消息会被投递到 DLQ
    @Bean
    public MessageRecoverer messageRecoverer(RabbitTemplate rabbitTemplate) {
        // 重试 3 次失败后发给 DLQ
        return new RepublishMessageRecoverer(rabbitTemplate, "order.retry.exchange",
            "order.retry.routingKey");
    }

    @Bean
    public RetryOperationsInterceptor retryInterceptor(MessageRecoverer recoverer) {
        return RetryInterceptorBuilder.stateless()
            .maxAttempts(3)
            .backOffOptions(2000, 2.0, 30000)  // 2s, *2, max 30s
            .recoverer(recoverer)
            .build();
    }
}
```

## 四、优先级队列与惰性队列

### 4.1 优先级队列

```java
@Bean
public Queue priorityQueue() {
    Map<String, Object> args = new HashMap<>();
    args.put("x-max-priority", 10);        // 优先级范围 0-10
    return QueueBuilder.durable("order.priority.queue")
        .withArguments(args)
        .build();
}

// 发送时指定优先级
rabbitTemplate.convertAndSend("order.exchange", "order.created",
    event, msg -> {
        msg.getMessageProperties().setPriority(event.getPriority());  // 0-10
        return msg;
    });
```

### 4.2 惰性队列（Lazy Queues）

```yaml
# 在 4.3.x 中 Classic Queue v2（CQv2）已是默认
# CQv2 始终以惰性模式运行，所有消息直接写入磁盘
# 无需额外配置 x-queue-mode=lazy
```

| 特性 | CQv1（已移除） | CQv2（默认） |
|------|--------------|-------------|
| 消息存储 | 内存优先，超限刷盘 | 始终写入磁盘 |
| 内存占用 | 未定，可能 OOM | 低且可预测 |
| 性能 | 突发高 | 稳定 |
| 适用场景 | - | 全部场景 |

## 五、事务消息

```java
// RabbitMQ 事务（不推荐，大幅降低性能）
// 建议用 Publisher Confirm 替代事务
@Bean
public RabbitTransactionManager transactionManager(ConnectionFactory cf) {
    return new RabbitTransactionManager(cf);
}

// 使用事务
@Transactional("rabbitTransactionManager")
public void sendInTransaction(OrderEvent event) {
    rabbitTemplate.convertAndSend("order.exchange", "order.created", event);
}
```

## 六、消息转换器

```java
@Bean
public MessageConverter messageConverter() {
    // JSON 序列化（推荐）
    Jackson2JsonMessageConverter converter = new Jackson2JsonMessageConverter(objectMapper);

    // 添加类信息到消息头（防止反序列化失败）
    converter.setTypeResolver(DefaultJackson2JavaTypeMapper.INSTANCE);
    converter.getJavaTypeMapper().addTrustedPackages("com.example.order.event");

    // 消息 ID 生成
    converter.setCreateMessageIds(true);

    return converter;
}
```

## 七、测试最佳实践

### 7.1 单元测试（Mock）

```java
@SpringBootTest
@AutoConfigureMockMvc
class OrderEventPublisherTest {

    @MockitoBean
    private RabbitTemplate rabbitTemplate;

    @Autowired
    private OrderEventPublisher publisher;

    @Test
    void testSendOrderCreatedEvent() {
        OrderEvent event = new OrderEvent(1L, "CREATED");

        publisher.send(event);

        // 验证消息发送
        ArgumentCaptor<CorrelationData> captor = ArgumentCaptor.forClass(CorrelationData.class);
        verify(rabbitTemplate).convertAndSend(
            eq("order.exchange"),
            eq("order.created"),
            eq(event),
            captor.capture()
        );
        assertNotNull(captor.getValue().getId());
    }
}
```

### 7.2 集成测试（TestContainers）

```java
@SpringBootTest
@Testcontainers
class OrderEventConsumerIT {

    @Container
    static RabbitMQContainer rabbit = new RabbitMQContainer(
        RabbitMQContainer.DEFAULT_IMAGE_NAME.withTag("4.3.1")
    )
        .withExposedPorts(5672, 15672)
        .withVhost("test-vhost")
        .withUser("test", "test");

    @DynamicPropertySource
    static void properties(DynamicPropertyRegistry registry) {
        registry.add("spring.rabbitmq.host", rabbit::getHost);
        registry.add("spring.rabbitmq.port", () -> rabbit.getMappedPort(5672));
        registry.add("spring.rabbitmq.username", () -> "test");
        registry.add("spring.rabbitmq.password", () -> "test");
    }

    @Autowired
    private RabbitTemplate rabbitTemplate;

    @Test
    void testConsumeOrderEvent() {
        OrderEvent event = new OrderEvent(1L, "CREATED");
        rabbitTemplate.convertAndSend("order.exchange", "order.created", event);

        // 等待消费完成
        await().atMost(5, TimeUnit.SECONDS)
            .until(() -> orderProcessedCount.get() == 1);
    }
}
```

```xml
<!-- TestContainers 依赖 -->
<dependency>
    <groupId>org.testcontainers</groupId>
    <artifactId>testcontainers</artifactId>
    <scope>test</scope>
</dependency>
<dependency>
    <groupId>org.testcontainers</groupId>
    <artifactId>rabbitmq</artifactId>
    <scope>test</scope>
</dependency>
```

## 八、性能调优清单

```yaml
# 高吞吐场景推荐配置
spring:
  rabbitmq:
    listener:
      simple:
        concurrency: 10
        max-concurrency: 30
        prefetch: 50               # 提高 Prefetch 减少网络往返
        acknowledge-mode: AUTO     # 自动 ACK（可接受少量重复）
        retry:
          enabled: false           # 关闭重试（交给下游幂等）
    template:
      retry:
        enabled: true
        max-attempts: 3
    publisher-confirm-type: NONE   # 能接受丢失则关闭 Confirm
    cache:
      channel:
        size: 50
```

| 参数 | 低延迟场景 | 高吞吐场景 |
|------|-----------|-----------|
| Prefetch | 1~3 | 50~200 |
| Concurrency | 2~5 | 10~30 |
| Ack Mode | MANUAL | AUTO（可重入） |
| Publisher Confirm | CORRELATED | NONE |
| 消息持久化 | 是 | 按需 |

## 注意事项

1. **RabbitMQ 4.3 兼容性**：Spring Boot 3.x 的 `spring-rabbit` 已完全适配 CQv2 和 Khepri，无需额外配置
2. **虚拟线程支持**：Spring Boot 3.2+ 启用虚拟线程后，RabbitMQ 监听器自动运行在虚拟线程中，`concurrency` 建议设为较小值（线程上下文切换成本低）
3. **Prefetch 与重试的关系**：开启消费重试时，Prefetch 建议不超过 `max-attempts × 5`，否则重试期间积压的消息可能导致消费者内存溢出
4. **DLQ 监控**：所有队列务必配置 DLQ，并监控 DLQ 积压（`rabbitmq-dlq-count` 指标告警）
5. **消息幂等**：消费者必须实现幂等（通过消息 ID 或业务主键去重），因为 ACK 超时可能导致重复投递
