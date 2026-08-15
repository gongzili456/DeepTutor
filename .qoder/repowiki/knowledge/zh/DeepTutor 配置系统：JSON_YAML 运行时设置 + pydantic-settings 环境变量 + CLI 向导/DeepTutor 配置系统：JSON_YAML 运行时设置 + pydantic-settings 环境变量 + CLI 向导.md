---
kind: configuration_system
name: DeepTutor 配置系统：JSON/YAML 运行时设置 + pydantic-settings 环境变量 + CLI 向导
category: configuration_system
scope:
    - '**'
source_files:
    - deeptutor/runtime/home.py
    - deeptutor/services/config/loader.py
    - deeptutor/services/config/runtime_settings.py
    - deeptutor/services/config/model_catalog.py
    - deeptutor/utils/config_manager.py
    - deeptutor/config/settings.py
    - deeptutor/config/schema.py
    - deeptutor/config/accessors.py
    - deeptutor/config/constants.py
    - deeptutor_cli/config_cmd.py
    - deeptutor_cli/init_wizard.py
    - .env.example
---

## 1. 整体方案

DeepTutor 采用**分层配置**策略，将“部署/进程级开关”“用户持久化设置”“模型/凭据目录”“应用内行为参数”四类配置解耦，分别由不同子系统负责：

| 层级 | 存储位置 | 格式 | 覆盖来源 | 主要用途 |
|---|---|---|---|---|
| 进程/部署覆盖 | 进程 `os.environ` | 键值对 | Docker / compose / shell | 端口、CORS、API base URL、认证开关等 |
| 运行时设置（用户） | `<runtime-home>/data/user/settings/*.json` | JSON | API / CLI 写入 | system、auth、integrations、document_parsing、llamaindex、graphrag、lightrag、pageindex |
| 模型与凭据目录 | `<runtime-home>/data/user/settings/model_catalog.json` | JSON | `deeptutor init` 向导 / Settings UI | LLM / embedding / search / tts / stt / imagegen / videogen 的 profile 与 active model |
| 能力/Agent 参数 | `<runtime-home>/data/user/settings/agents.yaml` | YAML | 手动编辑或默认注入 | temperature、max_tokens、chat loop 预算 |
| 通用主配置 | `<runtime-home>/data/user/settings/main.yaml` | YAML | ConfigManager 读写 | 工具开关、语言、paths 等 |
| 应用启动参数 | `pydantic-settings` | 环境变量 | `LLM_RETRY__*` 前缀 | LLM 重试策略（最大重试次数、退避延迟等） |
| 项目根 `.env` | 根目录 `.env` | dotenv | 仅开发参考 | 容器宿主机端口映射说明（代码不读取） |

## 2. 核心文件与职责

- `deeptutor/runtime/home.py`：解析 `DEEPTUTOR_HOME` 环境变量确定 runtime home，所有数据位于 `<home>/data`。
- `deeptutor/services/config/loader.py`：统一加载 `data/user/settings/*.yaml`，提供 `_deep_merge`、`load_config_with_main`、`get_agent_params`、`get_chat_params`、`parse_language` 等工具；自动注入 `paths.*` 运行时路径。
- `deeptutor/services/config/runtime_settings.py`：`RuntimeSettingsService` 是运行时设置的唯一入口，维护 system/auth/integrations/document_parsing/llamaindex/graphrag/lightrag/pageindex 等 JSON 文件的读/写/迁移/归一化，并通过 `_apply_*_process_overrides` 把进程环境变量覆盖到内存中的 settings。
- `deeptutor/services/config/model_catalog.py`：`ModelCatalogService` 管理 `model_catalog.json`，按服务（llm/embedding/search/tts/stt/imagegen/videogen）组织 profiles 与 models，并维护 active_profile_id / active_model_id。
- `deeptutor/utils/config_manager.py`：单例 `ConfigManager`，以原子写（temp + `os.replace`）方式读写 `main.yaml`，并提供 `validate_required_env` 等辅助。
- `deeptutor/config/settings.py`：基于 `pydantic-settings` 的 `Settings(BaseSettings)`，通过 `env_prefix="LLM_"` + `env_nested_delimiter="__"` 将 `LLM_RETRY__MAX_RETRIES` 等环境变量映射为结构化对象。
- `deeptutor/config/schema.py`：Pydantic 模型定义（`LLMConfig`、`PathsConfig`、`AppConfig`），含 `CURRENT_SCHEMA_VERSION` 与占位 `migrate_config`。
- `deeptutor/config/accessors.py`：`ConfigAccessor` 提供 `llm_model()` / `llm_provider()` / `user_data_dir()` 等只读访问器。
- `deeptutor_cli/config_cmd.py`：`deeptutor config show` 命令，聚合 system、LLM、embedding、search、language、tools 当前状态输出。
- `deeptutor_cli/init_wizard.py`：交互式 `init` 向导，驱动 provider 选择、API key 捕获（支持 env 自动探测）、调用 `/models` 列表、可选连通性 probe，最终写入 `model_catalog.json`。
- `.env.example`：仅作为 compose 宿主端口映射的示例注释，明确说明 `DEEPTUTOR_API_BASE_URL` 实际来自 `system.json`。

## 3. 架构与设计约定

### 3.1 单一事实源 + 进程覆盖
`RuntimeSettingsService._load_or_create` 从磁盘 JSON 加载，合并默认值后调用对应 `_normalize_*` 归一化；若存在差异则原子写回。每次 load 时可选择 `include_process_overrides=True`，通过 `_process_env_value` 把进程环境变量注入内存视图——**环境变量不持久化**，仅作为部署覆盖层。

### 3.2 原子写保障
所有 JSON/YAML 保存都使用 `tempfile.mkstemp` + `os.replace`（`ConfigManager.save_config`、`ModelCatalogService.save`、`RuntimeSettingsService` 使用的 `atomic_write_json`），避免部分写入导致配置文件损坏。

### 3.3 向后兼容与迁移
- `document_parsing.json` 从旧版 `mineru.json` 在首次加载时自动重命名（`_migrate_legacy_document_parsing_file`），并将 v1 扁平结构迁移到 v2 的 `engines.mineru` 子段。
- `model_catalog.json` 在 `load()` 时补齐缺失字段（profile id、model id、active_profile_id 等）并写回。
- `AppConfig` 中预留 `CURRENT_SCHEMA_VERSION=1` 与 `migrate_config` 占位函数，用于未来 schema 升级。

### 3.4 多租户/多用户隔离
`_global_settings_dir()` 优先尝试 `multi_user.paths.get_admin_path_service().get_settings_dir()`，失败则回退到普通 PathService；`ModelCatalogService.get_instance` 在非管理员上下文中也切换到 admin 路径。这使得多用户模式下每个用户的 settings 目录相互隔离。

### 3.5 前端/后端地址协商
`RuntimeSettingsService.render_environment` 生成 `DEEPTUTOR_API_BASE_URL`，优先级为 `next_public_api_base` → `next_public_api_base_external` → `http://127.0.0.1:{backend_port}`。Next.js 侧 `web/proxy.ts` 在请求时据此重写 `/api/*` 和 `/ws/*`，而非构建期注入，因此修改 `system.json` 后无需重新构建前端。

### 3.6 配置可见性与安全
- `config_cmd` 显示 API key 时统一用 `***` 或 `(not set)` 脱敏。
- `init_wizard.capture_api_key` 支持从已有保存值、环境变量、隐藏输入三种来源，并以掩码展示。
- `SENSITIVE_ENDPOINT_QUERY_KEYS` 与 `redact_embedding_endpoint_for_display` 用于在错误信息中脱敏 endpoint URL 中的敏感查询参数。

## 4. 约定与约束

- **运行时 home 解析顺序**：显式 `home` 参数 > `DEEPTUTOR_HOME` 环境变量 > `cwd()`（`runtime.home.get_runtime_home`）。
- **settings 目录固定路径**：`<runtime-home>/data/user/settings`，由 `loader.get_runtime_settings_dir` 统一返回。
- **进程覆盖开关**：`DEEPTUTOR_IGNORE_PROCESS_ENV_OVERRIDES` 为真时，`RuntimeSettingsService` 完全忽略进程环境变量覆盖。
- **端口范围约束**：`_coerce_port` 强制端口在 1–65535，非法值回退默认。
- **聊天附件限制范围**：`CHAT_ATTACHMENT_MAX_FILE_MB_RANGE=(1,1024)`、`CHAT_ATTACHMENT_MAX_TOTAL_MB_RANGE=(1,2048)`、`CHAT_ATTACHMENT_CHARS_RANGE=(10_000,5_000_000)`，超出范围会被钳制。
- **LLM 重试配置**：必须通过 `LLM_RETRY__MAX_RETRIES` / `LLM_RETRY__BASE_DELAY` / `LLM_RETRY__EXPONENTIAL_BACKOFF` 三个环境变量设置，`settings.llm_retry` 属性已标记为弃用并产生 `DeprecationWarning`。
- **agents.yaml 必需**：`get_agent_params` 在未找到 `agents.yaml` 时直接抛 `FileNotFoundError`，该文件是 agent temperature/max_tokens 的唯一来源。
- **`.env` 不被读取**：`RuntimeSettingsService` 文档明确“Project-root `.env` files are intentionally ignored”，部署覆盖只能通过进程环境变量或 `system.json`。
- **Docker/compose 端口绑定**：`.env.example` 注释强调容器内端口由 `data/user/settings/system.json` 的 `backend_port`/`frontend_port` 决定，`.env` 仅调整宿主机映射。

## 5. 关键文件清单

- `deeptutor/runtime/home.py`
- `deeptutor/services/config/loader.py`
- `deeptutor/services/config/runtime_settings.py`
- `deeptutor/services/config/model_catalog.py`
- `deeptutor/utils/config_manager.py`
- `deeptutor/config/settings.py`
- `deeptutor/config/schema.py`
- `deeptutor/config/accessors.py`
- `deeptutor/config/constants.py`
- `deeptutor_cli/config_cmd.py`
- `deeptutor_cli/init_wizard.py`
- `.env.example`