## Mongo 在 foundation 中的角色

在 foundation 的整体架构中，Mongo 承担**半结构化与动态内容**的存储：既接收来自 MinIO 的「要素」经解析后的表格与文本（主数据 Excel/CSV → `master_semi_structured_data`），也存储问卷定义与问卷响应（`questionnaires`、`questionnaire_responses`），为 Agent 提供**文件与关系库之外的上下文**。在「Foundation 对要素的分析理解」中，低层 Agent 的输入包含「文件、Prompt、**数据库**」——Mongo 即该「数据库」中的半结构化部分；在主光盘分析体系中，各类政策/流程/表单解析后的灵活结构、中间处理结果与问卷事实也可落在 Mongo，供多 Agent 协作与结果形成、数据关联阶段使用。

### 一、实际库与集合结构

- **连接信息（生产环境）**
  - 容器：`foundation-mongo-prod-container`（镜像 `mongo:latest`）。
  - 端口映射：主机 `27017` -> 容器 `27017`。
  - 根据 `env.prod.eim` 与服务代码：
    - 主机：`MONGO_HOST=mongo`（容器内部） / 外部访问为 `10.98.15.4`
    - 端口：`MONGO_PORT=27017`（mongo-express 暴露在 `8081`）
    - 数据库：`MONGO_DATABASE=foundation`
    - 账户：`MONGO_ROOT_USERNAME=admin`，`MONGO_ROOT_PASSWORD=password123`

- **当前数据库 `foundation` 中的主要集合（来自 `mongosh db.getCollectionNames()`）**
  - `master_semi_structured_data`
  - `questionnaires`
  - `questionnaire_responses`

### 二、承载的业务数据

- **1. 半结构化主数据：`master_semi_structured_data`**
  - 来源：
    - Airflow DAG `masterdata_semi_structured_to_mongodb.py` 将存放在 MinIO 的 Excel/CSV 主数据文件解析为行列结构，并写入该集合。
    - Postgres 表 `master_data_semi_structured_ingestion_records` 记录了与该集合的映射关系：
      - `masterdata_id` / `version_number`：指向主数据主表及其版本。
      - `document_id`：Mongo `_id`（十六进制字符串）。
      - `collection_name`：固定为 `master_semi_structured_data`。
  - 文档结构（来自 `MasterDataSemiStructuredService` 及 Airflow DAG 注释）大致包括：
    - `content`：按行拼接的文本内容（兼容旧版检索）。
    - `metadata.columns`：列名列表（表头）。
    - `payload.rows`：每一行数据的结构化字典数组（列名 -> 值）。
    - 适用性与权限字段：`applicability_type_business_unit`、`applicable_business_units`、`applicability_type_function_role`、`applicable_function_role_ids` 等。
  - 业务含义：
    - 承载 Excel/CSV 形态的“主数据表格”，如组织表、账户表、成本中心、采购类别等。
    - 是主数据领域中“半结构化部分”的事实存储，用于做基于表格的问答与分析。

- **2. 问卷定义：`questionnaires`**
  - 背景：
    - 在 `backend/app/models.py` 中明确注释：
      - 原 `Questionnaire` / `QuestionnaireResponse` 表已迁移到 Mongo。
      - `Questionnaire` -> Mongo 集合 `questionnaires`。
  - 典型字段（推断自 Airflow DAG `agent_flows/*dashboard_generate*.py`）：
    - 问卷元信息：名称、说明、所属应用 `app_node_id`、维度类型等。
    - 题目结构：问题列表/题型/选项配置等，便于前端/agent 动态渲染。
  - 业务含义：
    - 存储围绕应用、流程、风险维度设计的问卷“模板”，为后续收集问卷响应与生成治理仪表盘提供结构。

- **3. 问卷响应：`questionnaire_responses`**
  - 背景：
    - 在 `backend/app/models.py` 注释中标记：`QuestionnaireResponse` -> Mongo `questionnaire_responses`。
    - 多个 Airflow DAG（如 `agent_flows/dashboard_generate.py`、`dashboard_generate_optimized.py`、`app_hierarchy_processor.py`）中通过 `mongodb_handler.get_collection("questionnaire_responses")` 访问。
  - 典型字段（从 DAG 中的聚合逻辑推断）：
    - `app_node_id`：响应所属应用。
    - `questionnaire_id`：关联问卷定义。
    - 每题回答内容（选择题/自由文本等）、时间戳、回答人标识（可能做了匿名化或聚合处理）。
  - 业务含义：
    - 承载用户/业务方在各应用维度上的风险控制、自评估、流程成熟度等问卷结果。
    - 是生成应用风险画像、控制环境评分、业务健康度等仪表盘的核心数据来源之一。

### 三、支撑的核心业务流程

- **1. 半结构化主数据 RAG 检索流程**
  - 入口：
    - 前端或 API（如 master data 相关问答接口）将自然语言查询传入。
  - 流程（由 `MasterDataSemiStructuredService` 驱动）：
    1. 使用 `_get_client()` 基于环境变量构造 Mongo URI：
       - `mongodb://admin:password123@MONGO_HOST:MONGO_PORT/foundation?authSource=admin`。
    2. 在 `master_semi_structured_data` 上确保 `content` 字段的全文索引（`ensure_text_index`）。
    3. 结合用户在 Postgres 中的 BU / Function Role 信息，构造权限过滤条件 `_build_permission_filter(user)`。
    4. 通过 `$text` 查询 + 权限过滤，选出最相关的文档（表格）。
    5. 使用 LLM（LangGraph agent）与 `_filter_rows_by_conditions` 对 `payload.rows` 做行级过滤，得到与用户问题最相关的行。
    6. 将结果封装为统一的 `payload`，供后续 QA pipeline 消费。
  - 结果：
    - 用户可以基于自然语言对 Excel/CSV 主数据进行 RAG 检索，例如：
      - “某个 BU 下有哪些成本中心？”
      - “某个邮箱用户属于哪个学校/部门？”

- **2. 问卷数据驱动的 Dashboard 生成流程**
  - 入口：
    - Airflow DAG `agent_flows/dashboard_generate.py` / `dashboard_generate_optimized.py`。
  - 核心步骤（概括自 DAG 中的逻辑和注释）：
    1. 通过 `mongodb_handler.get_collection("questionnaires")`、`get_collection("questionnaire_responses")` 读取两张集合。
    2. 根据 `app_node_id` 查找对应应用的问卷定义与响应记录。
    3. 对响应进行统计、聚合，形成应用在风险、合规、流程成熟度等维度上的量化指标。
    4. 将这些指标再写回 Postgres 或其他结构（如 Neo4j、Qdrant、前端 JSON 配置），用于前端 Dashboard 展示。
  - 结果：
    - 形成围绕应用/流程的“问卷驱动洞察”，用于治理决策和风险评估。

- **3. Human-in-the-loop 与问卷联动**
  - 在 `backend/app/services/workflow/nodes/human_in_the_loop_node.py` 中，有关于通过 `questionnaire_responses` 触发或辅助人工节点决策的逻辑说明。
  - 典型场景：
    - 智能工作流在处理到某个决策节点时，读取问卷响应，给出辅助建议或预填决策选项，再由人工确认。

### 四、典型业务场景示例（Mongo 视角）

- **示例 1：上传主数据 Excel 后，按自然语言查某 BU 的成本中心**
  - 数据落库过程：
    - 用户在主数据模块上传“成本中心表 Excel”，文件原件进入 MinIO（详见 `minio.md`）。
    - Airflow DAG 把每一行解析成结构化记录，写入 `master_semi_structured_data`，并在 Postgres 中的 `master_data_semi_structured_ingestion_records` 建立 `masterdata_id + version_number -> document_id` 映射。
  - 查询过程：
    - 用户在聊天或搜索框输入：“张三所在 BU 下有哪些成本中心？”。
    - Agent 先在 Postgres 查出张三的 `business_unit_id` 与主数据版本，再调用 `MasterDataSemiStructuredService.search_by_user`：
      - 在 `master_semi_structured_data` 上做 `$text` 搜索，结合 BU / Function Role 权限过滤。
      - 基于 LLM 把“张三所在 BU”“成本中心”等实体映射到列名，对 `payload.rows` 做行级匹配。
    - 返回结果中，每一行就是一个成本中心记录（例如成本中心代码、名称、所属组织等），以结构化文本形式回给前端或传给后续推理步骤。
  - 作用：
    - Mongo 在这里承载了“主数据的表格事实层”，让 Agent 在保持强权限控制的前提下，对 Excel/CSV 进行接近 SQL 级别的按列检索和行级过滤。

- **示例 2：围绕某应用的风险自评问卷，生成 Dashboard 指标**
  - 数据落库过程：
    - 风险团队在前端配置一份问卷模板（如“采购合规自评问卷”），模板结构持久化在 `questionnaires` 集合，带上 `app_node_id`（指向 Postgres 的 `app_node`）。
    - 各业务负责人填写问卷，回答内容逐条进入 `questionnaire_responses`，包括问题 ID、选项/文本答案、时间戳等。
  - 分析过程：
    - Airflow DAG 周期性读取 `questionnaires` 和 `questionnaire_responses`，按应用/BU/职能等维度做聚合，形成：
      - 某应用在“控制环境”“流程合规”“风险识别”等维度上的得分。
      - 关键问题的集中意见与典型文本回答摘要。
    - 聚合结果可以写回 Postgres（作为指标表或 JSON 配置）、或映射到 Neo4j 的 `risk_*_dimension` 节点，供前端可视化。
  - 作用：
    - Mongo 在这里承担的是“问卷事实库”，既保留每一次回答的明细，又能被上游 DAG 灵活聚合成面向治理的指标体系。

- **示例 3：Human-in-the-loop 节点利用问卷结果辅助审批**
  - 当某个审批流程（例如“供应商资质评估”）走到关键决策节点时，Human-in-the-loop 节点可以：
    - 从 `questionnaire_responses` 中拉取该供应商/应用最近一段时间的自评结果。
    - 提炼关键信息（例如控制薄弱点、高风险反馈）作为提示，展示在审批界面中。
  - 作用：
    - 让审批人不需要在多个系统里翻历史记录，而是通过 Mongo 中的问卷事实，获得上下文丰富、结构清晰的“辅助证据”。

### 五、与架构图的对齐（要素解析 · Agent 上下文 · 主光盘）

- **要素输入与解析结果的落库**  
  「分析完的数据的存储」中，要素输入（如用户上传的采购订单、报销单据、供应商合同）先存于 MinIO，经解析后结构化或半结构化内容可进入 Mongo。`master_semi_structured_data` 中的 `payload.rows`、`content`、`metadata.columns` 即「要素」的表格化形态，供 Agent 做影响分析、结果形成时使用。

- **为 Agent 提供数据库侧上下文**  
  「Foundation 对要素的分析理解」里，低层 Agent 做无关信息过滤与初次加工时，除「文件」「Prompt」外还依赖「数据库」：Mongo 中的主数据表格与问卷响应为 Agent 提供按列/按问题的检索与聚合能力，与 Postgres 的结构化业务数据一起构成完整上下文。

- **主光盘体系中的半结构化与问卷**  
  主光盘要求「支持将各类政策/流程/表单解析建模为光盘逻辑」「输入数据可自动匹配光盘逻辑并生成多种输出」：解析后的表单结构、流程中间结果、问卷定义与响应等灵活结构适合存放在 Mongo，便于多 Agent 协作时读写和聚合，并可与 Neo4j 的图谱、Postgres 的审计日志配合，形成可追溯的治理数据链。

### 六、与其他存储的关系

- **与 Postgres 的关系**
  - 主数据主表与版本信息在 Postgres `master_data` / `master_data_revisions` 中，Mongo 只作为半结构化内容存储与检索引擎。
  - `master_data_semi_structured_ingestion_records` 通过 `masterdata_id + version_number + document_id + collection_name` 做跨库关联。
  - 用户、组织与权限信息全部在 Postgres，Mongo 检索严格依赖 Postgres 用户模型做权限过滤（BU / Function Role）。

- **与 Neo4j 的关系**
  - 问卷分析结果和半结构化主数据的洞察可能通过 Agent 流程投射到 Neo4j 的**维度节点**（如 `org_dimension`、`process_dimension`、`risk_business_dimension` 等），形成图谱化视图。foundation 中维度数量由 Postgres 的 `dimension_metadata` 与 Neo4j 中已有标签共同决定（当前约 10 类固定维度标签 + 若干可配置通用维度），详见 `neo4j.md` 与 `postgres.md`。
  - Mongo 自身不直接负责图结构，而是作为源数据，供后续图构建与分析使用。

- **与 MinIO 的关系**
  - 半结构化主数据的原始文件保存在 MinIO（masterdata / policy 等 Bucket），Mongo 保存的是解析后的行列结构与全文内容。
  - Airflow DAG 从 MinIO 读取文件 -> 解析 -> 写入 Mongo -> 在 Postgres 中记录 ingestion 映射。

### 七、四库协同带来的能力（Mongo 视角）

- **把“散落在 Excel / 问卷里的事实”转成可计算资产**：Mongo 接住来自 MinIO 的解析结果，并通过与 Postgres 的 ID 映射，成为主数据与问卷事实的集中仓库。
- **支撑复杂问题的“按内容理解”能力**：例如“张三所在 BU 的历史自评结果如何？”、“某类成本中心在近三个月是否频繁出现异常反馈？”，Agent 可以：
  - 先用 Postgres 确定“张三是谁、在哪个 BU、有哪些可见主数据与应用”；
  - 再用 Mongo 在 `master_semi_structured_data` / `questionnaire_responses` 上按内容检索和聚合；
  - 最后结合 Neo4j 的关系视角和 MinIO 的原始文件，为用户输出既有数字、又有文本证据的解释性答案。
- **企业数据治理视角**：Mongo 让“原本藏在表格和问卷里的隐性知识”变成显性、可检索、可汇总的数据层，与 Postgres 的结构化业务模型、Neo4j 的关系图谱、MinIO 的原始证据一起，支撑起“既懂企业业务，又能梳理零散信息”的企业数据治理能力。

