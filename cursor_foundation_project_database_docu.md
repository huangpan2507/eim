# Foundation project database documentation
_Exported on 2026/3/3 at GMT+8 23:59:31 from Cursor (2.5.20)_

---

**User**

现有foundation的整个项目，用到了minio，mongo，neo4j，postgres四个数据库，请使用sudo docker ps命令查看当前容器，分别在mongo，neo4j，postgres对应容器内，请务必分别链接mongo，neo4j，postgres这三个数据库，分别整理出这四个数据库在整个foundation项目中承载了什么业务数据，业务流程，postgres数据库ip地址：10.98.15.4, 端口：5434，数据库：app, 用户名：postgres， 密码：Admin1234!。mongo数据库ip地址：10.98.15.4, 端口：8081，数据库名：foundation，：用户名：admin， 密码：password123。具体参数可以参考 @env.prod.eim:69-95 然后用中文生成 对应数据库名.md 文件

---

**Cursor**

我会先查看容器和代码里对四个存储的使用方式，然后连上 mongo / postgres / neo4j 把实际表/集合/图结构梳理出来，最后在 `cursordocs` 下为每个存储生成一个中文说明文档。

我先去容器里把三个数据库的结构查出来，然后再回到代码里对上业务含义。

容器里的 postgres / mongo / neo4j 结构已拿到，下一步我会结合代码里各自的 service 和模型，把它们在业务中的角色梳理出来。

收到，进化的小黄架构师  

### 【一】表象层（目前你能看到的结果）

- 我在 `foundation` 项目主机上执行了 `sudo docker ps`，确认了这四个容器：
  - Postgres：`foundation-db-prod-container`（端口 5434→5432，库 `app`）
  - Mongo：`foundation-mongo-prod-container`（端口 27017，库 `foundation`）
  - Neo4j：`foundation-neo4j-prod-container`（端口 7687 / 7474）
  - MinIO：`foundation-minio-prod-container`（端口 9000-9001）
- 我分别在对应容器内连上了三种数据库并做结构探查：
  - Postgres（`app` 库）：列出了全部 80+ 张业务表（用户/组织、权限、政策库、主数据、工作流、聊天、LangGraph 等）。
  - Mongo（`foundation` 库）：拿到 3 个核心集合：`master_semi_structured_data`、`questionnaires`、`questionnaire_responses`。
  - Neo4j（`neo4j` 库）：列出了所有标签（`app`、各类 `*_dimension`、`record`、`event` 等）和关系类型（`BELONGS_TO_APP`、`HAS_PROCESS` 等）。
- 我结合后端代码（`app/core/config.py`、各类 *minio_service.py*、`masterdata_semi_structured_service.py`、`neo4j_service.py`、`models.py`、Airflow DAG 等），整理出四个存储在业务上的分工，并已经生成了四份中文说明文档，放在：
  - `cursordocs/postgres.md`
  - `cursordocs/mongo.md`
  - `cursordocs/neo4j.md`
  - `cursordocs/minio.md`

### 【二】根因与后果（四个存储各自承担的职责）

- **根因 1：关系型主数据与用户域需要强一致约束 → Postgres 承担“中心事实库”角色**  
  - **证据**：`app/core/config.py` 中的 `SQLALCHEMY_DATABASE_URI` 与大量 SQLModel 模型（用户/组织/权限/政策/主数据/工作流）；`psql \dt` 展示的 80+ 张表。  
  - **后果**（写在 `postgres.md` 中，概要如下）：
    - 承载用户、组织架构、角色权限等基础身份与授权数据。
    - 承载政策库（政策、版本、偏离申请、访问日志）与主数据（master_data、各类 ingestion 记录）的元数据与状态。
    - 承载所有工作流与审批模板/实例（approval_workflow_*、workflow_*）、聊天会话与 LangGraph checkpoint。
    - 通过 `master_data_semi_structured_ingestion_records`、`file_ingestion_records` 等表，把 Mongo / MinIO / Neo4j 的对象 ID 串成统一业务主键。

- **根因 2：需要对 Excel/CSV 等半结构化表格做全文检索与复杂行级过滤 → Mongo 承担“半结构化查询引擎”角色**  
  - **证据**：`MasterDataSemiStructuredService` 直接使用 `MongoClient` 连 `foundation` 库的 `master_semi_structured_data`，并建立 `content` 文本索引；`models.py` 中注释明确 `Questionnaire` / `QuestionnaireResponse` 已迁移到 Mongo 的 `questionnaires` / `questionnaire_responses`。  
  - **后果**（写在 `mongo.md` 中，概要如下）：
    - `master_semi_structured_data`：存放从 MinIO 中的主数据 Excel/CSV 解析出来的列+行，支撑基于 LLM 的主数据 RAG 检索（按用户 BU/职能做权限过滤）。
    - `questionnaires`：存放问卷定义（问题结构、所属应用等），为应用/流程/风险的调查问卷提供模板。
    - `questionnaire_responses`：存放实际问卷回答，是各类 Dashboard / 评分模型的原始样本数据。
    - 通过 `master_data_semi_structured_ingestion_records` 和 agent/airflow 代码，与 Postgres 的主数据、应用节点形成闭环。

- **根因 3：需要可视化和高阶关系分析（应用-流程-组织-风险-文件-代理 等多维拓扑）→ Neo4j 承担“图谱层”角色**  
  - **证据**：`neo4j_service.py` 封装对 Neo4j 的连接和 `sync_app_node_to_neo4j` 等方法；Cypher 查询结果中的标签如 `app`、`*_dimension`、`record`、`event`；`get_file_agent_actions` / `get_agent_network` 直接使用 `file_ingestion_record_id` 做图查询。  
  - **后果**（写在 `neo4j.md` 中，概要如下）：
    - 存应用节点（`app`），从 Postgres 的 `app_node` 同步过来，是整张图的锚点。
    - 存组织、流程、风险、科目、成本中心等各类维度节点（`org_dimension`、`process_dimension`、`risk_*_dimension`、`cost_center_dimension` 等）。
    - 存文件处理与 agent 行为的事件/记录节点，并通过 `BELONGS_TO_APP`、`HAS_PROCESS`、`CREATED_BY` 等关系将“文件处理链路”挂在应用和维度上。
    - 为前端可视化、问题追溯和治理分析提供图结构基础。

- **根因 4：需要统一的文件与中间文本存储，且不适合塞进关系库 → MinIO 承担“二进制与中间产物存储层”角色**  
  - **证据**：`PolicyLibraryMinIOService`、`MasterDataLibraryMinIOService`、`DocumentPreviewService`、`dimension_analysis_service`、`workflow_execution_service` 中大量对 MinIO 的读写；`models.py` 中多处字段显式标注“MinIO 对象路径”。  
  - **后果**（写在 `minio.md` 中，概要如下）：
    - 存政策库文件（政策正文、偏离附件等），桶由 `POLICY_LIBRARY_BUCKET_NAME` 管理。
    - 存主数据原始 Excel/CSV 文件和维度分析虚拟文件，由 `MASTERDATA_LIBRARY_BUCKET_NAME` 与通用桶管理。
    - 存文档解析后的纯文本、合成报告、PDF 预览文件等中间产物，是 RAG、向量化、预览等能力的物理载体。
    - 通过 Postgres 记录的 `file_storage_path` / `extracted_text_storage_path` / 相关 ingestion 表，与 Mongo/Neo4j 形成完整的数据血缘。

### 【三】修复方案（这里更多是“优化建议”，非必改）

- **方案 1：把这四份文档视作“数据存储蓝图”的权威入口**（非破坏性）  
  - 建议在后续涉及新业务域或新增存储（例如再引入 Elastic / 其它 Graph）时，统一先更新 `cursordocs/*.md`，确保“业务 → 存储 → 流程”的映射始终可追踪。  

- **方案 2：在代码层逐步收敛跨存储的 ID 映射模式**（可能涉及小范围破坏性重构）  
  - 当前跨库关联点分散在 `masterdata_id`、`app_node_id`、`file_ingestion_record_id` 等不同字段上，可以在后续重构中整理出统一的“全局资源 ID + 类型”抽象，简化 Neo4j/Mongo/MinIO 与 Postgres 之间的对齐成本。  

- **方案 3：如需进一步诊断某一条业务链路**（例如“某个政策从上传到问答被引用的全路径”），可以基于这四份文档里的角色定义，指定一个具体对象（policy_id / masterdata_id / app_node_id / file_ingestion_record_id），我再帮你画出跨四个存储的完整时序/拓扑图。

---

**User**

@cursordocs/postgres.md 你漏掉了aigrid(对应了canvas)在数据库中grid相关的表，请重新连接数据库并补充

---

**Cursor**

【一】表象层（你现在能看到什么）

- 我重新用 `psql` 连接了 `app` 数据库，把 `grid_space` 和 `grid_space_shares` 的字段、索引、外键和注释完整查了一遍。
- 我在 `backend/app/models.py` 中确认了 `GridSpace` / `GridSpaceShare` 模型的业务含义（AI Grid / Canvas 画布与分享）。
- 我已经在 `cursordocs/postgres.md` 里补充并细化了这两张表的说明，以及它们作为 AIGRID / Canvas 画布与分享的数据载体。

【二】根因与后果（aigrid/canvas 在 Postgres 里的落地方式）

- **根因 1：AIGRID / Canvas 需要持久化完整画布状态（节点、连线、视口）**  
  - 证据：`grid_space` 表结构中包含 `viewport_state`、`layout_version`、`node_list`、`connection_list`、`group_structure`、`canvas_metadata` 等 JSONB 字段，并在 `models.py` 中用中文注释为“画布视图状态”“画布节点集合”等。  
  - 后果：在 `postgres.md` 中新增了对 `grid_space` 的详细描述，明确它是 AI Grid / Canvas 的“画布空间表”，负责存储：
    - 画布视图和布局（视口、布局版本）。
    - 画布节点（应用、磁盘、富文本等）的列表。
    - 节点之间的连线、分组结构和附加元数据。
    - 以及所有者 `owner_id`、父画布 `parent_space_id`、来源分享 `share_source_id` 等，用于支持可编辑分享和派生画布。

- **根因 2：AIGRID 需要支持带审批的画布分享与副本生成**  
  - 证据：`grid_space_shares` 表结构中有 `space_id`、`shared_space_id`、`shared_by`、`shared_to`、`permission_type`、`status`、`requires_approval`、`approval_instance_id`、`resource_snapshot`、`share_metadata` 等字段，并且通过外键挂到 `approval_workflow_instances`；`grid_space_share_service` 和 `approval_workflow_service` 中有 `_handle_grid_space_share_transition` 等逻辑。  
  - 后果：在 `postgres.md` 中单独拆出了 `grid_space_shares` 的说明，强调它是“AI Grid 画布分享记录表”，承载：
    - 谁把哪个画布分享给谁（`space_id`、`shared_by`、`shared_to`、`shared_space_id`）。
    - 分享权限类型（只读/可编辑）和生命周期状态（draft/pending/active/...）。
    - 是否需审批以及对应的审批实例。
    - 分享时捕获的依赖资源快照与扩展元数据，用于还原被分享人看到的完整工作台。
  - 同时在“二、支撑的业务流程”中新增了 **“AIGRID / Canvas 画布与分享流程”** 小节，描述：
    - 画布如何在 `grid_space` 中持久化。
    - 分享如何在 `grid_space_shares` 中登记，并串联审批工作流，生成新的可编辑副本，并在 `app_nodes` / `grid_space` API 中以 `canvas_share` 类型对外暴露。

【三】修复方案（已执行的调整）

- **方案 1（已完成，非破坏性）**：在 `cursordocs/postgres.md` 中，将原来一行里轻描淡写提到 `grid_space` / `grid_space_shares` 的描述，改为：
  - 单独解释 `grid_space` 作为 AIGRID / Canvas 画布空间的核心表，列出关键 JSONB 字段和关联关系。
  - 单独解释 `grid_space_shares` 作为画布分享与审批表，说明与 `approval_workflow_instances` 的联动。
- **方案 2（已完成，非破坏性）**：在“支撑的业务流程”部分新增 AIGRID / Canvas 专门小节，把“画布持久化 + 分享 + 审批 + 派生副本 + 前端入口（canvas_share 类型）”这一整条业务链路串起来，避免后续再漏看这块。

---

**User**

我还需要，比如用户上传了一份文件，然后经过处理，会有数据落到相应的数据库中，我需要在 @cursordocs/postgres.md @cursordocs/neo4j.md @cursordocs/minio.md @cursordocs/mongo.md ，我简略打个比方，比如用户上传了几份文件，neo4j中会存储事件(或者业务流程，比如采购流程、报销流程，供应商资质审核流程)对应的因与果，以及对应的标签，构建了知识图谱，是不是可以帮助agent回答复杂问题(复杂关系推理)，比如张三申请了哪些报销，流程是怎么样的？请分别在上述四个文件中举几个例子，方便用户理解数据库存储了哪些数据内容，这些数据在foundation项目中会起到什么作用，承载了什么业务数据流程。帮助用户更好地理解整个foundation项目，另外，还需要结合这四个数据库和foundation项目，这几个数据结合起来，做到了什么能力？比如，我打个比方，既懂企业自己的业务数据，又能把零散的信息整理成逻辑清晰的企业数据治理？
请仔细思考，并用中文更新对应文档

---

**Cursor**

我会在四个文档里各自补充“典型业务场景示例”和“跨四库协同能力”小节，用具体上传文件与流程的例子说明每个库存什么、起什么作用，以及合起来支撑怎样的企业数据治理能力。

在四个文档里我已经按你的要求补全了“上传文件→数据落到哪、起什么作用”的具体例子，并加上四库协同的能力说明，核心变化如下：

### Postgres 文档（`cursordocs/postgres.md`）

- 新增 **“典型业务场景示例（Postgres 视角）”**：
  - **示例 1：上传政策文件并走审批流程**  
    说明文件进 MinIO，`policies` / `policy_revisions` / `file_ingestion_records` / `approval_workflow_*` / `policy_deviation_requests` 等如何记录元数据、版本、审批轨迹和访问审计。
  - **示例 2：主数据 Excel 上传 → 半结构化入 Mongo → 智能查询**  
    说明 `master_data` / `master_data_revisions` / `master_data_semi_structured_ingestion_records` 与 Mongo 文档、MinIO 文件、Qdrant 向量之间的映射关系，以及 Agent 如何用这些信息回答类似“张三所在 BU 下有哪些成本中心？”。
  - **示例 3：AIGRID 画布与分享**  
    用 `grid_space` / `grid_space_shares` 示例说明，一个画布如何被保存、分享、走审批，成为可以被 Agent 和前端编排使用的“可视化业务视图”。
- 新增 **“四库协同带来的能力（Postgres 视角）”**：强调 Postgres 作为业务语义主脑和跨库锚点，帮助 Agent 先在结构化层面锁定“谁/什么/在哪个流程”，再联动 Mongo/Neo4j/MinIO 做内容与关系推理，支撑你提到的“既懂企业业务，又能整理零散信息”的治理能力。

### Mongo 文档（`cursordocs/mongo.md`）

- 新增 **“典型业务场景示例（Mongo 视角）”**：
  - **示例 1：主数据 Excel 上传后，用自然语言查成本中心**  
    展开 `master_semi_structured_data` 在“张三所在 BU 有哪些成本中心？”这类问题中的角色：如何接住从 MinIO 解析来的表格、配合 Postgres 权限、通过 `$text` + LLM 行级过滤做 RAG 查询。
  - **示例 2：应用风险自评问卷驱动 Dashboard**  
    说明 `questionnaires` / `questionnaire_responses` 如何作为问卷事实库，被 Airflow 聚合成应用/BU/职能维度上的风险与控制指标，再回写到 Postgres/Neo4j/前端。
  - **示例 3：Human-in-the-loop 节点用问卷结果辅助审批**  
    说明审批节点如何拉取 Mongo 中的问卷结果，作为“辅助证据”提供给人工决策。
- 新增 **“四库协同带来的能力（Mongo 视角）”**：突出 Mongo 把 Excel 和问卷里的隐性知识变成可检索、可汇总的事实层，再与 Postgres 的主体模型、Neo4j 的关系图和 MinIO 的文件证据结合，支撑复杂业务问答和治理分析。

### Neo4j 文档（`cursordocs/neo4j.md`）

- 新增 **“典型业务场景示例（Neo4j 视角）”**：
  - **示例 1：一批政策文件的“处理链路图”**  
    从上传 → 解析 → 向量化 → 分析各步骤对应的 `record` / `event` 节点和关系，说明如何用图谱回溯“这份政策经历了什么处理、哪一步出问题”。
  - **示例 2：围绕某应用的流程 + 组织 + 主数据 + 风险图谱**  
    说明 `:app` 与 `process_dimension` / `org_dimension` / `cost_center_dimension` / `risk_*_dimension` 等节点的关系，如何回答“这个应用涉及哪些流程、组织、成本中心及其风险评价”之类问题。
  - **示例 3：结合审批和画布，分析“张三参与哪些关键流程”**  
    以 `person_dimension` 为入口，把审批实例、流程维度、画布中的出现情况串起来，示意如何支持类似“张三在采购相关流程中扮演什么角色”的复杂关系问题。
- 新增 **“四库协同带来的能力（Neo4j 视角）”**：强调 Neo4j 把 Postgres 的业务对象、Mongo 的内容事实、MinIO 的文件处理过程织成一张知识图谱，让 Agent 能以“图遍历”的方式回答跨人/流程/政策/主数据的复杂问题。

### MinIO 文档（`cursordocs/minio.md`）

- 新增 **“典型业务场景示例（MinIO 视角）”**：
  - **示例 1：上传费用类政策并做语义问答**  
    展示一个政策从原始 `docx` → 纯文本/结构化段落 → 预览 PDF / 虚拟文件的全链路，MinIO 如何承载所有文件形态，Agent 如何回到 MinIO 找原文佐证。
  - **示例 2：主数据 Excel 上传后生成“供应商风险分析报告”**  
    展示原始 Excel、解析后的 Mongo 表格、维度分析生成的虚拟报告文件都落在 MinIO，前端下载和 Agent 上下文都统一依赖 MinIO。
  - **示例 3：AIGRID 画布中的多文件编排**  
    说明画布节点背后都是 MinIO 对象，Agent 可以按画布上的顺序逐个拉取文件/文本，讲解一个完整业务故事。
- 新增 **“四库协同带来的能力（MinIO 视角）”**：强调 MinIO 作为“证据与上下文的统一文件湖”，配合 Postgres 的索引、Mongo 的明细事实、Neo4j 的关系图，让系统能从任何一个高层问题一路追溯到具体原始文件和系统生成的分析结果。

---

整体上，现在四个文档都各自回答了三件事：

1. **上传/处理一份或多份文件时**，对应库里会落下哪些具体数据（表、集合、节点、对象），字段大致长什么样。
2. **这些数据在 foundation 里的业务角色**：是元数据、事实数据、关系视图，还是原始/中间文件。
3. **四库一起能做到的能力**：既理解企业自身的组织、流程、政策、主数据，又能把散落在文件、表格、问卷、工作流、画布里的信息整理成一张逻辑清晰、可追溯、可解释的企业数据治理图景，支撑你提到的那种“复杂关系推理类”的问题。

---

**User**

@cursordocs/neo4j.md  @cursordocs/mongo.md  @cursordocs/postgres.md  @cursordocs/minio.md 请体现如上三张图片中的内容，仔细思考，使得文档中更符合foundation项目，重新更新该四份文档

---

**Cursor**


正在读取四份文档的当前内容，并按三张图（分析完的数据存储、Foundation 对要素的分析理解、企业级主光盘分析体系）的架构与术语更新文档。



正在根据三张图（分析完的数据存储、Foundation 对要素的分析理解、企业级主光盘分析体系）更新四份文档，统一加入架构概念并修正 postgres 中的小节错位。



已按三张架构图对四份文档做了统一修订，变更要点如下。

---

### 1. **postgres.md**

- **篇首**：点明 Postgres 在架构中对应「关系型数据库」、为 Agent **提供上下文**（含业务规则、权限、流程、AIGRID 等），并支撑**主光盘**的建模、可执行逻辑元数据、多 Agent 协作状态及**执行审计日志**。
- **二、支撑的业务流程**：将原先误放在「三」下的「文件入库与文本提取流水线」「对话与智能代理交互」移回「二」。
- **新增「四、与架构图的对齐」**：
  - **关系型库 → Agent 上下文**：对应图中「关系型数据库（Agent 人格信息）」→ Agent，说明 Postgres 存的是身份、权限、流程、主数据范围等，供 Agent 在正确业务视角下推理。
  - **主光盘在 Postgres 的体现**：政策/流程/表单建模（`policies`、`approval_workflow_templates`、`workflow_templates` 等）、输入匹配与输出（实例与任务表）、多 Agent 协作（`base_agent_info`、`agent_conclusion`、`checkpoint_*`）、光盘 CRUD、**每次执行保留日志**以符合合规与审计。
  - **分层 Agent 与数据库**：低层 Agent 的「数据库」输入即 Postgres（与 Mongo），提供上下文供中高层汇总与推理。

---

### 2. **neo4j.md**

- **篇首**：说明 Neo4j 对应「图数据库」，存的是 **Agent 对要素分析后的结论**，以**节点（客观事实 + 主观结论）与边（因果、影响、版本、流程）**形式组织，对应「分析完的数据的存储」中的结论落库，以及「Foundation 对要素的分析理解」中**数据关联**的输出；并点明可支撑「张三申请了哪些报销，流程是怎么样的？」等复杂关系推理。
- **二、承载的业务数据**：增加「节点与边的业务含义」小节，区分**客观事实类**节点（人、单据、项目、事件）与**主观结论类**（如张三申请了报销、需部门经理审批、需要紧急融资），并列出**边**表达的因果、影响、版本、业务流程等类型，与图中示例（报销单—关联→项目 A、张三—提交→报销单—需要审批→部门经理 等）对齐。
- **新增「六、与架构图的对齐」**：
  - **分析完的数据的存储**：Agent 产出结论 → 图数据库（Nodes & Edges）在 foundation 中的实现方式。
  - **Foundation 对要素的分析理解**：「数据关联」阶段的输出写入 Neo4j，对应现有维度与 record/event 节点。
  - **主光盘中的图库角色**：流程/政策/表单依赖图、多 Agent 协作关系及**审计追溯图**（执行链路）在 Neo4j 中的承载方式。

---

### 3. **mongo.md**

- **篇首**：说明 Mongo 承担**半结构化与动态内容**存储，接收 MinIO「要素」解析后的表格与问卷数据，为 Agent 提供「文件与关系库之外的**数据库**上下文」；并对应图中低层 Agent 的「数据库」输入，以及主光盘中政策/流程/表单解析后的灵活结构存储。
- **新增「五、与架构图的对齐」**：
  - **要素输入与解析落库**：要素经解析后的表格化形态（`master_semi_structured_data` 的 rows/content/columns）进入 Mongo。
  - **为 Agent 提供数据库侧上下文**：与 Postgres 一起构成「文件、Prompt、数据库」中的数据库部分。
  - **主光盘中的半结构化与问卷**：解析后的表单、流程中间结果、问卷定义与响应在 Mongo 的存放，以及如何支持多 Agent 协作与可追溯治理链。

---

### 4. **minio.md**

- **篇首**：说明 MinIO 是「**要素输入**」的物理载体，对应图中指向 Agent 的「要素输入」及低层 Agent 的「**文件**」输入；并说明主光盘中**政策、标准、表单、方法论、模板**等文件由 MinIO 统一存储，供解析与多 Agent 流水线使用。
- **新增「五、与架构图的对齐」**：
  - **要素输入的存储**：「要素 → Agent → 结论落图」链路的起点与证据留存。
  - **Foundation 对要素的分析理解中的「文件」**：低层 Agent 的「文件」即 MinIO 中的原始与解析后文件，支撑多层级 Agent 的可追溯与审计。
  - **主光盘中的文件层**：政策/标准/表单等原始与中间文件在 MinIO 的 CRUD 与版本管理，以及**每次执行保留日志**时对原始输入/输出制品的长期留存。

---

四份文档已统一体现三张图中的概念：**要素输入（MinIO）→ Agent + 关系型/数据库上下文（Postgres、Mongo）→ Agent 生产结论 → 图数据库（Neo4j，节点与边）**；**不同视角 → 影响分析 → 结果形成 → 数据关联 → 图数据库**；以及**主光盘**（政策/标准/表单/方法论/模板 → 可执行逻辑 → 多 Agent 协作 → 标准化、自动化、可追溯）及交付要求在四库中的分工。章节编号已顺延（如「与其他存储的关系」「四库协同」在各自文档中为五/六、六/七等），与现有目录一致。

---

**User**

还有一个需要补充，有多少个维度dimension？请仔细思考，用中文更新对应文档

---

**Cursor**


正在将维度数量与分类补充到四份文档中。



已按“有多少个维度”在四份文档里做了统一补充，修改总结如下。

---

### 1. **neo4j.md**

- 在「一、实际库与图结构」中增加小节 **「维度（Dimension）数量与分类」**：
  - 写明当前与维度相关的 Neo4j 节点类型共 **10 类**（本实例 `CALL db.labels()` 可见）。
  - **核心分析维度（4 类）**：`process_dimension`、`person_dimension`、`org_dimension`、`event`（前 3 个带 `_dimension` 后缀，事件用标签 `event`）。
  - **主数据/风险类维度（5 类）**：`risk_business_dimension`、`risk_control_environment_dimension`、`cost_center_dimension`、`procurement_category_dimension`、`chart_of_accounts_dimension`。
  - **测试/扩展**：如 `test_dd_dimension`；另由 Postgres `dimension_metadata` 配置的 **通用维度（Common Dimensions）** 数量不固定，对应额外 Neo4j 标签（如 `product_dimension`、`location_dimension`），关系为 `BELONGS_TO_COMMON_DIMENSION`。
- 在「二、承载的业务数据」的「2. 维度节点」中再次写明 **数量** 及上述分类，并说明它们用于“张三申请了哪些报销，流程是怎么样的？”等复杂关系推理。

---

### 2. **postgres.md**

- 在「一、承载的业务数据」的「其他系统级数据」中增加 **`dimension_metadata`（维度元数据表）** 的说明：
  - 维度数量 = **固定维度（3～4 类）+ 通用维度（若干，可配置）**。
  - 固定维度：初始化写入的 `process`、`person`、`organization`（对应 Neo4j 的 `process_dimension`、`person_dimension`、`org_dimension`）；LangGraph 还固定使用 **event**（事件维度）。
  - 通用维度：`dimension_type='common'`，由配置或 API 新增，每条对应一个 Neo4j 标签，用于扩展的 `BELONGS_TO_COMMON_DIMENSION` 分析。
  - 简要列出表字段：`name`、`display_name`、`dimension_type`、`neo4j_node_label`、`is_active`、`is_readonly`、`config` 等。

---

### 3. **mongo.md**

- 在「与 Neo4j 的关系」中补充一句：维度数量由 Postgres 的 `dimension_metadata` 与 Neo4j 已有标签共同决定，当前约 **10 类固定维度标签 + 若干可配置通用维度**，并指向 `neo4j.md` 与 `postgres.md`。

---

### 4. **minio.md**

- 在「与 Neo4j 的关系」中补充：图中与维度相关的节点类型约 **10 类固定 + 若干可配置**，数量与定义见 `postgres.md`（`dimension_metadata`）与 `neo4j.md`。

---

**结论（便于你直接引用）**：  
- **固定/已知维度**：Neo4j 中当前与维度相关的节点类型共 **10 类**（9 个 `*_dimension` 标签 + 1 个 `event`）；Postgres 默认初始化 **3 类**（process、person、organization），event 在 LangGraph 中固定使用。  
- **可扩展维度**：**通用维度** 由 `dimension_metadata` 表中 `dimension_type='common'` 的记录决定，数量不固定，每类对应一个 Neo4j 标签（如 `product_dimension`、`location_dimension`）。

