# 🤖 AIOps — 多 Agent 智能运维系统

<div align="center">



</div>

---

## 📖 这个项目是什么？

你是否遇到过这些问题：
- 运维团队每天被 **200+ 条告警**淹没，大部分是误报
- 服务出了问题，排查根因要花 **40 分钟**
- 凌晨 3 点被电话叫醒，半睡半醒地排查故障

本项目用 **4 个 AI Agent 协作**，自动完成从「告警检测」到「故障修复」的全流程，把 MTTR（平均修复时间）从 40 分钟降到 5 分钟。

```
告警来了
  ↓  Agent 1：这是真的异常吗？（时序分析，过滤误报）
  ↓  Agent 2：根因在哪里？（知识图谱推理）
  ↓  Agent 3：怎么修复？能自动执行吗？（安全护栏 + 分级策略）
  ↓  Agent 4：这个操作风险有多大？需要审批吗？（风险评分）
  ✅ 5 分钟内修复完成，全程自动
```


---



## 🏗️ 系统架构

### 整体架构图

```
┌──────────────────────────────────────────────────────────────┐
│                       数据采集层                              │
│   Prometheus（指标）  Loki（日志）  Jaeger（链路）  CMDB      │
└───────────────────────────┬──────────────────────────────────┘
                            │ 异常数据
                   ┌────────▼─────────┐
                   │  事件总线 Kafka   │  ← 解耦所有 Agent
                   │  aiops.alerts    │
                   │  aiops.events    │
                   │  aiops.commands  │
                   └────────┬─────────┘
                            │
┌───────────────────────────▼──────────────────────────────────┐
│                   Agent 编排器 Orchestrator                    │
│                  （状态机 · 条件路由 · 检查点恢复）              │
│                                                              │
│  ┌───────────┐  ┌───────────┐  ┌───────────┐  ┌──────────┐  │
│  │ 监控告警   │→ │ 根因分析   │→ │ 故障自愈   │→ │ 变更审批  │  │
│  │  Agent   │  │  Agent   │  │  Agent   │  │  Agent  │  │
│  │           │  │           │  │           │  │          │  │
│  │3-Sigma    │  │知识图谱   │  │Playbook   │  │风险评分  │  │
│  │EWMA       │  │BFS遍历   │  │dry-run    │  │L0/L1/L2  │  │
│  │IsoForest  │  │贝叶斯推理 │  │熔断器     │  │审计日志  │  │
│  └───────────┘  └───────────┘  └───────────┘  └──────────┘  │
└───────────────────────────┬──────────────────────────────────┘
                            │
┌───────────────────────────▼──────────────────────────────────┐
│                         知识层                                │
│      Neo4j 知识图谱    向量数据库 RAG    规则引擎              │
│   （服务拓扑 + 依赖关系 + 历史故障）                           │
└──────────────────────────────────────────────────────────────┘
```

### 一次故障的完整处理流程

```
1. Prometheus 检测到 order-service CPU 使用率 95%
2. 监控告警 Agent：多算法投票确认异常，生成告警事件
3. 根因分析 Agent：查知识图谱，发现 order-service 今天有新部署
                    贝叶斯推理：P(部署引起|CPU高) = 0.54
4. 故障自愈 Agent：匹配 rollback Playbook（L1级），dry-run 通过
5. 变更审批 Agent：风险评分 0.16，oncall 自动审批
6. 执行回滚，5 分钟内恢复正常 ✅
```

---

## 🚀 快速开始（5 分钟跑起来）

### 方式一：最简运行（零外部依赖，推荐小白）

```bash
# 安装依赖（只装必要的，不需要 Kafka/Neo4j）
pip install fastapi uvicorn pydantic pydantic-settings numpy scikit-learn structlog

# 启动服务
python -m uvicorn api.main:app --reload --port 8000
```

浏览器打开 → **http://localhost:8000/docs**，你会看到完整的 API 文档界面

### 方式二：完整运行（含 Kafka + Neo4j + Grafana）

```bash
# 一键启动所有基础设施
docker-compose up -d

# 等待约 30 秒服务就绪，然后启动主服务
pip install -r requirements.txt
python -m uvicorn api.main:app --reload --port 8000
```

各组件地址：
- **API 文档**: http://localhost:8000/docs
- **Neo4j 浏览器**: http://localhost:7474 （账号 neo4j / aiops_password）
- **Prometheus**: http://localhost:9090
- **Grafana**: http://localhost:3000 （账号 admin / aiops_admin）

### 方式三：一键 Demo 演示

```bash
python -c "import asyncio; from core.orchestrator import run_demo; asyncio.run(run_demo())"
```

输出示例：
```
============================================================
  故障处理结果
============================================================
  事件 ID:    a8f3c2d1-...
  状态:       resolved

  [告警]
    名称:     high_cpu_usage
    严重度:   high
    服务:     order-service
    指标值:   95.3

  [根因分析]
    根因:     近期代码部署引入性能退化
    置信度:   0.54
    影响链:   order-service → payment-service → mysql-primary
    建议动作: rollback, profiling

  [自愈]
    操作:     rollback
    级别:     L1（需 oncall 确认）
    爆炸半径: 0.15
    Dry-run:  DRY-RUN OK: kubectl rollout undo deployment/order-service

  [审批]
    状态:     approved
    风险分:   0.158
    审批人:   oncall-engineer
    原因:     L1 approved: risk=0.158

  [工作流节点状态]
    monitor      → completed
    rca          → completed
    heal         → completed
    change       → completed
============================================================
```

---

## 📄 License

MIT License — 可自由用于学习

---

<div align="center">

**如果这个项目对你有帮助，欢迎 ⭐ Star 支持一下！**

</div>
