# 项目文档中心

本文档中心旨在为项目所有参与者（包括人类开发者与 AI 编程助手）提供一个清晰、结构化的知识库。它概述了项目中的所有核心文档及其用途，并确立了它们之间的层级与引用关系。

---

## 1. 顶层文档

### 1.1. `README.md`

-   **文件路径**: `README.md`
-   **简介**: 项目的门户与高阶指南。它为人类开发者提供了项目的核心目标、技术栈选型以及三大核心架构模式的快速概览。此文件侧重于可读性与快速上手，并链接到更详细的规范文档。

---

## 2. AI 协作核心指令

### 2.1. `copilot-instructions.md`

-   **文件路径**: `.github/copilot-instructions.md`
-   **简介**: 这是指导 AI 编程助手（如 GitHub Copilot）的最高指令集。它定义了 AI 在本项目中的核心行为准则、技术栈约束，并作为所有详细规范的**高级索引**，通过链接指向各个“唯一事实来源”。

---

## 3. 核心规范 (Instructions)

以下文件定义了项目在各个垂直领域的**唯一事实来源 (Single Source of Truth)**。

### 3.1. 架构

#### `architecture.instructions.md`
-   **文件路径**: `.github/instructions/architecture.instructions.md`
-   **简介**: **【最高优先级】** 本项目核心架构模式的唯一权威定义。它详细描述了“同步旁路缓存”、“异步缓存失效”和“RAG 数据流”三种模式的识别方法、实现规则、工作流程及可视化图表。

#### `rag-architecture.instructions.md`
-   **文件路径**: `.github/instructions/rag-architecture.instructions.md`
-   **简介**: 专门阐述 RAG 架构的数据流转、组件职责边界和接口约定。它与 `architecture.instructions.md` 互为补充，聚焦于 RAG 模式的实现细节。

### 3.2. 开发与编码

#### `style-guide.instructions.md`
-   **文件路径**: `.github/instructions/style-guide.instructions.md`
-   **简介**: 代码风格和注释的权威指南。它详细规定了 `godoc` 注释的格式、多段落注释的写法以及代码格式化的标准（`goimports`）。

#### `workflow.instructions.md`
-   **文件路径**: `.github/instructions/workflow.instructions.md`
-   **简介**: 定义了开发工作流中的核心命令与流程。例如，在修改 `.proto` 或 `wire.go` 文件后必须运行 `go generate ./...`，以及如何使用 Docker Compose 管理环境。

### 3.3. 专项规范

#### `ai-service.instructions.md`
-   **文件路径**: `.github/instructions/ai-service.instructions.md`
-   **简介**: 针对 Python AI 服务的开发规范。内容涵盖模型选型（Qwen 系列）、API 设计、与 Go 服务的协作模式以及向量检索和答案生成的实现要求。

#### `error-handling.instructions.md`
-   **文件路径**: `.github/instructions/error-handling.instructions.md`
-   **简介**: 项目的系统韧性设计蓝图。它统一了从 `Data` 层到 `Service` 层的错误处理策略，并详细定义了针对外部服务（如 AI 服务、Elasticsearch）和数据管道（Kafka, Flink）的超时、重试、降级及熔断机制。

#### `frontend.instructions.md`
-   **文件路径**: `.github/instructions/frontend.instructions.md`
-   **简介**: 前端开发的规范指南。当项目涉及 UI 或 3D 可视化时，此文件定义了必须遵循的技术栈（React, Three.js, R3F）和核心开发原则（声明式、组件化）。
