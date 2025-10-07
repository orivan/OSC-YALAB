## 错误处理规范

### 规则：分层错误处理策略

本项目遵循 Kratos 框架的错误处理最佳实践，每一层都有明确的职责边界。

#### Data 层错误处理

- **职责**: 将底层错误（GORM、缓存、外部服务）转换为 Kratos 标准错误
- **规则**:
  - 使用 `errors.NotFound()` 表示资源不存在
  - 使用 `errors.Internal()` 表示内部错误（数据库连接失败、缓存异常等）
  - 使用 `errors.BadRequest()` 表示数据验证失败
  - 必须记录原始错误到日志，但不向上层暴露敏感信息

**示例**:
```go
func (r *userRepo) GetByID(ctx context.Context, id int64) (*biz.User, error) {
    var user User
    if err := r.db.WithContext(ctx).First(&user, id).Error; err != nil {
        if errors.Is(err, gorm.ErrRecordNotFound) {
            return nil, errors.NotFound("USER_NOT_FOUND", "user not found")
        }
        r.log.Errorw("failed to get user", "id", id, "error", err)
        return nil, errors.Internal("DATABASE_ERROR", "failed to get user")
    }
    return &biz.User{...}, nil
}
```

#### Biz 层错误处理

- **职责**: 处理业务逻辑错误，添加业务上下文
- **规则**:
  - 使用 `errors.BadRequest()` 表示业务规则违反
  - 使用 `errors.Forbidden()` 表示权限不足
  - 使用 `errors.Conflict()` 表示资源冲突（如重复创建）
  - 透传 Data 层的 NotFound 和 Internal 错误

**示例**:
```go
func (uc *UserUsecase) CreateUser(ctx context.Context, u *User) error {
    // 业务规则验证
    if u.Age < 18 {
        return errors.BadRequest("INVALID_AGE", "user must be at least 18 years old")
    }
    
    // 检查重复
    existing, err := uc.repo.GetByEmail(ctx, u.Email)
    if err != nil && !errors.IsNotFound(err) {
        return err // 透传 Data 层错误
    }
    if existing != nil {
        return errors.Conflict("USER_EXISTS", "user with this email already exists")
    }
    
    return uc.repo.Create(ctx, u)
}
```

#### Service 层错误处理

- **职责**: 将 Kratos 错误转换为 gRPC/HTTP 响应
- **规则**:
  - 框架会自动转换 Kratos 错误为对应的 HTTP 状态码和 gRPC 状态
  - 不需要手动处理，直接返回 Biz 层的错误即可
  - 必要时可以添加额外的元数据（metadata）

---

### 规则：外部服务与数据管道错误处理

#### Python AI 服务调用

- **超时控制**: 
  - 向量检索超时: 3 秒
  - LLM 生成超时: 10 秒
  - 使用 `context.WithTimeout()` 设置超时

- **重试策略**:
  - 对临时错误（网络超时、503 服务不可用）进行重试
  - 指数退避: 初始 100ms，最大 2 秒，最多重试 3 次
  - 对业务错误（400、404、422）不重试

- **降级策略**:
  - **向量检索失败**: 降级到关键词匹配（使用 Elasticsearch 的 `match` 查询）。如果 ES 也失败，返回热门文档。
  - **LLM 生成失败**: 返回检索到的原始文档片段（前 3 个最相关文档的摘要），并附带说明。
  - **用户体验**: 在降级响应中标注 `"mode": "degraded"`。

**示例**:
```go
func (s *aiService) VectorSearch(ctx context.Context, query string) ([]string, error) {
    ctx, cancel := context.WithTimeout(ctx, 3*time.Second)
    defer cancel()
    
    var lastErr error
    for i := 0; i < 3; i++ {
        resp, err := s.client.Post(ctx, "/api/v1/search", map[string]interface{}{
            "query": query,
        })
        
        if err == nil {
            return resp.DocumentIDs, nil
        }
        
        // 判断是否应该重试
        if !shouldRetry(err) {
            return nil, errors.Internal("AI_SERVICE_ERROR", err.Error())
        }
        
        lastErr = err
        time.Sleep(time.Duration(100*(1<<i)) * time.Millisecond) // 指数退避
    }
    
    s.log.Warnw("AI service retry exhausted, degrading", "error", lastErr)
    return s.degradeToKeywordSearch(ctx, query) // 降级
}
```

#### Elasticsearch 异常处理

- **查询失败**:
  - **检测**: 查询超时（> 5 秒）或连接错误。
  - **降级策略**: 依次尝试：1. 从缓存返回最近结果 -> 2. 降级到 PostgreSQL 全文搜索 -> 3. 返回固定的热门文档列表。
  - **恢复**: 定期探测 Elasticsearch 可用性，恢复后切回正常流程。

- **索引写入失败**:
  - **策略**: 将失败文档记录到**重试队列**（如 Redis），进行异步指数退避重试（最多 5 次）。超过次数后移入**死信队列**并触发告警。
  - **核心原则**: **不影响 Kafka 消费进度**，确保 Kafka offset 正常提交。

- **索引与映射**:
  - **首次启动**: 自动创建索引并设置正确的映射（mapping）。
  - **映射冲突**: 记录错误，阻止写入，并触发告警，需要人工介入。

#### Kafka 消息处理

- **消息发送失败**:
  - **策略**: 记录到本地队列或 Redis，异步重试。
  - **原则**: 不阻塞主业务流程，消息发送失败不应导致用户请求失败。

- **消息消费延迟过高 (Lag)**:
  - **检测**: 监控消费 Lag。
  - **策略**: Lag > 10,000 触发告警；Lag > 50,000 考虑自动扩展消费者实例。

- **消息格式错误/解析失败**:
  - **策略**: 记录错误消息到**死信队列 (DLQ)**，继续消费下一条消息，不阻塞整个流程。

#### Flink 作业失败
- **检测**: 监控 Flink 作业的 CheckPoint 状态。
- **策略**: 自动从最近的 CheckPoint 重启。若连续失败则告警并暂停作业，待人工修复。

---

### 规则：缓存操作错误处理

#### 模式一：同步旁路缓存

- **读取缓存失败**: 降级到数据库查询，记录告警。
- **写入缓存失败**: 记录告警但不影响主流程（数据库已写入成功）。
- **删除缓存失败**: 记录告警，可能导致短期数据不一致（可接受）。

**示例**:
```go
func (r *userRepo) Update(ctx context.Context, u *User) error {
    // 1. 更新数据库
    if err := r.db.WithContext(ctx).Save(u).Error; err != nil {
        return errors.Internal("DATABASE_ERROR", "failed to update user")
    }
    
    // 2. 删除缓存（失败不影响主流程）
    cacheKey := fmt.Sprintf("user:%d", u.ID)
    if err := r.cache.Delete(ctx, cacheKey); err != nil {
        r.log.Warnw("failed to invalidate cache", "key", cacheKey, "error", err)
        // 不返回错误，数据库已更新成功
    }
    
    return nil
}
```

#### 模式二：异步缓存失效

- **应用层无缓存失效逻辑**: 不需要处理缓存错误。
- **CDC 管道监控**: 通过独立的监控系统跟踪 Flink 作业状态。
- **延迟容忍**: 接受短期（< 1 秒）的缓存不一致。

---

### 规则：错误日志与监控

#### 日志规范

- **日志级别**:
  - **Error**: 需要立即处理的错误（DB 连接失败、服务持续不可用）。
  - **Warn**: 降级场景、重试成功、可容忍的失败（如缓存删除失败）。
  - **Info**: 正常业务流程关键节点。
  - **Debug**: 详细调试信息。

- **必须记录的字段**:
  ```go
  log.Errorw("operation failed",
      "operation", "create_user",      // 操作名称
      "user_id", userID,                // 业务标识
      "error", err,                     // 原始错误
      "stack", string(debug.Stack()),   // 堆栈信息（仅 Error 级别）
      "duration_ms", elapsed.Milliseconds(), // 操作耗时
  )
  ```
- **敏感信息**: 必须在日志记录前进行脱敏处理。

#### 监控与告警指标

| 类别 | 指标 | 告警阈值 | 严重阈值 |
|---|---|---|---|
| **数据管道** | Kafka 消费 Lag | > 10,000 | > 50,000 |
| | ES 索引延迟 | > 30s | > 60s |
| | ES/向量化成功率 | < 95% | < 90% |
| **AI 服务** | 向量检索/LLM 生成 P99 延迟 | > 2s / > 10s | > 5s / > 20s |
| | 降级率 / 错误率 | > 1% | > 5% |
| **端到端** | 总延迟 P99 | > 10s | > 20s |
| | 成功率 | < 99% | < 98% |

