# 核心架构模式 (Core Architecture Patterns)

本文件是 OSC-YALAB 项目核心架构模式的**唯一事实来源 (Single Source of Truth)**。所有关于数据处理模式的实现、修改和讨论，都必须以此文件为基准。

项目探索并实践了以下三种核心数据处理模式：

1.  **模式一：同步旁路缓存 (Synchronous Cache-Aside)**
2.  **模式二：异步缓存失效 (Asynchronous Cache Invalidation via CDC)**
3.  **模式三：RAG 数据流架构 (RAG Data Flow Architecture)**

---

## 模式一：同步旁路缓存 (应用层负责)

这是传统的、在应用代码中显式实现的缓存策略。

-   **识别方法**:
    -   在 `internal/data` 的写入方法（如 `Update`、`Delete`）中，你会看到对 `gocache` 的 `Delete()` 或 `Invalidate()` 的显式调用。
    -   `biz` 层的 `Repo` 接口定义中，写入方法可能会返回 `error`，暗示需要处理缓存操作的结果。

-   **你的任务**:
    -   **保持模式**: 当你修改这些方法时，必须维持 **“先写数据库，再删缓存”** 的顺序。
    -   **原子性**: 数据库操作和缓存操作应被视为一个整体，尽管它们不是事务性的。确保在数据库写入成功后才执行缓存失效。

-   **读取操作**: 在 `internal/data` 的 Repository 实现中，遵循 `Read -> Cache -> DB` 的顺序。先检查缓存，未命中时查询数据库，并将结果回填。
-   **写入操作**: 在 Repository 的更新/删除方法中，采用"先更新数据库，再使缓存失效"的策略。**这是应用代码的显式责任。**

---

## 模式二：异步缓存失效 (CDC + Flink 负责)

这是更高级的、解耦的事件驱动缓存策略。应用服务的写入操作**不再关心缓存失效逻辑**，只负责更新数据库。这使得业务代码更纯粹，系统组件间耦合度更低。

-   **识别方法**:
    -   在 `internal/data` 的写入方法中，**没有任何** `gocache.Delete()` 或 `Invalidate()` 调用。
    -   代码注释中可能标注 `// Cache invalidation handled by CDC pipeline`。

-   **你的任务**:
    -   **严禁添加缓存逻辑**: 在应用代码中**绝对不能**添加任何缓存失效代码。
    -   **只关注数据库**: 写入方法只需确保数据库操作的正确性和事务完整性。
    -   **信任 CDC 管道**: 缓存失效完全由外部的 CDC 管道自动化处理。

-   **工作流程**:
    1.  **应用写入**: `internal/data` 执行数据库 `INSERT/UPDATE/DELETE` 操作。
    2.  **CDC 捕获**: Debezium 监控 PostgreSQL 的预写日志 (WAL)，捕获所有行级别的变更事件。
    3.  **事件发布**: Debezium 将捕获到的变更事件发布到指定的 Kafka Topic。
    4.  **流式处理**: 一个独立的 Flink 作业订阅该 Kafka Topic，实时消费数据变更事件。
    5.  **执行失效**: Flink 作业解析事件内容，计算出需要失效的缓存键 (Cache Key)，然后调用 Memcached 的接口删除对应的缓存项。

-   **可视化流程**:
    ```mermaid
    flowchart LR
        KratosApp[Kratos App]
        PostgreSQL[(PostgreSQL)]
        Debezium[Debezium]
        Kafka[Kafka]
        FlinkJob[Flink Job]
        Memcached[(Memcached)]
        
        KratosApp -->|Write| PostgreSQL
        PostgreSQL -->|DB Change| Debezium
        Debezium -->|Event| Kafka
        Kafka -->|Stream| FlinkJob
        FlinkJob -->|API Call Invalidate| Memcached
    ```

---

## 模式三：RAG 数据流架构 (RAG Data Flow Architecture)

这是系统的核心查询链路，整合了搜索与 AI 生成能力。它明确了 Go 服务和 Python 服务的职责边界：Go (Kratos) 负责数据管道和业务流程编排，而 Python 负责核心 AI 计算。

### 数据写入流程 (Go 微服务负责)

1.  **数据捕获**: Debezium 监控 PostgreSQL 的变更日志，捕获原始数据变更。
2.  **数据富化**: Flink 作业消费原始数据，对其进行业务逻辑处理和数据增强。
3.  **富化数据发布**: 处理后的数据写回 Kafka 的新 Topic (如 `topic_rich_data`)。
4.  **向量处理**: Go 微服务集群订阅富化数据 Topic，通过调用向量化模型 (**Qwen-Embedding-0.6B**) 将文本转换为向量。
5.  **索引写入**: 将向量化的数据写入 Elasticsearch，建立可搜索的索引。

-   **可视化流程**:
    ```mermaid
    flowchart TD
        PostgreSQL[(PostgreSQL)]
        Debezium[Debezium]
        KafkaRaw[Kafka: topic_raw]
        FlinkJob[Flink Job]
        KafkaRich[Kafka: topic_rich]
        GoCluster[Go Microservices Cluster]
        QwenEmbed[Qwen-Embedding Model]
        ES[(Elasticsearch)]
        
        PostgreSQL -->|CDC| Debezium
        Debezium -->|Raw Data| KafkaRaw
        KafkaRaw -->|Consume| FlinkJob
        FlinkJob -->|Enrich & Rich Data| KafkaRich
        KafkaRich -->|Subscribe| GoCluster
        GoCluster -->|Vectorize| QwenEmbed
        GoCluster -->|Index| ES
    ```

### 数据读取流程 (Go + Python 协作)

1.  **用户查询**: `Kratos 服务`接收来自用户的查询请求。
2.  **向量检索**: `Kratos 服务`调用 `Python AI 服务`的接口，传递用户问题。`Python AI 服务`将问题向量化，并查询 `Elasticsearch` 获取最相关的文档 ID 列表。
3.  **上下文组装**: `Kratos 服务`接收到文档 ID 列表后，从 `gocache` 或 `PostgreSQL` 中获取完整的文档内容，组装成 LLM 需要的上下文 (Context)。
4.  **答案生成**: `Kratos 服务`再次调用 `Python AI 服务`，这次将**原始问题**和**组装好的上下文**一同传递过去。`Python AI 服务`内部构建最终的 Prompt，并调用 **Qwen2.5-7B-Instruct** 大语言模型。
5.  **返回结果**: `Kratos 服务`获取到 AI 生成的最终答案，返回给用户。

-   **可视化流程**:
    ```mermaid
    sequenceDiagram
        participant User
        participant KratosApp as Kratos App
        participant PythonAI as Python AI Service
        participant ES as Elasticsearch
        participant DB as PostgreSQL/Cache
        participant LLM as Qwen2.5-7B-Instruct
        
        User->>KratosApp: 1. Query
        KratosApp->>PythonAI: 1. Query
        PythonAI->>ES: 2. Vector Search
        ES-->>PythonAI: 2a. Return IDs
        PythonAI-->>KratosApp: 2b. Return Document IDs
        KratosApp->>DB: 3. Get Data by ID
        DB-->>KratosApp: 3a. Return Documents
        Note over KratosApp: 3b. Assemble Context
        KratosApp->>PythonAI: 4. Query + Context
        PythonAI->>LLM: 4. Call LLM with Prompt
        LLM-->>PythonAI: 4a. Generated Answer
        PythonAI-->>KratosApp: 5. Final Answer
        KratosApp-->>User: 6. Return
    ```
