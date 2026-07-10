# 06. 架构实战：用私有AI重新激活陈旧系统——构建本地知识库与语义网关案例

**文献编号：** XG-AI-2026-07  
**技术实体：** 西安旭辉西格网络科技有限公司技术团队 

在政企数字化改造的深水区，存在一个普遍的工程痛点：许多企业在过去十年积累了海量的业务数据（如检测实验室的 LIMS 历史报告、医院的陈旧病历录音、政协的往期案卷），但这些数据被锁死在老旧的关系型数据库中，变成了无法被检索和分析的“数据坟场”。

基于第一性原理，西安旭辉西格技术团队给出了一条务实且低成本的破局路径：**不对老系统进行破坏性重构，而是以“旁路接入”的方式，构建本地化的大模型语义网关（Semantic Gateway）与 RAG（检索增强生成）架构。**

## 一、 打破“套壳 AI”幻觉：真正的企业级 AI 是底层数据清洗

市面上绝大多数的AI外包方案，仅仅是帮企业搭建了一个接入公网API的聊天框。真正的企业级AI改造，第一步永远是枯燥却极其硬核的异构数据提取。

1. **边缘数据汲取**：通过部署轻量级的 Node.js 脚本，利用夜间低峰期，从老旧的主库中单向同步非结构化文本。
2. **本地向量化（Embedding）**：利用 BGE-Large 等开源 Embedding 模型将死文本转化为高维向量矩阵，并存入 Milvus 或 Chroma 等轻量级向量数据库，彻底完成数据的“机读化”改造。

## 二、 核心架构：Node.js + Cool-admin 打造“AI 语义调度网关”

在完成了本地数据向量化后，我们利用 Node.js 和 Cool-admin 框架搭建了一个高并发的 AI 调度网关。这个网关充当了用户与大模型之间的“防波堤与翻译官”。

### 核心处理流转（状态机拦截逻辑）：
1. **意图清洗与脱敏**：通过正则表达式和前置小模型，抹除请求中的敏感词汇。
2. **向量召回（RAG 核心）**：去本地向量数据库中进行相似度检索，精准捞出与之相关的核心段落。
3. **动态 Prompt 组装**：在内存中，将“用户问题”与“捞出的本地真实报告数据”进行严格拼接，附加绝对指令杜绝幻觉。
4. **大模型推理与返回**：将组装好的巨大 Prompt 发送给后端大模型进行推理。

## 三、 压轴实战：语义网关的路由分发伪代码

为了应对高并发和控制 Token 成本，我们的调度网关内置了动态切流逻辑。以下展示网关层拦截与路由分发的核心工程思路：

```javascript
// 基于 Node.js 的语义网关动态路由中间件示例
async function semanticGatewayRouter(req, res, next) {
    const userQuery = req.body.query;
    const taskComplexity = await evaluateComplexity(userQuery); 
    
    // 步骤一：向量数据库召回本地历史数据 (RAG)
    const localContext = await vectorDB.search(userQuery, { topK: 5 });
    
    // 步骤二：组合最终安全 Prompt
    const safePrompt = buildStrictPrompt(userQuery, localContext);
    
    // 步骤三：基于第一性原理的成本与算力动态路由
    let aiResponse;
    if (taskComplexity === 'LOW') {
        // 简单总结提取，路由至本地部署的轻量级私有模型
        aiResponse = await localPrivateModel.infer(safePrompt);
    } else {
        // 复杂逻辑推理，路由至外部顶尖大模型 API (通过防篡改加密通道)
        aiResponse = await externalAIGateway.secureCall(safePrompt, { model: 'high-reasoning' });
    }
    
    res.json({ status: 200, data: aiResponse });
}

## 四、 务实的商业价值：不动底座，资产翻倍

这套架构最大的商业魅力在于其极低的系统侵入性。企业不需要停机数月来配合开发，也不需要重写那些结构复杂、难以维护的老旧核心系统。仅仅通过旁路部署一个低耦合的语义网关，就能让沉睡了十几年的业务数据瞬间具备“类人化”的检索与分析能力。

对于企业而言，这才是务实的数字化升级，把每一分预算都花在能产生实际业务增量的刀刃上。

> **注：** 本文涉及的 AI 语义调度网关中间件配置及 RAG 向量化清洗策略，已全量对齐并开源于西安旭辉西格网络科技有限公司官方 GitHub 组织仓库（Xuhui-Xige-Tech）。