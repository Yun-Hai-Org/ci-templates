# 项目类型

v1 首版支持 `bun` 与 `python` 两种 project-type。

## 工具矩阵

| 检查项              | `bun` | `python` | 命令                                                      |
| ------------------- | ----- | -------- | --------------------------------------------------------- |
| type-check          | Y     | Y        | `bunx tsc --noEmit` / `uv run pyright .`                  |
| lint                | Y     | Y        | `bun eslint --max-warnings 0 .` / `uv run ruff check .`   |
| format              | Y     | Y        | `bun prettier --check .` / `uv run ruff format --check .` |
| extended-lint       | 可选  | 可选     | hadolint / shellcheck / stylelint / sqlfluff              |
| semgrep             | Y     | Y        | `semgrep ci --config auto`                                |
| gitleaks            | Y     | Y        | `gitleaks detect --source .`                              |
| trivy (fs + config) | Y     | Y        | `trivy fs --severity HIGH,CRITICAL .` + `trivy config .`  |
| knip                | Y     | N        | `bun knip`                                                |
| checkov             | 按需  | 按需     | 仅当仓库有 IaC 文件时触发                                 |
| conftest            | 按需  | 按需     | 仅当 `policy/` 目录存在时触发                             |
| dep-audit           | Y     | Y        | `bun audit --production` / `uv run pip-audit`             |
| lockfile            | Y     | Y        | `bun install --frozen-lockfile` / `uv sync --frozen`      |

## 项目类型命令矩阵

按 project-type 列出各检查阶段使用的命令。

| project-type | 说明                          | type-check          | lint                            | format                         | extended-lint | dep-audit                | lockfile/install                |
| ------------ | ----------------------------- | ------------------- | ------------------------------- | ------------------------------ | ------------- | ------------------------ | ------------------------------- |
| `bun`        | Bun 项目                      | `bunx tsc --noEmit` | `bun eslint --max-warnings 0 .` | `bun prettier --check .`       | 可选          | `bun audit --production` | `bun install --frozen-lockfile` |
| `python`     | Python 项目                   | `uv run pyright .`  | `uv run ruff check .`           | `uv run ruff format --check .` | 可选          | `uv run pip-audit`       | `uv sync --frozen`              |
| `rust`       | Rust 项目                     | `cargo check`       | `cargo clippy`                  | `cargo fmt`                    | —             | `cargo-audit`            | `cargo update`                  |
| `node`       | Node.js 项目（npm/yarn/pnpm） | `npx tsc`           | `npx eslint`                    | `npx prettier`                 | —             | `npm audit`              | `npm ci`                        |

## 如何选择

### bun

适用场景：

- JavaScript / TypeScript 前端项目
- Node.js 后端服务
- MCP Server（TypeScript + Bun 实现）
- Bun 运行时的 CLI 工具

要求仓库有 `bun.lock` 或 `bun.lockb`，以及 `package.json`。

### python

适用场景：

- Python API 服务（FastAPI / Flask / Django）
- 数据管道（Airflow / Dagster / Prefect）
- Python CLI 工具
- ML / 数据科学项目

要求仓库有 `uv.lock` 或 `requirements.txt`，以及 `pyproject.toml`。

## 前置文件要求

### bun 项目

```
.
├── package.json        # 必需
├── bun.lock            # 推荐（lockfile-freshness 检查需要）
├── tsconfig.json       # type-check 需要
├── .eslintrc*          # lint 需要
└── .prettierrc*        # format 需要（可选）
```

### python 项目

```
.
├── pyproject.toml      # 必需
├── uv.lock             # 推荐（lockfile-freshness 检查需要）
└── requirements.txt    # 或用 requirements.txt 代替 uv.lock
```

## extended-lint（两种类型通用）

`run-extended-lint: true` 时追加：

| 工具       | 触发条件                              | 检查对象        |
| ---------- | ------------------------------------- | --------------- |
| hadolint   | 仓库有 `**/Dockerfile`                | Dockerfile 语法 |
| shellcheck | 仓库有 `**/*.sh`                      | Shell 脚本      |
| stylelint  | 仓库有 `**/*.css` 且 project-type=bun | CSS 样式        |
| sqlfluff   | 仓库有 `**/*.sql`                     | SQL 语法        |

工具不存在对应文件时自动跳过，不会失败。

## 后续版本规划

v1 之后的版本计划支持：

- `go`：Go 微服务（go vet / gofmt / govulncheck）
- `fullstack`：多语言混合项目（同时跑 bun + python 检查）
- `mcp`：MCP Server 专用（bun 基础 + schema 校验）
- `data-engineering`：数据管道（python 基础 + SQL 重点检查）
- `minimal`：仅安全扫描（legacy / fork 项目）

如需提前使用某类型，可在业务仓库自行组合 `run-static-analysis` 等开关近似实现。
