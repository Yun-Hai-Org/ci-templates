# Central CI Templates

中心化 CI 模板仓库，通过 GitHub Actions **Reusable Workflows** 让业务项目用几行 `uses:` 调用统一的 CI 流水线。升级工具、调整规则只改本仓库，所有调用方自动生效——告别"复制粘贴地狱"。

## 如何在你的仓库接入（其他 repo 引用本项目）

本仓库对外暴露**唯一入口** [`standard-ci.yml`](.github/workflows/standard-ci.yml)（`on: workflow_call`）。你的仓库只需新建一个 `.github/workflows/ci.yml`，用一行 `uses:` 指向它即可——**无需 clone 本仓库、无需安装任何依赖**。

### 三步接入

**1. 在你的业务仓库新建 `.github/workflows/ci.yml`**（复制即用）：

```yaml
# 你的业务仓库 .github/workflows/ci.yml
name: CI
on:
  pull_request:
  push:
    branches: [main]
jobs:
  ci:
    uses: Yun-Hai-Org/ci-templates/.github/workflows/standard-ci.yml@main
    with:
      project-type: 'bun'        # 或 'python' / 'rust' / 'node'
    secrets: inherit              # 透传 org/repo secrets，未配置的自动跳过
```

**2. 选 `project-type`**：`bun`（JS/TS，含 MCP Server）/ `python` / `rust` / `node`（npm/yarn/pnpm）。其余 input 全部有默认值，无需填写。

**3.（可选）配 secrets**：企业微信通知、Semgrep、SonarQube、Snyk 等 token 在 GitHub Org/Repo Settings 配置后，`secrets: inherit` 自动透传；未配的检查自动跳过并 `::notice::`，不阻断 CI。

### 零配置自动检测

所有检查**默认开启**，工具靠**文件存在性**决定是否执行——你的仓库没有 `package.json` / `policy/` / `tests/` / `Dockerfile` 等对应文件时，相关检查自动跳过。即接入后**无需逐项开关**，不适用的一律跳过。

### `@main` 还是固定 tag

- `@main`：本仓库改动合入 main 即时对所有调用方生效，无需维护 tag（**当前默认**）。
- 固定 tag（如 `@v1`）：升级 GitHub Team 后用 Organization Ruleset 全局强制时建议固定，避免 main 不稳定改动即时影响。详见 [版本管理](#版本管理) 与 [Ruleset 全局强制](docs/ruleset-onboarding.md)。

> 想要 PR 强制阻断（不靠开发者自觉）？升级 GitHub Team 后用 [Organization Ruleset](docs/ruleset-onboarding.md) 全局强制，业务仓库可逐步删除 ci.yml。当前 GitHub Free 阶段用上面的 ci.yml 即可。

更多细节见下方 [Inputs](#inputs) · [Secrets 配置](#secrets-配置) · [接入方式](#接入方式) · [常见配置组合](#常见配置组合)。


## 架构

```
业务项目 .github/workflows/ci.yml
  │
  │  uses: Yun-Hai-Org/ci-templates/.github/workflows/standard-ci.yml@main
  │  with: { project-type: 'bun', ... }
  │  secrets: { WECOM_BOT_KEY: ..., SEMGREP_APP_TOKEN: ... }
  ▼
┌──────────────────────────────────────────────────────────────────────┐
│  standard-ci.yml (对外唯一入口)                                       │
│                                                                       │
│  lint ──────────────────────────┐                                    │
│                  security (B+B+) ┤                                    │
│                  dependency ─────┤                                    │
│                  release-gates(D)┤──► [ai-content, load-test,        │
│                                  │     db-benchmark] (E) ──► notify-end│
└──────────────────────────────────────────────────────────────────────┘
         │              │              │            │
         ▼              ▼              ▼            ▼
   lint-checks    security-scans  dependency   release-gates
   (A 类)         (B+B+ 类)        (C 类)        (D 类)
                                                       │
                                                       ▼
                                              ai-content / load-test
                                              / db-benchmark (E 类)

  F 类：PR 模板 / release checklist / commitlint+husky / 文档（非 CI job，是模板文件）
```

七类检查 + 通知：

| 类别   | 名称             | 默认                    | 说明                                                              |
| ------ | ---------------- | ----------------------- | ----------------------------------------------------------------- |
| **A**  | 静态分析与格式化 | ✅ 开                   | type-check / lint / format / extended-lint                        |
| **B**  | 安全扫描（开源） | ✅ 开                   | semgrep / gitleaks / trivy / knip / checkov / conftest            |
| **B+** | 外部安全服务     | ⚠️ 有 key 才跑          | SonarQube / Snyk / GitGuardian                                    |
| **C**  | 依赖审计         | ✅ 开                   | bun audit / pip-audit / lockfile-freshness                        |
| **D**  | 上线前卡点       | ✅ 开（无文件自动跳过） | OPA test / Semgrep 自定义 / 敏感数据 / Jira / Schema / commitlint |
| **E**  | 上线后验证       | ✅ 开（无文件自动跳过） | promptfoo AI 内容 / k6 压测 / pgbench DB 基准                     |
| **T**  | 测试执行         | ✅ 开（严格模式）       | bun test / pytest / cargo test / npm test + 集成 + E2E            |
| **F**  | 流程卡点         | 📋 模板                 | PR 模板 / release checklist / 本地钩子（复制即用）                |
| —      | 企业微信通知     | ✅ 开                   | CI 开始/结束发 markdown 到群机器人                                |

> **设计原则**：所有检查默认开启，工具靠**文件存在性**决定是否执行——无对应文件/不适用时自动跳过并输出 `::notice::` 说明原因。业务仓库接入后零配置即扫描。

---

## 检查项总览

### A 类：静态分析与格式化（默认开启）

| 检查项        | 检查什么                       | bun                                          | python                | 需要 key |
| ------------- | ------------------------------ | -------------------------------------------- | --------------------- | -------- |
| type-check    | TypeScript / Python 类型错误   | `bunx tsc --noEmit`                          | `uv run pyright`      | 否       |
| lint          | 代码规范                       | `eslint`                                     | `ruff check`          | 否       |
| format        | 格式化检查                     | `prettier --check`                           | `ruff format --check` | 否       |
| extended-lint | Dockerfile / Shell / CSS / SQL | hadolint / shellcheck / stylelint / sqlfluff | 同左                  | 否       |

**启用/关闭**：`run-static-analysis: true/false`；扩展 lint 用 `run-extended-lint: true`。

### B 类：安全扫描（开源工具，默认开启）

| 检查项         | 检查什么                                     | 需要 key                   | 缺失行为                       |
| -------------- | -------------------------------------------- | -------------------------- | ------------------------------ |
| Semgrep (auto) | 通用 SAST 规则集                             | `SEMGREP_APP_TOKEN`        | 跳过 + warning                 |
| Gitleaks       | Git 历史中的密钥泄露                         | `GITLEAKS_LICENSE`         | 用社区版（私有仓库功能受限）   |
| Trivy fs       | 文件系统漏洞 + 密钥                          | `DOCKERHUB_USERNAME/TOKEN` | 跳过登录，仍跑（可能慢）       |
| Trivy config   | IaC 配置错误（Terraform / K8s / Dockerfile） | —                          | 始终跑                         |
| Knip           | JS 死代码检测                                | —                          | 无 package.json 跳过（仅 bun） |
| Checkov        | IaC 策略扫描（tf / Dockerfile / k8s）        | —                          | 无 IaC 文件跳过 + notice       |
| Conftest       | OPA 策略校验 IaC 文件                        | —                          | 无 `policy/` 目录跳过 + notice |

**启用/关闭**：`run-security-scan: true/false`；Knip 用 `run-knip: false` 单独关。

### B+ 类：外部安全服务（有 key 才跑）

| 检查项      | 检查什么                         | 需要 key                                   | 缺失行为       |
| ----------- | -------------------------------- | ------------------------------------------ | -------------- |
| SonarQube   | 代码质量 + 跨文件漏洞 + 重复代码 | `SONAR_TOKEN` + `sonar-organization` input | 跳过 + notice  |
| Snyk        | 第三方依赖漏洞 + 许可证          | `SNYK_TOKEN`                               | 跳过 + warning |
| GitGuardian | 密钥泄露（SaaS，含历史扫描）     | `GITGUARDIAN_API_KEY`                      | 跳过 + warning |

**启用方式**：

- **Snyk / GitGuardian**：无需 input 开关，`secrets: inherit` 透传对应 token 即可。
- **SonarQube**：除 `SONAR_TOKEN` secret 外，还需在 ci.yml 的 `with:` 下配置 `sonar-organization`（SonarCloud organization key）。获取方式见 [docs/external-security-tools.md](docs/external-security-tools.md#获取-sonar-organization)。

```yaml
with:
  sonar-organization: 'your-sonarcloud-org' # SonarCloud organization key
secrets: inherit # 含 SONAR_TOKEN
```

### C 类：依赖审计（默认开启）

| 检查项             | 检查什么                      | bun                             | python             | 需要 key |
| ------------------ | ----------------------------- | ------------------------------- | ------------------ | -------- |
| dep-audit          | 依赖漏洞                      | `bun audit --production`        | `uv run pip-audit` | 否       |
| lockfile-freshness | lockfile 与 manifest 是否同步 | `bun install --frozen-lockfile` | `uv sync --frozen` | 否       |
| OSV Scanner        | 全量 OSV 漏洞库扫描（慢）     | 可选                            | 可选               | 否       |

**启用/关闭**：`run-dependency-audit: true/false`；OSV 用 `run-osv-scanner: true`（建议 schedule 触发）。

> **无 manifest 时**：无 `package.json`（bun）或 `pyproject.toml`/`setup.py`/`requirements.txt`（python）时，dep-audit 与 lockfile-freshness 自动跳过并输出 notice。

### D 类：上线前卡点（默认开启，无文件自动跳过）

| 检查项         | 检查什么                         | 触发条件                  | 需要 key |
| -------------- | -------------------------------- | ------------------------- | -------- |
| OPA fmt        | Rego 文件格式化                  | `policy/` 存在            | 否       |
| OPA test       | Rego 策略单元测试（白盒）        | `policy/` 存在            | 否       |
| Semgrep 自定义 | AI 代码 / 敏感数据 / Jira 注释   | `.semgrep/` 存在          | 否       |
| Jira ID 校验   | PR 标题 + commit 含工单 ID       | PR 事件                   | 否       |
| Schema 校验    | Agent 配置 JSON Schema           | `schema-check-paths` 非空 | 否       |
| commitlint     | Conventional Commits + Jira 规则 | PR 事件                   | 否       |

**启用/关闭**：`run-release-gates: true`；Jira 前缀用 `jira-prefix: 'PROJ'`；过渡期用 `jira-warning-only: true`（失败仅 warning）。

**与 B 类 Conftest 的分工**：`opa test` 验证 Rego 策略逻辑本身（白盒单元测试），`conftest test` 用 Rego 策略校验 IaC 文件（黑盒集成）。两者都读 `policy/` 目录。

### E 类：上线后验证（默认开启，无文件自动跳过，依赖 D 类通过）

| 检查项      | 检查什么                                 | 需要 key                                | 缺失行为             |
| ----------- | ---------------------------------------- | --------------------------------------- | -------------------- |
| promptfoo   | LLM prompt 注入 / 越狱 / 幻觉 / 敏感输出 | `OPENAI_API_KEY` 或 `ANTHROPIC_API_KEY` | 无配置/API key 跳过  |
| k6 / Locust | HTTP 压测 P99 延迟 / 错误率              | 否                                      | 无脚本跳过 + warning |
| pgbench     | PG 入库速度 TPS 基准                     | 否                                      | 跑本地容器，~60s     |

**启用/关闭**：`run-ai-content-test: true` / `run-load-test: true` / `run-db-benchmark: true`。

**成本提示**：E 类调用真实 LLM（有 token 成本）或压测目标服务。`pgbench` 在本地 docker 容器中跑，无外部副作用，适合每次 push 验证。`promptfoo` 和 `k6` 若无对应配置文件自动跳过，不会产生成本。建议高频 PR 用默认触发，低频全量测试用独立 workflow 配 `schedule:` 或 `workflow_dispatch:`。

### F 类：流程卡点（模板文件，复制即用）

| 文件            | 用途                                            | 位置                                          |
| --------------- | ----------------------------------------------- | --------------------------------------------- |
| PR 模板         | 提 PR 时自动填充上线前 checklist + 行政合规自查 | `.github/PULL_REQUEST_TEMPLATE.md`            |
| 发布 checklist  | 发版 issue 模板，覆盖技术/流程/合规三维度       | `.github/ISSUE_TEMPLATE/release-checklist.md` |
| commitlint 配置 | Conventional Commits + Jira ID 规则             | `templates/commitlint.config.js`              |
| husky 钩子      | 本地 `git commit` 时校验 message                | `templates/.husky/commit-msg` / `pre-commit`  |
| pre-commit 配置 | 多语言本地钩子框架                              | `templates/_pre-commit-config.yaml`           |

**启用方式**：业务仓库复制对应文件到自己的 `.github/` 或根目录。这些是仓库级文件，不走 reusable workflow，模板仓库升级不会自动同步。

---

## Secrets 配置

### Secret 总览

**全部可选**。缺失时对应检查自动跳过并 `::warning::`，不阻断 CI。

| secret                | 用途                     | 在哪获取                                | 缺失行为         |
| --------------------- | ------------------------ | --------------------------------------- | ---------------- |
| `WECOM_BOT_KEY`       | 企业微信通知             | 群机器人 webhook URL 中 `key=` 后的部分 | 跳过通知         |
| `SEMGREP_APP_TOKEN`   | Semgrep App 规则集       | semgrep.dev                             | 跳过 semgrep     |
| `GITLEAKS_LICENSE`    | Gitleaks 私有仓库许可    | gitleaks.io                             | 用社区版         |
| `DOCKERHUB_USERNAME`  | Trivy 拉镜像避限流       | hub.docker.com                          | 跳过登录         |
| `DOCKERHUB_TOKEN`     | 同上                     | hub.docker.com                          | 同上             |
| `SONAR_TOKEN`         | SonarQube 扫描           | sonarcloud.io 或自建 SonarQube          | 跳过 SonarQube   |
| `SNYK_TOKEN`          | Snyk 依赖扫描            | snyk.io                                 | 跳过 Snyk        |
| `GITGUARDIAN_API_KEY` | GitGuardian 密钥扫描     | dashboard.gitguardian.com               | 跳过 GitGuardian |
| `OPENAI_API_KEY`      | promptfoo 调用 OpenAI    | platform.openai.com                     | 跳过 AI 内容测试 |
| `ANTHROPIC_API_KEY`   | promptfoo 调用 Anthropic | console.anthropic.com                   | 同上             |

### 配置方式

#### 方式一：Organization-level（推荐，配一次通用）

业务仓库在同一 GitHub Org 下时，在 org 层面配一次，所有仓库自动继承：

1. GitHub Organization → Settings → Secrets and variables → Actions
2. New organization secret
3. Name: 如 `WECOM_BOT_KEY`
4. Value: 填入对应 token / key
5. Repository access: 选 `All repositories`
6. Add secret

业务仓库 ci.yml 里写 `secrets: WECOM_BOT_KEY: ${{ secrets.WECOM_BOT_KEY }}`，GitHub 自动从 org 取值——**业务仓库 Settings 不需要再配**。换 key 时只改 org 一处，所有仓库立即生效。

> **技术说明**：GitHub Actions 的 secret 查找是层级继承的（Repository → Organization → Environment）。reusable workflow 跨仓库调用时无法直接读取调用方未传递的 secret，所以"中心仓库代发"不可行——org-level secret 是实现"配一次通用"的唯一方式。

#### 方式二：Repository-level（仓库独立配置）

业务仓库不在同一 org，或需要不同仓库用不同 key 时，在每个仓库 Settings → Secrets and variables → Actions 单独配置。

### 各 secret 获取步骤

#### `WECOM_BOT_KEY`

1. 企业微信群 → 右上角 … → 群机器人 → 添加机器人
2. 复制 webhook URL，形如 `https://qyapi.weixin.qq.com/cgi-bin/webhook/send?key=abc12345-...`
3. 取 `key=` 后面的部分 `abc12345-...` 作为 secret 值

#### `SEMGREP_APP_TOKEN`

1. 登录 [semgrep.dev](https://semgrep.dev)
2. Settings → API Tokens → Create token
3. 复制 token

#### `SONAR_TOKEN`

1. 登录 [SonarCloud](https://sonarcloud.io)（或自建 SonarQube）
2. My Account → Security → Generate Tokens
3. 复制 token

#### `SNYK_TOKEN`

1. 注册 [snyk.io](https://snyk.io)
2. Account Settings → API Tokens → Show
3. 复制 token

#### `GITGUARDIAN_API_KEY`

1. 注册 [dashboard.gitguardian.com](https://dashboard.gitguardian.com)
2. API Access → Personal Access Tokens → Create
3. 复制 key

#### `OPENAI_API_KEY` / `ANTHROPIC_API_KEY`

- OpenAI：[platform.openai.com](https://platform.openai.com) → API Keys → Create
- Anthropic：[console.anthropic.com](https://console.anthropic.com) → API Keys

详细配置见 [docs/external-security-tools.md](docs/external-security-tools.md)。

---

## 启用与关闭检查

### 三种控制方式

#### 1. input 开关（最常用）

在 ci.yml 的 `with:` 下设置 boolean。所有检查**默认开启**，无对应文件/不适用时自动跳过并输出 `::notice::`：

```yaml
with:
  run-static-analysis: true # A 类（默认开）
  run-security-scan: true # B + B+ 类（默认开）
  run-dependency-audit: true # C 类（默认开）
  run-unit-tests: true # T 类（默认开，严格模式：有代码无测试即 fail）
  run-integration-tests: true # T 类（默认开，无 tests/integration/ 跳过）
  run-e2e-tests: true # T 类（默认开，无 E2E 配置跳过）
  run-release-gates: true # D 类（默认开，无文件自动跳过）
  run-ai-content-test: true # E 类（默认开，无配置自动跳过）
  run-load-test: true # E 类（默认开，无脚本自动跳过）
  run-db-benchmark: true # E 类（默认开，跑本地容器）
```

如需关闭某类检查（例如 CI 太慢），设为 `false` 即可。

#### 2. 目录存在性（细粒度控制）

某些 D 类检查在 `run-release-gates: true` 前提下，根据业务仓库是否有对应目录自动决定是否执行：

| 目录                              | 触发的检查                    | 不存在时      |
| --------------------------------- | ----------------------------- | ------------- |
| `policy/`                         | OPA fmt + OPA test + Conftest | 跳过 + notice |
| `.semgrep/`                       | Semgrep 自定义规则扫描        | 跳过 + notice |
| `tests/` / `test/` / `__tests__/` | 测试集 Git 跟踪校验           | 仅 warning    |

业务仓库可参考模板仓库的 `policy/` 和 `.semgrep/` 示例目录复制或自写。

#### 3. secret 存在性（B+ 类与 E 类 AI 测试）

B+ 类外部安全服务和 E 类 promptfoo 无需 input 开关，根据 secret 是否配置自动决定。推荐用 `secrets: inherit` 一次性透传所有 org/repo secrets：

```yaml
secrets: inherit # 自动透传 SONAR_TOKEN / SNYK_TOKEN / GITGUARDIAN_API_KEY / OPENAI_API_KEY 等
```

或逐个声明（等价写法）：

```yaml
secrets:
  SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }} # 有则跑 SonarQube
  SNYK_TOKEN: ${{ secrets.SNYK_TOKEN }} # 有则跑 Snyk
  GITGUARDIAN_API_KEY: ${{ secrets.GITGUARDIAN_API_KEY }} # 有则跑 GitGuardian
  OPENAI_API_KEY: ${{ secrets.OPENAI_API_KEY }} # E 类 AI 测试需要
```

未配置的 secret 对应步骤输出 `::warning::secret XXX not set, skipping ...`，CI 继续。

### 常见配置组合

#### 最小配置（全量检查，无文件自动跳过）

```yaml
jobs:
  ci:
    uses: Yun-Hai-Org/ci-templates/.github/workflows/standard-ci.yml@main
    with:
      project-type: 'bun'
    secrets: inherit
```

所有 A/B/C/D/E 七类检查默认开启，无对应文件的检查自动跳过并输出 `::notice::`。对纯模板仓库（无 `package.json`/`policy/`/`tests/`）实际只跑 B 类安全扫描 + 企业微信通知。

#### 标准 PR 检查（Jira 强制阻断）

```yaml
with:
  project-type: 'bun'
  jira-prefix: 'PROJ' # 强制 commit 含 PROJ-1234
  jira-warning-only: false # 失败即阻断（默认 warning-only: true）
  schema-check-paths: 'agents/*.json' # 有 Agent 配置时
```

D 类默认已开，只需调整 Jira 严格度。业务仓库需准备 `policy/` 和 `.semgrep/` 目录（参考模板仓库示例），否则对应步骤跳过。

#### 完整安全扫描（B+ 全开）

```yaml
with:
  project-type: 'python'
  run-security-scan: true
secrets: inherit # 自动透传 SEMGREP/SONAR/SNYK/GITGUARDIAN 等 token
```

#### 定时全量验证（E 类，独立 workflow）

E 类有成本，建议单独配 `schedule` workflow：

```yaml
# .github/workflows/weekly-e2e.yml
name: Weekly E2E
on:
  schedule:
    - cron: '0 3 * * 1' # 每周一凌晨
  workflow_dispatch:
jobs:
  ci:
    uses: Yun-Hai-Org/ci-templates/.github/workflows/standard-ci.yml@main
    with:
      project-type: 'bun'
      run-release-gates: true # E 类依赖 D 类通过
      run-ai-content-test: true
      run-load-test: true
      run-db-benchmark: true
      load-test-target-url: 'https://staging.example.com'
      promptfoo-fail-threshold: 0.8
      db-threshold-tps: 800
    secrets: inherit # 含 OPENAI/ANTHROPIC API key 供 promptfoo 调用
```

### 失败处理

- **HIGH/CRITICAL** 漏洞或检查失败 → 阻断 PR（默认）
- **MEDIUM** → 警告但不阻断
- **LOW** → 仅报告
- 用 `fail-on-severity` 调整阈值：`none` / `low` / `medium` / `high`（默认）/ `critical`

某个工具的 secret 没配？对应 step 自动跳过并 warning，不阻断其他检查。

---

## 接入方式

业务仓库有两种接入方式。**当前组织为 GitHub Free，默认采用方案 B（ci.yml）**，后续升级 GitHub Team 后可切换方案 A。

### 方案 B：仓库内 ci.yml（默认，当前采用）

在业务仓库放一个 `.github/workflows/ci.yml`，`uses:` 指向本仓库的 `standard-ci.yml@main`。适合任何 GitHub 版本，公开/私有仓库均可，无版本限制。

```yaml
# 业务项目 .github/workflows/ci.yml
name: CI
on:
  pull_request:
  push:
    branches: [main]
jobs:
  ci:
    uses: Yun-Hai-Org/ci-templates/.github/workflows/standard-ci.yml@main
    with:
      project-type: 'bun' # 或 'python'
      wecom-notify: true
    # secrets: inherit 自动透传 org/repo 级所有 secrets，未配置的自动跳过
    secrets: inherit
```

**优点**：零版本门槛、配置可见可调试、开发者可在 with 下自行调整开关。
**缺点**：每个业务仓库都要放一份 ci.yml；指向 `@main` 时所有调用方自动跟 main 最新，main 上的不稳定改动会立即影响所有业务仓库。

### 方案 A：Repository Ruleset 全局强制（升级路径，后续采用）

通过 GitHub **Repository Ruleset**，对仓库的 PR 强制运行 `standard-ci.yml`，业务仓库无需放 ci.yml。PR 不通过则 Merge 按钮锁定，开发者无法绕过。

> **当前不可用**：本组织是 GitHub Free，Organization 级 Ruleset 需升级到 **GitHub Team** 才能强制执行。升级前的过渡方案：
>
> - **短期**：继续用方案 B（ci.yml）
> - **升级 Team 后**：配置 Organization Ruleset，一处生效全组织，再逐步删除各仓库的 ci.yml

**版本限制**：

| GitHub 版本             | Organization Ruleset | Repository Ruleset（逐仓库） |
| ----------------------- | -------------------- | ---------------------------- |
| Free for Organizations  | ❌ 不可用            | ✅ 可用（公开仓库）          |
| Team / Enterprise Cloud | ✅ 可用（公/私仓库） | ✅ 可用（公/私仓库）         |

**升级 Team 后的配置步骤**（需组织管理员权限）：

1. Organization → Settings → Rulesets → New branch ruleset
2. Target repositories：选 `All repositories` 或按标签/语言动态筛选
3. Target branches：`Include default branch`
4. 勾选 `Require status checks to pass before merging` → `Require workflows to pass before merging`
5. Add workflow：
   - Repository: `Yun-Hai-Org/ci-templates`
   - Workflow: `.github/workflows/standard-ci.yml`
   - Ref: `v1`
6. Enforcement status：先 `Evaluate`（评估模式，不阻断）测试，确认无误后切 `Active`

详细说明见 [Organization Ruleset 全局强制接入](docs/ruleset-onboarding.md)。

## Quick Start

最小可用的业务仓库 ci.yml（方案 B，当前默认）：

```yaml
# 业务项目 .github/workflows/ci.yml
name: CI
on:
  pull_request:
  push:
    branches: [main]
jobs:
  ci:
    uses: Yun-Hai-Org/ci-templates/.github/workflows/standard-ci.yml@main
    with:
      project-type: 'bun' # 或 'python'
      wecom-notify: true
    secrets: inherit # 自动透传 org/repo secrets，未配置的自动跳过
```

提 PR 后看到的反馈：

- **企业微信群**：`🚀 CI 开始` → `✅ CI 完成` / `❌ CI 失败`，含各阶段状态
- **GitHub PR Checks**：`Lint & Format` / `Security Scan` / `Dependency Audit` / `Release Gates` 等 job 状态
- **Security 标签页**：Semgrep / SonarQube 的 SARIF 漏洞详情
- **PR 评论**：Gitleaks / GitGuardian 发现的泄露位置

---

## Project Types

| project-type | 适用场景                      | type-check       | lint           | format                 | knip | dep-audit     | lockfile                        |
| ------------ | ----------------------------- | ---------------- | -------------- | ---------------------- | ---- | ------------- | ------------------------------- |
| `bun`        | JS/TS 项目（含 MCP Server）   | `bunx tsc`       | `eslint`       | `prettier`             | ✅   | `bun audit`   | `bun install --frozen-lockfile` |
| `python`     | Python 项目                   | `uv run pyright` | `ruff check`   | `ruff format --check`  | —    | `pip-audit`   | `uv sync --frozen`              |
| `rust`       | Rust 项目                     | `cargo check`    | `cargo clippy` | `cargo fmt --check`    | —    | `cargo-audit` | `cargo update --dry-run`        |
| `node`       | Node.js 项目（npm/yarn/pnpm） | `npx tsc`        | `npx eslint`   | `npx prettier --check` | —    | `npm audit`   | `npm ci`                        |

所有 project-type 默认跑 B 类安全扫描。

## Inputs

完整列表见 [docs/inputs-reference.md](docs/inputs-reference.md)。常用：

| input                   | 默认    | 说明                                                      |
| ----------------------- | ------- | --------------------------------------------------------- |
| `project-type`          | `bun`   | `bun` / `python`                                          |
| `run-static-analysis`   | `true`  | A 类总开关                                                |
| `run-security-scan`     | `true`  | B + B+ 类总开关                                           |
| `run-dependency-audit`  | `true`  | C 类总开关                                                |
| `run-extended-lint`     | `false` | hadolint / shellcheck / stylelint / sqlfluff              |
| `run-unit-tests`        | `true`  | T 类单元测试（bun test / pytest / cargo test / npm test） |
| `run-integration-tests` | `true`  | T 类集成测试（无 tests/integration/ 跳过）                |
| `run-e2e-tests`         | `true`  | T 类 E2E 测试（Playwright/Cypress）                       |
| `fail-on-severity`      | `high`  | `none`/`low`/`medium`/`high`/`critical`                   |
| `run-release-gates`     | `false` | D 类上线卡点（OPA / Jira / Schema / commitlint）          |
| `run-ai-content-test`   | `false` | E 类 promptfoo AI 内容安全                                |
| `run-load-test`         | `false` | E 类 k6 / Locust 压测                                     |
| `run-db-benchmark`      | `false` | E 类 pgbench DB 基准                                      |
| `jira-prefix`           | `""`    | Jira 项目前缀（如 `PROJ`）                                |
| `wecom-notify`          | `true`  | 是否发送企业微信通知                                      |

## 版本管理

业务项目和内部子调用统一使用 `@main`：workflow 改动合入 main 即时对所有调用方生效，无需维护 tag。

```yaml
uses: Yun-Hai-Org/ci-templates/.github/workflows/standard-ci.yml@main
```

**Trade-off**：main 上的不稳定改动会立即影响所有调用方。缓解措施：PR 合入前必须通过 smoke-test（main push 触发，端到端验证 standard-ci.yml 编排链路）；改子 workflow 的 input schema 时必须同步修改 standard-ci.yml 调用处，code-review 重点检查。

## 文档

- [快速开始](docs/getting-started.md)
- [Inputs 与 Secrets 参考](docs/inputs-reference.md)
- [项目类型详解](docs/project-types.md)
- [企业微信通知配置](docs/wecom-notification.md)
- [迁移指南](docs/migration-guide.md)
- [FAQ](docs/faq.md)
- [Release Gates（D 类上线卡点）](docs/release-gates.md)
- [AI 内容安全测试](docs/ai-content-testing.md)
- [压测配置](docs/load-testing.md)
- [DB 基准测试](docs/db-benchmark.md)
- [测试执行详解](docs/tests.md)
- [commitlint + husky 本地钩子](docs/commitlint-husky.md)
- [外部安全服务](docs/external-security-tools.md)
- [PR 模板与发布 Checklist](docs/pr-checklist.md)
- [Organization Ruleset 全局强制接入](docs/ruleset-onboarding.md)

## License

MIT
