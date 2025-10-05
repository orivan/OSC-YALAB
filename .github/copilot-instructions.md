# 指导 GitHub Copilot - OSC-YALAB 项目

欢迎来到 OSC-YALAB！这是一个用于学习和实验现代 Go 微服务技术栈的后端项目。

**核心目标**: 通过最简单、最直接的实现，来验证和学习一套现代化的 Go 微服务技术栈和相关的数据架构模式。代码的清晰度和对设计模式的典型诠释，优先于复杂的业务逻辑。

为了让你能高效地协助开发，请遵循以下核心准则和架构模式。

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

## 2. 更多说明

更详细的说明请参考 `.github/instructions/` 目录下的文件。
