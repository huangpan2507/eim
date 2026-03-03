## Postgres 在 foundation 中的角色

在 foundation 的整体架构中，Postgres 对应「**关系型数据库**」角色：为 Agent **提供上下文**（含业务规则、用户与权限、流程定义、AIGRID 画布等），并承载**主光盘**体系中的政策/流程/表单建模、可执行逻辑的元数据、多 Agent 协作任务状态以及**每次执行的审计日志**，满足合规与可追溯要求。要素输入（如 MinIO 中的文件）与关系型库中的上下文共同驱动 Agent 产生「特定视角下的结论」，结论再落入 Neo4j 图数据库形成节点与边。

### 一、承载的业务数据

- **核心用户与组织数据**
  - `users` 及相关枚举字段：用户账户、类型（内部/外部/系统）、安全属性（MFA、锁定、密码策略等）。
  - `groups`、`divisions`、`business_units`、`functions`、`sub_functions`、`job_titles` 等：完整的组织架构与职能体系，用于权限与可见性计算。
  - 关联表如 `group_division_links`、`organization_import_records`：记录组织导入任务及集团‑事业部关系。

- **权限与角色体系**
  - `roles`、`role_hierarchies`、`permissions`、`role_permissions`、`user_roles`、`user_permissions`、`permission_templates`、`function_roles`、`sub_function_roles` 等：
    - 承载基于组织、职能、角色的细粒度权限模型。
    - 结合用户的 BU / Function / Job Title，控制政策库、主数据等资源的访问范围。

- **政策库（Policy Library）数据**
  - `policies`：政策主表，包含政策名称、摘要、生效/失效时间、责任人等元数据。
  - `policy_revisions`、`policy_history`：版本管理及变更历史。
  - `policy_favorites`、`policy_access_logs`：用户收藏与访问审计。
  - `policy_deviation_requests`、`policy_deviation_attachments`：偏离申请及其附件元数据。
  - `policy_vector_ingestion_records`、`policy_dify_sync_records`：向量化、对接外部智能应用的处理记录。

- **主数据（Master Data）相关数据**
  - `master_data`：主数据条目主表（概念级别的业务主数据，如科目表、成本中心等）。
  - `master_data_revisions`、`master_data_history`、`master_data_revisions`：主数据版本与历史。
  - `master_data_vector_ingestion_records`：主数据文件向量化与知识库构建记录。
  - `master_data_semi_structured_ingestion_records`：
    - 记录与 MongoDB `master_semi_structured_data` 文档的映射。
    - 字段 `collection_name` 明确 Mongo 集合名，`document_id` 存储 Mongo `_id`。
  - `master_data_access_logs`、`master_data_favorites`：访问审计与收藏。

- **文件与解析流水线数据**
  - `file_ingestion_records`：文件入库记录，包含 MinIO 对象名、解析状态等。
  - `file_scan_tasks`、`file_scan_items`：安全扫描与内容检测任务。
  - 与 MinIO 相关的字段：
    - `file_storage_path`：文件在 MinIO 的对象路径。
    - `extracted_text_storage_path`：解析后纯文本在 MinIO 的存储路径。

- **工作流与审批引擎数据**
  - `approval_workflow_templates`、`approval_workflow_flows`、`approval_workflow_node_definitions` 等：
    - 定义审批流程模板、节点、角色/职级/人员绑定。
  - `approval_workflow_instances`、`approval_workflow_steps`：
    - 记录每一次审批流程实例的执行状态、流转路径。
  - `approval_workflow_node_users`、`approval_workflow_node_roles`、`approval_workflow_node_job_titles`：
    - 描述某个审批节点的参与者配置。
  - `account_approval_requests`：账号开通/资料修改等审批请求。

- **通用工作流与 LangGraph 代理相关数据**
  - `workflow_templates`、`workflow_instances`、`workflow_execution_tasks`：
    - 定义与跟踪非审批类的“智能工作流”（包括文件处理、问答等）。
  - `base_agent_info`、`plan_agent_info`、`agent_conclusion`：
    - 描述智能代理的配置、规划结果与结论。
  - `langgraph_intent`、`langgraph_prompt`：
    - LangGraph 代理的意图与提示词配置。
  - `checkpoint_*` 系列表（`checkpoints`、`checkpoint_blobs` 等）：
    - 存储 LangGraph 编排执行的状态快照，支撑长流程恢复与回放。

- **聊天与交互日志**
  - `chat_sessions`、`chat_messages`：用户与系统的对话会话与消息明细。
  - `email_audit_logs`：邮件发送与通知审计。

- **其他系统级数据**
  - `system_configs`：动态配置中心（如邮件、开关项等）。
  - `resources`、`visions`：前端应用中的视图、仪表盘与资源配置。
  - **`dimension_metadata`**：**维度元数据表**，定义 foundation 中有多少类维度及其在 Neo4j 中的标签与配置。维度数量 = **固定维度（3～4 类）+ 通用维度（若干，可配置）**。其中：**固定维度** 在初始化时写入（如 `process`→`process_dimension`、`person`→`person_dimension`、`organization`→`org_dimension`）；LangGraph 推理还会固定使用 **event**（事件维度，Neo4j 标签为 `event`）。**通用维度** 的 `dimension_type='common'`，由管理员通过 API 或配置新增，每条记录对应一个 Neo4j 节点标签（如 `product_dimension`、`location_dimension`），用于 `(record)-[:BELONGS_TO_COMMON_DIMENSION]->(维度节点)` 的扩展分析。表字段包括 `name`、`display_name`、`dimension_type`、`neo4j_node_label`、`is_active`、`is_readonly`、`config` 等。
  - `grid_space`：AI Grid / Canvas 画布空间表，存放单个画布的完整状态，包括视口配置（`viewport_state`）、布局版本（`layout_version`）、节点列表（`node_list`，包含应用、磁盘、富文本等节点）、连线列表（`connection_list`）、分组结构（`group_structure`）、画布元数据（`canvas_metadata`），以及所有者 `owner_id`、父画布 `parent_space_id`、来源分享 `share_source_id` 等。
  - `grid_space_shares`：AI Grid 画布分享记录表，描述某个 GridSpace 的分享关系和审批状态，包含原始画布 `space_id`、可编辑副本 `shared_space_id`、分享人/被分享人 (`shared_by`/`shared_to`)、权限类型 `permission_type`（read/edit）、状态 `status`（draft/pending/active/...）、是否需审批 `requires_approval`、关联的 `approval_workflow_instances`、以及分享时捕获的依赖资源快照 `resource_snapshot` 和扩展元数据 `share_metadata`。

### 二、支撑的业务流程（高层视角）

- **用户与组织管理流程**
  - 通过 `organization_import_records` 等表导入 Excel 组织数据，并在 `groups`、`divisions`、`business_units`、`functions` 中形成结构化组织树。
  - 用户注册 / 导入后写入 `users`，并通过 `user_roles`、`user_permissions` 与组织结构表形成权限闭环。

- **政策库全生命周期管理**
  - 新建政策：在 `policies` 创建记录，上传附件到 MinIO（见 MinIO 文档），保存 MinIO 对象名到政策修订表。
  - 修订与历史：每次变化写入 `policy_revisions` 与 `policy_history`，并更新适用范围字段（BU / Function Role）。
  - 阅读与偏离：访问行为记录在 `policy_access_logs`，偏离申请保存在 `policy_deviation_requests` 及其附件表。
  - 智能检索：政策文本解析、向量化过程的记录存入 `policy_vector_ingestion_records`，支撑基于向量数据库/Qdrant 的语义检索。

- **主数据管理与跨库联动流程**
  - 主数据元信息及版本在 Postgres 的 `master_data`、`master_data_revisions` 中维护。
  - 半结构化文件（Excel/CSV）在 MinIO 存储，对应的解析与入 Mongo 流程写入 `master_data_semi_structured_ingestion_records`。
    - `masterdata_id + version_number` 对应一个 Mongo 文档 `_id` 和集合 `master_semi_structured_data`。
  - 结构化向量化流程写入 `master_data_vector_ingestion_records`，为向量检索与智能分析提供索引。

- **工作流与审批/智能流程编排**
  - 业务审批流程：审批模板在 `approval_workflow_templates` 等表中定义，实例化后以 `approval_workflow_instances` 记录进度，节点执行明细在 `approval_workflow_steps`、`approval_workflow_node_*` 中。
  - 智能工作流：LangGraph 或自定义 workflow 的定义写入 `workflow_templates`，运行实例写入 `workflow_instances` 与 `workflow_execution_tasks`。
  - 与 Neo4j 的联动：`app_node` 表中的应用节点部分会同步到 Neo4j（见 neo4j.md），形成图结构用于可视化与关系推理。

- **AIGRID / Canvas 画布与分享流程**
  - 画布本身：每个 AIGRID / Canvas 画布在 `grid_space` 中对应一条记录，前端编辑时会把视口状态、节点列表、连接关系、分组结构和元数据以 JSONB 形式落盘，用于恢复和协同编辑。
  - 画布分享：当用户分享某个画布时，会在 `grid_space_shares` 中写入一条分享记录（含权限类型、审批需求、资源快照等），必要时通过 `approval_workflow_instances` 走审批流程；批准后可为被分享人生成新的可编辑画布（`shared_space_id` 指向新的 `grid_space`），并在 `app_nodes` / `grid_space` 相关 API 中以 `canvas_share` 类型暴露出来，供前端工作台和 AIGRID 入口使用。

- **文件入库与文本提取流水线**
  - 文件元数据与解析状态记录在 `file_ingestion_records`、`file_scan_tasks`、`file_scan_items`。
  - 实际文件与萃取文本存放在 MinIO（路径字段存于上述表中），供后续工作流与智能代理使用。

- **对话与智能代理交互**
  - 用户在前端发起的对话会话及消息序列存储在 `chat_sessions`、`chat_messages`。
  - LangGraph agent 的意图、提示词与执行中间态通过 `langgraph_*` 和 `checkpoint_*` 系列表沉淀，为后续重放与分析提供基础。

### 三、典型业务场景示例（Postgres 视角）

- **示例 1：用户上传一份政策文件并走完审批流程**
  - 上传阶段：
    - 用户在前端上传政策文件，请求通过政策相关 API（如 `policies` / `policy_revisions`）。
    - 文件本身写入 MinIO（见 `minio.md`），Postgres 中的 `policies`、`policy_revisions`、`file_ingestion_records` 记录：
      - 政策基础信息（名称、摘要、适用范围等）。
      - 对应的 MinIO 对象路径、文件大小、MIME 类型。
      - 文件是否已完成解析/向量化等状态。
  - 审批阶段：
    - 管理员在审批配置界面为“新政策发布”配置审批模板，模板持久化在 `approval_workflow_templates` 及相关节点定义表。
    - 当有新政策提交时，系统为该请求创建一条 `approval_workflow_instances` 记录，并为每个节点生成 `approval_workflow_steps`。
    - 不同审批人（来自 `users`、`roles`、`function_roles`、`job_titles` 等）在前端逐步审批，结果实时写入上述实例与步骤表。
  - 使用阶段：
    - 普通员工在前端浏览政策列表，查询条件命中 `policies` / `policy_revisions`，访问行为记录到 `policy_access_logs`。
    - 当员工发起“政策偏离申请”时，请求写入 `policy_deviation_requests` 及其附件表，并通过同一套 `approval_workflow_*` 表走偏离审批。
  - 作用：
    - Postgres 在这个场景中承载了政策的**权威元数据**、**审批轨迹**与**访问审计**，为合规审计和策略变更追踪提供单一事实源。

- **示例 2：主数据 Excel 上传 → 半结构化入库 → 智能查询**
  - 上传阶段：
    - 用户上传一份“成本中心主数据 Excel”，文件写入 MinIO；同时在 `master_data` 中创建一条主数据记录，在 `master_data_revisions` 中新增一个版本。
  - 解析与入库：
    - 后台任务或 Airflow DAG 读取该文件，对应创建/更新：
      - `master_data_semi_structured_ingestion_records`：记录该版本主数据与 Mongo 集合文档的映射（`collection_name=master_semi_structured_data`，`document_id` 为 Mongo `_id`）。
      - 如有向量化，则写入 `master_data_vector_ingestion_records`，指向 Qdrant 中的向量集合。
  - 查询与问答：
    - 当用户在前端提问“张三所在 BU 下有哪些成本中心？”时：
      - 身份与权限来自 `users` + 组织/角色表。
      - Agent 根据问题路由到主数据服务，先从 Postgres 获取可见主数据版本，再通过 `master_data_semi_structured_ingestion_records` 锁定 Mongo 文档，最后由 Mongo 做行级过滤（见 `mongo.md`）。
  - 作用：
    - Postgres 把“主数据版本”“半结构化表格”“向量库索引”这些分散组件统一绑在一起，使得系统可以按业务视角（主数据条目）管理数据全生命周期。

- **示例 3：用户在 AIGRID 画布中编排多个系统与数据源**
  - 画布创建：
    - 用户在 aigrid/canvas 页面中新建一个画布，在其中拖拽多个应用节点（`app_node`）、文件/磁盘节点和说明性富文本。
    - 每次保存会把当前画布结构序列化为 JSON，写入 `grid_space` 的 `node_list` / `connection_list` / `group_structure` / `canvas_metadata` 字段。
  - 画布分享：
    - 用户将该画布分享给同事张三，系统在 `grid_space_shares` 中新增一条记录：
      - `space_id` 指向原始画布，`shared_to` 为张三的 `users.id`，`permission_type='edit'`。
      - 如配置了审批模板，则同步创建 `approval_workflow_instances` 记录，状态从 `draft` → `pending` → `active`。
  - 作用：
    - Postgres 让“可视化编排画布”本身成为一类一等公民资源，可被索引、分享、审批和审计，为后续在 Neo4j 中构建更丰富的图谱提供结构化入口。

### 四、与架构图的对齐（要素 · Agent 上下文 · 主光盘）

- **关系型数据库 → 为 Agent 提供上下文**  
  图中「关系型数据库」存储的 **Agent 人格信息 / 业务上下文**，在 foundation 中由 Postgres 实现：用户与组织、角色与权限、政策与主数据元数据、审批与工作流模板、AIGRID 画布结构等。Agent 在处理「要素」（如上传的文件、报表输入）时，从 Postgres 拉取这些结构化信息，从而在正确的业务视角下做判断与推理。

- **主光盘分析体系在 Postgres 中的体现**  
  「企业级主光盘」将政策、标准、表单、方法论、模板抽象为统一容器，经 AI 转为可执行逻辑并拆解为多 Agent 协作。Postgres 支撑其中多项交付要求：
  - **政策/流程/表单解析建模为光盘逻辑**：`policies`、`policy_revisions`、`approval_workflow_templates`、`workflow_templates` 等表存流程阶段、审批人、审批条件、表单结构等，构成可被 Agent 执行的逻辑元数据。
  - **输入匹配光盘逻辑并生成多种输出**：执行实例与任务状态落在 `approval_workflow_instances`、`workflow_instances`、`workflow_execution_tasks`，输出结果与报告元数据也可落表，供报表与可视化消费。
  - **光盘逻辑拆解为多 Agent 协作**：`base_agent_info`、`plan_agent_info`、`agent_conclusion`、`checkpoint_*` 等表管理 Agent 任务队列、执行状态与结论，与 LangGraph 编排一致。
  - **支持光盘新增、修改、删除**：政策、主数据、工作流模板、画布等均支持 CRUD，Postgres 为权威存储。
  - **每次执行保留日志，合规与审计**：审批步骤、工作流任务、访问日志、邮件审计等写入 Postgres，满足「每次执行可追溯」的要求。

- **分层 Agent 与数据库**  
  「Foundation 对要素的分析理解」中，低层 Agent 做无关信息过滤与初次加工，其输入包含「文件、Prompt、数据库」：这里的「数据库」即 Postgres（及 Mongo），提供身份、权限、流程定义、主数据范围等上下文，供中高层 Agent 汇总与综合推理时使用。

### 五、与其他存储的关系

- **与 Mongo 的关系**
  - Postgres 中的 `master_data_semi_structured_ingestion_records` 将主数据版本与 Mongo 集合 `master_semi_structured_data` 的文档 `_id` 绑定。
  - 问卷业务：问卷定义与响应已迁移到 Mongo（`questionnaires`、`questionnaire_responses`），Postgres 仅保留注释性说明，不再承担存储职责。

- **与 Neo4j 的关系**
  - `app_node` 表中的应用节点会通过 `Neo4jService.sync_app_node_to_neo4j` 同步到图数据库中的 `:app` 节点。
  - 文件入库记录（`file_ingestion_records`）的 ID 也被用于在 Neo4j 中建立 agent 处理轨迹节点，形成“应用 / 文件 / agent 行为”图谱。

- **与 MinIO 的关系**
  - Postgres 保存 MinIO 对象名、Bucket 信息和解析状态，是 MinIO 对象的“索引与控制平面”。
  - 文件真实内容（原始文件+解析文本+预览 PDF）均在 MinIO，业务逻辑通过 Postgres 中的记录找到具体对象。

### 六、四库协同带来的能力（Postgres 视角）

- **业务语义的“主脑”**：Postgres 中的用户、组织、角色、政策、主数据、工作流等表，定义了企业内部“是谁”“做什么”“在哪个流程里”“遵守什么规则”等核心语义。
- **跨库数据血缘的锚点**：通过 `file_ingestion_records`、`master_data_*_ingestion_records`、`grid_space_shares` 等表，Postgres 把 MinIO 文件、Mongo 文档、Neo4j 节点、Qdrant 向量等外部存储统一挂在同一套主键体系下。
- **对 Agent 友好的结构化视图**：当 Agent 需要回答类似“张三参与过哪些审批/看过哪些政策/访问过哪些主数据/在哪些画布中被提到？”这类复杂问题时，可以先在 Postgres 中基于结构化关联做第一轮过滤，再联动 Neo4j 做关系推理、联动 Mongo 做内容检索、联动 MinIO 回溯原始证据，从而在“既懂企业自身业务结构，又能整理零散信息”的前提下给出解释性很强的答案。

