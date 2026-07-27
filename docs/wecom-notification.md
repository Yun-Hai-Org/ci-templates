# 企业微信通知配置

CI 检查完成后，自动向企业微信群发送模板卡片（template_card）通知。

## 消息样式

通知采用企业微信「文本通知型模板卡片」（text_notice），点击卡片可跳转 CI 详情页。

```
┌───────────────────────────────────┐
│  ✅ CI 完成                        │  ← main_title.title
│  Yun-Hai-Org/ci-templates          │  ← main_title.desc (仓库名)
│                                    │
│         ✅                         │  ← emphasis_content.title (状态图标)
│         成功                       │  ← emphasis_content.desc (状态文字)
│                                    │
│  分支: feat/wecom-template-card    │  ← sub_title_text
│                                    │
│  触发者        pr9898              │  ← horizontal_content_list
│  事件          Push                │
│  静态分析      🟡 部分跳过          │
│  安全扫描      ✅ 成功              │
│  依赖审计      ⊘ 跳过              │
│  ...                               │
│                                    │
│  [点击卡片查看 CI 详情]            │  ← card_action.url
└───────────────────────────────────┘
```

### 标题与状态映射

| 场景     | title                  | status  | emphasis 图标 |
| -------- | ---------------------- | ------- | ------------- |
| 全部成功 | ✅ CI 完成             | success | ✅            |
| 部分跳过 | ✅ CI 完成（部分跳过） | success | ✅            |
| 有失败   | ❌ CI 失败             | failure | ❌            |

### 检查项状态图标

| 图标        | 含义       |
| ----------- | ---------- |
| ✅ 成功     | 检查通过   |
| 🟡 部分跳过 | 有步骤跳过 |
| ❌ 失败     | 检查失败   |
| ⊘ 跳过      | 整项跳过   |

## 配置步骤

### 1. 创建企业微信群机器人

1. 打开企业微信群
2. 右上角 `...` → 群机器人 → 添加机器人
3. 命名（如 "CI 通知"），完成添加
4. 复制 webhook URL，形如：

```
https://qyapi.weixin.qq.com/cgi-bin/webhook/send?key=<YOUR_BOT_KEY>
```

### 2. 提取 WECOM_BOT_KEY

webhook URL 中 `key=` 后面的部分即为 `WECOM_BOT_KEY`。上面的例子中：

```
WECOM_BOT_KEY=<YOUR_BOT_KEY>
```

### 3. 配置 Secret（推荐 Organization-level）

**推荐：Organization-level Secret（配一次，所有仓库通用）**

如果业务仓库都在同一个 GitHub Organization 下，在 org 层面配置一次，所有仓库自动继承，业务仓库无需各自配置：

1. 打开 GitHub Organization 页面
2. Settings → Secrets and variables → Actions
3. New organization secret
4. Name: `WECOM_BOT_KEY`
5. Secret: 上一步提取的 key 值
6. Repository access: 选 `All repositories`（或选择指定仓库）
7. Add secret

配置后，业务仓库的 ci.yml 写 `secrets: WECOM_BOT_KEY: ${{ secrets.WECOM_BOT_KEY }}`，GitHub 会自动从 org 级别取值，业务仓库 Settings 里不需要再配。

**备选：Repository-level Secret（每仓库各自配）**

如果业务仓库不在同一 org，或需要不同仓库发到不同群，在每个业务仓库单独配置：

1. 仓库 Settings → Secrets and variables → Actions
2. New repository secret
3. Name: `WECOM_BOT_KEY`
4. Secret: 上一步提取的 key 值
5. Add secret

### 4. 在 CI 样板中引用

`templates/bun-ci.yml` 和 `templates/python-ci.yml` 已默认引用 `WECOM_BOT_KEY`，无需额外修改：

```yaml
secrets:
  WECOM_BOT_KEY: ${{ secrets.WECOM_BOT_KEY }}
```

## 关闭通知

### 完全关闭

在业务仓库的 `.github/workflows/ci.yml` 中设置：

```yaml
with:
  wecom-notify: false
```

### 仅不配置 secret

删除 `secrets:` 下的 `WECOM_BOT_KEY` 行——未配置 secret 时通知自动跳过，输出 warning。

## 故障排查

### 没收到通知

1. 确认 `WECOM_BOT_KEY` secret 已配置（仓库 Settings → Secrets）
2. 确认 `wecom-notify: true`（默认 true）
3. 查看 CI 日志中 `notify-end` job 的输出：
   - `::warning::secret WECOM_BOT_KEY not set` → secret 未配置
   - `::warning::WeCom webhook returned errcode=...` → key 错误或机器人被禁用

### 通知延迟

结束通知在所有检查完成后发出。

### errcode 说明

| errcode | 含义                   | 解决方法                        |
| ------- | ---------------------- | ------------------------------- |
| 0       | 成功                   | —                               |
| 93000   | webhook 不存在或被禁用 | 重新创建群机器人                |
| 40014   | key 错误               | 检查 WECOM_BOT_KEY 是否完整复制 |
| 45009   | 超频（20 条/分钟）     | 合并 CI 触发，减少频率          |

### 通知内容乱码

模板卡片（template_card）为企业微信群机器人默认支持的消息类型。如需切换为 text 或 markdown，修改 `.github/workflows/standard-ci.yml` 中 notify-end job 的 Python payload。
