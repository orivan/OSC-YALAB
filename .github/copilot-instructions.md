# 指导 GitHub Copilot - OSC-YALAB 项目

欢迎来到 OSC-YALAB！这是一个用于学习和实验现代 Go 微服务技术栈的后端项目。为了让你能高效地协助开发，请遵循以下核心准则和架构模式。

## 1. 核心技术栈

本项目严格遵循以下技术选型。所有代码实现都应基于此技术栈：

- **微服务框架**: **Kratos (v2)**，用于构建 gRPC 和 HTTP 服务。
- **数据库与 ORM**: **PostgreSQL** 与 **GORM**。
- **缓存**: 使用 `gocache/v3` 作为统一抽象。
  - **L1 本地缓存**: Ristretto
  - **L2 分布式缓存**: Memcached
- **异步架构**:
  - **消息队列**: Apache Kafka (`segmentio/kafka-go` 客户端)
  - **CDC**: Debezium，用于捕获 PostgreSQL 的数据变更。
  - **流处理**: Apache Flink，用于处理来自 Debezium 的事件。
- **依赖注入**: 使用 `google/wire` 在 `cmd/{project-name}/wire.go` 中管理依赖。
- **API 定义**: 所有 API 均使用 Protobuf (`.proto` 文件) 进行定义。

## 2. 项目架构与开发规范

代码严格遵循 `kratos new` 生成的标准分层结构：

- **`internal/service`**: API 服务层。负责接收请求、调用 `biz` 层并返回响应。
- **`internal/biz`**: 业务逻辑层。封装核心业务规则 (Use Cases)，不关心数据如何存储。
- **`internal/data`**: 数据访问层。实现 `biz` 定义的 `Repo` 接口，负责与数据库和缓存交互。

### 关键开发模式

本项目探索两种缓存处理模式，请在开发时明确当前上下文属于哪一种。

#### 模式一：同步旁路缓存 (应用层负责)

这是传统的、在应用代码中显式处理缓存的模式。

- **读取流程 (`internal/data`)**:
  1.  首先尝试从 `gocache` 读取数据。
  2.  如果缓存未命中，则查询 PostgreSQL 数据库。
  3.  将从数据库中获取的数据回填到缓存中。

- **写入流程 (`internal/data`)**:
  1.  **先更新数据库** (PostgreSQL)。
  2.  **再使缓存失效**。这是 `data` 层方法的明确责任。

**示例**: 当实现一个 `UpdateUser` 的 `Repo` 方法时，你必须在 GORM 更新操作成功后，立即调用代码来删除相关的缓存键。

#### 模式二：异步缓存失效 (CDC + Flink 负责)

这是更高级的、解耦的模式。**应用服务本身不处理缓存失效**。

- **工作流**:
  1.  Kratos 应用（`internal/data`）的写入操作**只负责更新 PostgreSQL 数据库**。
  2.  Debezium 捕获数据库的变更，并将事件发送到 Kafka。
  3.  一个独立的 Flink 作业消费 Kafka 中的事件，并根据事件内容计算出需要失效的缓存键，然后调用 Memcached API 将其删除。

- **你的任务**: 当在此模式下工作时，**不要**在 Kratos 应用的写入/更新/删除方法中添加任何缓存失效逻辑。只需确保数据库操作是正确的。

### 注释与文档规范

为了生成清晰的 `godoc` 并为 AI 助手提供准确上下文，所有公开的（首字母大写）成员都必须有注释。

- **格式**: 注释必须以被注释的成员名开头，紧贴声明之上。
- **示例**:
  ```go
  // CreateUser 创建一个新用户。
  // 它会处理必要的验证和数据库插入操作。
  func (uc *UserUsecase) CreateUser(ctx context.Context, u *User) error {
      // ...
  }
  ```

## 3. 开发工作流

- **代码生成**: 在修改了 `.proto` 文件或 `wire.go` 后，必须在项目根目录运行 `go generate ./...` 来更新生成的代码。
- **运行环境**: 项目依赖的服务（数据库、缓存、Kafka 等）通过 `docker-compose.yml` 文件管理。使用 `docker-compose up -d` 启动。
- **运行项目**: 使用 `kratos run` 在前台启动应用。
- **配置**: 所有配置信息位于 `configs/config.yaml`。
- **测试**: 使用 `stretchr/testify` 编写单元测试和集成测试。

在开始编码前，请确认你已经理解了当前任务是应遵循“同步模式”还是“异步模式”，这将决定你的实现方式。
