# Data Analysis Agent 项目优化路线图

## 1. 项目定位

构建一个面向业务数据分析的 Text-to-SQL Agent，实现：

> 自然语言提问 → 意图理解 → 指标与 Schema 匹配 → 语义澄清 → SQL 生成与安全执行 → 图表规划与结论输出。

项目优化重点不是继续增加 Agent 数量，而是形成一条：

- 可控：关键步骤由 LangGraph 状态机编排；
- 可解释：能够说明指标、表和字段为什么被选择；
- 可澄清：信息不足时不盲目生成 SQL；
- 可恢复：SQL 失败后能够分类并有限修复；
- 可评测：通过测试集量化 Schema 召回率和 SQL 正确率。

---

## 2. 最终架构

```text
用户提问
  ↓
意图与原始槽位提取
  ↓
规则精确映射 + Schema RAG 候选召回
  ↓
Schema Linker LLM 受限精排
  ↓
语义澄清判断
  ├─ 存在歧义 → 向用户追问 → 恢复工作流
  └─ 信息充分
        ↓
Schema Registry 校验 + RelationGraph 补齐
        ↓
最小 Schema Context 构建
        ↓
SQL 生成
        ↓
AST 安全与 Schema 校验
        ↓
只读受限执行
  ├─ 执行失败 → 定向修复，最多 2 次
  └─ 执行成功
        ↓
结果校验、图表规划与结论输出
```

---

# P0：核心闭环，必须完成

目标：优先解决“SQL 为什么可信”这个面试核心问题，形成可以完整演示和评测的最小闭环。

## P0.1 收敛 LangGraph 工作流

- [ ] 保留 LangGraph 作为状态管理和流程编排框架。
- [ ] 将原来的 16 个节点收敛为 8～10 个职责明确的节点。
- [ ] 删除 ReAct 与 Coordinator 双架构分流。
- [ ] 使用条件边处理语义澄清、正常执行和失败修复。
- [ ] SQL 修复最多执行 2 次，避免无界 Agent Loop。
- [ ] 保存用户原问题、结构化意图、候选 Schema、SQL、错误信息和执行结果等状态。

建议节点：

```text
intent_parse
schema_retrieve
schema_link
clarification_gate
schema_context_build
sql_generate
sql_validate
sql_execute
sql_repair
result_analyze
```

验收标准：

- 普通查询可以完整走通生成、校验、执行和结果输出链路。
- SQL 生成或执行失败后最多修复 2 次并正常结束。
- 每个节点输入、输出和失败原因均可记录和回放。

---

## P0.2 统一意图与分析槽位

- [ ] 意图识别阶段不注入完整数据库 Schema。
- [ ] 只提取用户想分析的指标、维度、时间、过滤条件等原始表达。
- [ ] 不在意图识别阶段直接预测物理表名和字段名。
- [ ] 使用统一的 `AnalysisSpec` 在节点之间传递需求。

建议结构：

```json
{
  "intent_type": "ranking",
  "metric_mentions": ["销售额"],
  "dimension_mentions": ["城市"],
  "time_expression": "最近30天",
  "filter_mentions": [],
  "granularity": null,
  "comparison": null,
  "sort": "desc",
  "limit": 10
}
```

验收标准：

- 能识别指标查询、趋势、排行、对比、分布和明细查询。
- 能保留“销售额”“城市”等原始业务表达。
- 意图识别结果不包含模型猜测的数据库字段。

---

## P0.3 完善 MetricRegistry 指标知识库

继续使用现有 GMV、订单数、客单价等 8 类指标 YAML。

每个指标补充：

- [ ] 指标 ID 和标准名称；
- [ ] 自然语言别名；
- [ ] 业务定义；
- [ ] SQL 计算公式；
- [ ] 基础事实表；
- [ ] 默认时间字段；
- [ ] 依赖字段。

示例：

```yaml
id: gmv
name: GMV
aliases:
  - 成交额
  - 商品交易总额
  - 销售额
description: 用户下单产生的商品交易金额，不扣除退款
formula: SUM(orders.pay_amount)
base_table: orders
time_field: orders.paid_at
dependencies:
  - orders.pay_amount
  - orders.paid_at
```

设计边界：

- 指标知识库负责定义“指标怎么算”。
- 不为每个指标枚举所有可用维度。
- 不为每个指标维护大量查询模板。

验收标准：

- 8 类指标均能通过标准名称或别名定位。
- 每个指标引用的表和字段真实存在。
- SQL Generate 不能自行修改已定义的指标公式。

---

## P0.4 实现 Schema RAG 与 Schema Linking

目标：槽位本身不能通过规则直接定位 Schema，因此采用“确定规则 + 语义检索 + LLM 受限精排 + 程序校验”的四段式方案。

### SchemaRegistry：Schema 的事实来源

- [ ] 从 `information_schema` 同步表名、字段名、数据类型、字段注释和主外键。
- [ ] 为关键表补充业务 description 和数据粒度。
- [ ] 为关键字段补充 description、aliases 及与相似字段的区别。
- [ ] 所有表和字段使用稳定 ID，例如 `users.register_city`。
- [ ] 向量库只保存用于召回的语义文档和 Schema ID，不作为真实 Schema 的事实来源。

字段语义文档示例：

```json
{
  "id": "users.register_city",
  "type": "column",
  "table": "users",
  "column": "register_city",
  "data_type": "varchar",
  "description": "用户注册时填写的城市，用于分析用户来源地区",
  "aliases": ["注册城市", "用户城市"]
}
```

人工维护边界：

- 不要求一次性为所有字段编写完整业务描述。
- 数据库元数据和已有字段注释自动同步。
- 只重点维护高频、易混淆的业务字段。
- 根据评测错误逐步补充 aliases 和 description。

### CandidateRetriever：Schema RAG 候选召回

- [ ] 核心指标优先通过 MetricRegistry 的标准名称和别名精确匹配。
- [ ] 时间、Top-N、排序等确定表达通过规则解析。
- [ ] 表和字段分别建立 BM25/关键词索引与 Embedding 向量索引。
- [ ] 使用“精确匹配候选 ∪ BM25 Top-K ∪ Embedding Top-K”作为候选集合。
- [ ] 分别召回候选指标、候选表和候选字段。
- [ ] 记录每个候选来自精确规则、关键词检索还是语义检索。
- [ ] 初筛阶段追求高召回，不直接判定最终表字段。
- [ ] 不为每张表人工维护庞大的正则表达式集合。

设计原则：

```text
规则解决确定表达
BM25 解决关键词匹配
Embedding 解决长尾语义表达
候选并集保证正确 Schema 不被过早过滤
```

### SchemaLinker：LLM 受限精排

- [ ] 输入原始用户问题、AnalysisSpec、候选指标、候选表字段 description 和候选关系。
- [ ] 根据完整语义从候选集合中选择指标、表和字段。
- [ ] LLM 只能返回候选 ID，禁止自由生成表名、字段名和 JOIN 条件。
- [ ] 输出选择理由、歧义项及是否需要向用户澄清。
- [ ] 当“订单发生城市”“用户注册城市”等候选均合理时，进入 ClarificationGate。

建议输出：

```json
{
  "metric_id": "metric.gmv",
  "column_ids": [
    "orders.pay_amount",
    "dim_city.city_name"
  ],
  "relation_ids": ["orders_to_city"],
  "need_clarification": false,
  "ambiguities": [],
  "reason": "问题分析销售额排行，应按照订单发生城市聚合"
}
```

### SchemaValidator 与 RelationGraph

- [ ] 校验 LLM 返回的指标、表、字段和关系 ID 均属于候选集合。
- [ ] 从 SchemaRegistry 获取最新、完整的字段定义。
- [ ] 从主外键构建 RelationGraph。
- [ ] 对数据库未声明的关键逻辑关系补充少量配置。
- [ ] 根据已选表查找合法 JOIN 路径并补齐 JOIN Key。
- [ ] LLM 只能选择已有 relation ID，不能自行编造 JOIN 条件。
- [ ] 无合法连接路径时停止 SQL 生成并返回结构化原因。

Schema Linking 完整输出示例：

```json
{
  "metric": {
    "id": "metric.gmv",
    "formula": "SUM(orders.pay_amount)"
  },
  "dimensions": [
    {
      "mention": "城市",
      "column": "dim_city.city_name"
    }
  ],
  "tables": ["orders", "dim_city"],
  "join_path": [
    "orders.city_code = dim_city.city_code"
  ],
  "need_clarification": false
}
```

验收标准：

- 标准问题的正确表和字段能够进入检索 Top-K。
- Schema Linker 只能从候选 ID 中进行选择。
- 输出候选集合外 ID 时可以被程序拦截。
- 可以单独统计候选召回率和 LLM 精排准确率。
- JOIN 字段由 RelationGraph 校验和补齐，不依赖模型自由生成。

---

## P0.5 构建最小 Schema Context

新增 `SchemaContextBuilder`，只向 SQL Generate 注入必要内容。

需要合并的五类字段：

- [ ] 指标计算字段；
- [ ] 默认时间字段；
- [ ] 维度字段；
- [ ] 过滤字段；
- [ ] JOIN Key。

同时注入：

- [ ] 指标公式和业务定义；
- [ ] 相关表的数据粒度；
- [ ] 表关系；
- [ ] 已解析的时间范围；
- [ ] 排序和 Limit；
- [ ] SQL 安全约束。

设计原则：

- 不注入完整数据库 Schema。
- 不遗漏指标依赖字段、时间字段和 JOIN Key。
- Schema Linker LLM 负责从候选集合中精排表字段。
- SQL Generate 只接收校验后的最小 Schema，不再扫描候选全集。
- SchemaContextBuilder 从 Registry 获取最新 Schema，而不是直接信任向量检索文档。

验收标准：

- 每次 SQL 生成只注入相关表和字段。
- 对同一测试集统计 Prompt Token，并与完整 Schema 方案对比。
- 保留现有“单次 Prompt Token 降低约 40%”数据，并重新验证。

---

## P0.6 SQL 静态校验与安全执行

### SQLValidator

- [ ] 使用 SQL Parser/AST 解析 SQL。
- [ ] 只允许单条 `SELECT`。
- [ ] 禁止 INSERT、UPDATE、DELETE 和 DDL。
- [ ] 校验表名真实存在。
- [ ] 校验字段属于对应表。
- [ ] 校验 JOIN 关系合法。
- [ ] 校验指标公式没有被模型擅自修改。
- [ ] 禁止 `SELECT *`。
- [ ] 自动增加或限制 `LIMIT`。

### SafeExecutor

- [ ] 使用只读数据库账号。
- [ ] 设置查询超时。
- [ ] 限制最大返回行数。
- [ ] 拦截明显无约束的大表扫描。
- [ ] 对执行错误进行结构化分类。

### SQLRepair

- [ ] 修复上下文只包含原 SQL、错误类型、错误信息和相关 Schema。
- [ ] 修复后重新进入 AST 校验。
- [ ] 最多重试 2 次。
- [ ] 超过次数后返回明确失败原因。

验收标准：

- 写操作和不存在的字段可以在执行前被拦截。
- 字段名、语法等常见错误能够定向修复。
- Agent 不会无限循环或重复执行危险 SQL。

---

## P0.7 建立最小评测集

先构建 80～120 条 Text-to-SQL 测试样本，覆盖：

- [ ] 单表查询；
- [ ] 多表关联；
- [ ] 指标计算；
- [ ] 时间范围；
- [ ] 趋势、排行、对比；
- [ ] 过滤和分组；
- [ ] 错误字段；
- [ ] 无法回答的问题。

P0 核心指标：

- [ ] 指标映射准确率；
- [ ] Schema RAG 候选 Recall@3/Recall@5；
- [ ] Schema Linker 表选择与字段选择准确率；
- [ ] Schema Linker 候选集合外 ID 违规率；
- [ ] SQL 执行成功率；
- [ ] Execution Accuracy；
- [ ] SQL 修复成功率；
- [ ] 平均修复次数；
- [ ] 单次 Prompt Token；
- [ ] 端到端响应时间。

验收标准：

- 每次架构或 Prompt 修改后可以自动回归。
- 不只统计 SQL 能否执行，还要统计执行结果是否正确。
- 保留并重新验证当前“相关表结构命中率约 85%”的数据。

---

# P1：工程增强，提升项目亮点

目标：解决 Schema 变化、模糊语义和数据时效性问题，让系统从“Demo 可用”升级为“工程上更可靠”。

## P1.1 Schema 变更治理

将知识拆成两层：

- 数据库元数据：表、字段、类型、主外键，自动同步；
- 业务语义：指标口径、公式、别名，使用 YAML + Git 管理。

实现项：

- [ ] 启动时读取 `information_schema`。
- [ ] 为 Schema 计算 fingerprint/hash。
- [ ] Schema 变化后增量更新检索索引。
- [ ] 校验指标 YAML 引用的表和字段。
- [ ] 指标依赖失效后将其标记为不可用。
- [ ] 禁止使用失效指标继续生成 SQL。

验收标准：

- 新增表或字段后可以被索引发现。
- 删除或重命名字段后，对应指标不会静默生成错误 SQL。
- 能明确记录本次 Schema 变化影响了哪些指标。

---

## P1.2 语义澄清 ClarificationGate

按查询类型维护少量通用必要槽位：

| 查询类型 | 必要槽位 |
| --- | --- |
| 指标查询 | 指标 |
| 趋势分析 | 指标、时间范围 |
| 排行分析 | 指标、排行维度 |
| 对比分析 | 指标、对比对象 |
| 明细查询 | 业务对象 |
| 分布分析 | 指标、分组维度 |

触发澄清的情况：

- [ ] 必要槽位缺失且没有默认策略；
- [ ] 一个自然语言表达对应多个指标口径；
- [ ] 相对时间无法唯一解析；
- [ ] 候选字段代表不同业务含义；
- [ ] 候选表粒度不一致；
- [ ] 候选表无法通过关系图连接；
- [ ] 指标依赖已经失效。

澄清结果使用结构化原因：

```json
{
  "need_clarification": true,
  "reason_code": "MULTIPLE_METRIC_CANDIDATES",
  "slot": "metric",
  "candidates": ["GMV", "支付金额", "净收入"],
  "question": "你说的销售额是指 GMV、实际支付金额，还是扣除退款后的净收入？"
}
```

设计原则：

- 不使用 LLM 自报的统一置信度。
- 只有歧义会改变 SQL 语义时才追问。
- Top-N、默认排序等可安全推导的信息不追问。
- 用户回答后恢复原工作流，而不是重新开始一次请求。

验收标准：

- 能正确识别指标口径、时间和字段语义歧义。
- 澄清后可以恢复并完成 SQL 查询。
- 对清晰问题不会频繁产生无意义追问。

---

## P1.3 Schema Linking 效果增强

- [ ] 分析关键词检索与向量检索的召回差异。
- [ ] 对表级和字段级结果分别统计 Recall@K。
- [ ] 使用指标基础表作为 Schema 召回先验。
- [ ] 使用 RelationGraph 对候选表进行连通性过滤。
- [ ] 对高频错误字段补充 aliases 和 description。
- [ ] 记录无法召回、召回错误和候选冲突三类失败。

优化策略：

- 对精确别名匹配、BM25 和向量检索结果进行候选合并；
- 对语义检索候选保留 Top-K，避免初筛阶段过早丢失正确字段；
- 由 Schema Linker LLM 在候选范围内统一精排；
- 对 Schema Linker 输出执行候选 ID、字段存在性和关系连通性校验；
- 候选语义明显冲突时进入澄清，不让模型静默替用户决定口径。

验收标准：

- Schema Recall@3/Recall@5 高于 P0 基线。
- 错误案例能够反向指导 aliases 和 description 优化。
- 不通过增加大量 Prompt Token 换取召回率。

---

## P1.4 时间语义与数据新鲜度

新增两个轻量能力：

- [ ] `get_current_time()`：提供确定的当前时间；
- [ ] `get_data_freshness(table)`：查询数据的最新日期。

区分：

- “最近 7 天”：确定的相对时间，直接解析；
- “最新数据”：查询表的最大数据日期；
- “最近的数据”：范围不明确，追问 7 天或 30 天。

验收标准：

- 相对时间解析不依赖模型猜测当前日期。
- 查询结果能够展示数据更新截止时间。
- “最近”和“最新”不会被错误地当成相同概念。

---

## P1.5 扩充评测和错误分析

在 P0 测试集基础上增加：

- [ ] 模糊指标口径；
- [ ] 模糊时间范围；
- [ ] 同名字段；
- [ ] 不同数据粒度；
- [ ] Schema 新增、删除和重命名；
- [ ] 无法连接的候选表；
- [ ] 执行超时和结果为空；
- [ ] 用户澄清后的多轮恢复。

新增指标：

- [ ] 澄清触发准确率；
- [ ] 应澄清但未澄清的漏判率；
- [ ] 不必要澄清率；
- [ ] Schema 变化后的回归通过率；
- [ ] 各类错误占比；
- [ ] P50/P95 延迟。

验收标准：

- 每类失败都能归因到意图、指标、Schema、SQL 或执行层。
- 项目优化能够用评测数据证明，而不是只描述架构。

---

# P2：可选实验，有余力再做

目标：用于扩展技术深度或补充对照实验，不影响核心项目交付。

## P2.1 SQL 性能基础检查

- [ ] 使用 `EXPLAIN` 检查明显全表扫描。
- [ ] 对大表要求时间范围或其他高选择性条件。
- [ ] 设置扫描行数或执行成本阈值。
- [ ] 给出用户可理解的高成本查询提示。

边界：

- 不实现通用 SQL 自动优化器。
- 不自动进行可能改变语义的复杂 SQL 重写。

---

## P2.2 N 条 SQL 多采样对照实验

- [ ] 对同一问题分别生成 1 条和 N 条 SQL。
- [ ] 比较 Execution Accuracy、Token、延迟和成本。
- [ ] 研究候选 SQL 的选择标准。
- [ ] 只有评测证明明显有效时才加入默认链路。

边界：

- 多采样不能替代语义澄清。
- SQL 能执行不能作为候选正确的唯一判断。

---

## P2.3 复杂分析 Planner

只有出现以下场景时考虑：

- [ ] 一个问题必须拆成多条 SQL；
- [ ] 后一步查询依赖前一步结果；
- [ ] 需要跨多个数据源；
- [ ] 多个子结果需要汇总分析。

边界：

- 普通分组、趋势、排行和对比仍然使用单一工作流。
- 不重新引入泛化的 ReAct/Coordinator 双架构。
- 必须通过复杂任务评测证明 Planner 的收益。

---

## P2.4 MCP 外部数据接入

只在真实业务需要时接入：

- 天气数据；
- 节假日数据；
- CRM；
- 广告平台；
- 其他外部业务系统。

边界：

- “最近”不需要 MCP。
- “最新数据”优先通过数据库新鲜度查询解决。
- 不为了展示技术关键词强行加入 MCP。

---

## P2.5 可观测性

- [ ] 记录各 LangGraph 节点耗时。
- [ ] 记录检索候选及选择原因。
- [ ] 记录 Prompt Token 和模型调用次数。
- [ ] 记录 SQL 校验、执行和修复结果。
- [ ] 按错误类型生成简单统计报表。

---

# 3. 明确不做的内容

- 不维护“每张表一套庞大正则规则集”。
- 不让 LLM 直接扫描完整数据库 Schema。
- 不在意图识别阶段预测物理表和字段。
- 不将主外键、安全策略、时间规则等确定性规则交给 RAG 决定。
- 不把向量库当作 Schema 的事实来源。
- 不允许 Schema Linker 自由生成候选集合外的表名和字段名。
- 不把 LLM 自报置信度作为执行依据。
- 不默认生成 N 条 SQL。
- 不实现通用 SQL 自动优化器。
- 不使用无上限的 SQL 修复循环。
- 不为了 Multi-Agent 概念保留 ReAct 与 Coordinator。
- 不为了 MCP 概念强行接入无关外部数据。

---

# 4. 推荐实施顺序

```text
第一阶段：P0.1 ～ P0.3
收敛工作流、统一 AnalysisSpec、完善指标知识库

第二阶段：P0.4 ～ P0.5
实现 Schema Linking、RelationGraph 和最小上下文构建

第三阶段：P0.6
完成 AST 校验、只读执行和有限修复

第四阶段：P0.7
建立评测集，得到端到端基线数据

第五阶段：P1.1 ～ P1.2
补充 Schema 变化治理和语义澄清

第六阶段：P1.3 ～ P1.5
优化召回、时间语义和评测体系

第七阶段：按评测结果选择 P2 实验
```

---

# 5. 最终项目模块

| 模块 | 职责 |
| --- | --- |
| `IntentParser` | 提取意图和用户原始槽位 |
| `MetricRegistry` | 管理指标定义、别名、公式和依赖 |
| `SchemaRegistry` | 管理真实数据库表、字段和数据粒度，是 Schema 事实来源 |
| `SchemaSearchIndex` | 对表字段语义文档建立 BM25 与向量索引 |
| `CandidateRetriever` | 合并规则、关键词和向量检索结果，召回候选 Schema |
| `SchemaLinker` | 使用 LLM 在候选 ID 范围内精排表字段并识别歧义 |
| `SchemaValidator` | 校验 LLM 输出是否属于候选集合及真实 Schema |
| `RelationGraph` | 管理表关系并补齐 JOIN Key |
| `ClarificationGate` | 判断是否需要向用户澄清 |
| `SchemaContextBuilder` | 构建最小 SQL 生成上下文 |
| `SQLGenerator` | 基于受限 Schema 生成 SQL |
| `SQLValidator` | 进行 AST、安全、Schema 和指标校验 |
| `SafeExecutor` | 在只读受限环境中执行 SQL |
| `SQLRepair` | 根据结构化错误定向修复 SQL |
| `ResultAnalyzer` | 规划图表并生成分析结论 |
| `Evaluation` | 执行端到端回归和指标统计 |

---

# 6. 简历核心表达

> 基于 LangGraph 构建受控 Text-to-SQL 工作流，串联意图解析、Schema RAG、语义澄清、SQL 生成、AST 校验、只读执行及异常修复；将 GMV、订单数、客单价等 8 类指标沉淀为版本化 YAML，通过别名规则、BM25 与向量检索召回候选表字段，再由 Schema Linker LLM 在候选 ID 范围内受限精排，并利用表关系图校验和补齐 JOIN Key，仅向 SQL 生成节点注入必要 Schema，降低无关上下文和字段幻觉。

> 构建覆盖单表、多表关联、指标计算、模糊语义和异常查询的回归测试集，评估 Schema Recall@K、Execution Accuracy、澄清准确率及修复成功率；通过 Schema fingerprint 感知表结构变化，并校验指标对表字段的依赖关系，避免失效口径继续参与 SQL 生成。

---

# 7. 一句话总结

> 项目的核心不是堆叠 Multi-Agent，而是通过指标知识库、Schema RAG 候选召回、LLM 受限精排、最小上下文、语义澄清、SQL 安全执行和端到端评测，构建一条可解释、可恢复、可量化的 Text-to-SQL 工程链路。
