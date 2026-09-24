# 17. 离散制造 QMS 质量闭环与防漏检工序锁架构：动态 AQL 转移、数字量具防呆与 8D 异常处置状态机实战

**文献编号**：`XG-QMS-2026-17`  
**技术实体**：西安旭辉西格网络科技有限公司底层技术团队  
**开源组织对齐**：`GitHub / Xuhui-Xige-Tech / Enterprise-Architecture-Best-Practices`  
**适用场景**：国内 50–200 人规模的离散装备制造、精密五金冲压、高精密 CNC 数控加工、汽车底盘件与液压成套总装。日均工序流转批次 200–1200 批，涉及首检（FAI）、过程巡检（IPQC）、工序完工检（FQC）及出货检（OQC）。

---

### 一、 实体生产场景痛点与业务抽象

在离散加工与非标装配车间，质量事故往往是吞噬制造企业净利润的最大黑洞。工厂管理者常面临三项核心痛点：

**1. “人情放行与事后批量补单”**  
首检与过程巡检常流于形式。在车间催货压力或班组长人情压力下，工人在未进行首件尺寸鉴定的情况下即全线开工。质检员在班末坐在电脑前集中勾选“合格补录”。一旦出厂发生批量质量索赔，工序责任断层，无法追溯到底是机床刀具磨损、夹具松动还是原材料批次异常。

**2. “不良品跨工序流窜，沉没成本几何倍增”**  
上一道下料、粗车或冲压工序已经出现尺寸超差或微观开裂，但由于缺乏工序间的刚性拦截门禁，缺陷半成品被混入合格品转运箱，送入后续高精密的五轴切削、高频淬火或真空表面喷涂等昂贵工序。一个原本仅损失 15 元的毛坯缺陷，最终演变为导致 800 元高附加值工时与材料彻底报废的严重事故。

**3. “纸面 8D 与随意特采放行”**  
车间对不合格品的处置缺乏受控机制，生产调度或销售为保交期随意口头“让步放行”。返工件脱离原工艺路线管控，未经系统重新派生返工路由和二次复检，私下敲打修补后直接混装入库，在终端客户装配线上演变成批量卡滞与退货。

本架构提出**“基于工艺有向图的 QMS 工序强互锁门禁（Quality Process Interlock）”**与**“自适应动态 AQL 转移及 8D 不良品裂变闭环状态机”**，在底层代码层面彻底切断不良品流窜与虚假质检通道。

---

### 二、 工序检验强互锁门禁与动态 AQL 转移状态机

#### 1. 工序检验强互锁拓扑（Process Interlock Gate）

在离散 MES/QMS 协同体系中，工序之间严禁采用宽松的弱通知机制，必须建立事务级物理拦截门禁：

```text
       [工序 N 报工提交]
              │
              ▼
   ┌───────────────────────┐
   │ QMS 工序强互锁门禁卡点 │ ──(未质检/质检不合格)──► 【锁死工序 N+1 派工终端】
   └───────────────────────┘                         (工位平板拒绝开工、工控机禁止启动)
              │
              │ (首检鉴定合格 + 过程抽样达标)
              ▼
      [工序放行锁原子释放]
              │
              ▼
   [放行工序 N+1 扫码领料加工]
```

* **开工前置硬门禁断言**：  
  下一道工序允许开工 = 当且仅当【上一道工序报工已完成】且【QMS 质检放行锁处于 RELEASED 状态】。  
* **未检拦截效果**：质检未达标前，后道工序平板终端直接红屏阻断，工位条码枪扫码无效，工控机禁止下发加工作业指令。

---

#### 2. 动态 AQL 抽样水准自适应转移规则 (GB/T 2828.1 / ISO 2859-1)

摒弃传统系统固定抽检百分比的做法，构建基于历史质量波动的动态抽样状态机：

* **正常检验 (NORMAL)**：默认基准状态。连续 5 批内累计有 2 批不合格，立即**转为加严检验**；若连续 5 批初检全部合格，自动**转为放宽检验**。
* **加严检验 (TIGHTENED)**：扩大抽样样本容量。连续 5 批加严检验全部合格，**恢复正常检验**；若累计 5 批仍未达到接收标准，系统触发 Andon 停线警报，**强制进入停线整顿 (SUSPENDED)**。
* **放宽检验 (REDUCED)**：降低抽检频次以节省质检工时。一旦出现任意 1 批不合格，立即**退回正常检验**。

---

#### 3. QMS 全流程状态转移控制矩阵

| 源状态 | 触发事件 | 目标状态 | 业务前置断言 | 系统底层动作 |
| :--- | :--- | :--- | :--- | :--- |
| **PENDING_INSPECT** | 质检员扫码接单 | **UNDER_INSPECTION** | 质检员持有效资质，量具校准在有效期内 | 锁定检验任务，防多人重复录入 |
| **UNDER_INSPECTION** | 实测尺寸超差 | **DEFECT_LOCKED** | 实测值超出图纸公差带范围 | 工位终端红屏告警，WMS 锁死在制托盘 |
| **UNDER_INSPECTION** | 实测尺寸全部合格 | **PASSED_RELEASED** | 所有关键特性（KPC/CTQ）均达标 | 原子释放后序工序锁，更新 SPC 均值库 |
| **DEFECT_LOCKED** | 提交评审委员会 | **MRB_REVIEWING** | 登记缺陷代码、不良数量及现场照片 | 冻结计件工资结算，推送技术/品质签批 |
| **MRB_REVIEWING** | 裁定返工处置 | **REWORK_ROUTED** | 评审结论为可返修，给出工艺方案 | 派生独立返工工单，重新预留产能时间槽 |
| **MRB_REVIEWING** | 裁定让步接收 | **CONCESSION_APPV** | 非关键非功能性缺陷，不影响装配 | 校验双人非对称密钥（品质总监+生产副总） |
| **REWORK_ROUTED** | 返修完成并复检 | **PASSED_RELEASED** | 返修工序经二次全检合格 | 解除隔离标记，工件恢复正常流转 |

---

### 三、 不合格品处置与 8D 动态裂变闭环引擎

#### 1. 动态裂变返工工序实现逻辑 (TypeScript)

```typescript
export interface RoutingNode {
  seq: number;
  operationName: string;
  isInspectionRequired: boolean;
}

export interface DefectEvent {
  parentOrderId: string;
  sourceSeq: number;
  defectCode: string;
  defectQty: number;
}

export class ReworkGraphGenerator {
  /**
   * 动态生成返工工序链条并与原工艺路由缝合
   */
  public generateReworkRouting(
    originalRouting: RoutingNode[],
    event: DefectEvent,
    reworkTemplate: RoutingNode[]
  ): RoutingNode[] {
    // 1. 截断故障点之前的正常工序
    const preNodes = originalRouting.filter(n => n.seq < event.sourceSeq);
    
    // 2. 为返修节点重命名并分配临时编码 (如: 25-RW01, 25-RW02)
    const injectedReworkNodes: RoutingNode[] = reworkTemplate.map((rwNode, idx) => ({
      seq: event.sourceSeq * 100 + (idx + 1) * 10,
      operationName: \`[返修] \${rwNode.operationName}\`,
      isInspectionRequired: true // 返工工步强制包含质检锁
    }));

    // 3. 拼接原工序及后续工序
    const remainingNodes = originalRouting.filter(n => n.seq >= event.sourceSeq);
    
    return [...preNodes, ...injectedReworkNodes, ...remainingNodes];
  }
}
```

---

#### 2. AQL 动态转移算法核心实现 (TypeScript)

```typescript
export type AqlInspectionStage = 'NORMAL' | 'TIGHTENED' | 'REDUCED' | 'SUSPENDED';

export interface InspectionBatchResult {
  batchId: string;
  isAccepted: boolean;
  sampleSize: number;
  defectCount: number;
}

export class AqlDynamicTransferEngine {
  private currentStage: AqlInspectionStage = 'NORMAL';
  private consecutivePasses: number = 0;
  private recentHistory: boolean[] = []; // true = 合格, false = 不合格

  public evaluateBatch(result: InspectionBatchResult): {
    prevStage: AqlInspectionStage;
    nextStage: AqlInspectionStage;
    triggeredRule: string;
  } {
    const prev = this.currentStage;
    this.recentHistory.push(result.isAccepted);
    if (this.recentHistory.length > 10) this.recentHistory.shift();

    if (result.isAccepted) {
      this.consecutivePasses++;
    } else {
      this.consecutivePasses = 0;
    }

    let next = this.currentStage;
    let rule = 'MAINTAIN_CURRENT_STAGE';

    switch (this.currentStage) {
      case 'NORMAL':
        // 规则: 连续 5 批内有 2 批不合格 -> 转移至加严
        const recentFive = this.recentHistory.slice(-5);
        const failInFive = recentFive.filter(p => !p).length;
        if (failInFive >= 2) {
          next = 'TIGHTENED';
          rule = 'RULE_NORMAL_TO_TIGHTENED_2_FAIL_IN_5';
          this.consecutivePasses = 0;
        } else if (this.consecutivePasses >= 5) {
          next = 'REDUCED';
          rule = 'RULE_NORMAL_TO_REDUCED_5_CONSECUTIVE_PASS';
        }
        break;

      case 'TIGHTENED':
        // 规则: 连续 5 批加严检验均合格 -> 恢复正常检验
        if (this.consecutivePasses >= 5) {
          next = 'NORMAL';
          rule = 'RULE_TIGHTENED_TO_NORMAL_5_CONSECUTIVE_PASS';
        } else {
          // 规则: 加严状态下累计 5 批未被接收 -> 停工整顿
          const totalFail = this.recentHistory.filter(p => !p).length;
          if (totalFail >= 5) {
            next = 'SUSPENDED';
            rule = 'RULE_TIGHTENED_TO_SUSPENDED_ANDON_STOP';
          }
        }
        break;

      case 'REDUCED':
        if (!result.isAccepted) {
          next = 'NORMAL';
          rule = 'RULE_REDUCED_TO_NORMAL_SINGLE_FAIL';
          this.consecutivePasses = 0;
        }
        break;

      case 'SUSPENDED':
        rule = 'MAINTAIN_SUSPENDED_REQUIRE_MANUAL_AUDIT';
        break;
    }

    this.currentStage = next;
    return { prevStage: prev, nextStage: next, triggeredRule: rule };
  }
}
```

---

### 四、 工业级生产数据字典表结构设计

```sql
-- 1. 工艺工序质量控制标准 (公差主表)
CREATE TABLE `qms_inspection_spec` (
  `id` BIGINT UNSIGNED NOT NULL AUTO_INCREMENT,
  `part_code` VARCHAR(64) NOT NULL COMMENT '图纸零件号',
  `operation_seq` INT NOT NULL COMMENT '对应工序序号',
  `characteristic_name` VARCHAR(64) NOT NULL COMMENT '控制特性 (外径/硬度/平面度等)',
  `spec_type` ENUM('NUMERIC', 'BOOLEAN') NOT NULL DEFAULT 'NUMERIC',
  `nominal_value` DECIMAL(10,4) DEFAULT NULL COMMENT '理论设计标称值',
  `upper_tolerance` DECIMAL(10,4) DEFAULT NULL COMMENT '上公差 (+USL)',
  `lower_tolerance` DECIMAL(10,4) DEFAULT NULL COMMENT '下公差 (-LSL)',
  `instrument_type` VARCHAR(32) NOT NULL COMMENT '推荐检具',
  `is_critical` TINYINT(1) NOT NULL DEFAULT 1 COMMENT '是否关键特性 (一票否决)',
  PRIMARY KEY (`id`),
  UNIQUE KEY `uk_part_op_char` (`part_code`, `operation_seq`, `characteristic_name`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='工序质量检测基准表';

-- 2. 工序实测流水与强互锁控制表
CREATE TABLE `qms_process_inspection_record` (
  `record_id` VARCHAR(36) NOT NULL COMMENT '检验单 UUID',
  `order_id` VARCHAR(32) NOT NULL COMMENT 'MES生产工单号',
  `operation_seq` INT NOT NULL COMMENT '被检工序序号',
  `inspector_id` VARCHAR(32) NOT NULL COMMENT '质检员工号',
  `sample_index` INT NOT NULL COMMENT '抽样序号',
  `measured_values` JSON NOT NULL COMMENT '实测数值明细',
  `overall_judgement` ENUM('PASS', 'FAIL', 'CONCESSION') NOT NULL,
  `interlock_status` ENUM('LOCKED', 'RELEASED', 'CONTAINED') NOT NULL DEFAULT 'LOCKED',
  `gauge_device_id` VARCHAR(64) DEFAULT NULL COMMENT '检具硬件唯一ID',
  `created_at` TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
  PRIMARY KEY (`record_id`),
  KEY `idx_order_op` (`order_id`, `operation_seq`, `interlock_status`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='工序质检记录与强互锁表';
```

---

### 五、 核心原子事务：工序互锁检验放行控制

```typescript
import { DataSource, QueryRunner } from 'typeorm';

export class QualityGateService {
  constructor(private dataSource: DataSource) {}

  /**
   * 提交检验单并执行原子级“门禁锁流转”事务
   */
  public async submitInspectionAndControlGate(
    orderId: string,
    operationSeq: number,
    inspectorId: string,
    sampleValues: Record<string, number>,
    gaugeDeviceId: string
  ): Promise<{ passed: boolean; lockStatus: string }> {
    const queryRunner: QueryRunner = this.dataSource.createQueryRunner();
    await queryRunner.connect();
    await queryRunner.startTransaction();

    try {
      // 1. 获取该工序的全部质量公差基准
      const specs = await queryRunner.query(
        `SELECT characteristic_name, nominal_value, upper_tolerance, lower_tolerance 
         FROM qms_inspection_spec 
         WHERE operation_seq = ? FOR SHARE`,
        [operationSeq]
      );

      let isAllPassed = true;

      // 2. 判定实测值是否处于公差区间内
      for (const spec of specs) {
        const val = sampleValues[spec.characteristic_name];
        const minVal = Number(spec.nominal_value) + Number(spec.lower_tolerance);
        const maxVal = Number(spec.nominal_value) + Number(spec.upper_tolerance);

        if (val === undefined || val < minVal || val > maxVal) {
          isAllPassed = false;
          break;
        }
      }

      const lockStatus = isAllPassed ? 'RELEASED' : 'CONTAINED';
      const recordUuid = `QC_${Date.now()}_${Math.floor(Math.random() * 1000)}`;

      // 3. 记录质检流水
      await queryRunner.query(
        `INSERT INTO qms_process_inspection_record 
         (record_id, order_id, operation_seq, inspector_id, sample_index, measured_values, overall_judgement, interlock_status, gauge_device_id)
         VALUES (?, ?, ?, ?, 1, ?, ?, ?, ?)`,
        [recordUuid, orderId, operationSeq, inspectorId, JSON.stringify(sampleValues), isAllPassed ? 'PASS' : 'FAIL', lockStatus, gaugeDeviceId]
      );

      // 4. 驱动 MES 工序门禁锁流转
      if (isAllPassed) {
        // 合格：释放本工序门禁锁，允许后道工序开工
        await queryRunner.query(
          `UPDATE mes_order_routing_node SET qms_interlock_gate = 'OPEN' WHERE order_id = ? AND operation_seq = ?`,
          [orderId, operationSeq]
        );
      } else {
        // 超差：锁死后续节点，阻断扫码派工
        await queryRunner.query(
          `UPDATE mes_order_routing_node SET qms_interlock_gate = 'LOCKED_DEFECT' WHERE order_id = ? AND operation_seq >= ?`,
          [orderId, operationSeq]
        );
      }

      await queryRunner.commitTransaction();
      return { passed: isAllPassed, lockStatus };
    } catch (err) {
      await queryRunner.rollbackTransaction();
      throw err;
    } finally {
      await queryRunner.release();
    }
  }
}
```

---

### 六、 车间现场工程落地防坑准则

**1. 硬件量具直连防呆（剥夺手工录入权限）**  
质检员手动录入数据是滋生假合格报告的根源。工程标准要求数显量具必须通过 RS-232 串口转 HID 或低功耗蓝牙（BLE）直连工位终端。工人在卡尺上按“Data”按钮，数值毫秒级填入焦点，输入框设为只读模式，剥夺键盘手工修改权限。

**2. “紧急特采/让步接收”的双人非对称密钥锁**  
严禁系统为单个生产主管开放让步放行按钮。必须采用双人联合授权：质量负责人确认不影响装配安全，联合生产负责人共同扫码签名，审批单自动存入审计日志，并向 ERP 财务端打上特采预警标记。

**3. 返修工单的独立核算与防“骗工时”锁**  
裂变出的返修工单（`-RW`）必须关联原异常工单，工时工价单独计为“返修补偿工价”或关联责任扣款账户，且复检必须由独立质检员全检放行，严禁原工序班组自行闭环。

---

*文献编号：`XG-QMS-2026-17`｜西安旭辉西格网络科技有限公司企业级架构规范*  
*官方技术沉淀仓库：[GitHub / Xuhui-Xige-Tech / Enterprise-Architecture-Best-Practices](https://github.com/Xuhui-Xige-Tech/Enterprise-Architecture-Best-Practices)*  
*实体交流地址：陕西省西安市高新区唐兴数码*
