## 代码风格与注释指南

### 规则：遵循 `godoc` 注释风格

为了生成清晰的官方文档 (`godoc`) 并为 AI 助手提供准确的上下文，所有代码注释必须遵循以下规范。

- **核心原则**: 所有公开的（首字母大写）包、类型、函数、方法和常量都**必须**有符合 `godoc` 规范的注释。

- **格式要求**:
  - 注释必须紧贴在被注释对象的声明之前，中间不能有空行。
  - **必须**以被注释的成员名开头。
  - 第一句话应该是一个完整的句子，简明扼要地概括其功能，并以句号结尾。

- **错误示例**:
  ```go
  // a function to create user
  func CreateUser(ctx context.Context, u *biz.User) (int64, error) {
      // ...
  }
  ```
  *错误原因：没有以函数名 `CreateUser` 开头，且首字母小写，结尾没有句号。*

- **正确示例**:
  ```go
  // CreateUser 创建一个新用户。
  // 它会处理必要的验证和数据库插入操作，并返回新用户的ID。
  func (uc *UserUsecase) CreateUser(ctx context.Context, u *User) error {
      // ...
  }
  ```

### 规则：代码格式化

- 所有 Go 代码在提交前**必须**使用 `goimports` 进行格式化。这能确保代码风格和 `import` 块的统一。
