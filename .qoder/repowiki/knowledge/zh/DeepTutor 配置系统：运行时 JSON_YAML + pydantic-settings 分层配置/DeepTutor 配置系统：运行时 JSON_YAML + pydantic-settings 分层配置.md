---
kind: configuration_system
name: DeepTutor 配置系统：运行时 JSON/YAML + pydantic-settings 分层配置
category: configuration_system
scope:
    - '**'
source_files:
    - deeptutor/config/settings.py
    - deeptutor/services/config/loader.py
    - deeptutor/services/config/runtime_settings.py
    - deeptutor/services/config/model_catalog.py
    - deeptutor/utils/config_manager.py
    - deeptutor_cli/setup_config.py
    - deeptutor_cli/config_cmd.py
---

## 1. 采用的框架与工具

- **进程级环境变量 → Pydantic Settings**：`deeptutor/config/settings.py` 使用 `pydantic_settings.BaseSettings`，通过 `env_prefix="LLM_"` 与 `env_nested_delimiter="__"` 将 `LLM_RETRY__MAX_RETRIES`、`LLM_RETRY__BASE_DELAY` 等环境变量映射为嵌套的 `Settings.retry` 模型。
- **持久化运行时设置（JSON）**：`deeptutor/services/config/runtime_settings.py` 中的 `RuntimeSettingsService` 以 `data/user/settings/*.json` 为持久化存储，每个功能域一个文件（`system.json`、`auth.json`、`integrations.json`、`document_parsing.json`、`pageindex.json`、`query_rewrite.json`、`ima.json`、`llamaindex.json`、`graphrag.json`、`lightrag.json`、`lightrag_server.json`），所有写操作通过 `file_io.atomic_write_json` 原子写入。
- **模型/凭据目录（JSON）**：`deeptutor/services/config/model_catalog.py` 中的 `ModelCatalogService` 管理 `model_catalog.json`，维护 `connections` 与 `services.llm/embedding/search/tts/stt/imagegen/videogen` 的 profile 与 active_profile_id / active_model_id。
- **YAML 运行时行为配置**：`deeptutor/services/config/loader.py` 提供 `load_config_with_main`，从 `data/user/settings/main.yaml` 加载并注入运行时路径；`agents.yaml` 是 LLM 预算（temperature/max_tokens）的唯一来源。
- **CLI 配置入口**：`deeptutor_cli/config_cmd.py` 暴露 `deeptutor config providers|apply|show`，`setup_config.py` 用 Pydantic 校验用户输入后调用服务层写入 catalog/system。

## 2. 关键文件与包

- `deeptutor/config/settings.py` — 进程级 Pydantic Settings（LLM retry 等）
- `deeptutor/services/config/loader.py` — YAML 加载、路径解析、语言归一化、agent 参数读取
- `deeptutor/services/config/runtime_settings.py` — 所有 JSON 运行时设置的默认值、规范化器、进程环境变量覆盖、`render_environment`/`export_environment`
- `deeptutor/services/config/model_catalog.py` — model_catalog.json 的读写、连接复用、secret 脱敏/还原
- `deeptutor/utils/config_manager.py` — 对 `main.yaml` 的单例缓存 + mtime 热重载
- `deeptutor_cli/setup_config.py` — 非交互式 `deeptutor config apply` 的输入校验与 profile 构造
- `deeptutor_cli/config_cmd.py` — CLI 命令注册

## 3. 架构与约定

### 3.1 配置分层（由低到高优先级）

1. **内置默认值**：每个 `DEFAULT_*_SETTINGS` dict（如 `DEFAULT_SYSTEM_SETTINGS`、`DEFAULT_DOCUMENT_PARSING_SETTINGS`）定义字段、取值范围与注释。
2. **持久化 JSON/YAML 文件**：`data/user/settings/*.json` 与 `main.yaml`/`agents.yaml`。读时 `_load_or_create` 合并 defaults + 磁盘内容，若结构变化则回写。
3. **进程环境变量覆盖**：`RuntimeSettingsService._process_env_value` 在 `include_process_overrides=True` 时把 `DEEPTUTOR_*`、`BACKEND_PORT`、`AUTH_*`、`MINERU_*` 等变量映射到对应字段。`IGNORE_PROCESS_OVERRIDES_ENV = "DEEPTUTOR_IGNORE_PROCESS_ENV_OVERRIDES"` 可全局关闭。
4. **Pydantic Settings 环境变量**：`LLM_*` 前缀的嵌套变量直接注入 `Settings` 实例，独立于 JSON 层。

### 3.2 路径约定

- 运行时根目录由 `runtime.home` 决定（`DEEPTUTOR_HOME_ENV`），默认 `PROJECT_ROOT/data/user`。
- 所有用户设置统一位于 `<runtime_home>/data/user/settings/`，由 `get_runtime_settings_dir()` 返回。
- `main.yaml` 中 `paths.*` 与 `tools.exec.workspace` 由 `_inject_runtime_paths` 动态注入，不视为用户可编辑状态。

### 3.3 安全与机密

- `ModelCatalogService` 对 `api_key`/`apikey`/`token`/`secret`/`password` 及 `extra_headers` 字段统一脱敏为 `***`（`CATALOG_SECRET_MASK`），并提供 `restore_catalog_secrets` 在内部恢复。
- `setup_config._api_key` 要求 API Key 来自环境变量名，禁止在 `base_url` 中携带凭据。
- `settings_derived_keys()` 区分“由 settings 文件导出”和“operator 预先设置”的环境变量，避免子进程覆盖 operator 的值。

### 3.4 原子性与并发

- JSON 写均通过临时文件 + `os.replace`（或 `atomic_write_json`）实现原子替换。
- `ConfigManager` 使用 `RLock` 单例；`ModelCatalogService` 在 `load/save/update` 上持有 `threading.RLock`。
- `ConfigManager.load_config` 基于 `mtime` 缓存，支持 `force_reload` 热重载。

### 3.5 向后兼容与迁移

- `RuntimeSettingsService._migrate_legacy_document_parsing_file` 自动将旧 `mineru.json` 重命名为 `document_parsing.json`。
- `ModelCatalogService._normalize` 填充缺失字段、生成 UUID id、清理废弃的 `tasks` 指针与过期的 task overrides。
- `api_format`/`wire_api`/`binding` 三者在 normalize 中互相推导，保证旧文件可读。

### 3.6 职责边界（代码内明确声明的规则）

`loader.py` 的注释明确划分：
> “agents.yaml owns every LLM budget (temperature, max_tokens, per stage), main.yaml owns runtime behaviour (iteration caps, timeouts, retry policy, modes). A budget in main.yaml is a bug, not a style choice.”

即：**LLM 预算**（temperature、max_tokens、per-stage）只允许出现在 `agents.yaml`；**运行时行为**（迭代上限、超时、重试策略、模式开关）只允许出现在 `main.yaml`。

## 4. 约定与约束

- 所有运行时设置必须通过 `RuntimeSettingsService` 的 `load_*`/`save_*` 访问，禁止直接读写 `data/user/settings/*.json`。
- 新增设置需在 `DEFAULT_*_SETTINGS` 中声明默认值，并在 `_apply_*_process_overrides` 中添加对应的环境变量映射。
- 新增字段需配套 `_coerce_*` 规范化函数（bool/int/port/list）以保证脏数据不会破坏文件。
- 配置文件版本字段（`version`）用于向前兼容，升级逻辑放在 normalizer 中。
- 进程环境变量优先于文件，但被 `export_environment` 标记为“本进程导出的”键不会被视作 operator 覆盖。
- `main.yaml` 不是 operator 配置面（注释明确指出 project-root `main.yaml` 不应作为 operator 配置表面），operator 应使用 `DEEPTUTOR_*` 环境变量或 `system.json`。
- CLI `deeptutor config apply` 要求至少提供 `llm`/`embedding`/`search`/`system` 之一，且 `system.backend_port != system.frontend_port`。
- 所有 secret 字段在 API 响应中一律脱敏为 `***`，前端不得看到真实值。