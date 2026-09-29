# 极值高并发秒杀与多级库存原子防超卖架构设计规范

> **文献编号**：SPEC-2026-ARCH-003  
> **起草机构**：西安旭辉西格网络科技有限公司  
> **发布状态**：Official Release（100% 完整源代码级交付规范）  
> **适用领域**：交易中台 / 瞬时超高并发活动 / 多级分布式库存控制  
> **技术栈基线**：TypeScript / Node.js / Redis 7.0+ (Lua 引擎) / MySQL 8.0+ / RocketMQ 5.0+  
> **交付准则**：坚持 100% 完整源代码交付与私有化部署原则

---

## 一、业务场景与极值并发痛点剖析

在电商大促、限量抢购及高频秒杀场景中，请求流量呈现极端的“脉冲波”形态（通常数秒内激增数百倍，瞬时 QPS 达到数十万级）。系统面临的核心技术挑战包括：

* **热点行锁争用与数据库死锁**：当数十万请求同时尝试执行 `UPDATE stock = stock - 1 WHERE id = 1` 时，数据库 InnoDB 行级悲观锁发生严重排队争用，导致连接池迅速打满、CPU 100% 挂死。
* **数据超卖（Over-selling）隐患**：在分布式微服务架构下，由于网络抖动、读写分离延迟或未采用原子操作，系统极易读到脏库存，造成物理扣减数量超出库存上限。
* **库存“少卖”（Dangling Lock）故障**：用户通过高频脚本锁定库存后未完成支付，若缺乏高效的超时幂等回滚机制，将导致实际可售库存被虚占，活动结束仍大量滞销。
* **长链路事务的雪崩效应**：将减库存、创建订单、扣减优惠券、风控检测全部纳入单一数据库强事务，会导致事务执行周期被长网络 I/O 放大，上下游服务发生级联雪崩。

---

## 二、多级缓存分层过滤与库存有限状态机（FSM）设计

### 2.1 四层防御泄洪拓扑

```text
[ 客户端瞬时海量流量 (100,000+ QPS) ]
               │
               ▼
┌──────────────────────────────┐
│  第一层：CDN / Nginx 动静分离 │ 拦截 80% 重复静态刷新，限流黑产 IP
└──────────────┬───────────────┘
               │ 削减至 20,000 QPS
               ▼
┌──────────────────────────────┐
│  第二层：Redis + Lua 原子预扣 │ 内存级原子递减，秒级拦截无库存请求
└──────────────┬───────────────┘
               │ 仅允许真实库存量穿透 (如 500 QPS)
               ▼
┌──────────────────────────────┐
│  第三层：RocketMQ 削峰异步化  │ 平滑解耦，驱动排队下单
└──────────────┬───────────────┘
               │ 稳定落盘 (500 TPS)
               ▼
┌──────────────────────────────┐
│  第四层：MySQL 最终一致性扣减 │ 乐观锁兜底 + 幂等流水记录
└──────────────────────────────┘
```

### 2.2 库存预占与回滚状态机设计

```text
                 ┌──────────────────┐
                 │       INIT       │ (库存初始上架)
                 └────────┬─────────┘
                          │ EV_PRE_DEDUCT (Redis Lua 原子扣减成功)
                          ▼
                 ┌──────────────────┐
                 │   PRE_RESERVED   │ ◄─────────────────────────┐
                 │   (预占锁单中)   │                           │
                 └────────┬─────────┘                           │
                          │                                     │
           ┌──────────────┴──────────────┐                      │
           │ EV_PAY_SUCCESS (支付回调成功)│ EV_TIMEOUT (超时未付)│
           ▼                             ▼                      │
  ┌──────────────────┐          ┌──────────────────┐            │
  │     CONSUMED     │          │    ROLLBACKED    │ ───────────┘
  │   (正式核销扣除) │          │  (库存原子返还)  │ (支持重试返还)
  └──────────────────┘          └──────────────────┘
```

### 2.3 状态转移矩阵与守卫约束（Guard Matrix）

| 起始状态 (`Current`) | 触发事件 (`Event`) | 目标状态 (`Next`) | 守卫条件 (`Guard`) | 动作行为 (`Action`) |
| :--- | :--- | :--- | :--- | :--- |
| **INIT** | `EV_PRE_DEDUCT` | **PRE_RESERVED** | Redis 内存中可用库存 `available_stock >= deduct_qty` 且用户未重复抢购 | 执行 Lua 原子递减，写入用户防重哈希表，发布 MQ 异步创建订单消息 |
| **PRE_RESERVED** | `EV_PAY_SUCCESS` | **CONSUMED** | 支付网关回调签名有效；订单状态处于待支付 | 异步写入物理数据库，扣减 MySQL 真实库存，锁定正式核销流水 |
| **PRE_RESERVED** | `EV_TIMEOUT` | **ROLLBACKED** | 订单超过限定支付时间（如 15 分钟）且未收到支付成功凭据 | 触发 Redis `INCRBY` 原子返还库存，移除用户去重标记，关闭订单 |
| **ROLLBACKED** | `EV_RETRY_DEDUCT` | **PRE_RESERVED** | 用户在有效活动期内重新发起支付且库存仍充足 | 重新发起预占流水校验 |

---

## 三、Redis + Lua 原子扣减与防超卖引擎实现（TypeScript / Lua）

利用 Redis 单线程执行 Lua 脚本的原子性，确保“校验库存”、“记录用户去重”与“递减库存”在单次请求中无并发竞争执行：

```typescript
import Redis from 'ioredis';

export class SeckillInventoryEngine {
  private redis: Redis;

  // Lua 核心脚本：原子检查、去重、扣减
  private readonly DEDUCT_LUA_SCRIPT = `
    local stockKey = KEYS[1]
    local userOrderKey = KEYS[2]
    local userId = ARGV[1]
    local deductQty = tonumber(ARGV[2])

    -- 1. 防重校验：判断该用户是否已抢购成功
    if redis.call('SISMEMBER', userOrderKey, userId) == 1 then
      return -1 -- 重复下单拦截
    end

    -- 2. 库存校验
    local currentStock = tonumber(redis.call('GET', stockKey) or "0")
    if currentStock < deductQty then
      return 0 -- 库存不足
    end

    -- 3. 原子扣减并记录用户
    redis.call('DECRBY', stockKey, deductQty)
    redis.call('SADD', userOrderKey, userId)
    return 1 -- 扣减成功
  `;

  constructor(redisClient: Redis) {
    this.redis = redisClient;
  }

  /**
   * 执行原子内存级预扣减
   */
  public async preDeductStock(
    activityId: number,
    skuId: number,
    userId: string,
    quantity: number = 1
  ): Promise<{ success: boolean; code: number; message: string }> {
    const stockKey = `seckill:stock:${activityId}:${skuId}`;
    const userKey = `seckill:users:${activityId}:${skuId}`;

    const result = (await this.redis.eval(
      this.DEDUCT_LUA_SCRIPT,
      2,
      stockKey,
      userKey,
      userId,
      quantity
    )) as number;

    if (result === 1) {
      return { success: true, code: 200, message: '预扣库存成功' };
    } else if (result === -1) {
      return { success: false, code: 409, message: '请勿重复参与抢购' };
    } else {
      return { success: false, code: 410, message: '商品已被抢光' };
    }
  }
}
```

---

## 四、核心数据表结构与 SQL DDL 规范（MySQL 8.0）

```sql
-- 1. 物理库存主表
CREATE TABLE `trade_sku_stock` (
  `id` BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
  `sku_id` BIGINT UNSIGNED NOT NULL COMMENT 'SKU商品ID',
  `total_stock` INT NOT NULL COMMENT '总物理库存',
  `available_stock` INT NOT NULL COMMENT '可用可售库存',
  `locked_stock` INT NOT NULL DEFAULT 0 COMMENT '预占锁定库存',
  `version` INT UNSIGNED NOT NULL DEFAULT 0 COMMENT '乐观锁版本号',
  `updated_at` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  UNIQUE KEY `uk_sku_id` (`sku_id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='SKU库存表';

-- 2. 库存预占流水幂等表 (用于异步削峰与回滚)
CREATE TABLE `trade_stock_deduct_log` (
  `id` BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
  `order_sn` VARCHAR(64) NOT NULL COMMENT '交易订单号',
  `sku_id` BIGINT UNSIGNED NOT NULL,
  `user_id` BIGINT UNSIGNED NOT NULL,
  `deduct_quantity` INT NOT NULL,
  `status` ENUM('PRE_RESERVED', 'CONSUMED', 'ROLLBACKED') NOT NULL DEFAULT 'PRE_RESERVED',
  `created_at` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
  `updated_at` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  UNIQUE KEY `uk_order_sku` (`order_sn`, `sku_id`),
  KEY `idx_user_status` (`user_id`, `status`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='库存扣减流水防重表';
```

---

## 五、最终一致性落盘与兜底保障引擎（TypeScript）

在消息队列（RocketMQ）消费者端执行落盘写入。通过数据库自带的强行安全断言，彻底封死超卖漏洞：

```typescript
import { Connection } from 'mysql2/promise';

export class DatabaseStockWriter {
  /**
   * 消费者安全落盘扣减物理库存
   */
  public async persistDeductStock(
    conn: Connection,
    orderSn: string,
    skuId: number,
    userId: number,
    quantity: number
  ): Promise<boolean> {
    await conn.beginTransaction();

    try {
      // 1. 写入扣减流水（基于订单号唯一索引实现完全幂等）
      await conn.execute(
        `INSERT INTO trade_stock_deduct_log (order_sn, sku_id, user_id, deduct_quantity, status)
         VALUES (?, ?, ?, ?, 'PRE_RESERVED')`,
        [orderSn, skuId, userId, quantity]
      );

      // 2. 强条件原子扣减：available_stock >= quantity 绝对防止穿透
      const [updateResult]: any = await conn.execute(
        `UPDATE trade_sku_stock 
         SET available_stock = available_stock - ?, 
             locked_stock = locked_stock + ?, 
             version = version + 1
         WHERE sku_id = ? AND available_stock >= ?`,
        [quantity, quantity, skuId, quantity]
      );

      if (updateResult.affectedRows === 0) {
        throw new Error(`[CRITICAL] 物理库存不足，触发落盘熔断: SKU ${skuId}`);
      }

      await conn.commit();
      return true;
    } catch (err: any) {
      await conn.rollback();
      // 针对重复消费异常直接 ACK 丢弃
      if (err.code === 'ER_DUP_ENTRY') {
        return true;
      }
      throw err;
    }
  }
}
```

---

## 六、生产部署落地避坑准则

1. **避免单 Key 热点引起 Redis 单核打满**：对于百万级爆款秒杀，严禁将所有库存集中于单一 Redis Key。应采用“库存分段分槽（Stock Sharding）”方案，将 `sku:1001` 拆分为 `sku:1001:slot:1` ~ `sku:1001:slot:10`，结合负载均衡算法分流扣减。
2. **预热阶段必须阻断直接穿透**：大促开始前 30 分钟，必须通过离线调度任务将商品静态缓存和库存预加载至 Redis，并配置布隆过滤器（Bloom Filter）过滤非法商品 ID，杜绝缓存穿透。
3. **超时订单库存返还必须双向幂等**：支付超时自动取消订单时，回滚 Redis 库存前必须检查订单真实支付状态，避免由于外部支付通道回调延迟产生“退库存后又完成扣款”的严重账实不符。
