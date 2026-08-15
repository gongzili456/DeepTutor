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
    - deeptutor_cli/common.py
    - deeptutor/services/codex_auth/catalog.py
---

## 1. 系统/方法概述

DeepTutor 在后端 Python 层采用「自定义应用异常 + FastAPI `HTTPException`」的双轨模式：
- 业务/领域层通过 `deeptutor/core/errors.py` 定义的 `DeepTutorError` 及其子类（`ConfigurationError`、`ValidationError`、`ServiceError`、`LLMServiceError`、`LLMContextError`、`EnvironmentConfigError`）表达可被上层捕获的领域错误。
- API 边界统一通过 FastAPI 的 `HTTPException` 返回标准 HTTP 状态码，由框架自动序列化为 JSON 响应体。
- 进程/服务生命周期中的非致命故障（LLM 客户端初始化、EventBus 启动、Partner 自启动、Cron、PocketBase 探测、v1 内存迁移、MCP 关闭、LLM provider pool 关闭等）在 `lifespan` 中用 `try/except Exception` 包裹并降级为 `logger.warning`，确保单组件失败不会拖垮整个服务。
- CLI 侧使用 Typer 命令 + 自定义 `TurnInterrupted` 异常配合 SIGINT 拦截实现交互式中断；通用工具 `format_exception_message` 会尝试从异常消息中提取 JSON 块并格式化可读信息。

仓库中**没有**全局 `@app.exception_handler` 注册，也没有统一的异常到 HTTP 状态码映射器——所有 HTTP 错误都在路由内部显式 `raise HTTPException(...)`。

## 2. 关键文件与包

| 文件 | 作用 |
|---|---|
| `deeptutor/core/errors.py` | 定义 `DeepTutorError` 基类及 `ConfigurationError`、`ValidationError`、`ServiceError`、`LLMServiceError`、`LLMContextError`、`EnvironmentConfigError` 等细分异常 |
| `deeptutor/utils/error_utils.py` | `format_exception_message`：从异常字符串中解析 JSON 块并提取 `message/type/code` 字段，用于日志可读化 |
| `deeptutor/api/main.py` | FastAPI 应用入口：`lifespan` 中各子服务启动/关闭均 try/except 降级；`selective_access_log` 中间件仅记录非 200 请求；`SafeOutputStaticFiles` 对越界静态访问返回 404 |
| `deeptutor/api/routers/auth.py` | 认证相关路由：登录失败返回 401、未授权/无管理员权限返回 403、资源不存在返回 404；OAuth 回调将 `CodexAuthError` 转为带 `http_status` 的 HTML 响应 |
| `deeptutor/api/routers/book.py` | Book Engine 路由：大量 `raise HTTPException(status_code=400/404/500, detail=...)` 示例 |
| `deeptutor_cli/common.py` | CLI 流渲染器：`TurnInterrupted` 异常 + SIGINT 拦截，Ctrl-C 取消当前 turn 而非退出 REPL |
| `deeptutor/services/codex_auth/*.py` | 局部领域异常 `CodexAuthError`，被 auth 路由捕获后映射为 HTTP 状态码 |

## 3. 架构与约定

### 3.1 异常层次
```
Exception
 └── DeepTutorError (core/errors.py)
      ├── ConfigurationError
      │    └── EnvironmentConfigError
      ├── ValidationError
      ├── ServiceError
      │    └── LLMServiceError
      │         └── LLMContextError
```
该层次用于在业务/服务层区分“配置问题”、“输入校验失败”、“下游服务不可用”、“LLM 上下文超限”等语义，便于调用方按类型 catch。

### 3.2 API 层错误传播
- 路由内直接 `raise HTTPException(status_code=..., detail=...)`，例如：
  - 401：`auth.py` 登录失败
  - 403：`require_admin` 依赖检查失败
  - 404：用户/头像/Book/Spine/Page/Attachment 等资源不存在
  - 400：参数缺失或格式非法（如 `user_intent is required`、`Invalid proposal/spine`）
  - 500/501：后端未实现或内部错误（如 `Attachment backend not servable`）
- 没有全局异常处理器，因此每个路由必须自行决定 HTTP 状态码。

### 3.3 启动期容错（Lifespan）
`lifespan` 中每个外部依赖的启动/关闭都独立 try/except：
```python
try:
    llm_client = get_llm_client()
except Exception as e:
    logger.warning(f"Failed to initialize LLM client at startup: {e}")
```
这种模式贯穿 LLM 客户端、EventBus、Partner 管理器、Cron、PocketBase 探测、v1 内存迁移、MCP 连接关闭、LLM provider pool 关闭、Agentic client pool 关闭等。目的是让 DeepTutor 在部分依赖不可用时仍能启动。

### 3.4 访问日志作为“错误可见性”通道
`selective_access_log` 中间件只记录非 200 响应，配合 uvicorn 默认 access log 被关闭（通过 `--no-access-log`），形成“只有错误才输出”的访问日志策略。

### 3.5 CLI 错误处理
- `TurnInterrupted` 是 CLI 专用的中断信号，`render_turn_stream` 捕获后调用 `_cancel_interrupted_turn` 安全终止后端 turn。
- SIGINT 通过 `_SigintInterceptor` 安装/移除，使 Ctrl-C 在 streaming 期间取消任务而不是退出 REPL。
- 配置解析失败时抛出 `ValueError`（如 `parse_config_items`、`parse_json_object`），Typer 会将其转换为友好的命令行错误提示。

### 3.6 局部领域异常
`services/codex_auth` 模块定义了 `CodexAuthError`，并在 `auth.py` 的 `/openai-codex/callback` 中被捕获，根据 `exc.http_status` 返回对应状态的 HTML 页面。这是“领域异常 → HTTP 响应”的典型转换点。

## 4. 约定与约束

- **业务层优先抛出自定义 `DeepTutorError` 子类**，以便上层按语义区分错误来源（配置/校验/服务/LLM）。
- **API 路由层不吞异常**：所有需要对外暴露的错误都显式 `raise HTTPException`，由 FastAPI 统一序列化；未捕获异常会走 FastAPI 默认 500 处理。
- **启动期依赖失败一律降级为 warning**：`lifespan` 中任何 `except Exception` 都记录警告并继续启动，不允许单个子系统失败导致服务无法上线。
- **访问日志只关注非 200**：`selective_access_log` 中间件过滤掉 200 响应，减少噪音，使运维聚焦错误请求。
- **CLI 交互中断使用专用异常**：`TurnInterrupted` 配合 SIGINT 拦截，保证 Ctrl-C 只取消当前 turn 而不退出 REPL。
- **JSON 错误消息可读化**：`format_exception_message` 会尝试从异常字符串中抽取 `{...}` JSON 块并格式化 `message/type/code`，用于日志输出更友好。
- **静态资源访问白名单**：`SafeOutputStaticFiles.get_response` 对不在公开路径列表中的请求返回 404，防止任意文件读取。

## 5. 观察到的缺口

- 没有全局 `@app.exception_handler(Exception)` 做兜底错误归一化，新增路由容易遗漏状态码选择。
- `DeepTutorError` 及其子类目前仅在 `core/errors.py` 定义，grep 未发现业务代码广泛 raise 这些类型（更多直接使用 `HTTPException` 和 `ValueError`），说明该异常体系尚未完全下沉到服务层。
- 没有统一的错误码枚举或错误响应模型（如 Pydantic `ErrorResponse`），错误详情以纯字符串 `detail` 传递，前端需硬编码匹配。
