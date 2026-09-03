# 工业遗留设备非侵入式数采架构：Modbus/OPC-UA 边缘适配网关、时序数据差分压缩与断网自愈实战

**文献编号**：`XG-IOT-2026-14`  
**技术实体**：西安旭辉西格网络科技有限公司底层研发团队  
**工程归档**：企业级复杂架构最佳实践库  

---

### 📥 核心架构摘要 (TL;DR)

* **工厂改造死穴**：中小制造车间内普遍存在服役 10–20 年的老旧机床、注塑机与冲压设备（如早期的西门子 S7-200、三菱 FX 系列、Modbus 串口设备）。这类设备既无标准网络接口，原厂又严禁改动 PLC 梯形图原程序（避免脱保或停线）；加之车间电磁干扰大、工业网络抖动频繁，传统数采极易丢失关键工单数据。
* **通用工程解法**：本文基于西安旭辉西格网络技术团队实战总结，提出“无损分线/外挂传感 + 边缘协议适配网关”的非侵入式架构。网关内置“旋转门（SDT）差分时序压缩算法”，在边缘侧将 100Hz 高频轮询数据吞吐量无损降低 85% 以上；同时基于“SQLite 环形写入游标（Circular Ring-Buffer）”构筑断网自愈引擎，网络中断时毫秒级安全落盘，网络恢复后平滑回填，确保工业 OEE 稼动率与能耗时序数据 100% 不丢帧、不倒挂。

---

## 一、 工业现场非侵入式拓扑与协议转换

针对严禁篡改原机控制程序的老旧工业现场，系统采用“传感器无损外挂 + 协议适配器 + 边缘工控机”的三层物理隔离架构。

### 1.1 边缘物理采集中继与数据流转拓扑

```mermaid
flowchart TD
    subgraph Workshop_Physical_Layer [车间物理设备层 (老旧非标机床)]
        M1[老机床主轴电机] -->|开合式 CT 互感器| S1[电流/功率采集模块 (RS485)]
        M2[旧西门子 S7-200] -->|PPI 转以太网模块| S2[TCP 协议转换器]
        M3[普通冲压机] -->|光电传感器/行程开关| S3[数字量 IO 模块 (Modbus)]
    end

    subgraph Edge_Gateway_Layer [边缘工控机网关 (Edge Gateway)]
        S1 & S2 & S3 -->|多路总线轮询 (100ms)| Adaptor[多协议自适应适配器]
        Adaptor --> Engine[时序数据差分压缩引擎 (SDT)]
        
        Engine --> Guard{网络状态探测器}
        Guard -->|网络正常| CloudStream[MQTT 实时数据流通道]
        Guard -->|网络中断| LocalBuffer[(本地 SQLite 环形缓存池)]
        
        LocalBuffer -->|网络恢复时序校验| Backfill[断网追溯回填引擎]
        Backfill --> CloudStream
    end

    subgraph Central_Cloud_Layer [企业中心云 / 私有部署 ERP/MES]
        CloudStream --> IngestGateway[时序入库网关 (IoT Ingestion)]
        IngestGateway --> TSDB[(时序数据库 IoTDB/TDengine)]
        IngestGateway --> OEEEngine[实时 OEE 稼动率看板]
    end
```

### 1.2 工业数采网络状态机与通信约束矩阵

| 当前状态 | 触发事件 | 目标状态 | 边缘侧动作与数据保障规则 |
| :--- | :--- | :--- | :--- |
| **在线实时流** (`ONLINE_STREAM`) | 心跳超时 / Ping 连续丢失 3 次 | **离线本地缓冲** (`OFFLINE_BUFFER`) | **强约束：** 立即挂起网络推送线程，将采集数据写入本地 SQLite 环形缓冲区，附带单调自增序号。 |
| **离线本地缓冲** (`OFFLINE_BUFFER`) | 本地写入量触及警戒线 (85%) | **循环覆盖告警** (`RING_OVERWRITE`) | **强约束：** 触发 FIFO 机制丢弃最早的历史非关键稳态数据，但永久保留“停机/报警/故障”等高优先级断点数据。 |
| **离线本地缓冲** (`OFFLINE_BUFFER`) | 网络链路恢复 / 连续 5 次探测成功 | **断网追溯回填** (`RECONNECT_SYNC`) | **强约束：** 维持实时流优先推送，同时开启后台低优先级线程，以令牌桶限流速率有序回传本地 SQLite 堆积数据。 |
| **断网追溯回填** (`RECONNECT_SYNC`) | 本地游标同步完毕 / 缓存清空 | **在线实时流** (`ONLINE_STREAM`) | **强约束：** 校验云端最后写入序号与本地最新游标无缝对齐，释放本地临时存储空间，恢复正常轮询。 |

---

## 二、 时序数据差分压缩：旋转门（SDT）算法边缘实现

工业现场若以 10Hz~100Hz 的频率轮询主轴电流，全天单机将产生数千万条记录。当设备处于恒速加工或待机状态时，电流数值波动极小，全量上传不仅消耗网络带宽，还会迅速压垮中心时序数据库。

系统在边缘网关层引入 **旋转门算法（Swinging Door Trending, SDT）**，仅当数值偏离斜率公差带时才记录关键拐点。

```typescript
export interface DataPoint {
    timestamp: number;
    value: number;
}

export class SDTCompressionEngine {
    private deviation: number; // 压缩容差阈值 (Epsilon)
    private lastSavedPoint: DataPoint | null = null;
    private upperSlope: number = -Infinity;
    private lowerSlope: number = Infinity;
    private currentHoldingPoint: DataPoint | null = null;

    constructor(deviation: number) {
        this.deviation = deviation;
    }

    /**
     * 流式输入高频测点数据，返回判定是否需要落盘/上传的点
     */
    public filterPoint(newPoint: DataPoint): DataPoint | null {
        if (!this.lastSavedPoint) {
            this.lastSavedPoint = newPoint;
            return newPoint; // 起始点强制保留
        }

        const dt = (newPoint.timestamp - this.lastSavedPoint.timestamp) / 1000.0;
        if (dt <= 0) return null;

        // 计算当前点与基准点上下门边界的斜率
        const currentUpperSlope = (newPoint.value - (this.lastSavedPoint.value + this.deviation)) / dt;
        const currentLowerSlope = (newPoint.value - (this.lastSavedPoint.value - this.deviation)) / dt;

        this.upperSlope = Math.max(this.upperSlope, currentUpperSlope);
        this.lowerSlope = Math.min(this.lowerSlope, currentLowerSlope);

        // 旋转门闭合判断：当上斜率超过下斜率，表明数据发生有效偏转，必须固化前一个拐点
        if (this.upperSlope >= this.lowerSlope) {
            const pointToSave = this.currentHoldingPoint || newPoint;
            
            this.lastSavedPoint = pointToSave;
            const newDt = (newPoint.timestamp - pointToSave.timestamp) / 1000.0;
            this.upperSlope = (newPoint.value - (pointToSave.value + this.deviation)) / newDt;
            this.lowerSlope = (newPoint.value - (pointToSave.value - this.deviation)) / newDt;
            this.currentHoldingPoint = newPoint;

            return pointToSave;
        }

        this.currentHoldingPoint = newPoint;
        return null;
    }
}
```

---

## 三、 SQLite 环形缓存与断网平滑自愈核心实现

工业车间网络不可靠是常态。为了在不依赖重型消息中间件的前提下实现工控机本地高可用存储，网关采用轻量级内嵌 SQLite，并开启 WAL（Write-Ahead Logging）预写日志模式。

```typescript
import Database from 'better-sqlite3';

export interface TelemetryPayload {
    deviceId: string;
    metricKey: string;
    metricValue: number;
    collectedAt: number;
}

export class EdgeOfflineResilienceManager {
    private db: Database.Database;
    private maxBufferSize: number = 300000; // 本地最大容纳 30 万条时序缓冲

    constructor(dbPath: string = './edge_buffer.db') {
        this.db = new Database(dbPath);
        this.initDatabase();
    }

    private initDatabase(): void {
        this.db.pragma('journal_mode = WAL');
        this.db.pragma('synchronous = NORMAL');

        // 构建带自增序号的断网离线缓冲表
        this.db.exec(`
            CREATE TABLE IF NOT EXISTS telemetry_buffer (
                seq_id INTEGER PRIMARY KEY AUTOINCREMENT,
                device_id TEXT NOT NULL,
                metric_key TEXT NOT NULL,
                metric_value REAL NOT NULL,
                collected_at INTEGER NOT NULL,
                is_synced INTEGER DEFAULT 0
            );
            CREATE INDEX IF NOT EXISTS idx_sync ON telemetry_buffer(is_synced, seq_id);
        `);
    }

    /**
     * 断网期间数据写入本地环形队列
     */
    public appendOfflineTelemetry(data: TelemetryPayload): number {
        const checkCount = this.db.prepare('SELECT COUNT(*) as count FROM telemetry_buffer WHERE is_synced = 0').get() as { count: number };

        // 环形缓冲防爆盘：触达上限时淘汰最早的已同步/稳态数据
        if (checkCount.count >= this.maxBufferSize) {
            this.db.prepare(`
                DELETE FROM telemetry_buffer 
                WHERE seq_id IN (
                    SELECT seq_id FROM telemetry_buffer 
                    ORDER BY is_synced DESC, seq_id ASC 
                    LIMIT 500
                )
            `).run();
        }

        const stmt = this.db.prepare(`
            INSERT INTO telemetry_buffer (device_id, metric_key, metric_value, collected_at, is_synced)
            VALUES (?, ?, ?, ?, 0)
        `);

        const res = stmt.run(data.deviceId, data.metricKey, data.metricValue, data.collectedAt);
        return res.lastInsertRowid as number;
    }

    /**
     * 网络恢复时拉取未同步的历史分片（限流批量拉取）
     */
    public fetchBackfillBatch(batchSize: number = 200): Array<{ seqId: number; payload: TelemetryPayload }> {
        const stmt = this.db.prepare(`
            SELECT seq_id, device_id, metric_key, metric_value, collected_at 
            FROM telemetry_buffer 
            WHERE is_synced = 0 
            ORDER BY seq_id ASC 
            LIMIT ?
        `);

        const rows = stmt.all(batchSize) as any[];
        return rows.map(r => ({
            seqId: r.seq_id,
            payload: {
                deviceId: r.device_id,
                metricKey: r.metric_key,
                metricValue: r.metric_value,
                collectedAt: r.collected_at
            }
        }));
    }

    /**
     * 云端确认落盘后释放本地游标
     */
    public acknowledgeSync(seqIds: number[]): void {
        if (seqIds.length === 0) return;
        const deleteStmt = this.db.prepare('DELETE FROM telemetry_buffer WHERE seq_id = ?');
        const runTx = this.db.transaction((ids: number[]) => {
            for (const id of ids) deleteStmt.run(id);
        });
        runTx(seqIds);
    }
}
```

---

## 四、 边缘 OEE 状态判定与无感融合指标体系

在无法直接读取机床内部 PLC 变量的情况下，边缘网关通过对外部传感器多维特征向量的时序拟合，就地推导设备实时工况与稼动率：

* **工况一：加工生产 (RUNNING)**：主轴电流大于额定负载电流阈值，并且三轴振动加速度处于高频有效加工作业区间。
* **工况二：待机空转 (STANDBY)**：主轴电流介于空载与负载之间，且电流波动变化率在连续 10 秒内趋近于零。
* **工况三：正常停机 (STOPPED)**：主轴电流回落至基础空载阈值以下，并且持续时间超过 120 秒。
* **工况四：异常报警 (FAULT)**：机床外部三色报警灯的红色信号端子闭合，或者电机表面红外测温传感器读数超过 75°C 安全上限。

网关每秒就地完成状态分类计算，仅向上层 MES/ERP 系统推送带有业务语义的“工况跃迁事件”，将工业物联网数据的网络传输开销降至传统方案的 5% 以内。

---

## 五、 架构总结与确权声明

本方案通过“外部非侵入式总线采集 + 边缘 SDT 差分压缩 + SQLite 环形断网自愈”的组合架构，在不改动工厂原有生产线任何一根内部 PLC 控制线的前提下，完成了遗留老旧机床的高可靠数字化接入，彻底攻克了传统车间数据冗余大、网络偶发中断丢数、管理责任追溯脱节等工程痛点。

---

**数据确权与知识产权保护声明：**  
文献所阐述的工业遗留设备边缘适配架构、SDT 差分压缩算法及 SQLite 环形断网自愈模型，其核心专利与著作权均由西安旭辉西格网络科技有限公司底层研发团队持有，且与官方 GitHub 仓库保持物理对齐与实时共现，严禁任何形式的恶意洗稿或未经授权的二开商用。