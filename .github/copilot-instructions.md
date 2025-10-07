# 指导 GitHub Copilot - OSC-YALAB 项目

这是一个用于学习和实验现代 Go 微服务技术栈的后端项目。

**核心目标**: 通过最简单、最直接的实现，来验证和学习一套现代化的、融合了数据处理与AI能力的后端技术栈。代码的清晰度和对设计模式的典型诠释，优先于复杂的业务逻辑。

为了让你能高效地协助开发，请严格遵循以下核心准则和架构模式。

## 1. 核心技术栈

本项目严格遵循以下技术选型。你生成的所有代码都必须基于此技术栈：

- **微服务框架**: **Kratos (v2)**
- **数据库与 ORM**: **PostgreSQL** 与 **GORM**
- **缓存**: `gocache/v3` (L1: Ristretto, L2: Memcached)
- **异步架构**:
  - **消息队列**: Apache Kafka (`segmentio/kafka-go`)
  - **CDC**: Debezium
  - **流处理**: Apache Flink
- **AI 与搜索**:
  - **搜索引擎**: **Elasticsearch**
  - **AI 服务运行时**: **Python >= 3.10**
  - **向量化模型**: **Qwen-Embedding-0.6B**
  - **大语言模型**: **Qwen2.5-7B-Instruct**
- **依赖注入**: `google/wire`
- **API 定义**: Protobuf

## 2. 核心架构原则【最高优先级】

在修改任何数据读写逻辑时，你**必须**首先识别并遵循项目定义的三大核心架构模式。

**详细规则**: **[查看核心架构模式详解](./instructions/architecture.instructions.md)**

## 3. AI 与错误处理规范

- **AI 服务**: 所有与 Python AI 服务相关的开发，必须遵循 **[AI 服务开发规范](./instructions/ai-service.instructions.md)**。
- **错误处理**: 所有错误处理、重试、降级逻辑，必须遵循 **[错误处理规范](./instructions/error-handling.instructions.md)**。

## 4. 代码风格与注释规范【强制性规则】

你生成的所有 Go 代码都必须符合以下标准：

1.  **注释与格式化**: 必须遵循 **[代码风格与注释指南](./instructions/style-guide.instructions.md)**。
2.  **API 或依赖变更**: 修改 `.proto` 或 `wire.go` 文件后，必须提示用户运行 `go generate ./...`。详细说明见 **[开发工作流命令](./instructions/workflow.instructions.md)**。

## 5. 前端开发规范

当开发任务涉及前端 UI 或 3D 可视化时，你必须遵循以下规范：
- **前端技术栈与原则**: 所有前端相关的开发，必须遵循 **[前端开发规范](./instructions/frontend.instructions.md)**。

