# OSC-YALAB: Open Source Concepts - Yet Another LABoratory

## 1. 项目简介 (Project Introduction)

**OSC-YALAB** 是一个纯粹的、实验性的后端项目。

本项目的核心目标是：**通过最简单、最直接的实现，来验证和学习一套现代化的 Go 微服务技术栈和相关的数据架构模式。**

我们不追求复杂的业务逻辑，而是专注于以下两个核心领域的验证：

1.  **同步服务架构**: 基于 Kratos 框架，构建提供 gRPC/HTTP 接口的同步服务，并实践经典的旁路缓存模式。
2.  **异步数据流架构**: 基于 CDC (Change Data Capture) 和 Flink，构建事件驱动的、异步的数据处理管道，用于实现更高级的缓存一致性策略和数据同步。

本项目是一个用于快速原型设计和技术探索的“实验室”，旨在对比和理解不同架构模式的优劣。

---

## 2. 核心技术栈 (Core Technology Stack)

本项目将严格围绕以下技术选型进行构建。所有代码生成和开发决策都必须基于此技术栈。

| 类别 | 技术/库 | 版本/说明 |
| :--- | :--- | :--- |
| **微服务框架** | **Kratos** (`github.com/go-kratos/kratos/v2`) | 项目的骨架，提供 gRPC & HTTP 服务 |
| **语言** | Go | >= 1.22 |
| **数据库** | PostgreSQL | 主持久化存储 |
| **ORM** | GORM (`gorm.io/gorm`) | 数据库交互层 |
| **缓存** | `gocache` (`github.com/eko/gocache/v3`) | 统一缓存抽象层 |
| | ├─ L1 本地缓存 | Ristretto (`github.com/dgraph-io/ristretto`) |
| | └─ L2 分布式缓存 | Memcached |
| **消息队列** | Apache Kafka | 用于异步任务和事件驱动 |
| | └─ Kafka 客户端 | `segmentio/kafka-go` |
| **实时数据捕获**| **Debezium** | 用于实现对 PostgreSQL 的 CDC |
| **流处理引擎** | **Apache Flink** | 消费 Kafka 中的数据变更流 |
| **日志** | Kratos 内置日志接口 | 底层可适配 `uber-go/zap` |
| **配置** | Kratos 内置配置接口 | 支持多数据源 |
| **测试** | `stretchr/testify` | 断言和 Mock 工具 |
| **代码风格** | `goimports` | 格式化与包管理 |

---

## 3. 架构与开发规范 (Architecture & Development Guidelines)

本项目将同时探索并验证以下两种核心数据处理模式。

### 3.1. Kratos 标准项目结构
-   严格遵循 `kratos new` 命令生成的标准项目布局。
-   **`internal/biz`**: 存放业务逻辑的 Use Case。
-   **`internal/data`**: 存放数据访问逻辑 (Repository)，封装对 GORM 和 `gocache` 的操作。
-   **`internal/service`**: 存放 Service 定义，将 `biz` 暴露为 gRPC/HTTP 接口。

### 3.2. 模式一：同步旁路缓存 (Synchronous Cache-Aside Pattern)
这是传统的、在应用代码中实现的缓存策略。
-   **读取操作**: 在 `internal/data` 的 Repository 实现中，遵循 `Read -> Cache -> DB` 的顺序。先检查缓存，未命中时查询数据库，并将结果回填。
-   **写入操作**: 在 Repository 的更新/删除方法中，采用“先更新数据库，再使缓存失效”的策略。**这是应用代码的显式责任。**

### 3.3. 模式二：异步缓存失效 (Asynchronous Cache Invalidation via CDC & Flink)
这是更高级的、解耦的事件驱动缓存策略。
-   **工作流**:
    1.  **捕获变更**: Debezium 监控 PostgreSQL 的预写日志 (WAL)，捕获所有行级别的 `INSERT`, `UPDATE`, `DELETE` 操作。
    2.  **事件发布**: Debezium 将捕获到的变更事件发布到指定的 Kafka Topic。
    3.  **流式处理**: 一个独立的 Flink 作业订阅该 Kafka Topic，实时消费数据变更事件。
    4.  **执行失效**: Flink 作业解析事件内容，计算出需要失效的缓存键 (Cache Key)，然后调用 Memcached 的接口删除对应的缓存项。
-   **优势**: 应用服务（Kratos）的写入操作**不再关心缓存失效逻辑**，只负责更新数据库。这使得业务代码更纯粹，系统组件间耦合度更低。
-   **可视化流程**:
    ```
    [Kratos App] --(Write)--> [PostgreSQL]
                                    |
                                    +--(DB Change)--> [Debezium] --(Event)--> [Kafka] --(Stream)--> [Flink Job] --(API Call)--> [Invalidate Memcached]
    ```

### 3.4. API 定义与错误处理
-   **API-First**: 使用 Protobuf (`.proto` 文件) 定义所有 API 接口。
-   **错误码**: 使用 Kratos 提供的 `errors` 包来创建和返回标准化的业务错误码。

### 3.5. 依赖注入 (Dependency Injection)
-   所有组件的初始化和依赖关系，都在 `internal/server` 和 `cmd/{project-name}/wire.go` 中通过 `wire` 工具进行管理。

### 3.6. 注释与文档规范 (Comments & Documentation)
为了生成清晰的官方文档 (`godoc`) 并为 AI 助手提供准确的上下文，所有代码注释必须遵循以下规范：

-   **核心原则**: 所有公开的（首字母大写）包、类型、函数、方法和常量都**必须**有符合 `godoc` 规范的注释。
-   **格式要求**: 注释必须紧贴在被注释对象的声明之前，中间不能有空行。
-   **函数/方法注释**:
    -   必须以被注释的函数或方法名开头。
    -   第一句话应该是一个完整的句子，简明扼要地概括其功能，以句号结尾。
    -   如果需要更详细的说明、参数解释、背景信息或使用示例，可以在第一个句子后空一行，然后添加更多段落。

-   **示例**:

    **正确的注释风格:**
    ```go
    // UserRepo defines the data access methods for a User.
    // It is responsible for all database and cache operations related to users.
    type UserRepo interface {
        // CreateUser creates a new user in the database.
        // It returns the ID of the newly created user or an error if the operation fails.
        CreateUser(ctx context.Context, u *biz.User) (int64, error)
    }
    ```

    **错误的注释风格:**
    ```go
    // a function to create a user
    func CreateUser(ctx context.Context, u *biz.User) (int64, error) {
        // ...
    }
    ```
    *(错误原因：没有以函数名 `CreateUser` 开头，且首字母小写)*

-   **包注释**: 每个包（每个目录）都应该有一个 `doc.go` 文件或者在其中一个 Go 文件里包含包注释，用以说明整个包的功能。

    ```go
    // Package data handles all data access logic for the application,
    // including database operations and cache interactions.
    package data
    ```

---

## 4. 如何开始 (Getting Started)

1.  **安装 Kratos 工具链**:
    ```bash
    go install github.com/go-kratos/kratos/cmd/kratos/v2@latest
    ```
2.  **创建项目**:
    ```bash
    kratos new osc-yalab
    ```
3.  **依赖环境**: 使用 Docker Compose 启动完整的依赖环境。`docker-compose up -d` 将会启动 PostgreSQL, Memcached, Kafka, Debezium (Kafka Connect) 和 Flink。
4.  **配置**: 修改 `configs/config.yaml` 文件，配置数据库和缓存的连接信息。
5.  **生成代码**:
    ```bash
    # 在项目根目录执行
    go generate ./...
    ```
6.  **运行项目**:
    ```bash
    # 在项目根目录执行
    kratos run
    ```

---

*This README is specifically tailored for the OSC-YALAB project. It serves as a foundational document for human developers and as a precise context source for AI assistants like GitHub Copilot to generate project-specific instructions and code.*