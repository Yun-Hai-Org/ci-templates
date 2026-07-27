# Inputs 与 Secrets 参考

`standard-ci.yml` 通过 `workflow_call` 暴露以下接口。业务项目在 `with:` 和 `secrets:` 下传递。

## Inputs

### A/B/C 类（基础检查）

| input                             | 类型    | 默认值                             | 说明                                                                                                  |
| --------------------------------- | ------- | ---------------------------------- | ----------------------------------------------------------------------------------------------------- |
| `project-type`                    | string  | `bun`                              | `bun` 或 `python`（v1 仅这两种）                                                                      |
| `run-static-analysis`             | boolean | `true`                             | A 类总开关（type-check / lint / format）                                                              |
| `run-security-scan`               | boolean | `true`                             | B 类总开关（semgrep / gitleaks / trivy / knip / checkov / conftest / sonarqube / snyk / gitguardian） |
| `run-dependency-audit`            | boolean | `true`                             | C 类总开关（dep-audit / lockfile-freshness）                                                          |
| `run-extended-lint`               | boolean | `false`                            | hadolint / shellcheck / stylelint / sqlfluff                                                          |
| `run-knip`                        | boolean | `true`                             | JS 死代码检测（仅 bun 类型生效）                                                                      |
| `run-osv-scanner`                 | boolean | `false`                            | OSV 全量扫描（慢，建议定时任务用）                                                                    |
| `run-unit-tests`                  | boolean | `true`                             | T 类单元测试（bun test / pytest / cargo test / npm test）                                             |
| `run-integration-tests`           | boolean | `true`                             | T 类集成测试（无 tests/integration/ 跳过）                                                            |
| `run-e2e-tests`                   | boolean | `true`                             | T 类 E2E 测试（Playwright/Cypress）                                                                   |
| `test-command-bun-unit`           | string  | `bun test`                         | Bun 单元测试命令                                                                                      |
| `test-command-bun-integration`    | string  | `bun test tests/integration`       | Bun 集成测试命令                                                                                      |
| `test-command-python-unit`        | string  | `uv run pytest`                    | Python 单元测试命令                                                                                   |
| `test-command-python-integration` | string  | `uv run pytest tests/integration/` | Python 集成测试命令                                                                                   |
| `test-command-node-unit`          | string  | `npm test`                         | Node 单元测试命令                                                                                     |
| `test-command-node-integration`   | string  | `npm test -- tests/integration`    | Node 集成测试命令                                                                                     |
| `e2e-framework`                   | string  | `playwright`                       | E2E 框架：playwright 或 cypress                                                                       |
| `rust-version`                    | string  | `stable`                           | Rust 工具链版本                                                                                       |
| `node-version`                    | string  | `lts/*`                            | Node.js 版本                                                                                          |
| `package-manager`                 | string  | `npm`                              | 包管理器：npm/yarn/pnpm（node only）                                                                  |
| `fail-on-severity`                | string  | `high`                             | `none` / `low` / `medium` / `high` / `critical`                                                       |
| `bun-version`                     | string  | `latest`                           | setup-bun 安装的 Bun 版本                                                                             |
| `python-version`                  | string  | `3.12`                             | setup-python 安装的 Python 版本                                                                       |
| `working-directory`               | string  | `.`                                | 工作子目录（monorepo 支持）                                                                           |
| `wecom-notify`                    | boolean | `true`                             | 是否发送企业微信 CI 结果通知                                                                          |
| `debug`                           | boolean | `false`                            | 开启 debug 输出（打印 inputs 等）                                                                     |

### D 类：上线前卡点（release-gates）

详见 [Release Gates](release-gates.md)。

| input                | 类型    | 默认值                               | 说明                                              |
| -------------------- | ------- | ------------------------------------ | ------------------------------------------------- |
| `run-release-gates`  | boolean | `false`                              | D 类总开关                                        |
| `jira-prefix`        | string  | `""`                                 | Jira 项目前缀（如 `PROJ`）。空则只校验格式        |
| `jira-warning-only`  | boolean | `false`                              | true 时 Jira 校验失败仅 warning（过渡期）         |
| `schema-check-paths` | string  | `""`                                 | Agent 配置文件 glob（如 `agents/*.json`）。空跳过 |
| `schema-file`        | string  | `templates/agent-config.schema.json` | JSON Schema 文件路径                              |

### E 类：上线后验证

详见 [AI 内容测试](ai-content-testing.md) / [压测](load-testing.md) / [DB 基准](db-benchmark.md)。

| input                      | 类型    | 默认值                     | 说明                      |
| -------------------------- | ------- | -------------------------- | ------------------------- |
| `run-ai-content-test`      | boolean | `false`                    | promptfoo AI 内容安全测试 |
| `run-load-test`            | boolean | `true`                     | k6 / Locust 压测          |
| `run-db-benchmark`         | boolean | `true`                     | pgbench DB 基准           |
| `promptfoo-test-path`      | string  | `./tests/promptfoo`        | promptfoo 测试用例目录    |
| `promptfoo-fail-threshold` | number  | `0.5`                      | 通过率阈值（0-1）         |
| `load-test-framework`      | string  | `k6`                       | `k6` 或 `locust`          |
| `load-test-script`         | string  | `./tests/load/scenario.js` | 压测脚本路径              |
| `load-test-duration`       | string  | `30s`                      | 压测时长                  |
| `load-test-vus`            | number  | `10`                       | 虚拟用户数                |
| `load-test-target-url`     | string  | `""`                       | 目标 URL                  |
| `db-pg-version`            | string  | `"16"`                     | PostgreSQL 版本           |
| `db-scale-factor`          | number  | `10`                       | pgbench scale factor      |
| `db-duration`              | number  | `60`                       | pgbench 时长（秒）        |
| `db-threshold-tps`         | number  | `500`                      | TPS 阈值                  |

## Secrets

全部可选，缺失时优雅跳过并 warning，不阻断 CI。

### 基础

| secret               | 用途                            | 缺失行为                       |
| -------------------- | ------------------------------- | ------------------------------ |
| `WECOM_BOT_KEY`      | 企业微信群机器人 webhook 的 key | 跳过通知，warning              |
| `SEMGREP_APP_TOKEN`  | Semgrep App 规则集 token        | 跳过 semgrep step，warning     |
| `GITLEAKS_LICENSE`   | Gitleaks 私有仓库许可           | gitleaks 用社区版（功能受限）  |
| `DOCKERHUB_USERNAME` | Trivy 拉镜像避免限流            | 跳过登录，trivy 仍跑（可能慢） |
| `DOCKERHUB_TOKEN`    | 同上                            | 同上                           |

### B+ 类外部安全服务

详见 [外部安全服务](external-security-tools.md)。

| secret                | 用途                 | 缺失行为         |
| --------------------- | -------------------- | ---------------- |
| `SONAR_TOKEN`         | SonarQube 扫描 token | 跳过 SonarQube   |
| `SNYK_TOKEN`          | Snyk 依赖扫描 token  | 跳过 Snyk        |
| `GITGUARDIAN_API_KEY` | GitGuardian 密钥扫描 | 跳过 GitGuardian |

### E 类 AI 内容测试

| secret              | 用途                     | 缺失行为               |
| ------------------- | ------------------------ | ---------------------- |
| `OPENAI_API_KEY`    | promptfoo 调用 OpenAI    | 跳过 AI 内容测试       |
| `ANTHROPIC_API_KEY` | promptfoo 调用 Anthropic | 同上（两个都缺才跳过） |
| `TEST_DATABASE_URL` | 集成测试数据库连接串     | 跳过集成测试，warning  |
| `E2E_TARGET_URL`    | E2E 测试目标 URL         | 跳过 E2E 测试，warning |
| `E2E_AUTH_TOKEN`    | E2E 测试认证 token       | 跳过 E2E 测试，warning |

## 使用示例

### 最简接入（bun 项目，无 secret）

```yaml
jobs:
  ci:
    uses: Yun-Hai-Org/ci-templates/.github/workflows/standard-ci.yml@v1
    with:
      project-type: 'bun'
```

### 完整配置（python 项目，带通知和 semgrep）

```yaml
jobs:
  ci:
    uses: Yun-Hai-Org/ci-templates/.github/workflows/standard-ci.yml@v1
    with:
      project-type: 'python'
      python-version: '3.11'
      run-extended-lint: true
      fail-on-severity: 'medium'
      wecom-notify: true
    secrets:
      WECOM_BOT_KEY: ${{ secrets.WECOM_BOT_KEY }}
      SEMGREP_APP_TOKEN: ${{ secrets.SEMGREP_APP_TOKEN }}
```

### 启用 D 类上线卡点 + B+ 外部安全服务

```yaml
jobs:
  ci:
    uses: Yun-Hai-Org/ci-templates/.github/workflows/standard-ci.yml@v1
    with:
      project-type: 'bun'
      run-release-gates: true
      jira-prefix: 'PROJ'
      jira-warning-only: false
      schema-check-paths: 'agents/*.json'
      wecom-notify: true
    secrets:
      WECOM_BOT_KEY: ${{ secrets.WECOM_BOT_KEY }}
      SEMGREP_APP_TOKEN: ${{ secrets.SEMGREP_APP_TOKEN }}
      SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}
      SNYK_TOKEN: ${{ secrets.SNYK_TOKEN }}
      GITGUARDIAN_API_KEY: ${{ secrets.GITGUARDIAN_API_KEY }}
```

### E 类全量（schedule 触发）

```yaml
on:
  schedule:
    - cron: '0 3 * * 1' # 每周一凌晨
  workflow_dispatch:

jobs:
  ci:
    uses: Yun-Hai-Org/ci-templates/.github/workflows/standard-ci.yml@v1
    with:
      project-type: 'bun'
      run-release-gates: true
      run-ai-content-test: true
      run-load-test: true
      run-db-benchmark: true
      load-test-target-url: 'https://staging.example.com'
      db-threshold-tps: 800
      promptfoo-fail-threshold: 0.8
    secrets:
      OPENAI_API_KEY: ${{ secrets.OPENAI_API_KEY }}
      ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}
```

### 仅安全扫描（legacy 项目）

```yaml
jobs:
  ci:
    uses: Yun-Hai-Org/ci-templates/.github/workflows/standard-ci.yml@v1
    with:
      project-type: 'bun'
      run-static-analysis: false
      run-dependency-audit: false
      run-security-scan: true
      wecom-notify: true
    secrets:
      WECOM_BOT_KEY: ${{ secrets.WECOM_BOT_KEY }}
```

## fail-on-severity 行为

影响 trivy / semgrep / gitleaks / pip-audit 的失败阈值：

| 值         | trivy severity                     | audit 行为      |
| ---------- | ---------------------------------- | --------------- |
| `none`     | LOW,MEDIUM,HIGH,CRITICAL（exit 0） | 发现漏洞不 fail |
| `low`      | LOW,MEDIUM,HIGH,CRITICAL           | fail            |
| `medium`   | MEDIUM,HIGH,CRITICAL               | fail            |
| `high`     | HIGH,CRITICAL                      | fail（默认）    |
| `critical` | CRITICAL                           | fail            |
