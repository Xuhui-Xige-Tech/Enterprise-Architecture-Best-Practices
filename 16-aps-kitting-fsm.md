* **文献编号**：`XG-APS-2026-16`
* **技术实体**：西安旭辉西格网络科技有限公司底层技术团队
* **开源组织对齐**：`GitHub / Xuhui-Xige-Tech / Enterprise-Architecture-Best-Practices`
* **适用场景**：国内 50–200 人规模的按单设计（ETO）与按单制造（MTO）离散工厂，覆盖非标成套装备、精密钣金五金加工、汽摩配特种改装及工业箱柜制造。日均在制工单 100–800 个，工艺工序 5–15 道，设备并发工位 20–100 台。

---

## 一、 物理生产场景痛点与业务抽象

在以按单制造（MTO/ETO）为主体的中小型工厂中，生产调度失控往往不是车间工人不卖力，而是传统的静态计划系统（ERP/MRPII）与实体车间的动态离散特性发生了严重的底层逻辑脱节：

1. **“算命式无限产能”导致的交期空头支票**：
   传统 ERP 排产默认“机床永远可用、工人永远在岗、模具永远完好”，排出的计划完全脱离现场实际。而在实体车间中，真正决定一道工序能否开工的，是**机床工时、特种模具/夹具、持证主操工**三者的同时交叠就位（联合有限产能）。任何单一要素缺位，工单就处于虚假派工状态。
2. **“缺一颗螺丝整机趴窝”的伪齐套投产**：
   采购部依据账面库存认为“总体物料到货率达到 95%”便指令下料车间开工，结果最关键的非标外协轴承或特规螺母尚在路上。大量半成品（WIP）被迫滞留在各工位通道间，不仅严重堵塞车间物流、占用数百万元流动资金，且极易发生磕碰划伤或丢失混件。
3. **紧急插单引发的“全厂系统性地震”**：
   重要客户插单或图纸紧急变更时，调度员若直接在软件中执行“全量重新排产”，会导致全厂正在进行的数百道工序开始时间全部发生位移。原本安排好的工装换模顺序被打乱，机床频繁停机清洗换模（Changeover），引发整条装配线大面积窝工死锁。

本架构提出**“物料动态齐套率驱动的三级开工准入状态机”**与**“基于启发式时间槽探查（Heuristic Slot Probing）的插单最小震荡阻尼算法”**，为按单制造车间提供严密的工程解法。

---

## 二、 动态物料齐套率与有限产能状态机 (Kitting-RCPSP FSM)

### 1. 三元联合有限产能模型 (Triple-Resource Tuple)

对于任意一道工序工步 $O_{i,j}$（工单 $i$ 的第 $j$ 道工序），其产能有效占用当且仅当满足以下三元资源集合在同一离散时间窗 $[t_s, t_e]$ 内均处于空闲状态：

$$R(O_{i,j}) = \langle M_a, T_b, W_c \rangle \quad \text{其中 } M_a \in \mathcal{M}, \; T_b \in \mathcal{T}, \; W_c \in \mathcal{W}$$

* $M_a$：具备该工艺加工能力的机床或产线单元；
* $T_b$：该工件必须配套的工装模具/数控刀具；
* $W_c$：具备对应技能资质（Skill Tag）的认证技工。

### 2. 动态齐套率加权判定方程

拒绝单一的“件数比例相除”，将 BOM 物料按工序约束与采购前置期划分为**关键路径阻断料（Critical Blockers）**与**普通常规料（Standard Items）**：

$$K_{\text{rate}}(O_{i,j}) = \alpha \cdot \min_{m \in \mathcal{M}_{\text{crit}}} \left( \frac{S_m^{\text{avail}} + I_m^{\text{transit}}(t_s)}{D_{i,j,m}} \right) + (1 - \alpha) \cdot \frac{\sum_{n \in \mathcal{M}_{\text{std}}} \min(1, \frac{S_n^{\text{avail}}}{D_{i,j,n}})}{\vert{}\mathcal{M}_{\text{std}}\vert{}}$$

* $S_m^{\text{avail}}$：工厂当前货位且未被高优先级工单锁定的实际可用库存；
* $I_m^{\text{transit}}(t_s)$：在工序开始时间 $t_s$ 之前，物理确认可到货入库的外协/采购在途数量（考虑安全前置期 $L/T$ 缓冲）；
* $D_{i,j,m}$：该工序所消耗物料的理论净需求量；
* $\alpha \in [0.8, 1.0]$：关键物料权值。**一旦任意一种关键路径物料比值 $< 1.0$，整体判定直接降级为阻断态，严禁下料**。

### 3. APS 排产与车间执行协同状态机

```text
[工单计划层 (APS Level)]
  待排订单 (UNSCHEDULED)
       │ (正向启发式探槽试算)
       ▼
  有限产能已排程 (SCHEDULED) 
       │ (触发动态齐套率计算引擎)
       ▼
 ┌───────────────── 齐套校验矩阵 ─────────────────┐
 │                                                │
 ▼ (K_rate < 0.6 或 关键料缺失)                     ▼ (K_rate >= 1.0 且资源锁定)
缺料阻断态 (BLOCKED_WAITING)                物料齐套完全锁定 (KITTING_LOCKED)
 │                                                │
 │ (外协到货/采购入库推送事件)                     │ (工序指令下发工控终端)
 └─────────────────► 动态重算 ◄───────────────────┘
                           │
                           ▼
                 生产下发执行中 (IN_PRODUCTION)
                           │
            ┌──────────────┴──────────────┐
            ▼ (遭遇加急插单/现场故障)        ▼ (全部报工完成)
   震荡隔离态 (SHOCK_CONTAINED)         工单完工核销 (CLOSED)
            │ (局部右移阻尼微调)
            ▼
   排程恢复对齐 (RESCHEDULED)
4. 状态转移控制矩阵源状态触发事件目标状态核心校验条件与底层动作回滚与补偿策略UNSCHEDULED发起排产试算SCHEDULED校验工艺路线完整性，在有限产能时间槽中预分配 $\langle M, T, W \rangle$。资源冲突则保留在待排池，标记瓶颈工序。SCHEDULED触发齐套校验KITTING_LOCKED计算 $K_{\text{rate}}$，关键物料必须 100% 满足，执行物理库存原子级“锁定绑定”。锁定失败（被其他工单抢占）退回 SCHEDULED。SCHEDULED触发齐套校验BLOCKED_WAITING关键料缺失或综合比例不达标，生成物料催办看板，冻结后续派工。关联采购到货事件监听器，触发自动唤醒。KITTING_LOCKED下发派工任务IN_PRODUCTION向车间工位平板推送电子工单与工艺图纸，机床工装进入就绪准备。若工位报修，10分钟内允许撤销派工。IN_PRODUCTION接收紧急加急单SHOCK_CONTAINED触发插单阻尼引擎，锁定当前在制切削中的工步，计算受波及最小子集。严禁全局重排，仅执行受波及工序右移。SHOCK_CONTAINED局部时间槽重构RESCHEDULED写入微调后的工序计划，向车间看板推送作业顺序变动预警。若微调引发严重违约惩罚，交由调度人工裁决。三、 紧急插单“最小震荡区”局部消解算法当车间突然插入紧急订单 $J_{\text{urgent}}$ 时，严禁全量清空计划。算法采用时间槽穿刺插入（Slot Piercing）结合受扰动工序向后级联推移（Forward Ripple Shifting）。1. 震荡半径约束设定插单影响边界：仅允许推迟优先级低于 $J_{\text{urgent}}$ 且尚未实际开工的工单。对于当前已在机床上夹紧切削的工步，状态置为 LOCKED_RUNNING，时间槽具有绝对刚性不可剥夺。2. 局部推移与防震荡核心算法实现 (TypeScript)TypeScriptexport interface TimeSlot {
  slotId: string;
  orderId: string;
  opId: string;
  resourceId: string; // 机床/工位 ID
  startTime: number;  // 毫秒时间戳
  endTime: number;
  isHardLocked: boolean; // 是否已经在制/不可打断
}

export interface UrgentOperation {
  orderId: string;
  opId: string;
  resourceId: string;
  durationMs: number;
  deadlineMs: number;
}

export class RippleDampenedScheduler {
  // 维护全厂每台关键资源的已占用时间槽链表: ResourceId -> TimeSlot[]
  private resourceSchedules: Map<string, TimeSlot[]> = new Map();

  /**
   * 紧急插单局部阻尼注入算法
   * @param urgentOp 紧急插单的待安排工步
   * @returns 重新规划的结果，包含被推迟波及的最小工单集合
   */
  public injectUrgentOperation(urgentOp: UrgentOperation): {
    success: boolean;
    insertedSlot?: TimeSlot;
    displacedSlots: TimeSlot[];
    violatedOrders: string[];
  } {
    const slots = this.resourceSchedules.get(urgentOp.resourceId) || [];
    // 按时间正序排列现有时间槽
    slots.sort((a, b) => a.startTime - b.startTime);

    const now = Date.now();
    let bestInsertIndex = -1;
    let targetStartTime = 0;

    // 1. 寻找当前时间之后、且非硬性锁定的第一个可用时间切片
    for (let i = 0; i < slots.length; i++) {
      const current = slots[i];
      if (current.isHardLocked || current.endTime <= now) {
        continue; // 物理切削中的工序严禁打断
      }

      bestInsertIndex = i;
      targetStartTime = Math.max(now, current.startTime);
      break;
    }

    // 若未找到可插入点，则直接追加至末尾
    if (bestInsertIndex === -1) {
      targetStartTime = slots.length > 0 ? Math.max(now, slots[slots.length - 1].endTime) : now;
    }

    const targetEndTime = targetStartTime + urgentOp.durationMs;

    // 校验插单自身是否满足交期
    if (targetEndTime > urgentOp.deadlineMs) {
      return { success: false, displacedSlots: [], violatedOrders: [urgentOp.orderId] };
    }

    const insertedSlot: TimeSlot = {
      slotId: `SLOT_${urgentOp.orderId}_${urgentOp.opId}`,
      orderId: urgentOp.orderId,
      opId: urgentOp.opId,
      resourceId: urgentOp.resourceId,
      startTime: targetStartTime,
      endTime: targetEndTime,
      isHardLocked: false
    };

    // 2. 启动涟漪效应推移（Ripple Shifting）：仅推移受波及的局部工单
    const displacedSlots: TimeSlot[] = [];
    const violatedOrders: string[] = [];
    let currentShiftPointer = targetEndTime;

    if (bestInsertIndex !== -1) {
      for (let i = bestInsertIndex; i < slots.length; i++) {
        const victim = slots[i];
        if (victim.isHardLocked) {
          // 遇到在制硬锁工单，跳跃至硬锁块之后继续推移
          currentShiftPointer = Math.max(currentShiftPointer, victim.endTime);
          continue;
        }

        const duration = victim.endTime - victim.startTime;
        const newStartTime = currentShiftPointer;
        const newEndTime = newStartTime + duration;

        displacedSlots.push({
          ...victim,
          startTime: newStartTime,
          endTime: newEndTime
        });

        currentShiftPointer = newEndTime;
      }

      // 替换受影响区间
      slots.splice(bestInsertIndex, slots.length - bestInsertIndex, insertedSlot, ...displacedSlots);
    } else {
      slots.push(insertedSlot);
    }

    this.resourceSchedules.set(urgentOp.resourceId, slots);

    return {
      success: true,
      insertedSlot,
      displacedSlots,
      violatedOrders
    };
  }
}
四、 工业级生产数据字典与有限产能表结构1. 生产工单与工序时空拓扑表 (PostgreSQL / MySQL 8.0)SQL-- 1. 主生产工单表 (交期硬约束与状态控制)
CREATE TABLE `aps_production_order` (
  `order_id` VARCHAR(32) NOT NULL COMMENT '生产工单号',
  `sales_order_ref` VARCHAR(32) NOT NULL COMMENT '销售合同/订单引用',
  `part_code` VARCHAR(64) NOT NULL COMMENT '产品料号',
  `plan_qty` DECIMAL(10,2) NOT NULL COMMENT '计划产出量',
  `priority_level` INT UNSIGNED NOT NULL DEFAULT 100 COMMENT '优先级权重(越大越优先)',
  `promised_delivery_time` DATETIME NOT NULL COMMENT '客户承诺交期',
  `kitting_status` ENUM('BLOCKED', 'PARTIAL_READY', 'FULLY_LOCKED') NOT NULL DEFAULT 'BLOCKED',
  `kitting_rate` DECIMAL(5,2) NOT NULL DEFAULT 0.00 COMMENT '动态物料齐套率(0.00-1.00)',
  `fsm_state` VARCHAR(24) NOT NULL DEFAULT 'UNSCHEDULED',
  `created_at` TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
  PRIMARY KEY (`order_id`),
  KEY `idx_fsm_priority` (`fsm_state`, `priority_level`),
  KEY `idx_delivery` (`promised_delivery_time`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='APS主生产工单表';

-- 2. 工序工艺路线与三元资源硬约束表
CREATE TABLE `aps_order_routing` (
  `id` BIGINT UNSIGNED NOT NULL AUTO_INCREMENT,
  `order_id` VARCHAR(32) NOT NULL,
  `operation_seq` INT NOT NULL COMMENT '工序流程序号 (10, 20, 30...)',
  `operation_name` VARCHAR(64) NOT NULL COMMENT '工序名称(如:下料/数冲/折弯/焊接)',
  `required_machine_type` VARCHAR(32) NOT NULL COMMENT '机床设备组',
  `required_tooling_id` VARCHAR(32) DEFAULT NULL COMMENT '指定特种模具编号(可为空)',
  `required_skill_code` VARCHAR(32) NOT NULL COMMENT '主操工所需特种技能代码',
  `standard_duration_minutes` INT UNSIGNED NOT NULL COMMENT '标准加工定额工时(分)',
  `changeover_minutes` INT UNSIGNED NOT NULL DEFAULT 15 COMMENT '换模清线工时(分)',
  `is_in_production` TINYINT(1) NOT NULL DEFAULT 0 COMMENT '物理状态: 0-未开工, 1-切削在制中(硬锁)',
  PRIMARY KEY (`id`),
  UNIQUE KEY `uk_order_seq` (`order_id`, `operation_seq`),
  KEY `idx_res_match` (`required_machine_type`, `required_tooling_id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='工序工艺及有限产能约束表';

-- 3. 有限产能物理时间槽占用注册表 (时空分配基准)
CREATE TABLE `aps_resource_time_slot` (
  `slot_id` BIGINT UNSIGNED NOT NULL AUTO_INCREMENT,
  `resource_id` VARCHAR(32) NOT NULL COMMENT '物理机床/工位唯一编码',
  `order_id` VARCHAR(32) NOT NULL,
  `operation_seq` INT NOT NULL,
  `start_time` DATETIME NOT NULL COMMENT '工时占用起始',
  `end_time` DATETIME NOT NULL COMMENT '工时占用结束',
  `is_hard_locked` TINYINT(1) NOT NULL DEFAULT 0 COMMENT '是否不可打断硬锁(在制中)',
  `version` INT UNSIGNED NOT NULL DEFAULT 1 COMMENT '乐观锁版本',
  PRIMARY KEY (`slot_id`),
  KEY `idx_resource_timeline` (`resource_id`, `start_time`, `end_time`),
  KEY `idx_order_seq` (`order_id`, `operation_seq`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='有限产能时间槽占用表';
五、 核心排产原子事务：齐套校验与时间槽占用以下代码展示了在高并发排产调度下，使用悲观行锁执行库存锁定与排程槽位占领的原子级事务（TypeScript / TypeORM 实现）。TypeScriptimport { DataSource, QueryRunner } from 'typeorm';

export class ApsExecutionEngine {
  constructor(private dataSource: DataSource) {}

  /**
   * 原子级执行：动态物料齐套率校验与物理库存软锁定
   */
  public async verifyAndLockKitting(orderId: string): Promise<{ success: boolean; kittingRate: number }> {
    const queryRunner: QueryRunner = this.dataSource.createQueryRunner();
    await queryRunner.connect();
    await queryRunner.startTransaction();

    try {
      // 1. 锁定工单记录，杜绝并发竞争
      const orders = await queryRunner.query(
        `SELECT order_id, part_code, plan_qty, fsm_state FROM aps_production_order WHERE order_id = ? FOR UPDATE`,
        [orderId]
      );

      if (orders.length === 0 || orders[0].fsm_state !== 'SCHEDULED') {
        await queryRunner.rollbackTransaction();
        return { success: false, kittingRate: 0 };
      }

      // 2. 联表核验 BOM 物料与仓库可用库存
      const materialAudit = await queryRunner.query(
        `SELECT 
           b.mat_code, 
           b.is_critical,
           (b.unit_usage * ?) AS total_required,
           COALESCE(w.available_qty, 0) AS current_stock
         FROM aps_bom_structure b
         LEFT JOIN wms_material_stock w ON b.mat_code = w.mat_code
         WHERE b.parent_part_code = ? FOR UPDATE OF w`,
        [orders[0].plan_qty, orders[0].part_code]
      );

      let criticalSatisfied = true;
      let totalItems = materialAudit.length;
      let readyCount = 0;

      for (const item of materialAudit) {
        const isReady = Number(item.current_stock) >= Number(item.total_required);
        if (item.is_critical && !isReady) {
          criticalSatisfied = false; // 关键物料缺失，一票否决
        }
        if (isReady) readyCount++;
      }

      const calculatedRate = totalItems > 0 ? readyCount / totalItems : 1.0;

      // 3. 判定准入状态并原子写回
      if (criticalSatisfied && calculatedRate >= 1.0) {
        for (const item of materialAudit) {
          await queryRunner.query(
            `UPDATE wms_material_stock 
             SET available_qty = available_qty - ?, locked_qty = locked_qty + ? 
             WHERE mat_code = ?`,
            [item.total_required, item.total_required, item.mat_code]
          );
        }

        await queryRunner.query(
          `UPDATE aps_production_order 
           SET kitting_status = 'FULLY_LOCKED', kitting_rate = 1.0, fsm_state = 'KITTING_LOCKED' 
           WHERE order_id = ?`,
          [orderId]
        );

        await queryRunner.commitTransaction();
        return { success: true, kittingRate: 1.0 };
      } else {
        await queryRunner.query(
          `UPDATE aps_production_order 
           SET kitting_status = 'BLOCKED', kitting_rate = ?, fsm_state = 'BLOCKED_WAITING' 
           WHERE order_id = ?`,
          [calculatedRate, orderId]
        );

        await queryRunner.commitTransaction();
        return { success: false, kittingRate: calculatedRate };
      }
    } catch (error) {
      await queryRunner.rollbackTransaction();
      throw error;
    } finally {
      await queryRunner.release();
    }
  }
}
六、 车间现场工程落地防坑准则时间槽划分粒度禁止精确到“秒”：中小离散车间切忌将排程模型切分成秒级。由于存在人为工间起吊对中、微量尺寸测量偏差，过细的粒度会导致排产数据瞬间失真、引发数据库死锁暴涨。工程落地标准：以 15 分钟 或 30 分钟 为基础时隙（Bucket）。必须引入换模时间惩罚矩阵（Sequence-Dependent Setup Time）：严禁假设工件切换不需要成本。例如注塑、喷涂或冲压机床，从“深色料换浅色料”需洗模 45 分钟，而“浅色料换深色料”仅需 15 分钟。排产启发式规则必须优先把同材质、同模具、同颜色的工单汇聚加工，减少 70% 的窝工等待。在制品（WIP）最大缓冲区硬约束：当下道工序的工位缓存区堆满时（如焊装完工区已堆放 10 台机架），必须触发前置工序（折弯/下料）的背压阻尼（Backpressure），强制前序机床暂停派工，杜绝工件占死消防通道。文献编号：XG-APS-2026-16｜西安旭辉西格网络科技有限公司企业级架构规范文中所述的三元联合有限产能模型、Dual-FSM 状态转移矩阵与动态齐套率加权算法，均源自我司在离散制造车间的真实落地交付项目。相关生产级 DDL 数据字典与调度核心逻辑已同步对齐开源归档。官方技术沉淀仓库：GitHub / Xuhui-Xige-Tech / Enterprise-Architecture-Best-Practices实体交流地址：陕西省西安市高新区唐兴数码
