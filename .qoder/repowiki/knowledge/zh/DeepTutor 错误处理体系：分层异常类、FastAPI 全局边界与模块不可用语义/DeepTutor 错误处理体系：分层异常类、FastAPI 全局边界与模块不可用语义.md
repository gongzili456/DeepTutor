---
kind: error_handling
name: DeepTutor 错误处理体系：分层异常类、FastAPI 全局边界与模块不可用语义
category: error_handling
scope:
    - '**'
source_files:
    - deeptutor/core/errors.py
    - deeptutor/modules/catalog.py
    - deeptutor/book/errors.py
    - deeptutor/utils/error_utils.py
    - deeptutor/api/main.py
    - tests/api/test_main_exception_handler.py
---

## 1. 采用的方案

DeepTutor 在后端（FastAPI + Python）采用「自定义异常层次 + FastAPI `exception_handler` + HTTP middleware 兜底」的组合方式，没有引入第三方错误框架。核心思路是：
- 业务层抛出领域异常；
- API 层将特定异常映射为结构化 JSON 响应；
- 所有未捕获异常由一个注册在 CORS 内部的 HTTP middleware 统一包装成 `{detail, type}` 的 JSON 500。

CLI 侧（`deeptutor_cli/`）和 Web 前端（`web/`）各自有独立的错误呈现逻辑，本卡片聚焦后端平台。

## 2. 关键文件与包

| 路径 | 作用 |
|---|---|
| `deeptutor/core/errors.py` | 应用级基础异常族（`DeepTutorError` 及其子类） |
| `deeptutor/modules/catalog.py` | 模块清单校验与 `ModuleUnavailable` 语义异常 |
| `deeptutor/book/errors.py` | Book 运行时轻量公开异常（`BookPausedError`） |
| `deeptutor/utils/error_utils.py` | 通用异常消息格式化（从 LLM/API 返回的 JSON 片段中抽取 message/type/code） |
| `deeptutor/api/main.py` | FastAPI 应用入口，集中注册 `AccessError` / `ModuleUnavailable` 处理器与全局 JSON 错误边界 |
| `tests/api/test_main_exception_handler.py` | 断言未处理异常始终返回 JSON 且携带 CORS 头 |

## 3. 架构与约定

### 3.1 异常层次（`deeptutor/core/errors.py`）

```text
Exception
 └── DeepTutorError(message, details={})
      ├── ConfigurationError
      │    └── EnvironmentConfigError
      ├── ValidationError
      ├── ServiceError
      │    └── LLMServiceError
      │         └── LLMContextError
      └── AccessError(status_code, detail)
```

- `DeepTutorError` 是所有应用异常的根，除 `message` 外还附带可选的 `details: Dict[str, Any]`，`__str__` 会把 details 拼到末尾。
- `AccessError` 是「传输无关的访问失败」，通过 `status_code` + `detail` 表达，HTTP 适配层负责把它转成对应状态码。
- `LLMContextError` 专门表示 prompt 超出模型上下文窗口这一可恢复场景。

### 3.2 模块不可用语义（`deeptutor/modules/catalog.py`）

- `ModuleUnavailable(ValueError)` 是一个语义型异常，用于表达「功能模块未安装/未启用」。
- `require_module(name)` 在模块不可用时抛出 `ModuleUnavailable`，并给出 `deeptutor module install/enable` 的修复提示。
- `module_enabled(name)` 对未知模块直接抛 `ValueError(f"Unknown module: {name}")`，作为清单/配置错误的快速失败点。
- 模块清单加载时会对 schema_version、依赖环、重复 owner、非法 path/prefix、非法 route auth policy 等抛出 `ValueError`，把清单错误集中在 catalog 层。

### 3.3 FastAPI 错误路由（`deeptutor/api/main.py`）

#### 专用异常处理器

```python
@app.exception_handler(AccessError)
async def access_error_handler(request, exc):
    return JSONResponse(status_code=exc.status_code, content={"detail": exc.detail})

@app.exception_handler(ModuleUnavailable)
async def module_unavailable_handler(request, exc):
    return JSONResponse(status_code=409, content={"detail": str(exc)})
```

- `AccessError` → 透传其 `status_code`（常见 401/403），body 仅含 `detail`。
- `ModuleUnavailable` → 固定 409 Conflict，body 含人类可读的安装提示。

#### 全局 JSON 错误边界（middleware）

`json_error_boundary` 以 `@app.middleware("http")` 注册，位于 CORSMiddleware **内部**（先于它注册），保证 500 响应仍经过 CORS 中间件，从而带上 `access-control-allow-origin` 等跨域头。

行为：
- 捕获任意 `Exception`；
- 记录 `logger.error(..., exc_info=True)`；
- 返回 `JSONResponse(500, {"detail": f"{type(exc).__name__}: {exc}", "type": type(exc).__name__})`。

测试 `tests/api/test_main_exception_handler.py` 显式断言：
1. 未处理异常返回 500；
2. `Content-Type` 以 `application/json` 开头；
3. 响应体等于 `{"detail": "RuntimeError: ...", "type": "RuntimeError"}`；
4. 跨域请求下 `access-control-allow-origin` 与正常 200 响应一致。

注释明确解释了为什么不能用 `@app.exception_handler(Exception)`（Starlette 的 `ServerErrorMiddleware` 会绕过所有已注册 middleware，导致 CORS 丢失）。

#### 选择性访问日志

`selective_access_log` 只记录非 200 响应，并通过独立 logger `deeptutor.access` 输出到 stdout，避免被 root handler 的 WARNING 级别吞掉。

### 3.4 其他领域异常

- `deeptutor/book/errors.BookPausedError(RuntimeError)`：Book 暂停时拒绝编译工作。
- `agents/math_animator/agents/code_generator_agent.py` 中的 `GeneratedCodeOutputError(ValueError)`：代码生成器产出结构不合法时的局部异常。

这些异常遵循「具体场景继承标准基类或内置异常」的模式，而不是全部挂到 `DeepTutorError` 下。

### 3.5 错误消息格式化（`deeptutor/utils/error_utils.py`）

`format_exception_message(exc)` 尝试从异常字符串中提取内嵌 JSON 块（例如上游 LLM 返回的 `{"error":{"message","type","code"}}`），并格式化为 `Message: ... | Type: ... | Code: ...` 的可读形式；解析失败则原样返回。该工具主要用于 CLI 或日志中对第三方 API 错误进行二次展示。

## 4. 观察到的约定与约束

- **应用异常应继承 `DeepTutorError`**：`core/errors.py` 的 docstring 明确声明其为「consistent error handling across the application」的基础类，并提供区分 bug / 可恢复错误 / 配置问题的标准化方式。
- **HTTP 层禁止裸抛未处理异常**：`api/main.py` 的全局 middleware 强制所有 500 响应为 JSON，并由测试覆盖；新增路由不应绕过此边界。
- **跨域 500 必须带 CORS 头**：这是通过「middleware 而非 Starlette 顶层 exception handler」实现的硬性约束，测试 `test_unhandled_exception_response_carries_cors_headers` 验证。
- **模块不可用使用 `ModuleUnavailable`**：通过 `require_module()` 抛出，HTTP 层统一映射为 409，调用方不应自行构造 404/503 来表示模块缺失。
- **权限/访问失败使用 `AccessError`**：由上层适配器设置 `status_code`，`AccessError` 本身是 transport-independent 的，HTTP 层只做透传。
- **配置/清单错误使用 `ValueError`**：catalog 对非法 manifest、循环依赖、未知模块、非法 path/route policy 等直接抛 `ValueError`，作为启动期快速失败。
- **LLM 相关错误细分**：`LLMServiceError` → `LLMContextError` 形成专用分支，便于上层做降级或重试策略。
- **错误详情结构化**：`DeepTutorError.details` 允许附加结构化信息，供日志/监控消费；`AccessError.detail` 则是面向用户的短文本。
- **CLI 侧对第三方错误做友好格式化**：通过 `error_utils.format_exception_message` 从嵌套 JSON 中抽取 message/type/code，避免直接把原始 API body 暴露给用户。

## 5. 未覆盖的范围

- 前端 Next.js 应用的错误处理（React Error Boundary、SWR/TanStack Query 错误、Playwright E2E 错误）不在本卡片范围内。
- 桌面客户端（Electron）的错误处理同样未展开。
- 日志系统（`deeptutor/logging/`）负责错误输出，但属于 logging 范畴而非 error_handling 范畴。