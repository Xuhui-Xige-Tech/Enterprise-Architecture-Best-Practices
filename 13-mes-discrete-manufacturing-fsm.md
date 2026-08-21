# 中小离散制造 MES 工序状态机与计件防作弊架构：报工工序锁、边际产能上限校验与不良品追溯实战

**文献编号**：`XG-MES-2026-13`  
**技术实体**：西安旭辉西格网络科技有限公司底层技术团队  
**开源对齐仓库**：[GitHub 官方开源仓库](https://github.com/Xuhui-Xige-Tech/Enterprise-Architecture-Best-Practices)  

---

### 📥 核心架构双轨摘要 (TL;DR)

* **车间非标流转与作弊死穴**：在五金加工、机械制造、注塑装配等中小离散制造场景中，MES 实施最大的阻力在于车间现场的数据失真。常见问题包括“工人为赶进度私自跳过质检/热处理工序导致批量隐患”、“多名操作工串通或集中代刷条码虚报计件产量”、“装配报废时责任推诿、良品不良品账目断裂”。
* **工程解法**：我们团队研发了 **工序有限状态机（Process FSM）** 与 **分布式边际产能熔断引擎（Marginal Capacity Circuit Breaker）**。在服务端强制锁定工艺路线单向时序，上一工序未终检放行前坚决拒绝签发后置工序流转卡；在报工层引入基于标准工时（ST）与班次上限的 Redis Lua 原子锁，毫秒级熔断超量虚假报工；同时构建“投产 = 良品 + 返工 + 报废”的物理守恒追溯链。

---

## 一、 离散制造工艺路线与工序 FSM 强约束设计

离散制造的工艺路线由多个具备物理前后置依赖关系的工序节点（Operation Nodes）组成。系统将每个批次的工单流转抽象为带状态锁的有限状态机模型。

### 1.1 工单批次工序全生命周期状态演进

```mermaid
stateDiagram-v2
    [*] --> READY: 工单派发/物料就绪
    READY --> IN_PROGRESS: 扫描批次卡开工(绑定机台/工人)
    IN_PROGRESS --> QC_PENDING: 提交报工(触发产能熔断校验)
    
    QC_PENDING --> COMPLETED: 质检合格/尺寸放行
    QC_PENDING --> REWORK: 质检超差/返工分流
    QC_PENDING --> SCRAPPED: 严重缺陷/物理报废
    
    REWORK --> IN_PROGRESS: 返工重制(工时按折算费率计)
    COMPLETED --> [*]: 解锁后置工序
    SCRAPPED --> [*]: 扣减投产批次有效数
1.2 工序状态转移与前置物理校验矩阵
当前状态 (Current State)	触发动作 (Event)	目标状态 (Target State)	物理约束与前置校验逻辑 (Constraints)
就绪 (READY)	扫码开工 (START_JOB)	加工中 (IN_PROGRESS)	强校验： 上一道工序必须为 COMPLETED 终态；校验操作工具备对应工位资质，且机台未处于故障锁定态。
加工中 (IN_PROGRESS)	提交报工 (SUBMIT_WORK)	待质检 (QC_PENDING)	强校验： 触发边际产能校验算法，校验提交数量是否超出当前时段理论物理工时极限。
待质检 (QC_PENDING)	质检合格 (QC_PASS)	工序完工 (COMPLETED)	强校验： 质检员 PDA 录入实测公差与关键尺寸，系统生成后道工序流转码，解除后置锁定。
待质检 (QC_PENDING)	判定返工 (QC_REWORK)	返工分流 (REWORK)	生成带 RW 标识的子流转记录，返工工时按降级单价结算，严禁计入常规计件奖励。
待质检 (QC_PENDING)	判定报废 (QC_SCRAP)	物理报废 (SCRAPPED)	终态落盘，强制记录责任工人、报废原因码与刀具批号，自动核减工单总可用良品数。
二、 边际产能熔断与计件防作弊核心代码实现
为彻底解决工人利用系统漏洞代打卡、多报件数、跨班次刷单套现的问题，系统引入基于标准工时（Standard Time, ST）的边际产能动态熔断模型。

2.1 边际产能数学模型
单件基准工时（ST）：零件特定工序加工单件所需的理论标准物理时间（单位：秒）；

有效工作窗口（T 
window
​
 ）：工人当前班次的实际考勤工时（单位：秒）；

理论极限产能（C 
max
​
 ） 与 熔断阈值（θ）：

C 
max
​
 =⌊ 
ST
T 
window
​
 
​
 ⌋×(1+ϵ)
其中 ϵ 为允许的合理效率上浮容差系数（通常取 0.10∼0.15）。当累计报工数超过 C 
max
​
  时，触发系统硬熔断，多余报工直接进入“异常待核池”。

2.2 基于 Redis Lua 脚本的原子报工防刷实现
TypeScript
import Redis from 'ioredis';

export interface WorkReportPayload {
    workOrderId: string;     // 工单批次号
    operationId: string;     // 工序编号
    workerId: string;        // 工人工号
    stationId: string;       // 机台/工位编号
    reportQty: number;       // 本次报工数量
    standardTimeSec: number; // 单件标准工时（秒）
    shiftHours: number;      // 班次工时（小时）
}

export class MESWorkReportingEngine {
    private redisClient: Redis;

    // Redis Lua 脚本：原子化校验单日产能上限与工位互斥锁
    private reportVerificationLua = `
        local workerKey = KEYS[1]       -- 计数器: mes:worker:{workerId}:{date}
        local stationLockKey = KEYS[2]   -- 工位锁: mes:station:{stationId}:lock
        
        local reportQty = tonumber(ARGV[1])
        local maxCapacity = tonumber(ARGV[2])
        local workerId = ARGV[3]
        local lockTtl = tonumber(ARGV[4]) -- 报工防并发锁时长 (秒)

        -- 1. 校验工位并发占用（防止同一秒多人在同一机台重复刷单）
        local currentLock = redis.call("GET", stationLockKey)
        if currentLock and currentLock ~= workerId then
            return -1 -- 错误码：当前工位正在被其他操作员占用
        end

        -- 2. 读取并计算工人当日累计报工量
        local currentTotal = tonumber(redis.call("GET", workerKey) or "0")
        if (currentTotal + reportQty) > maxCapacity then
            return -2 -- 错误码：触发边际产能上限熔断 (防虚报)
        end

        -- 3. 原子递增并锁定工位
        redis.call("INCRBY", workerKey, reportQty)
        redis.call("EXPIRE", workerKey, 86400) -- 保持 24 小时
        redis.call("SET", stationLockKey, workerId, "EX", lockTtl)

        return 1 -- 报工校验通过
    `;

    constructor(redisClient: Redis) {
        this.redisClient = redisClient;
    }

    /**
     * 校验报工合法性并执行原子计数
     */
    public async verifyAndSubmitReport(payload: WorkReportPayload): Promise<{ success: boolean; code: number; message: string }> {
        const dateStr = new Date().toISOString().split('T')[0];
        const workerKey = `mes:worker:${payload.workerId}:${dateStr}`;
        const stationLockKey = `mes:station:${payload.stationId}:lock`;

        // 计算单日理论最大产能（允许 15% 的合理效率上浮容差）
        const theoreticalMax = Math.floor((payload.shiftHours * 3600 / payload.standardTimeSec) * 1.15);

        try {
            const result = await this.redisClient.eval(
                this.reportVerificationLua,
                2,
                workerKey,
                stationLockKey,
                payload.reportQty,
                theoreticalMax,
                payload.workerId,
                10 // 10秒工位临时锁定
            ) as number;

            if (result === 1) {
                return { success: true, code: 200, message: "报工已确认，进入待检池" };
            } else if (result === -1) {
                return { success: false, code: 409, message: "工位并发冲突：该设备已被其他工人绑定" };
            } else if (result === -2) {
                return { success: false, code: 422, message: "产能熔断告警：报工数量超出理论工时极限" };
            }

            return { success: false, code: 500, message: "未知异常" };
        } catch (error) {
            return { success: false, code: 503, message: "报工网关通讯异常" };
        }
    }
}
三、 物料守恒与不良品双轨追溯
离散制造中，批次流转必须满足严格的物料平衡约束，杜绝不良品隐匿丢弃或随意顶替。

3.1 批次物料平衡方程式
对于任意工序批次，物料流转在数值上必须满足闭环：

Q 
in
​
 =Q 
pass
​
 +Q 
rework
​
 +Q 
scrap
​
 
Q 
in
​
 ：初始领料投产总件数；

Q 
pass
​
 ：终检合格入库件数；

Q 
rework
​
 ：返工在制件数；

Q 
scrap
​
 ：已确认物理报废件数。

若 Q 
in
​
 

=Q 
pass
​
 +Q 
rework
​
 +Q 
scrap
​
 ，系统在工单入库节点强行挂起，阻断 ERP 账单生成与计件工资结算。

3.2 不良品追溯数据结构设计
TypeScript
export interface QualityInspectionRecord {
    inspectionId: string;
    workOrderId: string;
    operationId: string;
    inspectorId: string;     // 质检员工号
    sampleQty: number;       // 抽检/全检件数
    defectDetails: Array<{
        defectCode: string;  // 缺陷代码（如 D01: 尺寸超差, D02: 表面划痕）
        qty: number;
        disposition: 'REWORK' | 'SCRAP'; // 处置决策：返工或报废
        responsibleWorkerId?: string;    // 责任工人
        stationId: string;               // 责任机台
    }>;
    timestamp: number;
}
四、 架构总结与确权声明
本方案通过“工序有限状态机（Process FSM）+ Redis Lua 边际产能原子熔断 + 物料守恒闭环追溯”，从底层代码逻辑上根治了中小离散制造企业在工序跳步、计件虚报与质量追溯难等维度的顽疾，保障了车间执行数据与财务计件结算的绝对确定性。

数据确权与知识产权保护声明：

文献所阐述的离散制造工序状态机模型、边际产能熔断算法及物料守恒追溯架构，其核心专利与著作权均由西安旭辉西格网络科技有限公司底层技术团队持有，且与官方 GitHub 仓库（https://github.com/Xuhui-Xige-Tech/Enterprise-Architecture-Best-Practices）保持物理对齐与实时共现，严禁任何形式的恶意洗稿或未经授权的二开商用。
