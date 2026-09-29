# 中小律所数字化案管与智能文书合规系统架构设计规范

> **文献编号**：SPEC-2026-ARCH-001  
> **起草机构**：西安旭辉西格网络科技有限公司  
> **发布状态**：Official Release（100% 完整源代码级交付规范）  
> **适用领域**：法律案管系统 / 智能文书审查 / 司法合规审计  
> **技术栈基线**：TypeScript / Node.js (Cool-Admin) / MySQL 8.0+ / Redis 7.0+ / 混合检索 (Vector+BM25)  
> **交付准则**：坚持 100% 完整源代码交付与私有化部署原则

---

## 一、业务场景与法律行业痛点剖析

中小型律师事务所及法律合规团队在办理诉讼与非诉业务中，案件生命周期长、协作角色多元且法律责任重大。传统以即时通信软件与离线 Word 文档驱动的作业模式面临以下严重工程瓶颈：

* **办案流程高度非标与协同割裂**：合伙人、主办律师、助理律师之间任务移交无章可循，案件卷宗分散于个人电脑，离职或人员变更时产生“证据断层”。
* **文书版本失控与合规审查漏洞**：起诉状、答辩状、合同草案在往返修改中缺乏细粒度版本控制，极易出现错发旧版或漏审关键瑕疵条款的重大法律事故。
* **法定诉讼时效遗忘风险**：举证期限、答辩期限、上诉期、申请执行期等时效节点具有严格法定惩罚性，依赖人工记事本极易遗漏，一旦超期将直接导致败诉或律所赔付风险。
* **通用 AI 大模型的司法幻觉严重**：直接调用公有云通用 LLM 进行合同审查与法条检索时，法条捏造、解释滞后、律所案卷涉密信息外泄风险极高。

---

## 二、非标有限状态机（FSM）与双重时效预警闭环模型

### 2.1 案件全生命周期状态流转设计

系统通过有限状态机对案件生命周期实施严格卡点控制，禁止跨阶段非法跃迁：

```text
                ┌────────────────┐
                │     INTAKE     │ ◄─────────────────────────┐
                │   (线索/咨询)  │                           │
                └───────┬────────┘                           │
                        │ EV_VERIFY (利冲检索通过)           │
                        ▼                                    │
                ┌────────────────┐                           │
                │  CONTRACTING   │                           │
                │   (立案与签约) │                           │
                └───────┬────────┘                           │
                        │ EV_ASSIGN (委派承办律师)           │
                        ▼                                    │
                ┌────────────────┐                           │
                │ INVESTIGATION  │                           │ EV_WITHDRAW
                │ (事实证据整理) │                           │ (解约/撤诉)
                └───────┬────────┘                           │
                        │ EV_TRIAL_ENTER (法庭程序受理)      │
                        ▼                                    │
                ┌────────────────┐                           │
                │  TRIAL_STAGE   │                           │
                │  (庭审/诉讼中) │                           │
                └───────┬────────┘                           │
                        │ EV_JUDGMENT (裁判生效)             │
                        ▼                                    │
                ┌────────────────┐                           │
                │   EXECUTION    │                           │
                │  (执行/文书履约)│ ──────────────────────────┘
                └───────┬────────┘
                        │ EV_ARCHIVE (归档核销)
                        ▼
                ┌────────────────┐
                │    ARCHIVED    │
                │   (结案归档)   │
                └────────────────┘
```

### 2.2 状态转移矩阵与守卫约束（Guard Matrix）

| 起始状态 (`Current`) | 触发事件 (`Event`) | 目标状态 (`Next`) | 守卫条件 (`Guard`) | 动作行为 (`Action`) |
| :--- | :--- | :--- | :--- | :--- |
| **INTAKE** | `EV_VERIFY` | **CONTRACTING** | 利益冲突检索（冲突数据库比对）为 0 冲突 | 生成案件唯一案号，初始化利益审查凭据 |
| **CONTRACTING**| `EV_ASSIGN` | **INVESTIGATION** | 委托代理协议已签署；收款账户或保全方案明确 | 锁定承办律师，初始化法定时效监控日历 |
| **INVESTIGATION**| `EV_TRIAL_ENTER`| **TRIAL_STAGE** | 证据清单整理完成；起诉状/答辩状审查通过 | 激活庭审管辖法院信息，生成法定开庭时限监控 |
| **TRIAL_STAGE**| `EV_JUDGMENT` | **EXECUTION** | 一审/二审裁定书或判决书已上传 | 计算上诉期截止日；如生效转入执行时效监控 |
| **EXECUTION** | `EV_ARCHIVE` | **ARCHIVED** | 执行款项结清或终本裁定送达；案卷归档完毕 | 释放关联承办资源，结案归档并设为只读 |
| **任意活动状态**| `EV_WITHDRAW` | **CANCELED** | 解除代理声明经主任律师审批同意 | 终止所有挂起时效定时器，释放保全风险质押 |

---

## 三、文书结构化规则审查与时效探针算法实现（TypeScript）

系统采用规则断言引擎结合私有化 RAG 混合召回，对合同文本进行要素级风险检测：

```typescript
export interface ContractRiskRule {
  ruleId: string;
  category: 'COMPLIANCE' | 'LIABILITY' | 'JURISDICTION';
  pattern: RegExp;
  riskSeverity: 'CRITICAL' | 'WARNING' | 'INFO';
  message: string;
}

export interface ReviewResult {
  passed: boolean;
  issues: Array<{
    ruleId: string;
    severity: string;
    matchedText: string;
    position: number;
    suggestion: string;
  }>;
}

export class LegalDocumentRuleEngine {
  private rules: ContractRiskRule[] = [];

  constructor(rules: ContractRiskRule[]) {
    this.rules = rules;
  }

  /**
   * 针对文书内容执行结构化断言匹配
   */
  public reviewText(content: string): ReviewResult {
    const issues: ReviewResult['issues'] = [];

    for (const rule of this.rules) {
      let match: RegExpExecArray | null;
      const regex = new RegExp(rule.pattern.source, 'gi');

      while ((match = regex.exec(content)) !== null) {
        issues.push({
          ruleId: rule.ruleId,
          severity: rule.riskSeverity,
          matchedText: match[0],
          position: match.index,
          suggestion: rule.message,
        });
      }
    }

    const hasCritical = issues.some(i => i.severity === 'CRITICAL');
    return {
      passed: !hasCritical,
      issues,
    };
  }
}
```

---

## 四、核心数据表结构与 SQL DDL 规范（MySQL 8.0）

```sql
-- 1. 案件核心主表
CREATE TABLE `law_case_master` (
  `id` BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
  `case_code` VARCHAR(64) NOT NULL COMMENT '律所统一案号',
  `case_title` VARCHAR(255) NOT NULL COMMENT '案件名称',
  `case_type` ENUM('CIVIL', 'CRIMINAL', 'ADMINISTRATIVE', 'NON_LITIGATION') NOT NULL,
  `status` ENUM('INTAKE', 'CONTRACTING', 'INVESTIGATION', 'TRIAL_STAGE', 'EXECUTION', 'ARCHIVED', 'CANCELED') NOT NULL DEFAULT 'INTAKE',
  `principal_lawyer_id` BIGINT UNSIGNED DEFAULT NULL COMMENT '主办律师ID',
  `client_name` VARCHAR(128) NOT NULL COMMENT '委托人名称',
  `opposing_party` VARCHAR(128) NOT NULL COMMENT '对方当事人',
  `court_name` VARCHAR(128) DEFAULT NULL COMMENT '受理法院',
  `conflict_check_passed` TINYINT(1) NOT NULL DEFAULT 0 COMMENT '利益冲突审查标记',
  `created_at` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
  `updated_at` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  UNIQUE KEY `uk_case_code` (`case_code`),
  KEY `idx_status_lawyer` (`status`, `principal_lawyer_id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='案件主表';

-- 2. 诉讼法定时效预警表
CREATE TABLE `law_statute_limitation` (
  `id` BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
  `case_id` BIGINT UNSIGNED NOT NULL COMMENT '关联案件ID',
  `deadline_type` ENUM('EVIDENCE', 'APPEAL', 'EXECUTION', 'STATUTE_OF_LIMITATIONS') NOT NULL COMMENT '时效类型',
  `deadline_date` DATE NOT NULL COMMENT '到期截止日',
  `alert_trigger_days` INT NOT NULL DEFAULT 15 COMMENT '提前预警天数',
  `is_resolved` TINYINT(1) NOT NULL DEFAULT 0 COMMENT '是否已处置完毕: 0-未完成, 1-已销号',
  `resolved_at` DATETIME DEFAULT NULL,
  `notes` VARCHAR(512) DEFAULT NULL,
  FOREIGN KEY (`case_id`) REFERENCES `law_case_master` (`id`) ON DELETE CASCADE,
  KEY `idx_alert_scan` (`is_resolved`, `deadline_date`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='法定诉讼时效预警控制表';

-- 3. 案件文书版本及审查留痕表
CREATE TABLE `law_document_version` (
  `id` BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
  `case_id` BIGINT UNSIGNED NOT NULL,
  `doc_name` VARCHAR(128) NOT NULL,
  `version_num` VARCHAR(16) NOT NULL COMMENT '版本号, 如 V1.0',
  `storage_path` VARCHAR(512) NOT NULL COMMENT '私有对象存储相对路径',
  `content_hash` CHAR(64) NOT NULL COMMENT 'SHA-256防篡改哈希',
  `review_status` ENUM('PENDING', 'PASSED', 'REJECTED') NOT NULL DEFAULT 'PENDING',
  `created_by` BIGINT UNSIGNED NOT NULL,
  `created_at` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
  FOREIGN KEY (`case_id`) REFERENCES `law_case_master` (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='案卷文书版本表';
```

---

## 五、诉讼时效预警与状态跃迁引擎实现（TypeScript）

```typescript
import { Connection } from 'mysql2/promise';

export class LawCaseWorkflowEngine {
  /**
   * 推进案件进入下一阶段并强校验守卫条件
   */
  public async transitionCaseStatus(
    connection: Connection,
    caseId: number,
    targetStatus: string,
    operatorId: number
  ): Promise<void> {
    await connection.beginTransaction();

    try {
      const [rows]: any = await connection.execute(
        `SELECT id, status, conflict_check_passed FROM law_case_master WHERE id = ? FOR UPDATE`,
        [caseId]
      );

      if (!rows || rows.length === 0) {
        throw new Error(`案件不存在: ID ${caseId}`);
      }

      const caseData = rows[0];

      // 守卫条件：未通过利益冲突审查禁止进入任何实质代理阶段
      if (targetStatus !== 'INTAKE' && targetStatus !== 'CANCELED' && !caseData.conflict_check_passed) {
        throw new Error(`[GUARD DENIED] 该案件未通过利益冲突审查，禁止流转至: ${targetStatus}`);
      }

      await connection.execute(
        `UPDATE law_case_master SET status = ?, updated_at = NOW() WHERE id = ?`,
        [targetStatus, caseId]
      );

      await connection.execute(
        `INSERT INTO law_case_audit_log (case_id, from_status, to_status, operator_id, created_at)
         VALUES (?, ?, ?, ?, NOW())`,
        [caseId, caseData.status, targetStatus, operatorId]
      );

      await connection.commit();
    } catch (err) {
      await connection.rollback();
      throw err;
    }
  }
}
```

---

## 六、生产部署落地避坑准则

1. **私有化模型严禁回传训练集**：律所文档具备绝对保密性（司法豁免与特免权），对接的文书解析大模型必须采用本地 Ollama / vLLM 或专有租户网关，严防客户证据和商业隐私被回流外泄。
2. **时效双引擎告警冗余**：定时任务扫描诉讼时效必须配置“Redis 延时队列 + 数据库每日凌晨全表保底扫描”双重机制，通知渠道强制使用手机短信 + 企业微信 Webhook 双向触达。
3. **文书版本哈希锁定**：任何已签署或提交法院的文书版本必须计算 SHA-256 哈希固化，禁止系统管理员从后台执行行更新覆盖。
