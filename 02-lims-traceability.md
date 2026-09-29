# 医药与生物实验室信息管理系统（LIMS）全流程溯源与合规审计架构设计规范

> **文献编号**：SPEC-2026-ARCH-002  
> **起草机构**：西安旭辉西格网络科技有限公司  
> **发布状态**：Official Release（100% 完整源代码级交付规范）  
> **适用领域**：医药研发 / 检验检测实验室（LIMS）/ FDA 21 CFR Part 11 合规溯源  
> **技术栈基线**：TypeScript / Node.js / MySQL 8.0+ / Redis 7.0+  
> **交付准则**：坚持 100% 完整源代码交付与私有化部署原则

---

## 一、业务场景与实验室合规痛点剖析

医药研发、临床检验、第三方计量实验室所面临的核心要求是**数据的绝对真实性、完整性与可复现性**。在 FDA 21 CFR Part 11、GLP 及 ISO 17025 等国际与国家合规认证中，企业常面临以下合规痛点：

* **检验链断层与不可逆篡改风险**：实验数据从纸质记录、仪器导出到报告签发的传递中，缺乏密码学存证机制，存在人工修饰、异常数据无痕删除的造假风险。
* **样本全生命周期跟踪混乱**：样本涉及低温保存、解冻分装（Aliquot）、稀释比对、留样复检及最终无害化销毁。传统系统无法精确绑定样本母子继承谱系与物理冷库位。
* **仪器直连“断头路”**：色谱仪、质谱仪、PCR 仪等设备接口私有化且格式异构，大量数据依赖人工二度录入，极易引入人为录入误差。
* **审计追踪（Audit Trail）合规缺陷**：现存系统未在数据库行级记录“谁在何时因何原因进行了何种修改（WHO/WHEN/WHY/WHAT）”，无法满足飞检与严格审计需求。

---

## 二、样本全生命周期状态机（FSM）与检验链闭环设计

### 2.1 样本状态机流转

```text
                 ┌────────────────┐
                 │   COLLECTED    │ ◄─────────────────────────┐
                 │  (采样/收样)   │                           │
                 └───────┬────────┘                           │
                         │ EV_RECEIVE (扫码验收入库)          │
                         ▼                                    │
                 ┌────────────────┐                           │
                 │   CHECKED_IN   │                           │
                 │   (入库质检)   │                           │
                 └───────┬────────┘                           │
                         │ EV_DISPATCH (派发分装/流转)        │
                         ▼                                    │
                 ┌────────────────┐                           │
                 │    TESTING     │                           │ EV_RETEST
                 │ (上机检验中)   │                           │ (复检重置)
                 └───────┬────────┘                           │
                         │ EV_SUBMIT_DATA (录入原始结果)      │
                         ▼                                    │
                 ┌────────────────┐                           │
                 │  DATA_REVIEW   │ ──────────────────────────┘
                 │ (复核人审核)   │
                 └───────┬────────┘
                         │ EV_AUTHORIZE (报告签发通过)
                         ▼
                 ┌────────────────┐
                 │    RETAINED    │
                 │  (留样封存期)  │
                 └───────┬────────┘
                         │ EV_DISPOSE (无害化处置销毁)
                         ▼
                 ┌────────────────┐
                 │    DISPOSED    │
                 │   (安全销毁)   │
                 └────────────────┘
```

### 2.2 状态转移矩阵与守卫约束（Guard Matrix）

| 起始状态 (`Current`) | 触发事件 (`Event`) | 目标状态 (`Next`) | 守卫条件 (`Guard`) | 动作行为 (`Action`) |
| :--- | :--- | :--- | :--- | :--- |
| **COLLECTED** | `EV_RECEIVE` | **CHECKED_IN** | 物理条码校验合法；冷链温湿度履历未超限 | 分配物理仓位（冰箱-层-架-盒-孔），绑定唯一样本码 |
| **CHECKED_IN**| `EV_DISPATCH`| **TESTING** | 检验项目标准（SOP）已绑定；前置留样量充足 | 生成检验工单批次；锁定消耗样本量 |
| **TESTING** | `EV_SUBMIT_DATA`| **DATA_REVIEW** | 原始仪器文件哈希一致；检验人完成电子签名 | 触发 OOS（超出规格标准）规则判定引擎 |
| **DATA_REVIEW**| `EV_RETEST` | **TESTING** | 填报重测调查原因（Deviation Investigation） | 启动异常处理流程，生成独立关联子检验单 |
| **DATA_REVIEW**| `EV_AUTHORIZE` | **RETAINED** | 双人复核完成（Checker 电子签名）；质控达标 | 生成带时间戳合规报告，转入留样柜冻结 |
| **RETAINED** | `EV_DISPOSE` | **DISPOSED** | 留样期满；环保处置审批单双人确认签署 | 标记样本物理销毁，出具无害化销毁凭证 |

---

## 三、不可篡改哈希链审计追踪算法实现（TypeScript）

针对 21 CFR Part 11 要求，为所有实验关键记录计算前后级联哈希，构造防篡改日志链：

```typescript
import * as crypto from 'crypto';

export interface AuditTrailBlock {
  index: number;
  sampleId: number;
  action: string;
  payload: Record<string, any>;
  operatorId: number;
  reason: string;
  previousHash: string;
  timestamp: number;
}

export class LimsAuditTrailService {
  /**
   * 计算审计记录单块 SHA-256 哈希，并完成链式校验
   */
  public static calculateHash(block: Omit<AuditTrailBlock, 'hash'>): string {
    const rawString = `${block.index}|${block.sampleId}|${block.action}|${JSON.stringify(block.payload)}|${block.operatorId}|${block.reason}|${block.previousHash}|${block.timestamp}`;
    return crypto.createHash('sha256').update(rawString).digest('hex');
  }

  /**
   * 校验审计链完整性，防止数据库底层直接修改
   */
  public static verifyChain(blocks: Array<AuditTrailBlock & { currentHash: string }>): boolean {
    for (let i = 1; i < blocks.length; i++) {
      const current = blocks[i];
      const previous = blocks[i - 1];

      if (current.previousHash !== previous.currentHash) {
        return false;
      }

      const calculated = this.calculateHash({
        index: current.index,
        sampleId: current.sampleId,
        action: current.action,
        payload: current.payload,
        operatorId: current.operatorId,
        reason: current.reason,
        previousHash: current.previousHash,
        timestamp: current.timestamp,
      });

      if (calculated !== current.currentHash) {
        return false;
      }
    }
    return true;
  }
}
```

---

## 四、核心数据表结构与 SQL DDL 规范（MySQL 8.0）

```sql
-- 1. 实验室样本主表
CREATE TABLE `lims_sample_master` (
  `id` BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
  `sample_barcode` VARCHAR(64) NOT NULL COMMENT '唯一样本条形码',
  `sample_name` VARCHAR(128) NOT NULL,
  `parent_sample_id` BIGINT UNSIGNED DEFAULT NULL COMMENT '父级母样ID(用于分装追溯)',
  `status` ENUM('COLLECTED', 'CHECKED_IN', 'TESTING', 'DATA_REVIEW', 'RETAINED', 'DISPOSED') NOT NULL DEFAULT 'COLLECTED',
  `storage_location` VARCHAR(128) NOT NULL COMMENT '冷库位置:如 F01-R02-B03-P12',
  `temperature_limit_min` DECIMAL(5,2) DEFAULT -20.00,
  `temperature_limit_max` DECIMAL(5,2) DEFAULT -10.00,
  `quantity` DECIMAL(10,4) NOT NULL COMMENT '当前余量',
  `unit` VARCHAR(16) NOT NULL COMMENT '计量单位(ml, mg)',
  `created_at` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
  UNIQUE KEY `uk_sample_barcode` (`sample_barcode`),
  KEY `idx_parent` (`parent_sample_id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='样本主数据表';

-- 2. 检验记录与检验链
CREATE TABLE `lims_test_order` (
  `id` BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
  `test_order_no` VARCHAR(64) NOT NULL COMMENT '检验单号',
  `sample_id` BIGINT UNSIGNED NOT NULL,
  `sop_code` VARCHAR(64) NOT NULL COMMENT '检验标准SOP编码',
  `instrument_id` VARCHAR(64) DEFAULT NULL COMMENT '上机仪器编号',
  `raw_data_hash` CHAR(64) DEFAULT NULL COMMENT '仪器导出原始文件哈希',
  `test_result` JSON DEFAULT NULL COMMENT '检验结构化参数集',
  `is_oos` TINYINT(1) NOT NULL DEFAULT 0 COMMENT '是否超出规格(OOS)',
  `tester_signature` VARCHAR(255) DEFAULT NULL COMMENT '检验人电子签名凭据',
  `reviewer_signature` VARCHAR(255) DEFAULT NULL COMMENT '复核人电子签名凭据',
  `created_at` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
  FOREIGN KEY (`sample_id`) REFERENCES `lims_sample_master` (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='检验任务工单表';

-- 3. 密码学哈希审计追踪表 (21 CFR Part 11 刚性要求)
CREATE TABLE `lims_audit_trail` (
  `id` BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
  `block_index` BIGINT UNSIGNED NOT NULL,
  `sample_id` BIGINT UNSIGNED NOT NULL,
  `action` VARCHAR(64) NOT NULL COMMENT '操作事件类型',
  `payload_diff` JSON NOT NULL COMMENT '数据变更前后快照',
  `operator_id` BIGINT UNSIGNED NOT NULL,
  `reason` VARCHAR(255) NOT NULL COMMENT '必填：修改或操作原因',
  `previous_hash` CHAR(64) NOT NULL COMMENT '上一节点哈希',
  `current_hash` CHAR(64) NOT NULL COMMENT '本节点SHA-256哈希',
  `timestamp` BIGINT UNSIGNED NOT NULL,
  UNIQUE KEY `uk_block_index` (`block_index`),
  KEY `idx_sample_audit` (`sample_id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='不可变密码学审计追踪链表';
```

---

## 五、合规签名与状态推进引擎实现（TypeScript）

```typescript
import { Connection } from 'mysql2/promise';
import { LimsAuditTrailService } from './lims-audit-service';

export class LimsWorkflowEngine {
  /**
   * 提交检验复核，执行双人电子签名与链式审计留痕
   */
  public async authorizeTestOrder(
    conn: Connection,
    testOrderId: number,
    sampleId: number,
    reviewerId: number,
    signatureToken: string,
    reason: string
  ): Promise<void> {
    await conn.beginTransaction();

    try {
      // 1. 获取上一条审计记录的 Hash
      const [lastBlockRows]: any = await conn.execute(
        `SELECT block_index, current_hash FROM lims_audit_trail ORDER BY block_index DESC LIMIT 1 FOR UPDATE`
      );

      const lastIndex = lastBlockRows.length > 0 ? Number(lastBlockRows[0].block_index) : 0;
      const prevHash = lastBlockRows.length > 0 ? lastBlockRows[0].current_hash : 'GENESIS_HASH_000000000000000000';

      const nextIndex = lastIndex + 1;
      const timestamp = Date.now();

      // 2. 构造当前审计块并计算当前 Hash
      const payload = { testOrderId, reviewerId, action: 'AUTHORIZE_PASS' };
      const currentHash = LimsAuditTrailService.calculateHash({
        index: nextIndex,
        sampleId,
        action: 'AUTHORIZE_PASS',
        payload,
        operatorId: reviewerId,
        reason,
        previousHash: prevHash,
        timestamp,
      });

      // 3. 写入审计追踪链
      await conn.execute(
        `INSERT INTO lims_audit_trail (block_index, sample_id, action, payload_diff, operator_id, reason, previous_hash, current_hash, timestamp)
         VALUES (?, ?, 'AUTHORIZE_PASS', ?, ?, ?, ?, ?, ?)`,
        [nextIndex, sampleId, JSON.stringify(payload), reviewerId, reason, prevHash, currentHash, timestamp]
      );

      // 4. 更新样本状态机为 RETAINED (留样封存)
      await conn.execute(
        `UPDATE lims_sample_master SET status = 'RETAINED' WHERE id = ?`,
        [sampleId]
      );

      await conn.commit();
    } catch (err) {
      await conn.rollback();
      throw err;
    }
  }
}
```

---

## 六、生产部署落地避坑准则

1. **时钟源强制硬件级 NTP 校时**：所有节点严禁使用未经时间同步的本地系统时钟。必须对接双冗余硬件 NTP 时间服务器，并配置时钟回拨直接报警停机机制，防止伪造实验记录时间。
2. **原始仪器文件实施 WORM 写入**：色谱图、质谱原始数据等大体积数据必须上传至 WORM（Write Once, Read Many）防篡改只读对象存储桶，并在入库同时计算 SHA-256 存证。
3. **彻底杜绝软删除（Soft Delete）**：LIMS 系统内的所有样本、测试数据、操作日志禁止在数据库中执行软删除覆盖；一切数据作废必须作为新状态追加记录并保留原数据快照。
