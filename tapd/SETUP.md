# TAPD Skill 首次配置指南

## 前置条件

- 已安装 Kimi Code CLI / Claude Code / Cursor 等支持 Agent Skill 的 AI 工具
- 已有 TAPD 账号，且能访问目标项目

---

## 第一步：配置 TAPD 认证（必须）

### 方式一：配置文件（推荐）

编辑 `skills/tapd/config.json`，添加 `tapd_token`：

```json
{
  "tapd_token": "你的个人访问令牌"
}
```

1. 登录 [TAPD 开放平台](https://www.tapd.cn/my_access_token/)
2. 创建个人访问令牌
3. 复制令牌，填入 `config.json` 的 `tapd_token` 字段

### 方式二：环境变量（备用）

```bash
# Windows
setx TAPD_ACCESS_TOKEN "你的个人访问令牌"

# macOS / Linux
export TAPD_ACCESS_TOKEN="你的个人访问令牌"
```

> **注意**：环境变量配置后需要**重启 AI CLI 工具**才能生效。

---

## 第二步：配置项目信息（必须）

在同一份 `skills/tapd/config.json` 中，继续填写项目信息：

```json
{
  "project": {
    "name": "你的项目名称",
    "workspace_id": "你的项目ID"
  }
}
```

| 字段 | 说明 | 示例 |
|------|------|------|
| `name` | 项目显示名称 | 弹弹堂经典版 |
| `workspace_id` | TAPD 项目 ID | 39222963 |

---

## 第三步：配置企业微信机器人（可选）

如需推送通知到企业微信群：

1. 在企业微信群中添加群机器人
2. 复制 Webhook 地址
3. 配置到 `config.json`：

```json
{
  "bot_url": "https://qyapi.weixin.qq.com/cgi-bin/webhook/send?key=你的key"
}
```

---

## 第四步：配置关注成员（可选）

只推送特定成员相关的 Bug：

```json
{
  "filter_members": ["张三", "李四"]
}
```

留空 `[]` 则推送所有 Bug。

---

## 配置检查

启动流程前，AI 会自动检查以下配置：

| 检查项 | 位置 | 是否强制 | 缺失时行为 |
|--------|------|---------|-----------|
| `tapd_token` | `config.json` | ✅ 必须 | 提示配置令牌（或配环境变量 `TAPD_ACCESS_TOKEN`） |
| `workspace_id` | `config.json` | ✅ 必须 | 提示填写项目 ID |
| `bot_url` | `config.json` | ❌ 可选 | 仅不发群通知 |
| `filter_members` | `config.json` | ❌ 可选 | 默认推送所有 Bug |

---

## 快速验证

配置完成后，输入：

> "查一下我参与的项目"

如果返回项目列表，说明配置成功。
