---
kind: logging_system
name: DeepTutor 日志系统：基于 stdlib logging + loguru 桥接的结构化 JSONL 日志
category: logging_system
scope:
    - '**'
source_files:
    - deeptutor/logging/__init__.py
    - deeptutor/logging/config.py
    - deeptutor/logging/configure.py
    - deeptutor/logging/context.py
    - deeptutor/logging/formatters.py
    - deeptutor/logging/loguru_bridge.py
    - deeptutor/logging/process_stream.py
    - tests/logging/test_configure.py
    - tests/logging/test_context.py
    - tests/logging/test_formatters.py
    - tests/logging/test_loguru_bridge.py
    - tests/logging/test_process_logs.py
---

## 1. 使用的系统与框架

- **核心框架**：Python 标准库 `logging`，作为唯一被配置和路由的日志后端。
- **可选依赖**：`loguru` 通过 `install_loguru_bridge` 桥接到 stdlib logging；若未安装则静默跳过（返回 `False`）。
- **输出格式**：
  - 控制台：`ConsoleFormatter`，人类可读，包含 level、logger、stage、task_id 等上下文片段。
  - 文件：`JsonlFormatter`，每行一个 JSON 对象，字段包括 `timestamp`、`level`、`logger`、`message`、`context`、`exception`。
- **文件轮转**：`logging.handlers.RotatingFileHandler`，默认 `maxBytes=10MB`、`backupCount=5`。
- **进程级结构化事件**：`process_stream.ProcessLogEvent` / `ProcessLogHandler`，将匹配的 stdlib 记录转换为可异步发送的 `ProcessLogEvent`，用于向用户暴露“进程日志”流。

## 2. 关键文件与包

| 路径 | 职责 |
|---|---|
| `deeptutor/logging/__init__.py` | 对外统一导出 `LoggingConfig`、`configure_logging`、`bind_log_context`、`capture_process_logs`、`LLMStats` 等 |
| `deeptutor/logging/config.py` | `LoggingConfig` dataclass + 从 `data/user/settings/main.yaml` 加载 `logging.*` 配置 |
| `deeptutor/logging/configure.py` | 进程启动时一次性调用 `configure_logging()`，装配 console/file handler 并安装 loguru 桥 |
| `deeptutor/logging/context.py` | 基于 `contextvars.ContextVar` 的请求级结构化字段绑定（`request_id`/`turn_id`/`session_id`/`task_id`/`capability`/`stage`/`sink`） |
| `deeptutor/logging/formatters.py` | `ContextFilter`（注入 `log_context`）、`JsonlFormatter`、`ConsoleFormatter` |
| `deeptutor/logging/loguru_bridge.py` | 将 loguru logger 的 sink 重定向到 stdlib logging |
| `deeptutor/logging/process_stream.py` | `ProcessLogEvent`、`ProcessLogHandler`、`capture_process_logs` 上下文管理器 |
| `tests/logging/` | 覆盖 configure、context、formatters、loguru bridge、process logs、LLM stats 的测试集 |

## 3. 架构与设计决策

### 3.1 单一入口初始化
`configure_logging(force=False)` 是全局唯一入口。内部用 `_CONFIGURED` 标志防止重复配置；`force=True` 会先移除所有带 `_deeptutor_managed` 属性的 handler，再重新装配。根 logger 级别固定设为 `DEBUG`，业务模块通过各自的 logger 名称（如 `deeptutor.xxx`）控制自身级别。

### 3.2 双 Sink 策略
- **console_output=True** → `StreamHandler(sys.stdout)` + `ConsoleFormatter`
- **file_output=True** → `RotatingFileHandler(data/user/logs/deeptutor.jsonl)` + `JsonlFormatter`
两个 handler 都经过 `_managed()` 包装，自动附加 `ContextFilter`。

### 3.3 结构化上下文
`LOG_CONTEXT_FIELDS = ("request_id", "turn_id", "session_id", "task_id", "capability", "stage", "sink")` 定义了允许写入 `record.log_context` 的白名单。`bind_log_context(**fields)` 使用 `contextvars` 在协程/线程范围内临时绑定这些字段；`ContextFilter.filter` 把当前 context 合并到每个 LogRecord 上，供 formatter 序列化。

### 3.4 配置来源
`load_logging_config()` 从 `main.yaml` 读取 `logging.level`、`logging.console_output`、`logging.save_to_file`、`logging.max_bytes`、`logging.backup_count`，以及 `user_log_dir`（映射到 `get_path_service().get_logs_dir()`）。任何异常都会回退到默认 `LoggingConfig(log_dir=...)`，保证日志子系统不会阻止应用启动。

### 3.5 loguru 兼容层
第三方库可能直接使用 `loguru.logger`。`install_loguru_bridge(logging.DEBUG)` 在启动时被调用，移除 loguru 默认 sink 并添加一个转发到 stdlib logging 的 sink，同时标记 `extra={"loguru": True}` 以便后续过滤。

### 3.6 进程日志事件
`ProcessLogHandler` 支持按 `task_id`/`turn_id` 过滤，并将匹配的记录转为 `ProcessLogEvent`，通过 `emit` 回调（可协程）异步投递。为避免 worker 线程无运行 loop 导致丢日志，handler 记住创建时的 event loop，并通过 `asyncio.run_coroutine_threadsafe` 调度。

## 4. 约定与约束

- **日志级别解析**：`_level()` 接受字符串或整数，字符串会被 `getattr(logging, value.upper(), logging.INFO)` 解析，未知值降级为 `INFO`。
- **日志目录权限**：文件输出前调用 `ensure_private_directory(log_dir)` 和 `ensure_private_file(log_path)`，确保日志目录与文件具有私有权限。
- **受管 handler 清理**：所有由 `configure_logging` 创建的 handler 都打上 `_MANAGED_ATTR="_deeptutor_managed"` 标记，便于 `force` 模式或测试中安全移除。
- **结构化字段白名单**：只有 `LOG_CONTEXT_FIELDS` 中的键才会被 `ContextFilter` 从 record 属性复制到 `log_context`，避免任意字段污染 JSONL。
- **进程日志隔离**：`PROCESS_LOG_PRIVATE_ATTR = "deeptutor_process_log_private"` 标记内部产生的记录，使其被 `ProcessLogHandler.emit` 直接丢弃，不进入用户可见的进程日志流。
- **异常堆栈保留**：`ConsoleFormatter` 显式调用 `formatException(record.exc_info)`，避免仅打印 message 而丢失 traceback。
- **配置容错**：`load_logging_config()` 捕获全部异常并返回最小可用配置，保证即使 settings 损坏也不影响程序启动。
- **命名空间约定**：业务代码应使用 `logging.getLogger("deeptutor.<module>")`，根 logger 下 `deeptutor` 命名空间被显式设为 DEBUG 且 propagate=True。
- **测试覆盖**：`tests/logging/` 中对 configure、context、formatters、loguru bridge、process logs、LLM stats 均有独立测试文件，验证行为契约。

## 5. 不适用性说明

本仓库存在完整的 `deeptutor/logging/` 子系统，具备独立的配置、格式化、上下文、桥接与进程事件能力，因此该类别适用。