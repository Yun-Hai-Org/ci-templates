# Release Gates（D 类上线前卡点）

D 类卡点在 `release-gates.yml` 中实现，由 `standard-ci.yml` 的 `run-release-gates: true` 触发。
与 B 类 security-scans 的分工：security-scans 跑通用安全规则，release-gates 跑项目特定合规卡点。

## 检查项

| 步骤               | 工具                         | 触发条件                  | 失败行为                                          |
| ------------------ | ---------------------------- | ------------------------- | ------------------------------------------------- |
| OPA fmt            | `opa fmt`                    | `policy/**` 存在          | 未格式化则 error                                  |
| OPA test           | `opa test`                   | `policy/**` 存在          | 单元测试失败则 error                              |
| Semgrep 自定义规则 | `semgrep --config .semgrep/` | `.semgrep/*.yml` 存在     | WARNING 以上 error                                |
| Jira ID 校验       | shell + grep                 | PR 事件                   | 失败 error（`jira-warning-only=true` 时 warning） |
| Schema 校验        | `ajv`                        | `schema-check-paths` 非空 | 校验失败 error                                    |
| commitlint         | `commitlint`                 | PR 事件                   | 非 Conventional Commits 格式 error                |

## OPA 策略

### 与 conftest 的分工

| 工具            | 命令                        | 验证对象                                        |
| --------------- | --------------------------- | ----------------------------------------------- |
| `opa test`      | `opa test ./policy/... -v`  | Rego 策略本身的逻辑（白盒单元测试）             |
| `conftest test` | `conftest test . --combine` | 用 Rego 策略校验实际 IaC / 配置文件（黑盒集成） |

二者都从 `policy/` 读取策略。`*_test.rego` 文件被 `opa test` 自动发现，conftest 自动忽略。

### 编写策略

参考 `policy/` 示例目录，三组场景：

- `authz/`：部署授权（K8s Deployment 必须带 approved-by 标签）
- `config/`：镜像仓库白名单
- `sensitive/`：禁止手机号明文（Stargate ACL 场景）

业务仓库可直接复制或参考编写。

### 本地调试

```bash
# 格式化
opa fmt -w policy/

# 跑单元测试
opa test ./policy/... -v

# 用 conftest 校验某个 IaC 文件
conftest test examples/deployment.yaml --policy policy/
```

## Jira 工单 ID 校验

### 校验范围

- PR 标题：必须包含 `[A-Z]+-\d+`（如 `PROJ-1234`）
- 所有 PR commit message：必须包含上述模式

### 配置

```yaml
with:
  run-release-gates: true
  jira-prefix: 'PROJ' # 强制匹配 PROJ-\d+；空则只校验格式
  jira-warning-only: false # true 时失败仅 warning（团队过渡期）
```

### 过渡策略

1. 上线初期：`jira-warning-only: true`，仅告警不阻断
2. 团队习惯后：改为 `false`，正式卡点

## Semgrep 自定义规则

### 与 Semgrep auto 的分工

| 步骤           | 位置                 | 配置                 | 覆盖范围                               |
| -------------- | -------------------- | -------------------- | -------------------------------------- |
| Semgrep auto   | `security-scans.yml` | `config: auto`       | 通用安全规则（需 `SEMGREP_APP_TOKEN`） |
| Semgrep 自定义 | `release-gates.yml`  | `--config .semgrep/` | 项目特定规则（无需 token）             |

### 规则文件

`.semgrep/` 下三组规则：

- `ai-code.yml`：AI 生成代码常见问题（幻觉 import、上下文注入、重复函数）
- `sensitive-data.yml`：手机号 / 身份证 / AK-SK 明文、弱加密算法
- `jira-required.yml`：TODO/FIXME 未关联工单 ID

### 本地验证

```bash
semgrep --config .semgrep/ --validate
semgrep --config .semgrep/ .
```

## Schema 校验

针对 Agent 配置文件（JSON）。

```yaml
with:
  schema-check-paths: 'agents/*.json'
  schema-file: 'templates/agent-config.schema.json'
```

业务仓库可参考 `templates/agent-config.schema.json` 编写自己的 Schema，或直接复用。

## commitlint

### CI 中

PR 事件触发，校验 PR 所有 commit message 是否符合 Conventional Commits + Jira 规则。

业务仓库提供 `commitlint.config.js` 时优先使用，否则用模板默认配置。

### 本地钩子

参考 [commitlint + husky 配置](commitlint-husky.md)。

## 全部跳过的情形

- `run-release-gates: false`（默认）—— 整个 D 类不执行
- `policy/` 不存在 —— OPA 步骤跳过
- `.semgrep/` 不存在 —— 自定义规则跳过
- 非 PR 事件 —— Jira 与 commitlint 跳过
- `schema-check-paths` 为空 —— Schema 校验跳过

跳过时输出 `::notice::` 说明原因，不阻断 CI。
