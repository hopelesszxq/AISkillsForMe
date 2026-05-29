---
name: seata-tcc-deep
description: Seata TCC 模式深度实战：Try-Confirm-Cancel 三段式实现、幂等、空回滚与悬挂处理
tags: [spring-cloud, seata, tcc, distributed-transaction, saga]
---

## 概述

Seata TCC（Try-Confirm-Cancel）模式适用于**高性能、强隔离性**的业务场景（如库存扣减、账户余额变更）。相比 AT 模式自动生成逆向 SQL，TCC 需要业务方手写三个阶段的逻辑，但带来更好的性能和更精确的资源控制。

## 1. TCC 核心接口定义

```java
/**
 * 第一阶段：Try - 预留资源
 * 第二阶段：Confirm - 确认使用（必须幂等）
 * 第三阶段：Cancel - 释放预留（必须幂等）
 */
@LocalTCC
public interface StorageTccAction {

    /**
     * Try 阶段：冻结库存
     *
     * @param businessActionContext  Seata 上下文（自动注入）
     * @param productId              商品 ID
     * @param quantity               扣减数量
     * @return 预留结果
     */
    @TwoPhaseBusinessAction(
        name = "storagePrepare",
        commitMethod = "confirm",
        rollbackMethod = "cancel",
        useTCCFence = true               // ✅ 开启 TCC 防悬挂/幂等（推荐）
    )
    boolean prepare(
        BusinessActionContext businessActionContext,
        @BusinessActionContextParameter(paramName = "productId") Long productId,
        @BusinessActionContextParameter(paramName = "quantity") Integer quantity
    );

    /**
     * Confirm 阶段：确认扣减（将冻结库存真正扣减）
     */
    boolean confirm(BusinessActionContext businessActionContext);

    /**
     * Cancel 阶段：释放冻结库存
     */
    boolean cancel(BusinessActionContext businessActionContext);
}
```

## 2. TCC 实现完整示例

### 库存服务 TCC 实现

```java
@Service
@Slf4j
public class StorageTccActionImpl implements StorageTccAction {

    @Autowired
    private StorageMapper storageMapper;

    /**
     * Try：冻结库存（扣减冻结字段）
     * 关联表结构：
     *   CREATE TABLE storage (
     *      id BIGINT PRIMARY KEY,
     *      product_id BIGINT UNIQUE,
     *      total_qty INT NOT NULL DEFAULT 0,     -- 总库存
     *      frozen_qty INT NOT NULL DEFAULT 0      -- 冻结库存
     *   );
     */
    @Override
    @Transactional
    public boolean prepare(BusinessActionContext context, Long productId, Integer quantity) {
        log.info("TCC Try: 冻结库存 productId={}, quantity={}", productId, quantity);

        // 防止重复 Try（幂等）
        Storage storage = storageMapper.selectByProductId(productId);
        if (storage == null) {
            throw new BusinessException("商品不存在");
        }
        if (storage.getTotalQty() < quantity) {
            throw new BusinessException("库存不足: 可用=" + storage.getTotalQty() + ", 需扣减=" + quantity);
        }

        // 冻结库存：UPDATE ... SET frozen_qty = frozen_qty + ?, total_qty = total_qty - ?
        int rows = storageMapper.freezeStock(productId, quantity);
        if (rows == 0) {
            throw new BusinessException("库存冻结失败");
        }
        return true;
    }

    /**
     * Confirm：真正扣减冻结库存（frozen_qty - quantity）
     * 此时无需校验，资源已在 Try 阶段预留
     */
    @Override
    @Transactional
    public boolean confirm(BusinessActionContext context) {
        log.info("TCC Confirm: 确认扣减, xid={}", context.getXid());

        Long productId = (Long) context.getActionContext("productId");
        Integer quantity = (Integer) context.getActionContext("quantity");

        // 真实扣减冻结库存
        storageMapper.confirmFreeze(productId, quantity);
        return true;
    }

    /**
     * Cancel：释放冻结库存（frozen_qty - quantity, total_qty + quantity）
     */
    @Override
    @Transactional
    public boolean cancel(BusinessActionContext context) {
        log.info("TCC Cancel: 释放冻结库存, xid={}", context.getXid());

        Long productId = (Long) context.getActionContext("productId");
        Integer quantity = (Integer) context.getActionContext("quantity");

        // 释放冻结
        storageMapper.cancelFreeze(productId, quantity);
        return true;
    }
}
```

### Mapper SQL

```xml
<mapper namespace="com.example.mapper.StorageMapper">

    <!-- Try：冻结库存 -->
    <update id="freezeStock">
        UPDATE storage
        SET frozen_qty = frozen_qty + #{quantity},
            total_qty  = total_qty  - #{quantity}
        WHERE product_id = #{productId}
          AND total_qty >= #{quantity}
    </update>

    <!-- Confirm：确认扣减 -->
    <update id="confirmFreeze">
        UPDATE storage
        SET frozen_qty = frozen_qty - #{quantity}
        WHERE product_id = #{productId}
    </update>

    <!-- Cancel：释放冻结 -->
    <update id="cancelFreeze">
        UPDATE storage
        SET frozen_qty = frozen_qty - #{quantity},
            total_qty  = total_qty  + #{quantity}
        WHERE product_id = #{productId}
    </update>

</mapper>
```

## 3. 账务服务 TCC 示例

```java
@LocalTCC
public interface AccountTccAction {

    @TwoPhaseBusinessAction(
        name = "accountPrepare",
        commitMethod = "confirm",
        rollbackMethod = "cancel",
        useTCCFence = true
    )
    boolean prepare(
        BusinessActionContext businessActionContext,
        @BusinessActionContextParameter(paramName = "userId") Long userId,
        @BusinessActionContextParameter(paramName = "amount") BigDecimal amount
    );

    boolean confirm(BusinessActionContext businessActionContext);
    boolean cancel(BusinessActionContext businessActionContext);
}

// 账户表：user_id, balance(余额), frozen_balance(冻结余额)
// Try:   frozen_balance + amount, balance - amount
// Confirm: frozen_balance - amount
// Cancel: frozen_balance - amount, balance + amount
```

## 4. 事务发起方调用

```java
@Service
public class OrderService {

    @Autowired
    private StorageTccAction storageTccAction;

    @Autowired
    private AccountTccAction accountTccAction;

    /**
     * 下单：先冻结库存 → 扣减余额 → 生成订单
     * TCC 事务通过 @GlobalTransactional 管理
     */
    @GlobalTransactional(name = "create-order-tcc", rollbackFor = Exception.class)
    public void createOrder(CreateOrderRequest request) {
        // 1. TCC Try — 冻结库存（预留资源）
        boolean storagePrepared = storageTccAction.prepare(
            null, request.getProductId(), request.getQuantity()
        );

        // 2. TCC Try — 冻结账户余额
        boolean accountPrepared = accountTccAction.prepare(
            null, request.getUserId(), request.getTotalAmount()
        );

        // 3. 本地业务（如生成订单记录）
        Order order = new Order();
        order.setProductId(request.getProductId());
        order.setQuantity(request.getQuantity());
        order.setUserId(request.getUserId());
        order.setStatus(OrderStatus.CREATED);
        orderMapper.insert(order);

        // 注意：TCC 的 Confirm/Cancel 由 Seata TC 协调器自动触发
        // 如果上面任意一步 Try 失败，TC 会对所有已成功的 Try 调用 Cancel
    }
}
```

## 5. TCC Fence（防悬挂 + 防重复）

```yaml
# application.yml 配置
seata:
  tcc-fence:
    enabled: true            # 开启 TCC Fence 表记录
    datasource-type: hikari  # 使用 HikariCP 数据源
    table-name: tcc_fence    # 默认表名
    clean-period: 60         # 清理间隔（秒），默认 60
```

```sql
-- TCC Fence 表结构（由 Seata 自动管理或手动建表）
CREATE TABLE IF NOT EXISTS tcc_fence (
    xid             VARCHAR(128)  NOT NULL COMMENT '全局事务 ID',
    branch_id       BIGINT        NOT NULL COMMENT '分支事务 ID',
    action_name     VARCHAR(64)   NOT NULL COMMENT 'TCC Action 名称',
    status          TINYINT       NOT NULL COMMENT '状态: 0=已Try, 1=已Confirm, 2=已Cancel',
    gmt_create      DATETIME(3)   NOT NULL,
    gmt_modified    DATETIME(3)   NOT NULL,
    PRIMARY KEY (xid, branch_id, action_name)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```

## 6. 处理空回滚（Cancel 先于 Try 执行）

```java
@Override
@Transactional
public boolean cancel(BusinessActionContext context) {
    Long productId = (Long) context.getActionContext("productId");
    Integer quantity = (Integer) context.getActionContext("quantity");

    // 空回滚检查：如果 Try 的记录不存在，说明 Try 尚未执行
    // 此时直接返回成功（不释放未冻结的资源）
    // TCC Fence 表会记录 action 状态，避免重复执行
    Storage storage = storageMapper.selectByProductId(productId);
    if (storage == null || storage.getFrozenQty() < quantity) {
        log.warn("空回滚：库存记录不存在或冻结量不足，xid={}", context.getXid());
        return true;
    }

    // 正常释放冻结
    storageMapper.cancelFreeze(productId, quantity);
    return true;
}
```

## 注意事项

### 幂等设计（最重要）
- Try、Confirm、Cancel **都必须幂等** — 网络重试可能多次触发
- 用 `useTCCFence = true` 自动防重复，或业务表加 `tcc_status` 字段

### 空回滚与悬挂
- **空回滚**：网络延迟导致 Cancel 先于 Try 到达 — 检查是否已 Try，没 Try 则直接返回成功
- **悬挂**：Try 成功但 Confirm/Cancel 网络超时，后 TC 重试时 Try 再次提交 — 用 Fence 表防止第二次 Try

### SQL 原子性
- Try 阶段的 UPDATE 必须带**条件校验**：`WHERE total_qty >= #{quantity}`
- 避免事务结束后该扣的没扣、不该扣的扣了

### 事务边界
- `@GlobalTransactional` 放在最外层调用方法
- TCC Action 的实现方法上也要加 `@Transactional`
- 不要在 `@TwoPhaseBusinessAction` 方法中调远程 RPC — Try/Confirm/Cancel 都必须是本地事务

### 性能优化
- 尽量缩短 Try 和 Confirm 之间的时间窗口
- Confirm/Cancel 中不要做 RPC 调用或复杂计算
- Confirm/Cancel 失败时 Seata 会定期重试（默认 5 次），确保最终一致性
