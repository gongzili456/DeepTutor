---
kind: logging_system
name: 基于 stdlib logging + JSONL 的 DeepTutor 统一日志系统
category: logging_system
scope:
    - '**'
source_files:
    - deeptutor/logging/__init__.py
    - deeptutor/logging/configure.py
    - deeptutor/logging/config.py
    - deeptutor/logging/formatters.py
    - deeptutor/logging/context.py
    - deeptutor/logging/process_stream.py
    - deeptutor/logging/loguru_bridge.py
    - deeptutor/logging/adapters/llamaindex.py
    - deeptutor/logging/stats/llm_stats.py
    - deeptutor/api/main.py
---

## 1. 使用的系统与框架

DeepTutor 采用 Python 标准库 `logging` 作为统一的日志基础设施，并通过自定义处理器与格式化器实现结构化输出。核心依赖包括：
- `logging` + `logging.handlers.RotatingFileHandler`：进程级日志路由、按大小轮转的文件输出。
- `contextvars.ContextVar`：请求/任务级别的上下文变量，用于在异步/多线程环境中附加结构化字段（request_id、turn_id、session_id、task_id、capability、stage、sink）。
- 可选的 `loguru` 桥接：通过 `install_loguru_bridge` 将 loguru 的日志转发到 stdlib logger，使第三方库（如 LlamaIndex）的日志也能进入统一管道。
- 外部库适配：`adapters.llamaindex.LlamaIndexLogContext` 将 LlamaIndex 的多个子 logger 重定向到 `deeptutor.{scene}` 命名空间。

## 2. 关键文件与职责

| 文件 | 职责 |
|---|---|
| `deeptutor/logging/configure.py` | 进程启动时调用 `configure_logging()`，一次性配置 root logger、控制台与 JSONL 文件处理器，并安装 loguru 桥 |
| `deeptutor/logging/config.py` | `LoggingConfig` dataclass（level/console_output/file_output/log_dir/max_bytes/backup_count），从 `data/user/settings/main.yaml` 的 `logging` 段加载配置 |
| `deeptutor/logging/formatters.py` | `JsonlFormatter`（每行一个 JSON 对象）、`ConsoleFormatter`（开发用可读格式）、`ContextFilter`（把 contextvar 注入 LogRecord.log_context） |
| `deeptutor/logging/context.py` | `LOG_CONTEXT_FIELDS` 白名单 + `bind_log_context()` 上下文管理器 + `current_log_context()` |
| `deeptutor/logging/process_stream.py` | `ProcessLogEvent` dataclass + `ProcessLogHandler` + `capture_process_logs()` 上下文管理器，把匹配的 stdlib 记录转为可序列化的事件，供 WebSocket/UI 消费 |
| `deeptutor/logging/loguru_bridge.py` | 检测并安装 loguru → stdlib 的 sink |
| `deeptutor/logging/adapters/llamaindex.py` | 临时接管 `llama_index*` logger，将其输出转发到 `deeptutor.{scene}` |
| `deeptutor/logging/stats/llm_stats.py` | `LLMStats` 追踪 token 用量与成本，最终通过 `logging.getLogger(f"deeptutor.stats.{module}")` 输出汇总 |
| `deeptutor/api/main.py` | FastAPI 入口调用 `configure_logging()` 完成全局初始化 |

## 3. 架构与约定

### 3.1 启动流程
1. `api.main` 启动时调用 `configure_logging(force=False)`。
2. `load_logging_config()` 读取 `main.yaml` 中 `logging.*` 键；若失败则回退到默认值（level=INFO、console/file 均开启、log_dir 来自 path_service）。
3. 为 root logger 添加两个受管处理器：
   - `StreamHandler(sys.stdout)` → `ConsoleFormatter`（仅 console_output 为真时）。
   - `RotatingFileHandler("data/user/logs/deeptutor.jsonl")` → `JsonlFormatter`（仅 file_output 为真时）。
4. 设置 `logging.getLogger("deeptutor").setLevel(DEBUG)` 且 propagate=True，确保业务模块以 DEBUG 级别输出但由上层 handler 过滤。
5. 调用 `install_loguru_bridge(logging.DEBUG)` 把 loguru 输出汇入同一管道。

### 3.2 结构化上下文
- 所有业务代码通过 `with bind_log_context(task_id=..., capability=..., stage=..., sink=...)` 包裹一段逻辑。
- `ContextFilter` 在每个 LogRecord 上填充 `record.log_context = {field: value for field in LOG_CONTEXT_FIELDS if record 上有该属性 or 存在于 contextvar}`。
- `JsonlFormatter` 将 `timestamp`、`level`、`logger`、`message`、`context` 以及可选的 `exception` 序列化为一行 JSON。
- `ConsoleFormatter` 在开发终端显示 `LEVEL name@stage #task_id - message` 形式。

### 3.3 面向 UI 的进程日志流
- `capture_process_logs(emit, task_id=..., turn_id=..., min_level=INFO)` 动态挂载一个 `ProcessLogHandler` 到 root logger。
- Handler 只转发匹配当前 task_id/turn_id 的记录，并把它们包装成 `ProcessLogEvent` 通过 `emit` 回调（支持同步或异步 awaitable）。
- 典型用法：API 路由中以 `bind_log_context(task_id=..., capability="...", sink="ui")` 包裹整个请求处理，内部 `logging.info(...)` 即可被捕获并推送到前端。

### 3.4 第三方库日志隔离
- `LlamaIndexLogContext(scene="llamaindex", min_level="INFO")` 在进入作用域时移除 `llama_index*` 的 StreamHandler，禁用 propagate，并挂上 `LlamaIndexLogForwarder` 转发到 `deeptutor.{scene}`；退出时恢复原状。
- 这保证 LlamaIndex 的 verbose 输出不会污染主日志，而是进入独立命名空间。

### 3.5 日志级别策略
- root logger 始终设为 `DEBUG`，实际过滤由各 handler 的 level 控制。
- 全局级别由 `main.yaml` 的 `logging.level` 决定（默认 INFO）。
- `get_global_log_level()` 暴露当前配置以便其他模块按需调整行为。

## 4. 约定与约束

- **唯一入口**：进程内必须通过 `configure_logging()` 初始化；重复调用会跳过（除非 `force=True`），避免重复注册处理器。
- **结构化字段白名单**：只有 `LOG_CONTEXT_FIELDS` 中的键会被自动注入到每条记录的 `context` 字典；新增字段需在此元组声明。
- **JSONL 文件路径固定**：文件输出固定写入 `deeptutor.jsonl`，目录默认为 `data/user/logs`（可通过 `user_log_dir` 覆盖）。
- **轮转策略**：默认 `max_bytes=10MB`、`backup_count=5`，通过 `RotatingFileHandler` 管理磁盘占用。
- **异步安全**：`ProcessLogHandler.emit` 对异步 emit 使用 `asyncio.ensure_future` 非阻塞派发；异常被 `handleError` 吞掉，避免影响主流程。
- **命名空间规范**：业务 logger 统一使用 `deeptutor.*` 前缀（如 `deeptutor.stats.<module>`、`deeptutor.llamaindex`），便于单独调高/调低级别。
- **测试覆盖**：`tests/logging/` 下包含 `test_configure.py`、`test_context.py`、`test_loguru_bridge.py`、`test_process_logs.py`、`test_task_log_stream.py`，验证配置加载、上下文传播、loguru 桥接与进程日志捕获的正确性。

## 5. 与其他组件的关系

- `agents/base_agent.py` 等智能体模块通过 `from deeptutor.logging import LLMStats` 统计 token 用量，并在结束时 `log_summary()` 输出到 `deeptutor.stats.*`。
- API 路由（question、plugins_api、task_log_stream 等）是 `bind_log_context` + `capture_process_logs` 的主要使用者，将后端运行过程实时推送给前端。
- `services/` 下的各服务（RAG、embedding、sandbox 等）通过标准 `logging.getLogger(__name__)` 输出，自动继承根 logger 的处理器与上下文过滤器。