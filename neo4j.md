## Neo4j 在 foundation 中的角色

在 foundation 的整体架构中，Neo4j 对应「**图数据库**」角色：存储 **Agent 对要素分析后产生的结论**，以**节点（Nodes）与边（Edges）**形式组织。节点既包括客观事实（如人、单据、项目、事件），也包括主观结论（如「张三申请了报销」「该报销单需要部门经理审批」「需要紧急融资」）；边表达因果、影响、版本、流程等关系。这一结构对应「分析完的数据的存储」中「特定视角下该要素产生的结论 → 图数据库」的落库路径，以及「Foundation 对要素的分析理解」中**不同视角 → 影响分析 → 结果形成 → 数据关联**的最后一环「数据关联」的输出。在主光盘分析体系中，Neo4j 还可承载流程/政策/逻辑之间的依赖关系图谱及审计追溯图，支撑「张三申请了哪些报销，流程是怎么样的？」等复杂关系推理。

### 一、实际库与图结构

- **连接信息**
  - 容器：`foundation-neo4j-prod-container`（镜像 `neo4j:5.22.0`）。
  - 端口映射：主机 `7687` -> 容器 `7687`（Bolt），主机 `7474` -> HTTP 控制台。
  - 应用配置（`app/core/config.py`）：
    - `NEO4J_URI=bolt://localhost:7687`
    - `NEO4J_USERNAME=neo4j`
    - `NEO4J_PASSWORD=Admin1234!`（生产环境来自 `env.prod.eim`）。

- **数据库列表（`cypher-shell "SHOW DATABASES"`）**
  - `neo4j`（标准业务库，read-write，online，default）
  - `system`（系统库）

- **主要节点标签（`CALL db.labels()`）**
  - `app`
  - `record`
  - `event`
  - `original_data`
  - `process_dimension`
  - `org_dimension`
  - `person_dimension`
  - `risk_business_dimension`
  - `risk_control_environment_dimension`
  - `cost_center_dimension`
  - `procurement_category_dimension`
  - `chart_of_accounts_dimension`
  - `test_dd_dimension`

- **维度（Dimension）数量与分类**  
  foundation 中与“维度”相关的 Neo4j 节点类型共 **10 类**（当前实例 `CALL db.labels()` 所见）：
  - **核心分析维度（4 类）**：用于 LangGraph 多视角推理与证据链，关系为 `HAS_PROCESS` / `HAS_PERSON` / `HAS_EVENT` 及组织归属。
    - `process_dimension`：流程维度（如采购流程、报销流程、供应商资质审核流程）。
    - `person_dimension`：人员维度。
    - `org_dimension`：组织维度（集团/BU/部门等）。
    - `event`：事件维度（与 record 通过 `HAS_EVENT` 等关联，无 `_dimension` 后缀）。
  - **主数据/风险类维度（5 类）**：将主数据与风险控制抽象为图中的维度节点，便于跨维度关联与治理分析。
    - `risk_business_dimension`、`risk_control_environment_dimension`：风险业务维度、控制环境维度。
    - `cost_center_dimension`、`procurement_category_dimension`、`chart_of_accounts_dimension`：成本中心、采购类别、会计科目表维度。
  - **测试/扩展用**：`test_dd_dimension` 等。  
  此外，**通用维度（Common Dimensions）** 由 Postgres 表 `dimension_metadata`（`dimension_type='common'`）配置，数量不固定；每个通用维度对应一个 Neo4j 标签（如 `product_dimension`、`location_dimension`），关系为 `BELONGS_TO_COMMON_DIMENSION`。因此“有多少个维度”= 固定 10 类（当前图库中已存在的维度标签）+ 若干可配置的通用维度。

- **主要关系类型（`CALL db.relationshipTypes()`）**
  - `HAS_EVENT`
  - `FROM_ORIGINAL_DATA`
  - `CHILD`
  - `FATHER`
  - `HAS_PROCESS`
  - `BELONGS_TO`
  - `CREATED_BY`
  - `BELONGS_TO_APP`
  - `HAS_PERSON`
  - `BELONGS_TO_COMMON_DIMENSION`

### 二、承载的业务数据（节点与边的业务含义）

- **节点（Nodes）的两种形态（与架构图对应）**
  - **客观事实类**：如人员（张三）、单据（报销单号 XYZ123）、项目（项目 A）、事件（罚款 500 万）等实体，在图中对应 `person_dimension`、`record`、`event`、`original_data` 以及各类 `*_dimension` 节点。
  - **主观结论类**：Agent 根据规则与上下文推导出的结论，如「张三申请了报销」「该报销单需要部门经理审批」「公司面临资金周转压力，需要紧急融资」等，可通过 `event`、`record` 或与维度节点的属性/关系表达，并与其他节点通过边连接，便于回答「谁在什么流程里做了什么决策」。
- **边（Edges）表达的关系类型**
  - **因果 / 关联**：如报销单 —关联→ 项目 A；罚款 500 万 —导致→ 需要紧急融资。
  - **影响**：如财务规定变更 —影响→ 某结论或流程。
  - **版本 / 历史**：如报销单 —版本→ 第一版 —更新为→ 第二版。
  - **业务流程**：如张三 —提交→ 报销单 —需要审批→ 部门经理 —审批通过→ 财务处理；对应图中的 `HAS_EVENT`、`BELONGS_TO_APP`、`CREATED_BY`、`HAS_PROCESS` 等关系类型。

- **1. 应用（App）节点**
  - 来源：Postgres 中的 `app_node` 表。
  - 同步方式：
    - `Neo4jService.create_app_node(app_id: str, name: str, description: Optional[str])`
      - 在图中创建/更新 `(:app {app_id, name, description, create_time, update_time})`。
    - `Neo4jService.sync_app_node_to_neo4j(app_node_id: int, name: str, description: Optional[str])`
      - 使用 `id` 字段（`app_node.id`）作为 Neo4j 中 `:app {id: ...}` 的业务主键。
  - 业务含义：
    - 每个应用（如一个业务系统、数据域或组合视图）在图中对应一个 `:app` 节点，是应用级关系拓扑的锚点。

- **2. 维度节点（Dimension）**
  - **数量**：Neo4j 中与维度相关的节点标签共 **10 类**（9 个带 `_dimension` 后缀 + 1 个 `event`）；另可通过 Postgres `dimension_metadata` 配置若干**通用维度**（common），对应额外标签如 `product_dimension`、`location_dimension` 等。
  - **标签与含义**：
    - **核心分析维度（4 类）**：`process_dimension`（流程，如采购/报销/供应商资质审核）、`person_dimension`（人员）、`org_dimension`（组织）、`event`（事件）。
    - **主数据/风险类维度（5 类）**：`risk_business_dimension`、`risk_control_environment_dimension`、`cost_center_dimension`、`procurement_category_dimension`、`chart_of_accounts_dimension`。
    - **可扩展**：`test_dd_dimension` 及由 `dimension_metadata` 注册的通用维度标签。
  - 业务含义：
    - 这些节点把 Postgres 与 Mongo 中的主数据实体（科目、成本中心、组织、流程、风险等）抽象为图中的“维度点”，用于做跨维度关联与可视化；Agent 回答“张三申请了哪些报销，流程是怎么样的？”等复杂关系问题时，依赖对上述维度的遍历与推理。

- **3. 事件与记录节点**
  - `event`、`record`、`original_data` 标签通常用于描述：
    - 某个文件的处理记录（ingestion、解析、向量化等）。
    - 某个 agent 在文件或应用上的操作事件。
  - `Neo4jService.get_file_agent_actions(file_id)`：
    - 通过 `MATCH (n) WHERE n.file_ingestion_record_id = $file_id RETURN n` 查询所有与某一 `file_ingestion_record_id` 相关的节点。
  - `Neo4jService.get_agent_network(file_id)`：
    - 通过 `MATCH (n)-[r]->(m) WHERE n.file_ingestion_record_id = $file_id OR m.file_ingestion_record_id = $file_id RETURN n, r, m` 获取文件相关的 agent 网络。
  - 业务含义：
    - 把“文件 -> 解析 -> 向量化 -> 分析 -> 结论”的流水线，从线性日志提升为可视化的图结构。
    - 支撑故障排查（看到哪个 agent 在哪一步失败）、审计（谁对某个文件做了什么）以及可视化展示。

### 三、支撑的核心业务流程

- **1. 应用与维度图谱构建流程**
  - 入口：
    - 后端在创建/更新 `app_node`（Postgres 表）时，通过 `Neo4jService.sync_app_node_to_neo4j` 将其同步到 Neo4j。
    - Airflow / LangGraph 任务在维度分析时，创建或更新对应的维度节点（如 `process_dimension`、`org_dimension` 等）。
  - 流程概览：
    1. Postgres 中的业务实体（应用、主数据、组织等）通过服务层被抽象并映射到 Neo4j 节点。
    2. 不同维度之间通过如 `BELONGS_TO`、`CHILD`、`FATHER`、`BELONGS_TO_COMMON_DIMENSION` 等关系连接。
    3. 某个 `:app` 节点可通过 `BELONGS_TO_APP` 链接到相关维度结点，形成“应用视角”的多维度知识图谱。
  - 结果：
    - 前端页面（如 `visualization.tsx` 相关路由）可以基于 Neo4j 的图数据渲染应用‑维度关系网络。
    - 支撑“从应用看风险/流程/组织/成本”的立体化视图。

- **2. 文件‑Agent 网络与追踪流程**
  - 入口：
    - 文件入库与处理（File Ingestion + Workflow Execution）过程中，当某个 agent 针对文件执行步骤时，将节点与关系写入 Neo4j。
  - 流程概览：
    1. 文件被上传到 MinIO 并登记在 Postgres 的 `file_ingestion_records`。
    2. 工作流执行服务（`workflow_execution_service.py` 等）在处理文件时：
       - 为每个文件、每个 agent 操作创建 `record` / `event` 节点，并附上 `file_ingestion_record_id`。
       - 用关系（如 `CREATED_BY`、`HAS_EVENT`、`BELONGS_TO_APP`）把这些节点与 `:app`、维度节点连接起来。
    3. 通过 `Neo4jService.get_file_agent_actions` & `get_agent_network`，前端或诊断工具可拉取整个处理链路。
  - 结果：
    - 提供一个“从文件视角回溯整个 AI/工作流处理过程”的图谱，便于解释、审计和调优。

- **3. 维度驱动的风险与控制分析**
  - 标签组合（`risk_business_dimension`、`risk_control_environment_dimension`、`process_dimension` 等）表明：
    - 风险点、控制措施、流程节点等被作为图中的一等公民。
    - 可以沿着 `HAS_PROCESS`、`BELONGS_TO` 等关系做跨维度导航：
      - 从一个应用跳到相关流程。
      - 从流程跳到对应的业务风险。
      - 再跳到与之相关的组织单元或成本中心。
  - 结果：
    - 为“从点到面”的风险治理分析提供基础设施，比单表 SQL 查询更适合复杂关系场景。

### 四、典型业务场景示例（Neo4j 视角）

- **示例 1：上传一批政策文件，构建“政策处理链路图”**
  - 数据流：
    - 用户上传多份政策文件，文件与解析状态记录在 Postgres（`file_ingestion_records`），文件本体与提取文本在 MinIO。
    - 工作流服务按顺序对每个文件执行 OCR/文本抽取、向量化、合规规则识别等步骤。
  - 图谱构建：
    - 每处理一个关键步骤，会在 Neo4j 中创建一个或多个 `record` / `event` 节点，并打上 `file_ingestion_record_id`、`step`、`status` 等属性。
    - 这些节点通过 `CREATED_BY` / `BELONGS_TO_APP` / `HAS_EVENT` 等关系挂到对应的 `:app` 节点与维度节点上。
  - 效果：
    - 在前端或运维工具中，可以“一眼看清”：
      - 某份政策从上传到完成解析/向量化经历了哪些步骤、耗时如何、在哪一步失败。
      - 哪些 Agent/流程节点参与了处理，形成一条可视化的处理链路。
    - 当 Agent 需要解释“为什么这份政策检索不到”“为什么回答依赖的是旧版本文本”时，可以直接基于图谱做追溯。

- **示例 2：围绕某应用的流程与风险/控制图谱**
  - 数据流：
    - Postgres 中的流程定义、组织单元、主数据（如成本中心、科目表）通过定制任务投射到 Neo4j，对应 `process_dimension`、`org_dimension`、`cost_center_dimension`、`chart_of_accounts_dimension` 等节点。
    - Mongo 中的问卷结果经分析后，形成与“控制环境”“业务风险”相关的指标，被挂载到 `risk_business_dimension`、`risk_control_environment_dimension` 等节点。
  - 图谱构建：
    - 某个应用的 `:app` 节点通过 `BELONGS_TO_APP` 链接到若干流程维度节点（例如“采购流程”“报销流程”），流程节点再通过 `BELONGS_TO` 连接到相关组织、成本中心、风险维度节点。
  - 效果：
    - 支撑类似问题：“这个应用涉及哪些关键流程？这些流程覆盖了哪些组织与成本中心？对应的风险与控制评价如何？”。
    - Agent 可以在图中沿着多跳关系走完整条链，而不是在多张表里做手工 join。

- **示例 3：结合审批与画布，回答“张三参与了哪些关键流程？”**
  - 结构：
    - 张三的身份信息在 Postgres 的 `users` 表。
    - 与张三相关的审批实例在 `approval_workflow_instances` / `approval_workflow_steps`，可投射为 Neo4j `event` 节点或与 `process_dimension`、`person_dimension` 之间的关系。
    - 张三被分享或自己创建的 AIGRID 画布在 `grid_space` / `grid_space_shares` 中，可对应特定的 `:app` 或维度组合。
  - 图谱视角：
    - 通过 `person_dimension` + 关系边，图中可以看出张三：
      - 参与过哪些流程/审批链条。
      - 经常与哪些应用、政策、主数据维度共同出现。
  - 效果：
    - 当用户提问“张三在采购相关流程里扮演了什么角色？参与过哪些关键决策？”时，Agent 可以利用 Neo4j 的关系遍历能力在图上做复杂推理，而不仅是“查几张表 + 模糊搜索”。

### 五、与其他存储的关系

- **与 Postgres 的关系**
  - Postgres 是事实数据与配置的权威来源，Neo4j 主要保存“关系视图”：
    - `app_node` 表 -> `:app` 节点。
    - 文件入库记录（`file_ingestion_records` 的 ID）-> `record`/`event` 节点的属性 `file_ingestion_record_id`。
    - 各类维度数据（组织、流程、风险、主数据等）在 Postgres 中存主键与属性，在 Neo4j 中存图结构与跨维度关系。
  - 因此 Neo4j 更像是 Postgres 上的一层“图索引与可视化层”，而非替代关系数据库。

- **与 Mongo 的关系**
  - 问卷与半结构化主数据（在 Mongo）经过分析后，可被汇总为 Neo4j 中的一些维度或事件节点（例如某应用的问卷得分映射到相应的 `risk_*_dimension`）。
  - 这种关系通常由 Airflow / LangGraph 任务在 Agent 流程中显式创建，而不是系统级自动同步。

- **与 MinIO 的关系**
  - 文件真实内容仍在 MinIO，Neo4j 只记录与文件相关的“处理轨迹”和“上下文关系”：
    - 某一 `file_ingestion_record_id` 对应一组 Neo4j 节点与边。
    - 通过这些图结构，可以从文件追踪到其所属应用、相关维度、Agent 处理步骤等，再回到 Postgres / MinIO 获取细节。

### 六、与架构图的对齐（结论存储 · 数据关联 · 主光盘）

- **分析完的数据的存储**  
  Agent 在接收「要素输入」（如 MinIO 中的文件）和「关系型数据库上下文」（Postgres）后，产出的**特定视角下该要素产生的结论**写入 Neo4j：即图中「Agent 生产的结果 → 图数据库（Nodes & Edges）」在 foundation 中的实现。查询「张三申请了哪些报销，流程是怎么样的？」时，通过遍历图中与张三、报销单、审批人、项目等相关的节点与边即可还原完整流程与结论链。

- **Foundation 对要素的分析理解**  
  要素经「不同视角 → 影响分析 → 结果形成」后，在「**数据关联**」阶段被建立实体与结论之间的关系，其输出即落入图数据库。Neo4j 中的 `process_dimension`、`org_dimension`、`person_dimension`、`risk_*_dimension` 以及 `record`/`event` 等，正是多视角分析结果与关联关系的持久化形态。

- **主光盘分析体系中的图库角色**  
  主光盘将政策/流程/表单建模为可执行逻辑并拆解为多 Agent 协作；Neo4j 可表示流程步骤、审批节点、政策与表单之间的依赖图，以及「每次执行保留日志」所需的审计追溯图（如某次报销执行涉及的用户、单据、策略、Agent 步骤之间的链路），支撑合规与审计查询。

### 七、四库协同带来的能力（Neo4j 视角）

- **把“结构化表 + 文本内容 + 处理过程”连接成一张知识图谱**：
  - Postgres 负责“谁、什么业务对象、在哪个流程里”的结构化事实。
  - Mongo 负责“半结构化主数据与问卷反馈”的内容事实。
  - MinIO 负责“原始文件与解析中间件”的物理证据。
  - Neo4j 把 Agent 产出的**结论与关系**以节点和边的形式落库，使 Agent 可围绕任意一点（人、流程、政策、主数据、文件）做多步推理。
- **支撑复杂关系类问题**：例如“张三申请了哪些报销，流程是怎么样的？”“张三近半年参与的高风险审批都集中在哪些 BU/流程？”这类问题，在图库中通过多跳遍历即可回答，对应「既懂企业自己的业务数据，又能把零散信息整理成逻辑清晰的企业数据治理」中的关系推理能力。
- **企业数据治理视角**：在四库协同下，foundation 将零散在文件、表格、流程、问卷、画布里的信息提炼为「可解释的结论与关联」，形成企业级知识图谱，为内控与治理决策提供基础设施。

