---
name: design-patterns-java
description: Java 常用设计模式实战（创建型、结构型、行为型）
tags: [patterns, java, design-patterns, oop]
---

## 创建型模式（Creational）

### 单例模式（Singleton）

```java
// ✅ 枚举实现（最安全，防反射/序列化）
public enum DataSourceManager {
    INSTANCE;

    private final HikariDataSource dataSource;

    DataSourceManager() {
        HikariConfig config = new HikariConfig();
        config.setJdbcUrl("jdbc:postgresql://localhost:5432/db");
        config.setMaximumPoolSize(10);
        this.dataSource = new HikariDataSource(config);
    }

    public Connection getConnection() throws SQLException {
        return dataSource.getConnection();
    }
}

// ✅ 静态内部类（懒加载，线程安全）
public class ConfigHolder {
    private ConfigHolder() {}

    private static class Holder {
        static final ConfigHolder INSTANCE = new ConfigHolder();
    }

    public static ConfigHolder getInstance() {
        return Holder.INSTANCE;
    }
}
```

### 工厂模式（Factory Method）

```java
// 产品接口
public interface PaymentService {
    boolean pay(BigDecimal amount);
}

// 具体产品
public class WechatPay implements PaymentService {
    @Override
    public boolean pay(BigDecimal amount) {
        // 微信支付逻辑
        return true;
    }
}

public class AlipayPay implements PaymentService {
    @Override
    public boolean pay(BigDecimal amount) {
        // 支付宝支付逻辑
        return true;
    }
}

// 工厂类
public class PaymentFactory {
    public static PaymentService create(String type) {
        return switch (type) {
            case "wechat" -> new WechatPay();
            case "alipay" -> new AlipayPay();
            default -> throw new IllegalArgumentException("未知支付类型: " + type);
        };
    }
}
```

### 建造者模式（Builder）

```java
// 链式调用，适合多参对象构建
public class SearchRequest {
    private final String keyword;
    private final Integer page;
    private final Integer size;
    private final String sortBy;
    private final Boolean asc;
    private final List<Long> categoryIds;

    private SearchRequest(Builder builder) {
        this.keyword = builder.keyword;
        this.page = builder.page;
        this.size = builder.size;
        this.sortBy = builder.sortBy;
        this.asc = builder.asc;
        this.categoryIds = builder.categoryIds;
    }

    public static Builder builder() {
        return new Builder();
    }

    public static class Builder {
        private String keyword;
        private Integer page = 1;
        private Integer size = 20;
        private String sortBy = "createTime";
        private Boolean asc = false;
        private List<Long> categoryIds = Collections.emptyList();

        public Builder keyword(String keyword) {
            this.keyword = keyword; return this;
        }
        public Builder page(Integer page) {
            this.page = page; return this;
        }
        public Builder size(Integer size) {
            this.size = size; return this;
        }
        public SearchRequest build() {
            if (keyword == null || keyword.isBlank()) {
                throw new IllegalArgumentException("keyword 不能为空");
            }
            return new SearchRequest(this);
        }
    }
}

// 使用
SearchRequest request = SearchRequest.builder()
    .keyword("手机")
    .page(1)
    .size(10)
    .build();
```

## 结构型模式（Structural）

### 适配器模式（Adapter）

```java
// 旧接口
public class OldSmsService {
    public boolean sendSms(String phone, String msg) {
        System.out.println("旧短信: " + phone + " - " + msg);
        return true;
    }
}

// 新接口（目标）
public interface NewNotificationService {
    void send(String target, String title, String content);
}

// 适配器
public class SmsAdapter implements NewNotificationService {
    private final OldSmsService oldService;

    public SmsAdapter(OldSmsService oldService) {
        this.oldService = oldService;
    }

    @Override
    public void send(String target, String title, String content) {
        String msg = title + ": " + content;
        oldService.sendSms(target, msg);
    }
}

// 使用
NewNotificationService service = new SmsAdapter(new OldSmsService());
service.send("13800138000", "验证码", "您的验证码是 123456");
```

### 装饰器模式（Decorator）

```java
// 基础组件
public interface DataSource {
    String read();
    void write(String data);
}

public class FileDataSource implements DataSource {
    private final String path;

    public FileDataSource(String path) { this.path = path; }

    @Override
    public String read() {
        return Files.readString(Path.of(path));
    }

    @Override
    public void write(String data) {
        // 写入文件
    }
}

// 装饰器基类
public abstract class DataSourceDecorator implements DataSource {
    protected final DataSource wrappee;

    public DataSourceDecorator(DataSource wrappee) {
        this.wrappee = wrappee;
    }

    @Override
    public String read() { return wrappee.read(); }

    @Override
    public void write(String data) { wrappee.write(data); }
}

// 具体装饰器
public class EncryptDecorator extends DataSourceDecorator {
    public EncryptDecorator(DataSource wrappee) {
        super(wrappee);
    }

    @Override
    public void write(String data) {
        String encrypted = Base64.getEncoder().encodeToString(data.getBytes());
        wrappee.write(encrypted);
    }

    @Override
    public String read() {
        String encrypted = wrappee.read();
        return new String(Base64.getDecoder().decode(encrypted));
    }
}

public class CompressDecorator extends DataSourceDecorator {
    public CompressDecorator(DataSource wrappee) {
        super(wrappee);
    }

    @Override
    public void write(String data) {
        // 压缩后写入
        wrappee.write(compress(data));
    }

    @Override
    public String read() {
        return decompress(wrappee.read());
    }
}

// 使用：按需组合
DataSource source = new FileDataSource("/tmp/data.txt");
source = new CompressDecorator(source);
source = new EncryptDecorator(source);
source.write("敏感数据内容");
```

### 代理模式（Proxy）

```java
// Spring AOP 本质就是动态代理
// JDK 动态代理（需要接口）
public class MetricsProxy {
    @SuppressWarnings("unchecked")
    public static <T> T create(T target, Class<T> interfaceType) {
        return (T) Proxy.newProxyInstance(
            target.getClass().getClassLoader(),
            new Class[]{interfaceType},
            (proxy, method, args) -> {
                long start = System.nanoTime();
                try {
                    return method.invoke(target, args);
                } finally {
                    long cost = (System.nanoTime() - start) / 1_000_000;
                    System.out.printf("[METRICS] %s.%s took %dms%n",
                        interfaceType.getSimpleName(), method.getName(), cost);
                }
            }
        );
    }
}

// 使用
UserService proxy = MetricsProxy.create(new UserServiceImpl(), UserService.class);
proxy.getUser(1L);
```

## 行为型模式（Behavioral）

### 策略模式（Strategy）

```java
// 策略接口
@FunctionalInterface
public interface DiscountStrategy {
    BigDecimal apply(BigDecimal amount);

    // 工厂方法
    static DiscountStrategy of(String type) {
        return switch (type) {
            case "VIP" -> amount -> amount.multiply(new BigDecimal("0.8"));
            case "NEW_USER" -> amount -> amount.multiply(new BigDecimal("0.9"));
            case "BLACK_FRIDAY" -> amount -> amount.multiply(new BigDecimal("0.5"));
            default -> amount -> amount; // 无折扣
        };
    }
}

public class OrderService {
    public BigDecimal calculate(String orderType, BigDecimal amount) {
        return DiscountStrategy.of(orderType).apply(amount);
    }
}
```

### 观察者模式（Observer）

```java
// Spring 事件驱动完美体现观察者模式
// 事件
public class OrderCreatedEvent extends ApplicationEvent {
    private final Long orderId;

    public OrderCreatedEvent(Object source, Long orderId) {
        super(source);
        this.orderId = orderId;
    }

    public Long getOrderId() { return orderId; }
}

// 事件发布
@Service
public class OrderService {
    @Autowired
    private ApplicationEventPublisher publisher;

    public void createOrder(Order order) {
        // ... 保存订单 ...
        publisher.publishEvent(new OrderCreatedEvent(this, order.getId()));
    }
}

// 监听者 1：发送通知
@Component
public class NotificationListener {
    @EventListener
    public void handleOrderCreated(OrderCreatedEvent event) {
        // 发送短信/邮件通知
    }
}

// 监听者 2：更新库存
@Component
public class InventoryListener {
    @EventListener(condition = "#event.orderId > 0")
    public void deductStock(OrderCreatedEvent event) {
        // 扣减库存
    }
}
```

### 模板方法模式（Template Method）

```java
public abstract class AbstractDataExportor {

    // 模板方法：定义算法骨架
    public final File export(ExportRequest request) {
        List<?> data = queryData(request);       // 子类实现
        File tempFile = createTempFile();         // 公用
        try (OutputStream os = new FileOutputStream(tempFile)) {
            writeHeader(os);                      // 子类实现
            for (Object row : data) {
                writeRow(os, row);                // 子类实现
            }
            writeFooter(os);                      // 可选覆盖
        }
        return postProcess(tempFile);             // 公用
    }

    protected abstract List<?> queryData(ExportRequest request);
    protected abstract void writeHeader(OutputStream os) throws IOException;
    protected abstract void writeRow(OutputStream os, Object row) throws IOException;

    protected void writeFooter(OutputStream os) throws IOException {
        // 默认空实现，子类可选覆盖
    }

    private File createTempFile() { ... }
    private File postProcess(File file) { ... }
}

// 具体实现
@Component
public class OrderCsvExportor extends AbstractDataExportor {
    @Override
    protected List<?> queryData(ExportRequest request) {
        return orderMapper.selectList(...);
    }

    @Override
    protected void writeHeader(OutputStream os) {
        os.write("订单号,金额,状态,时间".getBytes(StandardCharsets.UTF_8));
    }

    @Override
    protected void writeRow(OutputStream os, Object row) {
        Order order = (Order) row;
        os.write(String.format("%s,%.2f,%s,%s%n",
            order.getOrderNo(), order.getAmount(),
            order.getStatus(), order.getCreateTime()).getBytes());
    }
}
```

## 实际项目中的模式组合

```java
// 工厂 + 策略 + 单例：支付引擎
@Component
public class PaymentEngine {
    private final Map<String, PaymentService> strategyMap;

    public PaymentEngine(List<PaymentService> services) {
        this.strategyMap = services.stream()
            .collect(Collectors.toMap(
                s -> s.getClass().getAnnotation(PaymentType.class).value(),
                Function.identity()
            ));
    }

    public PaymentResult pay(String type, PaymentRequest req) {
        PaymentService service = strategyMap.get(type);
        if (service == null) {
            throw new UnsupportedOperationException("不支持的支付方式: " + type);
        }
        return service.pay(req);
    }
}

@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
public @interface PaymentType {
    String value();
}
```

## 注意事项

1. **不要过度设计**：YAGNI（You Ain't Gonna Need It）原则，模式不是越多越好
2. **Singleton 慎用**：多线程环境确保线程安全，Spring Bean 默认就是单例
3. **工厂 vs 建造者**：对象参数<5个用工厂，参数多或有必填校验用建造者
4. **静态代理 vs 动态代理**：静态代理对每个类都写代理类 → 用 Spring AOP/动态代理
5. **策略模式的 Spring 实现**：通过 `@Autowired List<Strategy>` 自动注入所有实现，替代手工 map
6. **观察者避免循环依赖**：事件监听不要互相触发，防止栈溢出
7. **装饰器注意 equals/hashCode**：多层包装后比较要谨慎
