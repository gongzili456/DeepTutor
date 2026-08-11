---
kind: logging_system
name: 基于 stdlib logging + JSONL 的 DeepTutor 统一日志系统
category: logging_system
scope:
    - '**'
source_files:
    - deeptutor/logging/__init__.py
    - deeptutor/logging/config.py
    - deeptutor/logging/configure.py
    - deeptutor/logging/formatters.py
    - deeptutor/logging/context.py
    - deeptutor/logging/process_stream.py
    - deeptutor/logging/loguru_bridge.py
    - deeptutor/logging/adapters/llamaindex.py
    - deeptutor/logging/stats/llm_stats.py
    - deeptutor/api/main.py
    - deeptutor/api/run_server.py
    - deeptutor/api/routers/plugins_api.py
    - deeptutor/api/routers/question.py
    - deeptutor/api/utils/task_log_stream.py
---

## 1. 使用的系统与框架

DeepTutor 采用 Python 标准库 `logging` 作为唯一日志后端，通过自定义 Handler、Formatter 与 Filter 实现结构化输出；可选集成 `loguru`（通过桥接器转发到 stdlib）；第三方库 LlamaIndex 的日志通过专用适配器重定向到 DeepTutor 命名空间。所有日志最终汇聚到根 logger，由启动时统一配置。

## 2. 核心文件与包

- `deeptutor/logging/config.py`：定义 `LoggingConfig` dataclass（level/console_output/file_output/log_dir/max_bytes/backup_count），从 `data/user/settings/main.yaml` 的 `logging` 段加载配置，并提供 `get_default_log_dir()` / `load_logging_config()` / `get_global_log_level()`。
- `deeptutor/logging/configure.py`：进程级入口 `configure_logging(force=False)`，仅执行一次（全局 `_CONFIGURED` 标记），为 root logger 安装控制台 `StreamHandler`（`ConsoleFormatter`）与轮转文件 `RotatingFileHandler`（输出 `data/user/logs/deeptutor.jsonl`），并调用 `install_loguru_bridge(logging.DEBUG)`。
- `deeptutor/logging/formatters.py`：`ContextFilter` 把 `contextvars` 中的请求上下文字段注入每个 `LogRecord`；`JsonlFormatter` 输出单行 JSON（含 timestamp/level/logger/message/context/exception）；`ConsoleFormatter` 用于开发终端可读输出。
- `deeptutor/logging/context.py`：基于 `contextvars.ContextVar` 的请求级日志上下文，预定义 `LOG_CONTEXT_FIELDS = (request_id, turn_id, session_id, task_id, capability, stage, sink)`，提供 `bind_log_context(**fields)` 上下文管理器与 `current_log_context()`。
- `deeptutor/logging/process_stream.py`：`ProcessLogEvent` dataclass + `ProcessLogHandler` + `capture_process_logs()` 上下文管理器，将匹配的 stdlib 记录转换为事件并通过异步 emit 回调推送到前端/队列，支持按 `task_id`/`turn_id` 过滤。
- `deeptutor/logging/loguru_bridge.py`：`install_loguru_bridge(level=logging.DEBUG)` 移除 loguru 默认 handler，添加 sink 将 loguru 记录转发到 stdlib 同名 logger，使第三方使用 loguru 的代码也能进入统一管道。
- `deeptutor/logging/adapters/llamaindex.py`：`LlamaIndexLogContext` 上下文管理器临时接管 `llama_index*` 命名空间的 logger，将其重定向到 `deeptutor.{scene}` 下，避免噪音污染。
- `deeptutor/logging/stats/llm_stats.py`：`LLMStats` 追踪 token 用量与成本，汇总后通过 `logging.getLogger(f"deeptutor.stats.{module}")` 输出。

## 3. 架构与约定

- **单一入口**：所有服务在启动时调用 `configure_logging()`（见 `deeptutor/api/main.py`、`deeptutor/api/run_server.py`），该函数幂等且可 `force=True` 重置已安装的 managed handlers。
- **双通道输出**：
  - 控制台：`ConsoleFormatter`，格式如 `INFO deeptutor.xxx @stage #task_id - message`。
  - 文件：`JsonlFormatter` 写入 `RotatingFileHandler`，默认路径 `data/user/logs/deeptutor.jsonl`，按 `max_bytes`/`backup_count` 轮转。
- **结构化上下文**：业务代码通过 `with bind_log_context(task_id=..., capability=..., sink=...)` 绑定请求级元数据；`ContextFilter` 自动合并到每条记录的 `log_context` 字段，JSONL 中可见。
- **进程日志流**：通过 `capture_process_logs(emit, task_id=...)` 捕获匹配的记录并转为 `ProcessLogEvent`，由 API 层（如 `plugins_api.py`、`question.py`、`task_log_stream.py`）转发到 WebSocket/事件队列，供 UI 实时展示。
- **第三方日志收敛**：loguru 通过桥接器统一汇入 stdlib；LlamaIndex 通过 `LlamaIndexLogContext` 限定作用域转发到 `deeptutor.llamaindex` 等子 logger。
- **统计日志**：`LLMStats.log_summary()` 使用 `logging.getLogger(f"deeptutor.stats.{module}")` 输出，遵循同一管道。

## 4. 约定与约束

- 日志级别通过 `main.yaml` 的 `logging.level` 配置，默认 `INFO`；root logger 内部始终设为 `DEBUG`，具体 handler 再按配置级别过滤。
- 文件输出开关为 `logging.save_to_file`（默认 True），控制台输出开关为 `logging.console_output`（默认 True）。
- 日志目录可通过 `user_log_dir` 覆盖，否则回退到 `services.path_service.get_path_service().get_logs_dir()`。
- 受 configure_logging 管理的 handler 会打上 `_deeptutor_managed` 标记，便于 force 重启时清理。
- 请求级上下文字段必须来自 `LOG_CONTEXT_FIELDS` 白名单，确保 JSONL 结构稳定。
- `capture_process_logs` 的 emit 回调若返回 awaitable，会在当前 event loop 中以 `asyncio.ensure_future` 异步派发，避免阻塞日志路径。
- LlamaIndex 日志仅在 `LlamaIndexLogContext` 作用域内被转发，退出后恢复原 logger 状态。
- 测试套件位于 `tests/logging/`，覆盖 `configure`、`context`、`loguru_bridge`、`process_logs`、`task_log_stream` 等行为。