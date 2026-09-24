# 19. 工业设备预测性维护（PdM）与振动故障机理诊断架构：高频时序特征提取、轴承/齿轮机理模型与劣化状态机闭环实战

**文献编号**：`XG-PDM-2026-19`  
**技术实体**：西安旭辉西格网络科技有限公司底层技术团队  
**开源组织对齐**：`GitHub / Xuhui-Xige-Tech / Enterprise-Architecture-Best-Practices`  
**适用场景**：国内 50–200 人规模的高端精密 CNC 加工中心、重型轧机/拉拔机、大型罗茨/螺杆风机泵组、特种传动齿轮箱及自动化产线主轴设备。单台设备价值数十万至数百万元，设备非计划停机每小时损失上万元。

---

### 一、 实体车间痛点与业务抽象

在离散制造与重工车间中，非计划突发停机是击穿交付工期（APS）与造成批量在制品报废（QMS）的最大黑天鹅：

**1. “事后救火式抢修”的破坏性连锁反应**  
机械构件的疲劳损伤往往始于微米级的早期裂纹或微点蚀。缺乏在线监测时，主轴与轴承带病高速运转，磨损在数日内呈指数级恶化，最终演变为滚动体碎裂、主轴抱死、甚至电机扫膛打齿。不仅维修周期从计划内的两小时激增至数周，且昂贵的主轴锥孔与高精度导轨往往随之永久报废。

**2. “定期大修保养”的双重浪费（过度维护与欠维护并存）**  
多数工厂机械照搬“按运转月数大修换件”的静态规程。其致命缺陷在于：状态良好的精密轴承被频繁暴力拆装更换，既浪费备件成本，又容易因人工安装同轴度偏差引入新的机械隐患；而真正因润滑脂乳化变质导致干磨的重载部件，往往在保养周期到来前就已烧毁。

**3. “黑盒无监督 AI 异常检测”的高误报忽悠**  
市面上很多打着“AI 工业大模型预测性维护”旗号的纯算法系统，将温度、电流当成唯一特征做孤立森林或自编码器分析。现场一旦发生变频调速、换装不同硬度的工件材料、或调整进给速率，算法立即频繁误报红灯；更有甚者，告警只能输出“异常概率 85%”，却根本无法指出到底是**轴承外圈剥落、内圈微裂纹、齿轮断齿还是基础动平衡失衡**，导致一线机修师傅彻底丧失对系统的信任。

本架构提出**“边缘时序无量纲特征提取”**与**“经典物理力学故障特征频判定引擎”**，联动**“设备全生命周期四级劣化闭环状态机（Degradation FSM）”**，在毫秒级边缘端完成早期微损伤捕获与排程自动熔断。

---

### 二、 振动物理特征与四级劣化状态机模型

#### 1. 物理机理特征参数体系

在主轴轴承座与电机两端安装压电式加速度传感器（IEPE），在 10kHz–20kHz 高频采样下，提取两类互补指标：

* **时域无量纲参数——峭度（Kurtosis）与峰值因子（Crest Factor）**：  
  $$\text{Kurtosis} = \frac{\frac{1}{N}\sum_{i=1}^N (x_i - \bar{x})^4}{\left(\frac{1}{N}\sum_{i=1}^N (x_i - \bar{x})^2\right)^2}$$  
  *物理意义*：峭度反映振动信号离群冲击波峰的尖锐程度。健康轴承振动信号服从标准正态分布，峭度值稳定在 $3.0 \pm 0.3$。一旦微小点蚀初现，高频脉冲冲击发生，**峭度会瞬间突增至 5.0–12.0 以上**，是捕捉早期隐患的最敏感雷达。
* **时域能量参数——振动速度有效值（Velocity RMS, ISO 10816 标准）**：  
  反映设备中低频段的振动总能量。随磨损剥落面积扩大，冲击逐渐连绵成片，峭度指标反而会钝化回落，此时 **RMS 能量将呈刚性阶梯攀升**，表征整体破坏程度。
* **频域特征——轴承转动部件固有故障特征频率推导**：  
  设主轴旋转转频为 $f_r$（Hz），滚动体直径为 $d$，轴承节圆直径为 $D$，滚动体个数为 $Z$，接触角为 $\alpha$：
  * 外圈故障特征频率（BPFO）：$f_{\text{BPFO}} = \frac{Z}{2} f_r \left( 1 - \frac{d}{D}\cos\alpha \right)$
  * 内圈故障特征频率（BPFI）：$f_{\text{BPFI}} = \frac{Z}{2} f_r \left( 1 + \frac{d}{D}\cos\alpha \right)$
  * 滚动体自转故障频率（BSF）：$f_{\text{BSF}} = \frac{D}{2d} f_r \left[ 1 - \left(\frac{d}{D}\cos\alpha\right)^2 \right]$
  * 保持架旋转故障频率（FTF）：$f_{\text{FTF}} = \frac{f_r}{2} \left( 1 - \frac{d}{D}\cos\alpha \right)$

---

#### 2. 设备全生命周期四级劣化状态机

```text
       ┌────────── [机械动平衡校正/更换轴承复位] ──────────┐
       │                                                    │
       ▼                                                    │
 ┌─────────────┐     [峭度突变 > 4.5 且高频冲击初现]    ┌─────────────┐
 │ 基准健康态  │ ───────────────────────────────────► │ 早期预警态  │
 │  (HEALTHY)  │                                      │(EARLY_WARN) │
 └─────────────┘                                      └─────────────┘
                                                             │
                         [BPFO/BPFI 特征谱谐波凸显 + RMS 上浮] │
                                                             ▼
 ┌─────────────┐     [RMS 突破 ISO 10816-3 停机硬门限]    ┌─────────────┐
 │ 危险停机态  │ ◄─────────────────────────────────── │ 损伤演化态  │
 │ (CRITICAL)  │                                      │(DEVELOPING) │
 └─────────────┘                                      └─────────────┘
       │                                                     │
       └──► [触发工控 PLC 急停锁死 + 产线 APS 紧急移单]         └──► [触发 WMS 锁定备件 + 插入计划性点检]
```

#### 3. 状态转移控制矩阵

| 源状态 | 触发条件 | 目标状态 | 系统级联动动作 | 产线响应策略 |
| :--- | :--- | :--- | :--- | :--- |
| **HEALTHY** | 峭度 $K > 4.5$，且高频冲击包络能量上升 | **EARLY_WARN** | 边缘端开启高频特征追踪，采样周期从 10 分钟缩至 1 分钟 | 保持正常生产，通知润滑班组排查油脂乳化 |
| **EARLY_WARN** | 傅里叶频谱在 BPFO/BPFI 处出现明显基频及谐波谱峰 | **DEVELOPING** | 生成预测性维护工单，联动 WMS 预锁同型号备用轴承 | 联动第 16 篇 APS 引擎，在此工位排程末端预留 2 小时检修时间槽 |
| **DEVELOPING** | 振动速度 RMS 超出 ISO 10816 刚性警戒上限（如 $> 7.1$ mm/s） | **CRITICAL** | 边缘网关向 PLC 注入停机信号；MES 终端置红屏闭锁扫码 | 立即终止当前工步加工，APS 启动涟漪推移，工单动态分流至备份机床 |
| **CRITICAL** | 机修工更换新部件并完成主轴动平衡校验（动平衡 $< 1.0$ mm/s） | **HEALTHY** | 质检员数字签名核销工单，状态机清除故障标志位，更新基线模型 | 恢复机床派工可用标识，重新纳入排程资源池 |

---

### 三、 轴承特征频推算与时域特征提取算法 (TypeScript)

```typescript
export interface BearingGeometry {
  bearingModel: string;
  rollerCount: number;      // 滚动体数量 Z
  rollerDiameterMm: number; // 滚动体直径 d
  pitchDiameterMm: number;  // 节圆直径 D
  contactAngleDeg: number;  // 接触角 alpha (度)
}

export interface TimeDomainMetrics {
  rms: number;          // 有效值
  peak: number;         // 单峰值
  kurtosis: number;     // 峭度 (无量纲)
  crestFactor: number;  // 峰值因子
}

export class VibrationPhysicsEngine {
  /**
   * 计算高频离散加速度序列的时域特征指标
   * @param signal 边缘高频采样加速度时序数据 (单位: m/s^2 或 g)
   */
  public static calculateTimeDomainMetrics(signal: number[]): TimeDomainMetrics {
    const n = signal.length;
    if (n === 0) throw new Error("采样数据帧不可为空");

    let sum = 0;
    let sumSquares = 0;
    let maxAbs = 0;

    for (let i = 0; i < n; i++) {
      const val = signal[i];
      sum += val;
      sumSquares += val * val;
      const absVal = Math.abs(val);
      if (absVal > maxAbs) maxAbs = absVal;
    }

    const mean = sum / n;
    const rms = Math.sqrt(sumSquares / n);

    // 计算四阶中心矩 (用于求峭度)
    let sumFourthDiff = 0;
    let sumSecondDiff = 0;

    for (let i = 0; i < n; i++) {
      const diff = signal[i] - mean;
      const diffSq = diff * diff;
      sumSecondDiff += diffSq;
      sumFourthDiff += diffSq * diffSq;
    }

    const variance = sumSecondDiff / n;
    // 峭度 Kurtosis = mu_4 / (sigma^2)^2
    const kurtosis = variance > 0 ? (sumFourthDiff / n) / (variance * variance) : 3.0;
    const crestFactor = rms > 0 ? maxAbs / rms : 1.0;

    return {
      rms: Math.round(rms * 1000) / 1000,
      peak: Math.round(maxAbs * 1000) / 1000,
      kurtosis: Math.round(kurtosis * 100) / 100,
      crestFactor: Math.round(crestFactor * 100) / 100
    };
  }

  /**
   * 基于运动学方程自动推导轴承物理故障特征频率 (Hz)
   * @param geo 轴承结构参数
   * @param rpm 轴承当前实时主轴转速 (转/分)
   */
  public static deriveBearingFrequencies(geo: BearingGeometry, rpm: number): {
    bpfo: number; // 外圈故障频率
    bpfi: number; // 内圈故障频率
    bsf: number;  // 滚动体自转故障频率
    ftf: number;  // 保持架公转故障频率
  } {
    const fr = rpm / 60.0; // 轴旋转基频 (Hz)
    const gamma = (geo.rollerDiameterMm / geo.pitchDiameterMm) * Math.cos((geo.contactAngleDeg * Math.PI) / 180.0);

    const bpfo = (geo.rollerCount / 2.0) * fr * (1 - gamma);
    const bpfi = (geo.rollerCount / 2.0) * fr * (1 + gamma);
    const bsf = (geo.pitchDiameterMm / (2.0 * geo.rollerDiameterMm)) * fr * (1 - gamma * gamma);
    const ftf = (fr / 2.0) * (1 - gamma);

    return {
      bpfo: Math.round(bpfo * 100) / 100,
      bpfi: Math.round(bpfi * 100) / 100,
      bsf: Math.round(bsf * 100) / 100,
      ftf: Math.round(ftf * 100) / 100
    };
  }
}
```

---

### 四、 工业级设备动力学与健康度状态数据字典

```sql
-- 1. 关键转动设备与轴承运动学参数主表
CREATE TABLE `pdm_device_kinematics` (
  `device_id` VARCHAR(32) NOT NULL COMMENT '设备统一资产编码 (如: CNC-MAZAK-01)',
  `component_code` VARCHAR(32) NOT NULL COMMENT '受监护部件 (如: SPINDLE_BEARING_FRONT)',
  `bearing_model` VARCHAR(64) NOT NULL COMMENT '轴承标准型号 (如: SKF 7014 CD/P4A)',
  `roller_count` INT NOT NULL COMMENT '滚动体数量 Z',
  `roller_diameter_mm` DECIMAL(6,3) NOT NULL COMMENT '滚动体直径 d',
  `pitch_diameter_mm` DECIMAL(6,3) NOT NULL COMMENT '轴承节圆直径 D',
  `contact_angle_deg` DECIMAL(4,1) NOT NULL DEFAULT 15.0 COMMENT '接触角(度)',
  `nominal_rpm` INT NOT NULL DEFAULT 6000 COMMENT '额定工作转速',
  `iso_vibration_class` ENUM('CLASS_I', 'CLASS_II', 'CLASS_III', 'CLASS_IV') NOT NULL DEFAULT 'CLASS_II' COMMENT 'ISO 10816 刚性基础设备分类',
  `created_at` TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
  PRIMARY KEY (`device_id`, `component_code`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='设备传动动力学机理参数表';

-- 2. 边缘时钟小时级聚合振温时序流水表
CREATE TABLE `pdm_vibration_feature_hourly` (
  `id` BIGINT UNSIGNED NOT NULL AUTO_INCREMENT,
  `device_id` VARCHAR(32) NOT NULL,
  `component_code` VARCHAR(32) NOT NULL,
  `velocity_rms` DECIMAL(6,3) NOT NULL COMMENT '振动速度有效值 (mm/s)',
  `acceleration_kurtosis` DECIMAL(6,2) NOT NULL COMMENT '加速度峭度值',
  `acceleration_crest` DECIMAL(6,2) NOT NULL COMMENT '峰值因子',
  `temperature_celsius` DECIMAL(5,1) NOT NULL COMMENT '轴承座表面温度',
  `dominant_freq_hz` DECIMAL(8,2) DEFAULT NULL COMMENT '最高能量谱峰频率(Hz)',
  `matched_fault_type` ENUM('NONE', 'BPFO', 'BPFI', 'BSF', 'UNBALANCE', 'MISALIGNMENT') NOT NULL DEFAULT 'NONE',
  `captured_at` TIMESTAMP(3) NOT NULL DEFAULT CURRENT_TIMESTAMP(3),
  PRIMARY KEY (`id`),
  KEY `idx_dev_time` (`device_id`, `component_code`, `captured_at`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='设备振温时序特征流水表';

-- 3. 设备劣化生命周期状态机控制表
CREATE TABLE `pdm_degradation_state` (
  `device_id` VARCHAR(32) NOT NULL,
  `component_code` VARCHAR(32) NOT NULL,
  `current_state` ENUM('HEALTHY', 'EARLY_WARN', 'DEVELOPING', 'CRITICAL') NOT NULL DEFAULT 'HEALTHY',
  `health_score` DECIMAL(4,1) NOT NULL DEFAULT 100.0 COMMENT '综合健康指数 (0-100)',
  `last_transition_time` TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
  `action_ticket_id` VARCHAR(32) DEFAULT NULL COMMENT '关联的 MES/WMS 检修或备件工单号',
  `is_hard_interlocked` TINYINT(1) NOT NULL DEFAULT 0 COMMENT '是否触发物理开工互锁',
  PRIMARY KEY (`device_id`, `component_code`),
  KEY `idx_state` (`current_state`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='设备劣化健康状态机管理表';
```

---

### 五、 核心原子事务：时序指标研判与排程自动熔断

```typescript
import { DataSource, QueryRunner } from 'typeorm';

export class PdmLifecycleManager {
  constructor(private dataSource: DataSource) {}

  /**
   * 提交小时级特征并执行状态机推进事务
   */
  public async ingestMetricsAndEvaluateFsm(
    deviceId: string,
    componentCode: string,
    velocityRms: number,
    kurtosis: number,
    matchedFault: string
  ): Promise<{ stateChanged: boolean; targetState: string }> {
    const queryRunner: QueryRunner = this.dataSource.createQueryRunner();
    await queryRunner.connect();
    await queryRunner.startTransaction();

    try {
      // 1. 获取当前状态并加悲观排他锁
      const states = await queryRunner.query(
        `SELECT current_state, is_hard_interlocked FROM pdm_degradation_state 
         WHERE device_id = ? AND component_code = ? FOR UPDATE`,
        [deviceId, componentCode]
      );

      const currentState = states.length > 0 ? states[0].current_state : 'HEALTHY';
      let nextState = currentState;

      // 2. 基于物理机理指标研判目标状态 (ISO 10816 Class II: 4.5 为警告, 7.1 为危险停机)
      if (velocityRms >= 7.1) {
        nextState = 'CRITICAL';
      } else if (velocityRms >= 4.5 || matchedFault !== 'NONE') {
        nextState = 'DEVELOPING';
      } else if (kurtosis >= 4.5) {
        nextState = 'EARLY_WARN';
      } else {
        nextState = 'HEALTHY';
      }

      const stateChanged = nextState !== currentState;

      if (stateChanged) {
        let ticketUuid: string | null = null;
        let lockDevice = false;

        if (nextState === 'DEVELOPING') {
          ticketUuid = `MAINT_PLAN_${Date.now()}`;
          // 联动 WMS 预锁备品备件库房库存
          await queryRunner.query(
            `INSERT INTO wms_spare_parts_reservation (ticket_id, device_id, component_code, status)
             VALUES (?, ?, ?, 'RESERVED_FOR_INSPECTION')`,
            [ticketUuid, deviceId, componentCode]
          );
        } else if (nextState === 'CRITICAL') {
          ticketUuid = `MAINT_EMERG_${Date.now()}`;
          lockDevice = true;

          // 核心联动：下发工序互锁，在 MES/APS 中熔断该机床可用性
          await queryRunner.query(
            `UPDATE aps_resource_time_slot 
             SET is_hard_locked = 1 
             WHERE resource_id = ? AND start_time >= NOW()`,
            [deviceId]
          );
        }

        // 更新状态机主表
        await queryRunner.query(
          `UPDATE pdm_degradation_state 
           SET current_state = ?, action_ticket_id = ?, is_hard_interlocked = ?, last_transition_time = NOW()
           WHERE device_id = ? AND component_code = ?`,
          [nextState, ticketUuid, lockDevice ? 1 : 0, deviceId, componentCode]
        );
      }

      await queryRunner.commitTransaction();
      return { stateChanged, targetState: nextState };
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

**1. 传感器现场安装刚度陷阱（磁吸座 vs 螺纹打孔连接）**  
现场实施时，很多外协团队为图省事，直接拿强力磁吸座将高频加速度传感器吸附在电机机壳漆面上。  
*物理原理大坑*：磁吸座在超过 2kHz–3kHz 频率段时会发生机械共振脱离，导致采集的高频加速度冲击信号出现严重的假峰值和高频滤波失真，彻底废掉峭度计算。  
*工程落地标准*：轴承座监护点必须在设备死区处**机械打孔并攻 M5/M8 螺纹，使用双螺纹双向不锈钢螺栓刚性拧紧连接**，接触面打磨涂抹硅脂消除微间隙，保证传感器在 10Hz–10kHz 频响范围内绝对刚性。

**2. 变频与变负载工况的“动态转速基准归一化”**  
数控机床主轴转速是实时变化的（如车削与精铣不同切削工步），轴承特征频（BPFO/BPFI）是转频 $f_r$ 的线性函数。如果在固定转速假设下做特征匹配，转速微调 50 转就会导致特征谱峰完全偏离，产生灾难性误判。  
*底层设计准则*：边缘网关必须通过 Modbus-TCP 或 PLC 内部变量读取**实时主轴编码器转速（S指令与实际反馈）**。只有在主轴转速平稳运行超过 30 秒的时隙内，方可启动特征频比对；并将频谱横轴归一化为阶次（Order Analysis），杜绝转速波动引发的假阳性报警。

**3. 现场齿轮箱“边频带调制（Sideband）”识别逻辑**  
对于减速齿轮箱，单一齿轮局部断齿或点蚀除了产生齿轮啮合基频（$\text{GMF} = Z_{\text{gear}} \times f_r$）外，最核心的物理特征是产生以啮合频率为中心、以齿轮所在轴转频为间距的**对称边频带簇**。状态机在研判齿轮故障时，严禁仅盯 GMF 能量大小，必须编写算法自动扫描主频两侧是否存在等间距侧峰，方可确诊断齿。

---

*文献编号：`XG-PDM-2026-19`｜西安旭辉西格网络科技有限公司企业级架构规范*  
*官方技术沉淀仓库：[GitHub / Xuhui-Xige-Tech / Enterprise-Architecture-Best-Practices](https://github.com/Xuhui-Xige-Tech/Enterprise-Architecture-Best-Practices)*  
*实体交流地址：陕西省西安市高新区唐兴数码*
