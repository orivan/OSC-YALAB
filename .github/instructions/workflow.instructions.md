## 开发工作流与核心命令

### 规则：API 或依赖注入变更后必须生成代码

- **触发条件**: 当你修改了任何 `.proto` 文件（API 定义）或 `wire.go` 文件（依赖注入）时。
- **必须执行**: 在项目根目录下运行 `go generate ./...` 命令。
- **原因**: 此命令会重新生成 Protobuf 的 gRPC/HTTP桩代码、验证代码以及 Wire 的依赖注入容器。**如果忘记运行，你的代码将无法编译或运行在旧的逻辑上。**

### 规则：使用 Docker Compose 管理依赖环境

- **启动所有服务**: `docker-compose up -d`
  - 这会启动 PostgreSQL, Memcached, Kafka, Debezium (Kafka Connect), Flink, **Elasticsearch, Python AI 服务**。
- **查看日志**: `docker-compose logs -f <service_name>`
  - 例如，`docker-compose logs -f postgres`, `docker-compose logs -f python-ai-service`
- **停止所有服务**: `docker-compose down`

### 规则：使用 Kratos 内置命令运行和构建

- **前台运行 (开发时)**: `kratos run`
  - 这会启动服务并自动监听文件变化，实现热重载。
- **构建二进制文件**: `kratos build`

### 规则：配置管理

- 所有服务的配置都集中在 `configs/config.yaml` 文件中。
- 当你需要添加新的配置项时（例如，一个新的外部服务地址），请先在此文件中定义，然后通过 Kratos 的配置客户端在 `internal/data` 或 `internal/biz` 中加载。

### 规则：AI 服务管理

- **启动 AI 服务**: `docker-compose up -d python-ai-service`
- **重启 AI 服务**: `docker-compose restart python-ai-service`
- **查看 AI 日志**: `docker-compose logs -f python-ai-service`
- **AI 模型更新**: 当更新 Qwen-Embedding 或 Qwen2.5-7B 模型时，需要重启 Python 服务容器

### 规则：RAG 数据流测试

- **验证数据管道**: 检查 Kafka topics (`topic_raw`, `topic_rich_data`) 的消息流
- **验证向量索引**: 使用 Elasticsearch API 检查向量数据的索引状态
- **验证检索功能**: 测试 Python AI 服务的向量搜索接口
- **端到端测试**: 从数据写入到查询回答的完整流程测试
