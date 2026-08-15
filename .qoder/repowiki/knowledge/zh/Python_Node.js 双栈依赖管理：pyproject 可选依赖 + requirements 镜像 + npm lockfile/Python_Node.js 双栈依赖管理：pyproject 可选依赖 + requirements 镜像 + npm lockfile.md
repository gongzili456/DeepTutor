---
kind: dependency_management
name: Python/Node.js 双栈依赖管理：pyproject 可选依赖 + requirements 镜像 + npm lockfile
category: dependency_management
scope:
    - '**'
source_files:
    - pyproject.toml
    - requirements.txt
    - requirements/server.txt
    - requirements/partners.txt
    - requirements/cli.txt
    - requirements/dev.txt
    - requirements/math-animator.txt
    - requirements/matrix.txt
    - requirements/matrix-e2e.txt
    - packaging/deeptutor-cli/pyproject.toml
    - web/package.json
    - web/package-lock.json
---

## 1. 使用的系统与方法

仓库采用 **Python 与 Node.js 两套独立** 的依赖管理系统，并通过“单一事实来源 + 多份镜像”的方式保持一致性。

- Python 侧使用 `setuptools`（`build-backend = setuptools.build_meta`）+ `pyproject.toml` 声明依赖，通过 `[project]`、`[project.optional-dependencies]` 划分核心运行时与可选能力；CLI 单独发布为 `deeptutor-cli` 包（`packaging/deeptutor-cli/pyproject.toml`），复用同一份源码但排除 Web 资产。
- Node.js 前端位于 `web/`，使用 `package.json` + `package-lock.json`（lockfileVersion 3）锁定版本。
- 构建/部署脚本通过 Dockerfile / compose.yaml 调用这些清单进行安装。

## 2. 关键文件

| 文件 | 作用 |
|---|---|
| `pyproject.toml` | 主包 `deeptutor` 的依赖声明、可选 extra（`cli`/`server`/`partners`/`matrix`/`matrix-e2e`/`math-animator`/`graphrag`/`rag-lightrag`/`parse-*`/`dev`/`all`）、构建配置、Ruff/Black/pytest/mypy/bandit 工具链 |
| `requirements.txt` | 顶层入口，仅 `-r requirements/partners.txt`，注释中明确其为“single source of truth: pyproject.toml optional-dependencies”的镜像 |
| `requirements/server.txt`、`requirements/cli.txt`、`requirements/partners.txt`、`requirements/dev.txt`、`requirements/math-animator.txt`、`requirements/matrix*.txt` | 按用途切分的 requirements 子集，每个文件头部注释写明“Mirrors pyproject.toml … Keep in sync with pyproject.toml when adding/updating dependencies”，供 Docker/CI 在尚未拿到源码时直接 `pip install -r` |
| `packaging/deeptutor-cli/pyproject.toml` | CLI-only 分发包的元数据，依赖列表与主包一致但不包含 server/web 相关依赖 |
| `web/package.json` | Next.js 前端依赖（Next 16、React 19、Tailwind、Mermaid 等） |
| `web/package-lock.json` | 前端依赖锁定文件 |
| `Dockerfile`、`compose*.yaml` | 容器化安装入口，引用上述清单 |

## 3. 架构与约定

### 3.1 Python 依赖分层（可选 extra 体系）

`pyproject.toml` 将依赖拆分为“核心运行时 + 可选能力”：
- 核心 `dependencies` 包含 LLM SDK（openai、anthropic、dashscope、perplexityai）、RAG（llama-index 系列）、向量库（faiss-cpu）、文档解析（PyMuPDF、python-docx/openpyxl/python-pptx/pypdf/pdfplumber/reportlab）、Web 框架（fastapi、uvicorn、websockets）、鉴权（bcrypt、python-jose、pocketbase）、日志（loguru）、JSON 修复（json-repair）等。
- 可选 extra 按场景组合：`server`（API 服务）、`partners`（IM 渠道 SDK：Telegram、飞书、钉钉、Slack、企业微信、QQ、Zulip、Matrix、MCP 客户端等）、`matrix`/`matrix-e2e`、`math-animator`（Manim）、`parse-*`（markitdown/docling/pymupdf4llm/liteparse）、`dev`、`codebuddy`、`graphrag`、`rag-lightrag`、`all`。
- 重型或易冲突的依赖被刻意放在 extra 中，并在代码中以懒导入 + fallback 方式处理（例如 FAISS 缺失回退到 SimpleVectorStore；psutil 缺失走 `/proc` 路径；graphrag/rag-lightrag 带 `python_version < '3.14'` marker 避免在 Python 3.14 上触发不可满足依赖导致的静默降级）。

### 3.2 版本约束策略

- 普遍使用 `>=X.Y, <Z.0` 的半开区间，允许小版本更新但锁死大版本以防 API 破坏。
- 对已知冲突的包显式上限：如 `pdfplumber>=0.11.0,<0.11.8`（注释说明 0.11.8+ 会 pin pdfminer.six==20251230，与 mineru 冲突）、`faiss-cpu>=1.8.0,<2.0.0`、`llama-index-vector-stores-faiss>=0.4.0,<1.0.0`、`croniter>=6.0.0,<7.0.0`、`mcp>=1.26.0,<2.0.0` 等。
- Python 版本统一限制为 `>=3.11,<3.14`，原因注释指出 3.14 下 pip 无法找到某些编译依赖（如 faiss-cpu）的 wheel，会回退到失败的 source build；因此用 `requires-python` 给出可读的错误提示而非静默降级。
- 对第三方 SDK 的 minor 变更也做保护：如 `codebuddy-agent-sdk>=0.1.24,<0.2`，注释说明 provider 探针会读取其 turn-limit/permission-mode kwargs，minor 升级重命名会导致选项构建失败。

### 3.3 单一事实来源 + 镜像文件

`requirements.txt` 顶部注释明确：`Single source of truth: pyproject.toml [project.optional-dependencies]. Files under requirements/ mirror those extras for Docker/CI installs that don't yet have access to pyproject.toml + source code.` 各 `requirements/*.txt` 都带有相同注释并要求与 `pyproject.toml` 同步。这意味着新增依赖应优先改 `pyproject.toml`，再同步到对应 `requirements/*.txt`。

### 3.4 前端依赖

`web/package.json` 声明生产依赖（Next 16、React 19、framer-motion、mermaid、i18next、chart.js 等）与开发依赖（Playwright、eslint、tailwindcss、typescript），`web/package-lock.json` 锁定完整树。未检出私有 npm registry 或 `.npmrc` 配置，默认使用 npm 官方源。

### 3.5 无 vendoring / 私有注册表

仓库中未发现 `vendor/`、`third_party/` 等 vendored 目录，也未发现 `.npmrc`、`pip.conf`、`setup.cfg` 中的 `index-url`/`extra-index-url`、环境变量 `PIP_INDEX_URL`/`PIP_EXTRA_INDEX_URL` 等私有源配置。所有依赖均从 PyPI/npm 官方源拉取。

## 4. 约定与约束

- **Python 依赖必须通过 `pyproject.toml` 的 `[project]` 或 `[project.optional-dependencies]` 声明**，不得在源码中硬编码版本号；`requirements/*.txt` 仅为镜像，新增依赖需同时维护两者。
- **可选功能以 extra 形式提供**，并通过懒导入 + fallback 保证缺失时不阻塞主流程（FAISS、psutil、graphrag、rag-lightrag、parse-* 引擎等）。
- **禁止引入 Python 3.14**：`requires-python = ">=3.11,<3.14"` 在两个 `pyproject.toml` 中一致生效。
- **大版本变更需显式放宽范围并评估兼容性**：几乎所有依赖都带 `<X.0` 上限，新增依赖也应遵循此模式。
- **前端依赖通过 `web/package.json` + `web/package-lock.json` 管理**，不在根目录声明。
- **无 vendoring、无私有 PyPI/npm registry 配置**——依赖全部来自公共源。
- **安全扫描集成**：`dev` extra 包含 `safety<3.0.0` 与 `bandit>=1.8.0`，`pyproject.toml` 中 bandit 的 `skips` 列表记录了已知的误报豁免（如 B104 绑定 0.0.0.0、B301 pickle 用于内部缓存等）。
- **构建产物打包规则**：`tool.setuptools.package-data` 将 `**/*.yaml/.yml/.md/.json/.sh/.j2/.jinja` 等运行时资源打入 wheel，确保 `pip install deeptutor` 后 prompt YAML、SKILL.md、shell 脚本可用；CLI-only 包通过 `exclude = ["deeptutor_web*"]` 排除前端资源。
