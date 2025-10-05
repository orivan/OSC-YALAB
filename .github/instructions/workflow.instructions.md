## 开发工作流与核心命令

### 规则：API 或依赖注入变更后必须生成代码

- **触发条件**: 当你修改了任何 `.proto` 文件（API 定义）或 `wire.go` 文件（依赖注入）时。
- **必须执行**: 在项目根目录下运行 `go generate ./...` 命令。
- **原因**: 此命令会重新生成 Protobuf 的 gRPC/HTTP桩代码、验证代码以及 Wire 的依赖注入容器。**如果忘记运行，你的代码将无法编译或运行在旧的逻辑上。**

### 规则：使用 Docker Compose 管理依赖环境

- **启动所有服务**: `docker-compose up -d`
  - 这会启动 PostgreSQL, Memcached, Kafka, Debezium 和 Flink。
- **查看日志**: `docker-compose logs -f <service_name>`
  - 例如，`docker-compose logs -f postgres`
- **停止所有服务**: `docker-compose down`

### 规则：使用 Kratos 内置命令运行和构建

- **前台运行 (开发时)**: `kratos run`
  - 这会启动服务并自动监听文件变化，实现热重载。
- **构建二进制文件**: `kratos build`

### 规则：配置管理

- 所有服务的配置都集中在 `configs/config.yaml` 文件中。
- 当你需要添加新的配置项时（例如，一个新的外部服务地址），请先在此文件中定义，然后通过 Kratos 的配置客户端在 `internal/data` 或 `internal/biz` 中加载。
