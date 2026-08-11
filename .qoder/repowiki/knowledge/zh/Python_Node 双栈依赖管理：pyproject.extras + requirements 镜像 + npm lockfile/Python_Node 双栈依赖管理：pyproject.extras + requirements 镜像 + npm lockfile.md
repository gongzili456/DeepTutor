---
kind: dependency_management
name: Python/Node 双栈依赖管理：pyproject.extras + requirements 镜像 + npm lockfile
category: dependency_management
scope:
    - '**'
source_files:
    - pyproject.toml
    - requirements.txt
    - requirements/cli.txt
    - requirements/server.txt
    - requirements/partners.txt
    - requirements/matrix.txt
    - requirements/matrix-e2e.txt
    - requirements/math-animator.txt
    - requirements/dev.txt
    - web/package.json
    - web/package-lock.json
    - Dockerfile
    - Dockerfile.runner
    - .github/workflows/tests.yml
    - .github/workflows/docker-release.yml
    - .github/workflows/pypi-release.yml
    - packaging/deeptutor-cli/pyproject.toml
---

## 1. 使用的系统与工具

- **Python 侧**：以 `pyproject.toml`（PEP 621）为唯一权威源，使用 `setuptools.build_meta` 构建；通过 `[project]` 的 `dependencies` 声明核心运行时，并通过 `[project.optional-dependencies]` 将可选能力拆分为 `cli`、`server`、`partners`、`matrix`、`matrix-e2e`、`math-animator`、`graphrag`、`rag-lightrag`、`parse-*`、`dev`、`all` 等 extra。
- **requirements 镜像层**：根目录 `requirements.txt` 与 `requirements/*.txt`（`cli.txt`、`server.txt`、`partners.txt`、`matrix.txt`、`matrix-e2e.txt`、`math-animator.txt`、`dev.txt`）是对 `pyproject.toml` optional-dependencies 的逐行镜像，注释明确“Keep in sync with pyproject.toml”。这些文件是 Docker / CI 在尚未能访问源码时安装依赖的唯一入口（见 `Dockerfile` 中 `COPY requirements/ ./requirements/ && pip install -r requirements.txt`）。
- **前端（Next.js）**：位于 `web/`，使用 `package.json` 声明依赖，并通过 `package-lock.json`（lockfileVersion 3）锁定精确版本。脚本通过 `npm run dev/build/start` 调用 Next 内置命令，无自定义包管理器封装。
- **打包分发**：`packaging/deeptutor-cli/pyproject.toml` 提供独立的 CLI-only wheel 发布配置；主仓库通过 `.github/workflows/pypi-release.yml` 发布 PyPI 包，`docker-release.yml` 构建 Docker 镜像。

## 2. 关键文件

- `pyproject.toml`：定义 Python 包名 `deeptutor`、`requires-python = ">=3.11,<3.14"`、全部 core dependencies、所有 optional-dependencies、setuptools package-data（prompt YAML/模板随包分发）、Black/Ruff/pytest/mypy/bandit 工具配置。
- `requirements.txt` + `requirements/{cli,server,partners,matrix,matrix-e2e,math-animator,dev}.txt`：requirements 镜像层，被 Dockerfile 直接消费。
- `web/package.json` + `web/package-lock.json`：前端依赖声明与锁定。
- `Dockerfile` / `Dockerfile.runner`：容器化安装路径，优先使用 `requirements.txt` 而非 `pip install .`。
- `.github/workflows/{tests.yml,docker-release.yml,pypi-release.yml}`：CI 中触发测试、构建镜像与发布 PyPI。
- `packaging/deeptutor-cli/pyproject.toml`：CLI 独立 wheel 的发布元数据。

## 3. 架构与约定

### 3.1 分层可选依赖（Extras 体系）
项目把可选能力按运行场景切分，形成清晰的依赖树：
- `cli`：LLM SDK（openai/anthropic/dashscope/perplexity）、RAG（llama-index + faiss）、文档解析（docx/xlsx/pptx/pdf/reportlab/defusedxml）、搜索（ddgs）、CLI 框架（typer/rich/prompt_toolkit）。
- `server`：`deeptutor[cli]` + FastAPI/Uvicorn/websockets/python-multipart、认证（bcrypt/python-jose）、PocketBase 客户端、日志（loguru/json-repair）、croniter。
- `partners`：`deeptutor[server]` + MCP client + 各 IM SDK（telegram-bot/wecom/lark/dingtalk/slack/qq-botpy/zulip/socketio/msgpack/socks/websocket-client/aiohttp/PyJWT/qrcode）。
- `matrix` / `matrix-e2e`：Matrix 通道及其 E2EE 扩展（需要系统级 libolm）。
- `math-animator`：Manim（需系统 LaTeX/pkg-config/cairo/cmake/ffmpeg）。
- `graphrag` / `rag-lightrag`：重型 RAG 引擎，带 `python_version < '3.14'` marker 避免在 3.14+ 上静默回退。
- `parse-*`：可插拔文档解析引擎（markitdown/docling/pymupdf4llm），导入时懒加载，缺失即报告不可用。
- `dev`：pytest/pytest-asyncio/pre-commit/safety/bandit。
- `all`：`partners + matrix + math-animator + dev` 的聚合。

### 3.2 版本约束策略
- 核心库普遍采用 `>=X.Y, <Z` 形式（如 `fastapi>=0.100.0`、`uvicorn[standard]>=0.24.0`、`loguru>=0.7.3,<1.0.0`、`json-repair>=0.57.0,<1.0.0`），允许小版本演进但阻断破坏性升级。
- 对已知冲突的依赖显式封顶：`pdfplumber>=0.11.0,<0.11.8`（避免 pdfminer.six 冲突）、`faiss-cpu>=1.8.0,<2.0.0`、`numpy>=1.24.0,<3.0.0`、`llama-index-vector-stores-faiss>=0.4.0,<1.0.0` 等。
- Python 版本上限 `requires-python = ">=3.11,<3.14"`，并在 graphrag/rag-lightrag 上使用 `; python_version < '3.14'` marker，使 3.14+ 环境跳过不兼容的 heavy extras。
- 前端 `web/package.json` 使用 `^` 语义版本（如 `next ^16.2.3`、`react ^19.0.0`），由 `package-lock.json` 锁定实际安装版本。

### 3.3 多入口安装方式
- 公共 wheel：`pip install deeptutor`（含 server extras）或 `pip install deeptutor-cli`（仅 CLI）。
- 源码开发：`pip install -e .`、`pip install -e ".[partners]"`、`pip install -e ".[matrix]"`、`pip install -e ".[math-animator]"`、`pip install -e ".[dev]"`、`pip install -e ".[all]"`。
- Docker/CI：直接 `pip install -r requirements.txt`（顶层 requirements 指向 `requirements/partners.txt`）。

### 3.4 资源随包分发
`tool.setuptools.package-data` 将 `**/*.yaml|yml|md|json|sh|j2|jinja` 打入 wheel，确保 PromptManager 能在 `PACKAGE_ROOT/deeptutor/.../prompts/*.yaml` 找到提示词，否则首次使用会失败。

## 4. 约定与约束

- **单一权威源**：`pyproject.toml` 是 Python 依赖的唯一真实来源；`requirements/*.txt` 必须与其保持同步（每个文件头部注释明确要求 “Keep in sync with pyproject.toml when adding/updating dependencies”）。
- **Docker 不依赖源码**：`Dockerfile` 先 COPY `requirements/` 再 `pip install -r requirements.txt`，保证在无源码环境下也能构建镜像。
- **可选能力默认不装**：除 `server` 外，`partners`、`matrix`、`math-animator`、`graphrag`、`rag-lightrag`、`parse-*` 均作为 optional-dependencies，需显式选择安装，减小基础镜像体积。
- **重型/有系统依赖的能力隔离**：`math-animator` 要求系统级 LaTeX/pkg-config/cairo/cmake/ffmpeg；`matrix-e2e` 要求系统级 `libolm`；`graphrag`/`rag-lightrag` 通过 `python_version` marker 在 3.14+ 上静默跳过。
- **前端锁定**：`web/package-lock.json` 提交到仓库，确保全团队/CI 安装一致版本。
- **构建后端固定**：`build-system.requires = ["setuptools>=61.0", "wheel"]`，禁用 PEP 517 其他后端。
- **安全扫描集成**：`dev` extra 包含 `safety<3.0.0` 与 `bandit>=1.8.0`，配合 `pyproject.toml` 中 bandit skips 列表用于 CI 安全检查。
- **私有注册表/GOPRIVATE**：仓库未出现任何 `PIP_INDEX_URL`、`--extra-index-url`、`.netrc`、`pip.conf`、`GOFLAGS=-insecure` 或 `GOPRIVATE` 配置，表明当前依赖全部来自公开 PyPI/npm registry。

## 5. 适用性说明
本仓库同时存在 Python（pyproject + requirements 镜像）和 Node（package.json + package-lock.json）两套依赖管理系统，且通过 extras 机制实现可选能力的精细化控制，因此该类别完全适用。