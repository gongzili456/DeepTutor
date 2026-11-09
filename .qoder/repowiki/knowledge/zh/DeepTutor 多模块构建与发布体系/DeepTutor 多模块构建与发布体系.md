---
kind: build_system
name: DeepTutor 多模块构建与发布体系
category: build_system
scope:
    - '**'
source_files:
    - pyproject.toml
    - setup.py
    - build_metadata.py
    - build_support.py
    - scripts/build_modules.py
    - packaging/deeptutor-cli/pyproject.toml
    - Dockerfile
    - Dockerfile.runner
    - .github/workflows/tests.yml
    - .github/workflows/docker-release.yml
    - .github/workflows/pypi-release.yml
    - .github/workflows/repository-hygiene.yml
    - requirements/core.txt
    - requirements/full.txt
    - requirements/server.txt
    - requirements/partners.txt
    - requirements/video-learning.txt
    - requirements/matrix-e2e.txt
    - deeptutor/modules/manifests/core.json
---

## 1. 总体方案

DeepTutor 采用 **Python setuptools + Node.js/npm 双栈** 的混合构建，通过 GitHub Actions 驱动测试、Docker 镜像与 PyPI 包发布。核心思路是：

- 单一源码树（`deeptutor/`、`deeptutor_cli/`、`deeptutor_web/`）同时产出“核心 wheel”和若干“功能模块 wheel”，由 `scripts/build_modules.py` 按清单拆分。
- 版本来源唯一：`deeptutor/__version__.py`，由 `build_metadata.py` 用 AST 解析，禁止从 git tag 或环境变量推导。
- Docker 镜像使用多阶段构建，前端 Next.js 在 `frontend-builder` 阶段静态编译，后端 Python 依赖在 `python-base` 阶段安装，最终 `production` 镜像仅包含运行时产物。
- CI 矩阵覆盖 Python 3.11–3.14、Node 22、Windows/macOS/Linux，并通过 Redis 服务容器运行集成测试。

## 2. 关键文件与职责

| 文件 | 作用 |
|---|---|
| `pyproject.toml` | 项目元数据、可选依赖模板、Black/Ruff/pytest/mypy/bandit 配置；定义 `deeptutor` 入口脚本指向 `deeptutor_cli.main:main` |
| `setup.py` | 注册自定义 `CoreBuildPy` 命令类，将 `tool.deeptutor.optional-dependencies` 暴露为 `extras_require` |
| `build_metadata.py` | 纯标准库模块，从 AST 读取 `__version__`，展开可选依赖模板中的 `{version}` 占位符 |
| `build_support.py` | 基于 `deeptutor/modules/manifests/*.json` 计算每个源文件的“归属模块”，过滤根 wheel 只包含 core 文件 |
| `scripts/build_modules.py` | 遍历 manifests，为每个模块生成临时 `pyproject.toml`，调用 `pip wheel --no-deps` 产出独立 wheel |
| `packaging/deeptutor-cli/pyproject.toml` | 兼容包 `deeptutor-cli` 的元数据（空 packages，仅重定向到主包） |
| `Dockerfile` | 四阶段：`frontend-builder` → `node-runtime` → `python-base` → `production`（含 `development` 继承） |
| `Dockerfile.runner` | 沙箱执行器镜像（与主镜像分离） |
| `.github/workflows/tests.yml` | 测试矩阵：desktop shell、TypeScript chat-engine、Ruff lint、Web Playwright、import-check、Python tests、video-lesson sandbox |
| `.github/workflows/docker-release.yml` | Release 触发，QEMU + Buildx 构建 `linux/amd64, linux/arm64` 镜像推送到 `ghcr.io/hkuds/deeptutor` |
| `.github/workflows/pypi-release.yml` | Release 触发，验证 tag 与 `__version__.py` 一致，构建并上传所有模块 wheel |
| `.github/workflows/repository-hygiene.yml` | 检查受控生成文件 |
| `requirements/*.txt` | 分层依赖清单（core/cli/dev/full/server/partners/video-learning 等），对应 `pyproject.toml` 的可选依赖组 |
| `deeptutor/modules/manifests/*.json` | 每个模块的“所有权声明”（paths、distribution、requires、entry_points） |

## 3. 架构与约定

### 3.1 模块拆分与所有权

`deeptutor/modules/manifests/core.json` 定义 root wheel 的 `paths` 白名单；其他 manifest 定义各自代码路径。`build_support.CoreBuildPy.find_package_modules` / `find_data_files` 重写 setuptools 的文件发现，使普通 `pip install .` 只打包 core 文件。`scripts/build_modules.py` 再按同一份 manifest 把其余文件分发到 `deeptutor-audio-overview`、`deeptutor-server` 等独立 wheel。

规则：新增模块必须同时在 `deeptutor/modules/manifests/<name>.json` 中声明 `paths`、`distribution`、`requires`，否则该模块的代码不会被任何 wheel 包含。

### 3.2 版本策略

- 唯一来源：`deeptutor/__version__.py` 中的字符串字面量。
- `build_metadata.project_version()` 用 AST 解析，避免执行任意代码。
- `pyproject.toml` 中 `dynamic = ["version", "optional-dependencies"]`，由 `tool.setuptools.dynamic.version.attr = "deeptutor.__version__.__version__"` 注入。
- PyPI 发布流程在 `.github/workflows/pypi-release.yml` 中再次校验 tag（`vX.Y.Z`）与 `__version__.py` 一致，不一致直接失败。

### 3.3 可选依赖模板

`pyproject.toml` 的 `[tool.deeptutor.optional-dependencies]` 使用 `{version}` 占位符（如 `deeptutor-server=={version}`）。`build_metadata.optional_dependencies()` 在构建时替换为真实版本号，供 `setup.py` 的 `extras_require` 和 `build_modules.py` 的模块依赖列表共同消费。这保证了核心包与模块包的版本严格对齐。

### 3.4 Docker 多阶段构建

- Stage 1 (`frontend-builder`, `FROM node:22.23.4-slim`)：`npm ci --legacy-peer-deps` 后 `npm run build`，输出 Next.js standalone 模式产物。
- Stage 1b (`node-runtime`)：为目标平台拉取匹配的 Node 二进制，用于跨平台镜像。
- Stage 2 (`python-base`, `FROM python:3.11-slim`)：安装 Rust、系统依赖、`requirements.txt`。
- Stage 3 (`production`)：复制 Node 运行时、Python site-packages、Next.js standalone、应用源码；使用 supervisord 管理 backend (uvicorn) 和 frontend (node server.js) 两个进程；entrypoint 初始化数据目录、根据 PUID/PGID 重映射用户、加载 JSON 运行时设置并导出为环境变量。
- Stage 4 (`development`)：继承 production，拷贝完整 web 源码，启用 `--reload` 和 `next dev`。

镜像默认以非 root 用户 `deeptutor` (UID/GID 1000) 运行子进程；rootful 模式下 entrypoint 可将其 UID/GID 重映射到宿主机用户。

### 3.5 CI 流水线

`tests.yml` 并行运行以下 job（全部 fail-fast=false）：

- `desktop`：Ubuntu 24.04 + Windows latest，Electron shell 测试。
- `chat-engine`：TypeScript chat engine + Python 适配器双向测试。
- `lint`：Ruff check/format + import-linter + `scripts/check_architecture.py`。
- `web-tests`：Playwright Chromium 交互/无障碍/视觉审计。
- `multi-worker-web`：条件 job，需要 `DEEPTUTOR_MULTI_WORKER_E2E_URL` 变量。
- `import-check`：矩阵覆盖 Python 3.11/3.12/3.13/3.14（Linux/macOS/Windows），验证核心模块可导入且工具实现保持冷启动。
- `python-tests`：Redis 服务容器 + pytest，覆盖 `tests/` 与 `deeptutor/learning/tests/`。
- `video-lesson-sandbox`：跨三平台验证视频渲染隔离与取消。

`docker-release.yml` 仅在 release published 时触发，要求 tag 匹配 `^v(0|[1-9]\d*)\.(0|[1-9]\d*)\.(0|[1-9]\d*)(...)$`，构建 `linux/amd64, linux/arm64` 镜像，稳定版额外打 `latest`。

`pypi-release.yml` 要求 tag 在 origin/main 上，构建后先用 twine check，再上传；支持 `bootstrap` 模式自动等待 PyPI pending publisher 完成注册。

### 3.6 依赖分层

`requirements/` 下按功能域切分（`core.txt`、`cli.txt`、`server.txt`、`partners.txt`、`video-learning.txt`、`matrix-e2e.txt` 等），`full.txt` 聚合全部可选依赖。CI 对 import-check 和 python-tests 分别安装 `full.txt` 与 `partners.txt`，确保最小化依赖也能通过。

## 4. 约定与约束

- **版本单点**：版本号只能修改 `deeptutor/__version__.py`；PyPI 发布流程强制校验其与 git tag 一致（`.github/workflows/pypi-release.yml` 第 113–145 行）。
- **模块所有权**：所有源码文件必须能被 `deeptutor/modules/manifests/*.json` 中的 `paths` 前缀命中，否则会被归入 `core`；`build_support.owner()` 是判定逻辑的唯一来源。
- **可选依赖模板**：新增 feature 必须在 `pyproject.toml` 的 `[tool.deeptutor.optional-dependencies]` 中声明，并使用 `{version}` 占位符引用同版本模块包。
- **Python 版本范围**：`requires-python = ">=3.11,<3.15"`，CI 矩阵固定 3.11–3.14；部分依赖（graphrag、rag-rerank）通过 `; python_version < '3.14'` 标记排除 3.14。
- **Lint 规则**：Ruff 仅启用 E/F/I/B006，大量规则被显式 ignore；Black 行宽 100，目标 py311–py314。
- **Docker 安全**：生产镜像内 app 进程以非 root `deeptutor` 用户运行；entrypoint 拒绝 PUID/PGID=0；supervisord daemon 层不指定 `user=`，子进程通过 per-program `user=deeptutor` 降级。
- **前端构建**：Next.js 使用 standalone 模式，URL 不在构建期硬编码，运行时通过 `DEEPTUTOR_API_BASE_URL` 注入（`Dockerfile` 注释明确说明）。
- **CI 并发**：所有 workflow 使用 `concurrency.group` + `cancel-in-progress: true`，新 push 会取消同一 PR/branch 的旧任务。
- **Docker 发布**：仅接受以 `v` 开头的语义化版本 tag；预发布版本不会覆盖 `latest` 标签（`docker-release.yml` 第 93–95 行）。
- **模块 wheel 互斥**：`scripts/build_modules.py` 在构建完成后解压每个 wheel 并断言无重复路径，防止多个模块 wheel 包含同一文件（第 118–127 行）。
- **仓库卫生**：`scripts/check_workspace_hygiene.py` 在每次 push/PR 时被 `repository-hygiene.yml` 调用，检查受控生成文件是否已提交。