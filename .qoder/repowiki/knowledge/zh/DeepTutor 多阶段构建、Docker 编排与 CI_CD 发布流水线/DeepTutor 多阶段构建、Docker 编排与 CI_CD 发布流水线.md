---
kind: build_system
name: DeepTutor 多阶段构建、Docker 编排与 CI/CD 发布流水线
category: build_system
scope:
    - '**'
source_files:
    - pyproject.toml
    - Dockerfile
    - Dockerfile.runner
    - compose.yaml
    - requirements.txt
    - requirements/server.txt
    - requirements/partners.txt
    - packaging/deeptutor-cli/pyproject.toml
    - .github/workflows/tests.yml
    - .github/workflows/docker-release.yml
    - .github/workflows/pypi-release.yml
    - deeptutor/__version__.py
    - web/package.json
    - web/next.config.js
---

## 1. 使用的构建系统与工具

- **Python 包管理**：基于 `pyproject.toml` + setuptools（`setuptools.build_meta`），通过 `[project.optional-dependencies]` 提供 `cli`、`server`、`partners`、`matrix`、`math-animator`、`parse-*`、`graphrag`、`rag-lightrag`、`dev`、`all` 等可选依赖集；版本从 `deeptutor/__version__.py` 动态读取。
- **前端构建**：Next.js（Node 22）在 Docker 的 `frontend-builder` 阶段以 standalone 模式构建，产物 `.next/standalone/` 被复制到最终镜像，实现运行时环境变量注入。
- **容器化**：单仓库 `Dockerfile` 采用四阶段构建（`frontend-builder` → `node-runtime` → `python-base` → `production`，并衍生 `development`），使用 supervisord 同时启动 FastAPI 后端（uvicorn :8001）和 Next.js 前端（:3782）；另有一个独立的 `Dockerfile.runner` 作为最小化的沙箱执行 sidecar（端口 8900）。
- **编排**：`compose.yaml`（Podman Compose 兼容）定义 `deeptutor` 与可选 `pocketbase` 两个服务，默认只监听 127.0.0.1，数据通过 bind mount `./data:/app/data:U` 持久化。
- **CI/CD**：GitHub Actions 三个工作流——`tests.yml`（lint / web-node-tests / import-check / pytest）、`docker-release.yml`（触发 release 后构建 multi-platform linux/amd64+arm64 镜像推送到 GHCR `ghcr.io/hkuds/deeptutor`）、`pypi-release.yml`（仅发布 `deeptutor` 全量 wheel，CLI-only 包不发布到 PyPI）。
- **代码质量**：Ruff（lint+format，line-length=100，target-version py311）、Black（line-length=100，target py311–py314）、MyPy（relaxed）、Bandit（大量规则跳过）。pre-commit 由 `.pre-commit-config.yaml` 驱动。

## 2. 关键文件与位置

| 类别 | 关键文件 |
|---|---|
| Python 包配置 | `pyproject.toml`、`requirements.txt`、`requirements/*.txt` |
| CLI 独立包 | `packaging/deeptutor-cli/pyproject.toml` |
| 主镜像 | `Dockerfile`（含 production/development target）、`Dockerfile.runner` |
| 编排 | `compose.yaml`、`docker-compose.dev.yml`、`docker-compose.ghcr.yml`、`compose.codex-oauth.yaml` |
| 入口脚本 | `scripts/start_backend.sh`、`scripts/start_frontend.sh`、`scripts/entrypoint.sh`（内嵌于 Dockerfile） |
| 版本来源 | `deeptutor/__version__.py` |
| CI | `.github/workflows/tests.yml`、`.github/workflows/docker-release.yml`、`.github/workflows/pypi-release.yml` |
| 前端构建 | `web/package.json`、`web/next.config.js`、`web/.env.local` |
| 测试 | `tests/`（pytest 根目录 + `deeptutor/learning/tests`） |

## 3. 架构与约定

### 3.1 多阶段 Docker 构建
- **前端构建隔离**：`frontend-builder` 阶段固定为 `$BUILDPLATFORM`（避免 QEMU 交叉编译 JS），产出平台无关静态资源；`node-runtime` 阶段按目标平台拉取匹配的 Node 二进制，再拷贝进最终镜像。
- **Python 依赖分层**：`python-base` 安装系统库（OpenCV/Rust 编译依赖）与 pip 包；`production` 仅复制 site-packages，体积最小化。
- **supervisord 双程序**：`programs.conf` 分别定义 `backend`（uvicorn）和 `frontend`（Next.js standalone），均 drop 到非 root 用户 `deeptutor`（UID 1000）；开发 image 覆盖该文件启用 `--reload` 与 `next dev`。
- **运行时配置集中化**：`entrypoint.sh` 启动时调用 `export_runtime_settings_to_env`，从 `data/user/settings/*.json` 导出所有运行参数（端口、CORS、认证、模型目录等），容器内不再依赖 compose env var 注入。

### 3.2 版本与发布策略
- **单一版本源**：`deeptutor/__version__.py` 是 `pyproject.toml` 的 `dynamic version` 唯一来源；PyPI 发布前 CI 校验 Git tag（如 `v1.5.10`）与该文件一致，不一致则拒绝发布。
- **标签→镜像映射**：`docker-release.yml` 用 `docker/metadata-action` 将 release tag 转为镜像 tag（去掉 `v` 前缀）并附带 `latest`。
- **制品分发**：PyPI 仅发布全量 `deeptutor` wheel（包含打包好的 Next.js standalone 资产）；CLI-only 包 `deeptutor-cli` 仅在本地通过 `packaging/deeptutor-cli` 安装，不上传 PyPI。

### 3.3 沙箱执行隔离
- 主镜像默认启用受限 subprocess 后端；生产部署可通过 `DEEPTUTOR_SANDBOX_RUNNER_URL` 指向独立 `Dockerfile.runner` 构建的 sidecar，该镜像仅包含 stdlib HTTP server + 常用 CLI 工具（git/curl/jq/ripgrep/nodejs/gcc 等），无 DeepTutor 包，攻击面极小。

### 3.4 依赖管理约定
- `requirements.txt` 仅引用 `-r requirements/partners.txt`，注释说明“single source of truth”是 `pyproject.toml` 的 optional-dependencies；各 `requirements/*.txt` 用于 CI/Docker 在无源码环境下安装依赖。
- 对已知冲突的包显式 pin（如 `pdfplumber<0.11.8` 避免与 mineru 冲突、`faiss-cpu>=1.8.0,<2.0.0`、`llama-index-retrievers-bm25>=0.7.1,<0.8.0`）。
- Python 版本约束 `>=3.11,<3.14`，并在 graphrag/rag-lightrag 等重型 extra 上附加 `python_version < '3.14'` marker 防止 pip 静默回退。

### 3.5 CI 流水线约定
- `tests.yml` 并行执行 lint（ruff check/format）、web node tests（npm run test:node）、import check（3.11/312/313 稳定，3.14 experimental）、pytest（矩阵化 Python 版本）；任一失败则整体失败。
- 缓存策略：pip cache 基于 `requirements/server.txt` 等 hash，npm cache 基于 `web/package-lock.json`。
- 测试环境通过创建最小 `data/user/settings/main.yaml` 满足运行时初始化要求。

## 4. 约定与约束（描述性 + 可验证）

- **版本必须同步**：Git tag 与 `deeptutor/__version__.py` 必须一致，否则 `pypi-release.yml` 直接 exit 1（CI 强制）。
- **发布分支限制**：PyPI 发布前会校验 tag commit 是否为 `origin/main` 的祖先，非 main 分支 tag 拒绝发布。
- **Python 版本上限**：`requires-python = ">=3.11,<3.14"` 在 `pyproject.toml` 中显式声明，原因是 3.14 下部分编译依赖（faiss-cpu 等）无 wheel 且源码构建失败。
- **前端 URL 不硬编码**：Next.js 构建期不注入 API base，运行时由 `web/proxy.ts` 读取 `DEEPTUTOR_API_BASE_URL`（由 entrypoint 从 JSON 设置导出），Compose/Docker 也不通过 env var 注入 API base。
- **容器安全基线**：生产镜像以非 root 用户 `deeptutor` 运行；`compose.yaml` 默认 `read_only: true` + tmpfs 白名单；主机端口绑定限定 `127.0.0.1:`。
- **可选能力通过 extras 控制**：`partners`、`matrix`、`math-animator`、`graphrag`、`rag-lightrag` 等重型功能需显式 `pip install -e .[xxx]` 或 `pip install deeptutor[xxx]`，不会随基础包引入。
- **测试路径约定**：pytest 扫描 `tests` 与 `deeptutor/learning/tests`，async 测试需用 `@pytest.mark.asyncio` 标记，fixture loop scope 设为 function。
- **Lint/Format 统一**：Ruff 与 Black 共用 line-length=100；CI 对 ruff check 与 ruff format --check 都执行，格式不符即失败。