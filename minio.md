## MinIO 在 foundation 中的角色

在 foundation 的整体架构中，MinIO 是「**要素输入**」的物理载体：用户上传的文档、表格、图片等原始文件首先落入 MinIO，再经解析、萃取后变为可被 Agent 使用的文本与结构化数据（部分进入 Mongo/Postgres/Neo4j）。对应「分析完的数据的存储」图中指向 Agent 的「要素输入」箭头，其数据源即 MinIO 中的文件及解析后写回 MinIO 的中间产物。在「Foundation 对要素的分析理解」中，低层 Agent 的输入之一「**文件**」即来自 MinIO；在企业级主光盘分析体系中，**政策、标准、表单、方法论、模板**等以文件形式存在时，也由 MinIO 统一存储，供解析建模为光盘逻辑及多 Agent 协作流水线消费。

### 一、基础配置与存储桶

- **连接与凭证（来自 `app/core/config.py` 与 `env.prod.eim`）**
  - 容器：`foundation-minio-prod-container`（镜像 `minio/minio:latest`）。
  - 端口映射：主机 `9000-9001` -> 容器 `9000-9001`。
  - 配置项：
    - `MINIO_ENDPOINT`：内部访问端点（容器网络），如 `minio:9000`。
    - `MINIO_EXTERNAL_ENDPOINT`：对外暴露给前端或外部系统的端点（通常是 Nginx/域名）。
    - `MINIO_ROOT_USER` / `MINIO_ROOT_PASSWORD`：访问凭证。
    - `MINIO_BUCKET_NAME`：通用文件桶（例如文本萃取缓存等）。
    - `POLICY_LIBRARY_BUCKET_NAME`：政策库文件桶。
    - `MASTERDATA_LIBRARY_BUCKET_NAME`：主数据文件桶。
    - `FILE_RETENTION_BUCKET_NAME`：文件留存与归档桶。
    - `MINIO_DATA_VOLUME_PATH`：数据卷路径，用于磁盘空间监控。

- **服务封装**
  - `PolicyLibraryMinIOService`（`policy_library_minio_service.py`）
    - 专门负责政策库文件的上传、下载、列举、删除与预签名 URL。
    - 使用 `POLICY_LIBRARY_BUCKET_NAME` 作为 Bucket。
  - `MasterDataLibraryMinIOService`（`masterdata_library_minio_service.py`）
    - 专门负责主数据文件的上传、下载、列举、删除与预签名 URL。
    - 使用 `MASTERDATA_LIBRARY_BUCKET_NAME` 作为 Bucket。
  - `DocumentPreviewService`（`document_preview_service.py`）
    - 基于以上两个 Service，对 MinIO 中的 Office 文档进行 PDF 预览转换，并返回预签名链接或对象名。

### 二、承载的业务数据

- **1. 政策库文件（Policy Library）**
  - 业务流程：
    - 在创建或修订政策时，通过 `backend/app/api/v1/policies.py`、`policy_revisions.py`、`policy_library.py` 中的接口上传文件。
    - 上传逻辑：
      - 使用 `PolicyLibraryMinIOService.upload_file` 将文件存储到 Bucket 中，路径形式如：
        - `policy/{policy_id}/revisions/{revision_id}/{filename}`
        - 或偏离申请附件：`deviation/{deviation_id}/{filename}`
      - 返回的 `object_name` 与文件元数据写入 Postgres 相应表（如 `policies`、`policy_revisions`、`policy_deviation_attachments`）。
  - 承载内容：
    - 政策 PDF / Office 文档的原始文件。
    - 偏离申请、审批等相关的附件。
  - 下游使用：
    - 文档解析微服务（`document-parser` 容器）从 MinIO 拉取政策文件，提取文本后再写回 MinIO（见下文“提取文本文件”）。
    - 前端阅读时通过：
      - `policy_library` API 生成预签名 URL 或通过后台流式代理，避免前端直接暴露 MinIO 凭证。

- **2. 主数据文件（Master Data Library）**
  - 业务流程：
    - 在创建主数据或上传主数据版本时，通过 `masterdata.py`、`masterdata_revisions.py`、`masterdata_library.py` 上传文件。
    - 上传逻辑：
      - 使用 `MasterDataLibraryMinIOService.upload_file` 存储到 `MASTERDATA_LIBRARY_BUCKET_NAME`。
      - 对象路径形如：`masterdata/{masterdata_id}/revisions/{revision_id}/{filename}`。
      - 写入 Postgres 中的 `master_data` / `master_data_revisions` / `file_ingestion_records` 等表的路径字段。
  - 承载内容：
    - 各类主数据 Excel/CSV 文件（如财务科目表、成本中心表、供应商列表等）。
  - 下游使用：
    - Airflow DAG `masterdata_semi_structured_to_mongodb.py` 从 MinIO 拉取文件，解析为半结构化数据，写入 Mongo `master_semi_structured_data`。
    - 同时在 Postgres `master_data_semi_structured_ingestion_records` 中登记 ingestion 映射。

- **3. 文本萃取与虚拟文件（Extracted Text & Virtual Files）**
  - 相关服务与代码：
    - `backend/app/api/v1/report.py`：
      - 在上传文件或处理报告时，将文档解析得到的纯文本保存回 MinIO。
      - 字段说明：
        - `file_storage_path`：原文件在 MinIO 的对象路径。
        - `extracted_text_storage_path`：解析后的纯文本在 MinIO 的对象路径。
    - `backend/app/services/dimension_analysis_service.py`：
      - 提供“虚拟文件”能力，将维度分析生成的文本（如合并摘要、规则说明）以文件形式保存到 MinIO。
      - 同时保存一份纯文本版本，供后续 RAG / 向量化使用。
    - `backend/app/services/workflow/workflow_execution_service.py`：
      - 在工作流节点中读取 `FileIngestionRecord` 对应的 MinIO 文本文件，组合成一个大的上下文输入给 Agent。
  - 业务含义：
    - MinIO 不仅保存原始文件，还保存“解释过的中间产物”（纯文本、虚拟文件等），形成端到端的数据血缘：
      - 原始文件 -> 萃取文本 -> 虚拟合成文件 -> 向量与知识库。

- **4. 文档预览 PDF**
  - `DocumentPreviewService` 流程：
    1. 对 MinIO 中的 Office 文件（`.docx` / `.pptx` / `.xlsx` 等）下载到临时目录。
    2. 调用 `soffice`（LibreOffice headless）将其转换为 PDF。
    3. 将生成的 PDF 上传回同一 Bucket 的 `previews/` 前缀路径，如：
       - 原始对象：`policy/123/revisions/1/policy.docx`
       - 预览对象：`policy/123/revisions/1/previews/policy.pdf`
    4. 对于前端嵌入场景：
       - 通过 `get_presigned_url_for_preview` 返回内联（`Content-Disposition: inline`）的预签名 URL。
  - 业务意义：
    - 为前端提供统一的 PDF 预览能力，而不暴露 MinIO 内部结构。

- **5. 文件留存与磁盘空间管理**
  - 配置：
    - `FILE_RETENTION_BUCKET_NAME`：用于长期归档或按策略保留文件。
    - `FILE_RETENTION_DEFAULT_DAYS`、`ENABLE_AUTOMATIC_CLEANUP` 等配置：
      - 控制文件留存时间与自动清理策略。
    - `MINIO_DATA_VOLUME_PATH` 与磁盘阈值：
      - `DISK_SPACE_WARNING_THRESHOLD`、`DISK_SPACE_RECOVERY_TARGET`：
        - 当磁盘使用超出阈值时触发告警或清理。
  - 业务含义：
    - MinIO 是整个系统文件生命周期管理（上传、解析、预览、归档、清理）的物理载体。

### 三、支撑的核心业务流程

- **1. 文件上传与登记流程**
  - 入口：
    - 前端通过政策库、主数据、偏离申请等上传文件接口。
  - 流程：
    1. API 接口接收 `UploadFile`。
    2. 调用 `PolicyLibraryMinIOService` 或 `MasterDataLibraryMinIOService` 将文件写入对应 Bucket。
    3. 将 MinIO `object_name`、Bucket 名写入 Postgres 相关表。
    4. 触发后续工作流（如 Airflow DAG、LangGraph 流程）处理这些文件。

- **2. 文档解析与多模态流水线**
  - 入口：
    - 上传文件后，通过 Airflow DAG 或后台任务调用 `document-parser` 微服务。
  - 流程：
    1. 从 MinIO 下载原始文件。
    2. 通过外部解析服务提取文本、表格、结构化信息。
    3. 将结果（纯文本、JSON、虚拟文件）再写回 MinIO，路径保存在 Postgres。
    4. 下游任务（RAG、向量化、维度分析）从 MinIO 读取这些中间产物。

- **3. RAG 与智能问答工作流**
  - 虽然向量数据库使用 Qdrant，但原始文本与中间文件仍存放在 MinIO：
    - 向量化过程读取 MinIO 上的纯文本（通过 `file_ingestion_records.extracted_text_storage_path`）。
    - 在需要重新构建向量库或回溯数据来源时，可以根据 Qdrant 中的元数据反查 MinIO 对象。

- **4. 报表与维度分析虚拟文件**
  - `dimension_analysis_service` 从主数据、政策库、问卷等多源数据生成合成文本或报表：
    - 将这些结果视为“虚拟文件”写入 MinIO（既有原文，也有提炼后的文本）。
    - 方便后续作为上下文注入到聊天/工作流中，或作为证据材料被下载/归档。

### 四、典型业务场景示例（MinIO 视角）

- **示例 1：用户上传新政策并由 Agent 做语义问答**
  - 文件与中间产物：
    - 用户上传《费用报销管理办法》，原始 `docx` 文件进入政策库 Bucket，对象名如 `policy/1001/revisions/1/expense_policy.docx`。
    - 文档解析服务将其转换为纯文本/结构化段落，生成 `policy/1001/revisions/1/expense_policy.txt` 等中间文件。
    - 维度分析/向量化服务可能会额外生成“条款拆解文件”“示例问答文件”等虚拟文件，仍然全部落在 MinIO。
  - Agent 问答：
    - 当用户提问“报销单需要哪些附件？”时，Agent 会：
      - 通过 Postgres 找到这份政策对应的 MinIO 文本对象路径。
      - 从 MinIO 拉取解析文本作为上下文，再结合向量检索结果（Qdrant）做答案生成。
    - 若用户需要原文佐证，前端则通过文档预览服务生成该政策的 PDF 预览 URL，仍然来自 MinIO。
  - 作用：
    - MinIO 在这个场景中是“所有与该政策相关文件的单一存储位置”，Agent 与前端都只需要依赖 MinIO + Postgres 即可找到任意形式的证据。

- **示例 2：上传主数据 Excel，并生成可下载的合并分析报告**
  - 起点：
    - 用户上传“供应商主数据 Excel”，路径如 `masterdata/2001/revisions/1/suppliers.xlsx`。
    - Airflow 解析后把明细写入 Mongo 的 `master_semi_structured_data`，解析状态与 Mongo 映射记录在 Postgres。
  - 分析与报告生成：
    - `dimension_analysis_service` 基于 Mongo 中的明细、政策库中的要求、问卷中的反馈，生成一份“供应商分级与风险分析报告”。
    - 报告以虚拟文件形式写回 MinIO，例如 `masterdata/2001/revisions/1/reports/supplier_risk_summary.txt` 和 `supplier_risk_summary.docx`。
  - 使用：
    - 用户在前端点击“下载分析报告”时，API 直接从 MinIO 流式拉取该虚拟文件。
    - 在聊天场景下，Agent 也可以把这份报告作为上下文注入回答。
  - 作用：
    - MinIO 让“由系统生成的二次加工结果”与“用户上传的原始文件”同等对待，统一纳入文件生命周期与治理范围。

- **示例 3：AIGRID 画布下的多文件编排**
  - 场景：
    - 用户在某个 GridSpace 画布上拖入多个文件节点（政策文档、主数据报表、历史报告等），画布结构存储在 Postgres 的 `grid_space` 表。
  - 文件角度：
    - 每个节点背后的文件对象都位于 MinIO 对应 Bucket 中。
    - 当 Agent 需要按照画布中的顺序去“串讲”某个业务场景（例如从政策 → 主数据 → 历史偏离案例），会依次从 MinIO 拉取对应文件或其文本版本。
  - 作用：
    - MinIO 提供了一种“从画布到文件”的稳定映射，使得画布不仅是可视化工具，也是组合多源文件上下文的执行计划载体。

### 五、与架构图的对齐（要素输入 · 文件 · 主光盘）

- **要素输入的存储**  
  「分析完的数据的存储」中，进入 Agent 的「要素输入」在 foundation 中即用户上传的政策、主数据、偏离申请附件等文件：先写入 MinIO，再由文档解析等服务提取文本/表格，部分结果写回 MinIO（如 `extracted_text_storage_path`），部分进入 Mongo（半结构化表格）或驱动 Postgres 中的 ingestion 记录与 Neo4j 中的处理事件。MinIO 因此是整条「要素 → Agent → 结论落图」链路的起点与证据留存地。

- **Foundation 对要素的分析理解中的「文件」**  
  图中低层 Agent 的输入包含「文件、Prompt、数据库」：**文件**即 MinIO 中的原始对象与解析后的文本/虚拟文件。多层级 Agent（低层过滤与初次加工、中层汇总、高层综合推理）所依赖的原始证据与中间文件均可从 MinIO 按路径拉取，保证可追溯与可审计。

- **主光盘分析体系中的文件层**  
  主光盘将「政策、标准、表单、方法论、模板」抽象为统一容器并转化为可执行逻辑、多 Agent 协作任务。这些对象的**原始形态**（PDF、Word、Excel 等）及解析产生的中间文件（纯文本、结构化片段）均适合存放在 MinIO；支持光盘新增、修改、删除时，对应文件的 CRUD 与版本管理也依赖 MinIO 与 Postgres 的路径记录。每次执行保留日志、合规审计时，MinIO 可长期保留原始输入与输出制品，供事后核查。

### 六、与其他存储的关系

- **与 Postgres 的关系**
  - Postgres 是 MinIO 对象的“索引层”与“控制平面”：
    - 每个业务文件（政策、主数据、偏离附件等）在 Postgres 有一条记录，包含 MinIO 对象名和 Bucket。
    - 文件解析、扫描、向量化等任务的状态与日志也在 Postgres 中记录。
  - MinIO 不保存业务语义字段，语义由 Postgres 中的元数据表负责。

- **与 Mongo 的关系**
  - 半结构化主数据：原始 Excel/CSV 在 MinIO，解析后的行列数据在 Mongo `master_semi_structured_data`。
  - Airflow DAG 将 Postgres / MinIO / Mongo 串联起来：
    - 从 Postgres 中读取主数据记录与 MinIO 路径。
    - 从 MinIO 取文件 -> 解析 -> 写入 Mongo -> 回写映射到 Postgres。

- **与 Neo4j 的关系**
  - 文件和解析过程在 Neo4j 中表现为 `record` / `event` 等节点及关系，`file_ingestion_record_id` 作为跨存储的关键连接点。图中与“维度”相关的节点类型（如 `process_dimension`、`person_dimension`、`org_dimension`、`event` 及各类主数据/风险维度、通用维度）共约 **10 类固定 + 若干可配置**，数量与定义见 `postgres.md`（`dimension_metadata`）与 `neo4j.md`。
  - 通过 Neo4j 可以从某个文件处理事件追溯到：
    - MinIO 中的原始文件（经由 Postgres 的路径字段）。
    - 相关的应用、维度、风险节点等。

### 七、四库协同带来的能力（MinIO 视角）

- **证据与上下文的统一“文件湖”**：无论是用户上传的原始文档，还是解析出来的纯文本、虚拟报告、PDF 预览，全部集中在 MinIO；Postgres 负责指路，Mongo/Neo4j 负责从内容和关系上理解这些文件。
- **支撑“从问题回溯到原始证据”的能力**：当 Agent 回答类似“张三的某次报销为什么被拒绝？”这类问题时，可以：
  - 在 Postgres 中定位相关审批实例与政策/主数据/问卷记录；
  - 在 Neo4j 中沿着关系找出涉及的文件处理链路；
  - 在 Mongo/Qdrant 中查到对应的内容与向量检索结果；
  - 最终回到 MinIO，拿出当时的原始政策 PDF、报销说明文件或系统生成的分析报告，给出“有证据可查”的解释。
- **企业数据治理视角**：MinIO 把“看得见摸得着的文档证据”统一起来，与 Postgres 的结构化业务模型、Mongo 的事实表格、Neo4j 的知识图谱一起，让企业既能回答“数据怎么算出来的”，也能展示“依据哪些原始资料做出的判断”，从而完成从“数据存储”到“可解释治理”的升级。

