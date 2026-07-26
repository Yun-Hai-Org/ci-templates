# 测试执行详解

ci-templates 的 **T 类**检查负责执行测试，由 `tests.yml` workflow 实现，被 `standard-ci.yml` 调用。

## 三种测试类型

| 类型     | job 名              | 默认  | 跳过条件                     | 严格模式            |
| -------- | ------------------- | ----- | ---------------------------- | ------------------- |
| 单元测试 | `unit-tests`        | ✅ 开 | 无 manifest                  | 有代码无测试 → fail |
| 集成测试 | `integration-tests` | ✅ 开 | 无 `tests/integration/` 目录 | 否                  |
| E2E 测试 | `e2e-tests`         | ✅ 开 | 无 Playwright/Cypress 配置   | 否                  |

## 严格测试要求（单元测试）

单元测试 job 启用 **严格模式**：当项目存在 manifest 文件但未发现任何测试文件时，CI **失败**。

### 各语言的 manifest 与测试文件检测

| project-type | manifest 文件                                      | 测试文件检测                                                            |
| ------------ | -------------------------------------------------- | ----------------------------------------------------------------------- |
| `bun`        | `package.json`                                     | `tests/` / `test/` / `__tests__/` 目录，或 `*.test.*` / `*.spec.*` 文件 |
| `python`     | `pyproject.toml` / `setup.py` / `requirements.txt` | `test_*.py` / `*_test.py` / `conftest.py` 文件                          |
| `rust`       | `Cargo.toml`                                       | `tests/*.rs` 文件，或 `src/` 中含 `#[test]`                             |
| `node`       | `package.json`（且无 `bun.lockb`）                 | `tests/` / `test/` / `__tests__/` 目录，或 `*.test.*` / `*.spec.*` 文件 |

### 行为说明

- **有 manifest + 有测试文件** → 正常执行测试
- **有 manifest + 无测试文件** → `::error::` 并 `exit 1`（CI 失败）
- **无 manifest** → `::notice::` 跳过（非代码项目）

## 各语言测试命令

### Bun

| 测试类型 | 默认命令                                    | 可覆盖 input                   |
| -------- | ------------------------------------------- | ------------------------------ |
| 单元     | bun test                                    | `test-command-bun-unit`        |
| 集成     | bun test tests/integration                  | `test-command-bun-integration` |
| E2E      | `bunx playwright test` / `bunx cypress run` | `e2e-framework`                |

### Python

| 测试类型 | 默认命令                           | 可覆盖 input                      |
| -------- | ---------------------------------- | --------------------------------- |
| 单元     | `uv run pytest`                    | `test-command-python-unit`        |
| 集成     | `uv run pytest tests/integration/` | `test-command-python-integration` |
| E2E      | `uv run pytest tests/e2e/`         | —                                 |

### Rust

| 测试类型 | 默认命令                        | 可覆盖 input |
| -------- | ------------------------------- | ------------ |
| 单元     | cargo test --lib                | —            |
| 集成     | cargo test --test '\*'          | —            |
| E2E      | 不支持（用 load-test workflow） | —            |

### Node

| 测试类型 | 默认命令                                  | 可覆盖 input                    |
| -------- | ----------------------------------------- | ------------------------------- |
| 单元     | npm test                                  | `test-command-node-unit`        |
| 集成     | npm test -- tests/integration             | `test-command-node-integration` |
| E2E      | `npx playwright test` / `npx cypress run` | `e2e-framework`                 |

Node 支持 `package-manager` input（`npm` / `yarn` / `pnpm`），安装命令自动适配。

## 集成测试

集成测试 job 检查 `tests/integration/` 目录是否存在：

- **存在** → 安装依赖，执行集成测试命令
- **不存在** → `::notice::` 跳过

集成测试需要 `TEST_DATABASE_URL` secret。未配置时输出 `::warning::` 并跳过（不失败）。

## E2E 测试

E2E 测试 job 检查以下配置文件是否存在：

- `playwright.config.ts` / `playwright.config.js`
- `cypress.config.ts` / `cypress.config.js`
- `tests/e2e/` 目录

任一存在即触发 E2E。需要 `E2E_TARGET_URL` secret（未配置时 `::warning::` 跳过）。可选 `E2E_AUTH_TOKEN` 用于认证。

`e2e-framework` input 选择 `playwright`（默认）或 `cypress`。

## Secrets

| Secret              | 必填 | 用途                 |
| ------------------- | ---- | -------------------- |
| `TEST_DATABASE_URL` | 否   | 集成测试数据库连接串 |
| `E2E_TARGET_URL`    | 否   | E2E 测试目标 URL     |
| `E2E_AUTH_TOKEN`    | 否   | E2E 测试认证 token   |

所有 secret 通过 `secrets: inherit` 自动透传，未配置时自动跳过对应测试。

## 自定义测试命令

通过 `test-command-*` inputs 可覆盖默认测试命令：

```yaml
with:
  test-command-bun-unit: 'bun test --coverage'
  test-command-python-unit: 'uv run pytest -x --cov'
  test-command-node-unit: 'npm test -- --coverage'
```
