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

### 规则：外部服务调用错误处理

#### Python AI 服务调用

- **超时控制**: 
  - 向量检索超时: 3 秒
  - LLM 生成超时: 10 秒
  - 使用 `context.WithTimeout()` 设置超时

- **重试策略**:
  - 对临时错误（网络超时、503 服务不可用）进行重试
  - 指数退避: 初始 100ms，最大 2 秒，最多重试 3 次
  - 对业务错误（400、404）不重试

- **降级策略**:
  - 向量检索失败: 降级到关键词搜索
  - LLM 生成失败: 返回检索到的原始文档片段
  - 记录降级事件到监控系统

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

#### Elasticsearch 调用

- **超时控制**: 
  - 查询超时: 5 秒
  - 索引写入超时: 10 秒

- **错误分类**:
  - 连接错误: 记录告警，使用降级策略
  - 索引不存在: 自动创建索引（首次启动场景）
  - 查询语法错误: 返回 BadRequest 错误

- **降级策略**:
  - 查询失败: 降级到 PostgreSQL 全文搜索（使用 `LIKE` 或 `tsvector`）
  - 写入失败: 记录到重试队列，异步重试

#### Kafka 消息发送

- **错误处理**:
  - 发送失败: 记录到本地队列，异步重试
  - 不阻塞主业务流程: 消息发送失败不应导致请求失败
  - 记录失败事件: 使用日志和监控系统跟踪消息丢失

**示例**:
```go
func (p *kafkaProducer) SendAsync(ctx context.Context, topic string, msg []byte) {
    go func() {
        err := p.writer.WriteMessages(context.Background(), kafka.Message{
            Topic: topic,
            Value: msg,
        })
        if err != nil {
            p.log.Errorw("failed to send kafka message", "topic", topic, "error", err)
            // 记录到重试队列
            p.retryQueue.Add(topic, msg)
        }
    }()
}
```

---

### 规则：缓存操作错误处理

#### 模式一：同步旁路缓存

- **读取缓存失败**: 降级到数据库查询，记录告警
- **写入缓存失败**: 记录告警但不影响主流程（数据库已写入成功）
- **删除缓存失败**: 记录告警，可能导致短期数据不一致（可接受）

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

- **应用层无缓存失效逻辑**: 不需要处理缓存错误
- **CDC 管道监控**: 通过独立的监控系统跟踪 Flink 作业状态
- **延迟容忍**: 接受短期（< 1 秒）的缓存不一致

---

### 规则：错误日志规范

#### 日志级别

- **Error**: 需要立即处理的错误（数据库连接失败、外部服务持续不可用）
- **Warn**: 降级场景、重试成功、缓存操作失败
- **Info**: 正常业务流程关键节点
- **Debug**: 详细的调试信息（开发环境启用）

#### 必须记录的字段

```go
log.Errorw("operation failed",
    "operation", "create_user",      // 操作名称
    "user_id", userID,                // 业务标识
    "error", err,                     // 原始错误
    "stack", string(debug.Stack()),   // 堆栈信息（Error 级别）
    "duration_ms", elapsed.Milliseconds(), // 操作耗时
)
```

#### 敏感信息脱敏

- **禁止记录**: 密码、Token、信用卡号、身份证号
- **脱敏记录**: 邮箱（`u***@example.com`）、手机号（`138****1234`）

---

### 规则：错误监控与告警

#### 关键指标

- **错误率**: 各层级的错误率（Data/Biz/Service）
- **降级率**: 外部服务降级的频率
- **重试率**: 重试成功/失败的比例
- **P99 延迟**: 包含错误处理的端到端延迟

#### 告警阈值

- **错误率 > 5%**: 触发告警
- **降级持续 > 5 分钟**: 触发告警
- **Kafka 消息积压 > 10000**: 触发告警
- **Elasticsearch 索引延迟 > 30 秒**: 触发告警

#### 告警响应

- **自动恢复**: 重启失败的服务实例
- **自动扩容**: 根据负载自动扩展 Python AI 服务
- **人工介入**: 数据库主从切换、清理异常数据
