---
kind: configuration_system
name: DeepTutor 配置系统：分层 JSON/YAML + pydantic-settings + 运行时目录
category: configuration_system
scope:
    - '**'
source_files:
    - deeptutor/runtime/home.py
    - deeptutor/services/config/runtime_settings.py
    - deeptutor/services/config/model_catalog.py
    - deeptutor/services/config/provider_runtime.py
    - deeptutor/services/config/loader.py
    - deeptutor/utils/config_manager.py
    - deeptutor/config/settings.py
    - deeptutor/config/schema.py
    - deeptutor/config/accessors.py
    - deeptutor/config/constants.py
    - deeptutor/config/defaults.py
    - deeptutor_cli/config_cmd.py
    - deeptutor_cli/init_wizard.py
    - .env.example
---

## 1. 整体方案

DeepTutor 的配置系统采用**多源分层**设计，核心由以下组件构成：

- **运行时根目录（Runtime Home）**：通过 `DEEPTUTOR_HOME` 环境变量或当前工作目录确定，所有持久化配置位于 `<home>/data/user/settings/`。该路径由 `deeptutor/runtime/home.py` 的 `get_runtime_home()` / `get_runtime_data_root()` 统一解析。
- **JSON 运行时设置**：`RuntimeSettingsService`（`deeptutor/services/config/runtime_settings.py`）管理 `system.json`、`auth.json`、`integrations.json`、`document_parsing.json`、`llamaindex.json`、`graphrag.json`、`lightrag.json`、`pageindex.json` 等文件，每个都有默认值、规范化器（`_normalize_*`）和可选的环境变量覆盖。
- **模型目录（Model Catalog）**：`model_catalog.json` 集中保存 LLM / Embedding / Search / TTS / STT / Imagegen / Videogen 各服务的 Profile 与 Active Model，由 `ModelCatalogService` 以线程安全方式读写。
- **YAML 用户配置**：`main.yaml`（以及 `agents.yaml`）由 `ConfigManager`（`deeptutor/utils/config_manager.py`）加载，提供 `load_config` / `save_config` 原子写入（先写临时文件再 `os.replace`），并支持热重载（基于 mtime）。`loader.load_config_with_main()` 还会注入运行时路径到 `paths.*` 字段。
- **进程级环境变量**：`pydantic_settings.BaseSettings`（`deeptutor/config/settings.py`）通过 `env_prefix="LLM_"` 与 `env_nested_delimiter="__"` 将 `LLM_RETRY__MAX_RETRIES` 等环境变量映射为嵌套结构；`RuntimeSettingsService.render_environment()` 将 JSON 设置导出为 `BACKEND_PORT`、`DEEPTUTOR_API_BASE_URL`、`AUTH_ENABLED` 等子进程环境变量。
- **CLI 初始化向导**：`deeptutor_cli/init_wizard.py` 提供交互式 provider 选择、API Key 捕获（优先复用已保存值 → 检测环境变量 → 隐藏输入）、在线 `/models` 探测、连通性 ping，最终写入 catalog。
- **前端 `.env.example`**：仅定义宿主机端口绑定（`HOST_PORT_BACKEND`、`HOST_PORT_FRONTEND`、`HOST_PORT_POCKETBASE`、`TZ`），容器内端口来自 `system.json`。

## 2. 关键文件与包

| 文件 | 职责 |
|---|---|
| `deeptutor/runtime/home.py` | 解析 `DEEPTUTOR_HOME`，返回 runtime home 与 data root |
| `deeptutor/services/config/runtime_settings.py` | JSON 设置服务：system/auth/integrations/document_parsing/llamaindex/graphrag/lightrag/pageindex，含默认值、规范化、环境覆盖、`render_environment` |
| `deeptutor/services/config/model_catalog.py` | `model_catalog.json` 的 CRUD，维护 services.profiles.active_profile_id / active_model_id |
| `deeptutor/services/config/provider_runtime.py` | 从 catalog 解析出 `ResolvedLLMConfig` / `ResolvedEmbeddingConfig` / `ResolvedSearchConfig` / TTS / STT / Imagegen / Videogen 配置 |
| `deeptutor/services/config/loader.py` | YAML 加载（`main.yaml`、`agents.yaml`），注入运行时路径，语言代码归一化 |
| `deeptutor/utils/config_manager.py` | 单例 `ConfigManager`，对 `main.yaml` 做增量 deep-update 与原子保存 |
| `deeptutor/config/settings.py` | `pydantic_settings` 的 `Settings`（当前用于 LLM retry 等进程级开关） |
| `deeptutor/config/schema.py` | 旧版 `AppConfig` Pydantic 模型（保留兼容） |
| `deeptutor/config/accessors.py` | 面向 YAML 配置的轻量访问器（`llm_model`、`llm_provider`、`user_data_dir`） |
| `deeptutor_cli/config_cmd.py` | `deeptutor config show` 命令，汇总 system / llm / embedding / search / language / tools |
| `deeptutor_cli/init_wizard.py` | `deeptutor init` 向导：provider 菜单、API Key 捕获、模型列表拉取、连通探针 |
| `.env.example` | 宿主机端口与环境时区模板 |

## 3. 架构与设计约定

- **配置分层顺序**（高优先级在前）：进程环境变量 → JSON 运行时设置 → YAML 用户配置 → 内置默认值。`RuntimeSettingsService._apply_*_process_overrides` 显式列出可被 env 覆盖的键，避免任意键污染。
- **原子写入**：所有 JSON/YAML 保存都先写 `tempfile.mkstemp` 再 `os.replace`，保证崩溃不损坏配置文件。
- **向后兼容迁移**：`RuntimeSettingsService._migrate_legacy_document_parsing_file` 把旧 `mineru.json` 重命名为 `document_parsing.json`；`_normalize_document_parsing` 把 v1 扁平结构迁移为 v2 的 `engines.*` 形式；`model_catalog._normalize` 为缺失 profile/model 补全 id/name/default。
- **Provider 自动推断**：`provider_runtime._choose_resolved_provider` 按 hint → gateway 检测 → model 匹配 → 本地 base_url 启发式 → 有 api_key/is_local 的 pool 顺序选择 provider，使旧配置仍可工作。
- **运行时路径注入**：`loader._inject_runtime_paths` 在读取 YAML 时动态填充 `paths.user_data_dir`、`paths.knowledge_bases_dir`、`paths.user_log_dir` 等，使 YAML 无需硬编码绝对路径。
- **多用户隔离**：`get_model_catalog_service()` 在非 admin 场景下通过 `multi_user.paths.get_admin_path_service().get_settings_file("model_catalog")` 切换到用户专属 settings 目录。
- **启动时引导**：`ensure_runtime_settings_files()` 调用 `ensure_defaults()` 创建缺失 JSON 文件，随后加载 catalog，是应用启动的统一入口。

## 4. 约定与约束

- **配置存储位置固定**：所有持久化配置必须位于 `<runtime-home>/data/user/settings/`，由 `get_runtime_settings_dir()` 强制约束。
- **JSON 设置文件命名规范**：每个子系统一个独立 JSON 文件（`system.json`、`auth.json`、`integrations.json`、`document_parsing.json`、`llamaindex.json`、`graphrag.json`、`lightrag.json`、`pageindex.json`、`model_catalog.json`），文件名即 key。
- **版本化**：每个 JSON 设置包含 `version` 字段，便于未来升级时识别格式。
- **环境变量白名单**：只有 `_apply_*_process_overrides` 中列出的键会被进程环境变量覆盖，其余 JSON 设置不受 env 影响；`IGNORE_PROCESS_OVERRIDES_ENV = "DEEPTUTOR_IGNORE_PROCESS_ENV_OVERRIDES"` 可全局关闭 env 覆盖。
- **敏感信息脱敏**：`config_cmd.show` 输出 API Key 时显示 `***`；`init_wizard.capture_api_key` 使用 `_mask_secret` 显示前 4 + 后 4 字符；`probe_embedding` 用 `redact_embedding_endpoint_for_display` 脱敏 URL。
- **端口范围约束**：`_coerce_port` 限制在 1–65535；附件大小上限通过 `_coerce_clamped_int` 钳制到 `(1, 1024)` MB 等范围。
- **Chat 附件策略实时生效**：`get_chat_attachment_limits()` 每次调用重新读取 system.json，因此修改后下一条消息即生效，无需重启。
- **Web 前端 API Base 解析**：`render_environment` 生成 `DEEPTUTOR_API_BASE_URL`，其值为 `next_public_api_base` → `next_public_api_base_external` → `http://127.0.0.1:{backend_port}` 的三级回退，且明确使用 IPv4 环回地址以避免双栈主机上 `localhost` 解析到 `::1` 导致连接失败。
- **agents.yaml 为必填**：`get_agent_params` 在未找到 `agents.yaml` 时抛出 `FileNotFoundError`，因为它是 agent temperature/max_tokens 的唯一来源。
- **Pydantic Settings 前缀**：`settings.py` 中的 `Settings` 类使用 `env_prefix="LLM_"` 与 `env_nested_delimiter="__"`，因此 `LLM_RETRY__MAX_RETRIES=5` 会映射到 `settings.retry.max_retries`。

## 5. 适用性判断

本仓库存在完整、成体系的配置系统，涵盖运行时目录解析、JSON/YAML 持久化、pydantic-settings 环境变量映射、模型目录管理、CLI 初始化向导、启动引导与迁移逻辑，属于 **high** 置信度的配置系统实现。