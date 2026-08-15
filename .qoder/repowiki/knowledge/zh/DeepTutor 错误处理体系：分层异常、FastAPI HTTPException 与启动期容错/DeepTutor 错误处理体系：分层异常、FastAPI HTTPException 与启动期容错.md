---
kind: error_handling
name: DeepTutor 错误处理体系：分层异常、FastAPI HTTPException 与启动期容错
category: error_handling
scope:
    - '**'
source_files:
    - deeptutor/core/errors.py
    - deeptutor/utils/error_utils.py
    - deeptutor/api/main.py
    - deeptutor/api/routers/auth.py
    - deeptutor/api/routers/book.py
    - deeptutor/api/routers/attachments.py
    - tests/core/test_context.py
    - deeptutor/services/codex_auth/contracts.py
    - deeptutor/agents/math_animator/renderer.py
    - deeptutor/services/memory/ops.py
---

## 1. 使用的系统与总体方法

DeepTutor 在后端采用 **分层异常 + FastAPI HTTPException** 的混合模式：
- 应用层定义统一的 `deeptutor.core.errors` 异常族，作为业务/服务层抛出的结构化错误。
- API 路由层通过 `fastapi.HTTPException` 将错误映射为 HTTP 状态码（400/401/403/404/500/501），由 FastAPI/Uvicorn 统一序列化为 JSON 响应体。
- 进程启动阶段使用 `try/except Exception` 包裹各子系统初始化，失败时记录 warning 日志并继续启动，体现“非致命依赖降级”策略。
- 没有全局 `@app.exception_handler` 自定义处理器；所有 HTTP 错误均走 FastAPI 默认异常转换路径。
- CLI 侧未发现专门的异常类型，CLI 命令直接抛出 Python 内置异常并由 Typer/标准输出呈现。

## 2. 关键文件与包

| 文件 | 作用 |
|---|---|
| `deeptutor/core/errors.py` | 定义 `DeepTutorError` 基类及 `ConfigurationError`、`ValidationError`、`ServiceError`、`LLMServiceError`、`LLMContextError`、`EnvironmentConfigError` 等子类 |
| `deeptutor/utils/error_utils.py` | 提供 `format_exception_message`，从异常消息中抽取 JSON 块并格式化可读信息 |
| `deeptutor/api/main.py` | FastAPI 应用入口，包含启动/关闭生命周期中的 `try/except` 容错逻辑和选择性访问日志中间件 |
| `deeptutor/api/routers/*.py` | 路由层大量使用 `raise HTTPException(status_code=..., detail=...)` 表达客户端错误 |
| `tests/core/test_context.py` | 对 `DeepTutorError` 继承链进行断言测试，是异常体系的契约测试 |
| `deeptutor/services/codex_auth/contracts.py` | 局部领域异常 `CodexAuthError(RuntimeError)`，展示子模块可定义自有异常 |
| `deeptutor/agents/math_animator/renderer.py` | 定义 `ManimRenderError(RuntimeError)`，用于渲染管线特定错误 |
| `deeptutor/services/memory/ops.py` | 定义 `OpValidationError(Exception)`，体现局部领域异常模式 |

## 3. 架构与约定

### 3.1 异常层次结构
`deeptutor/core/errors.py` 定义了清晰的继承树：
```
DeepTutorError
├── ConfigurationError
│   └── EnvironmentConfigError
├── ValidationError
├── ServiceError
│   └── LLMServiceError
│       └── LLMContextError
```
每个异常都支持 `message` 和可选的 `details: Dict[str, Any]`，并通过 `__str__` 在包含 details 时拼接 `(details: {...})`。

### 3.2 API 层错误映射
路由层几乎全部通过 `HTTPException` 暴露错误，常见模式包括：
- 资源不存在：`HTTPException(status_code=404, detail="...")`（如 `attachments.py`、`auth.py`、`book.py`）
- 参数非法：`HTTPException(status_code=400, detail=...)`
- 功能未实现：`HTTPException(status_code=501, detail="Attachment backend not servable")`
- 服务端错误：`HTTPException(status_code=500, detail=str(exc))`

Pydantic 校验错误通过 `field_validator` 中 `raise ValueError(...)` 触发，再由 FastAPI 自动转换为 422 Unprocessable Entity。

### 3.3 启动期容错（Graceful Degradation）
`deeptutor/api/main.py` 的 `lifespan` 中对每个外部依赖初始化都使用独立的 `try/except Exception` 包裹，失败时仅 `logger.warning` 而不中断启动：
- LLM client 初始化失败 → 记录 warning
- EventBus 启动失败 → 记录 warning
- Partner auto-start 失败 → 记录 warning
- Cron 启动失败 → 记录 warning
- PocketBase ping 失败 → 记录 warning
- v1 memory 迁移失败 → 记录 warning
这种模式确保 DeepTutor 能在部分依赖不可用时仍启动。

### 3.4 关闭期资源释放
shutdown 阶段同样用 `try/except` 包裹 cron stop、partner stop、MCP shutdown、LLM provider pool close、Agentic client pool close、EventBus stop，全部以 `logger.warning` 记录失败，保证优雅退出。

### 3.5 访问日志过滤
`selective_access_log` 中间件只记录非 200 状态的请求，避免前端轮询产生的大量 INFO 噪音进入日志。

### 3.6 工具一致性校验
`validate_tool_consistency()` 在启动时检查 capability manifest 引用的工具是否已注册到 ToolRegistry，缺失时抛出 `RuntimeError` 并记录异常堆栈，阻止不一致配置上线。

## 4. 约定与约束

- **应用级异常必须继承 `DeepTutorError`**：`core/errors.py` 的文档说明其用途是“区分 bug、可恢复错误和配置问题”，`tests/core/test_context.py` 通过 `issubclass` 断言继承链，构成契约测试。
- **API 边界统一使用 `HTTPException`**：路由层不向客户端返回自定义对象，而是通过 `status_code` + `detail` 表达错误语义。
- **Pydantic 校验通过 `ValueError` 触发 422**：路由内 `field_validator` 中 `raise ValueError(...)` 是标准的 Pydantic 校验失败方式。
- **子模块可定义领域异常**：如 `ManimRenderError`、`CodexAuthError`、`OpValidationError`，但通常继承自 `RuntimeError` 或 `Exception`，而非强制继承 `DeepTutorError`——表明该基类并非强制约束，而是推荐规范。
- **无全局异常处理器**：仓库中搜索 `@app.exception_handler` 无结果，所有 HTTP 错误走 FastAPI 默认路径。
- **无 `panic/recover` 模式**：Python 生态下未见 `try/except BaseException` 捕获系统级异常的模式，仅在启动/关闭阶段用 `except Exception` 做降级。
- **错误消息可读性增强**：`error_utils.format_exception_message` 尝试从异常字符串中提取 JSON 块并格式化为 `Message: ... | Type: ... | Code: ...` 形式，便于日志阅读。
- **多用户认证错误集中化**：`auth.py` 中大量 `HTTPException(status_code=status.HTTP_404_NOT_FOUND, detail="User not found")` 复用常量，保持错误码一致。

## 5. 观察到的不足

- `DeepTutorError` 及其子类在路由层极少被直接使用，大部分错误以 `HTTPException` 直接抛出，导致业务异常与 HTTP 表示之间存在一层转换缺失。
- 没有统一的错误响应模型（如 `{"code", "message", "details"}`），不同路由的 `detail` 字段内容风格不一（有的纯字符串，有的可能为对象）。
- 未看到针对 `DeepTutorError` 的 FastAPI 异常处理器将其转换为结构化 JSON 响应的代码，意味着这些异常若冒泡到 API 层会被当作普通 `Exception` 处理。