# TAPD Bug 自动处理 Skill - 快速上手指南

## 一、安装 Skill

1. 克隆仓库到本地 skill 目录：
```bash
git clone https://github.com/nameLogen/tapd-skills.git
cp -r tapd-skills/skills/tapd ~/.claude/skills/tapd
# 或复制到你的 AI 工具对应的 skills 目录
```

2. 安装后重启 AI 工具（Claude Code / Kimi CLI / Cursor 等）

---

## 二、配置（必须）

编辑 `skills/tapd/config.json`，填写以下信息：

```json
{
  "project": {
    "name": "你的项目名称",
    "workspace_id": "你的TAPD项目ID"
  },
  "tapd_token": "你的TAPD个人访问令牌"
}
```

### 获取配置信息

| 配置项 | 获取方式 |
|--------|---------|
| `workspace_id` | 打开 TAPD 项目，URL 中的数字，如 `https://www.tapd.cn/39222963` → `39222963` |
| `tapd_token` | 登录 [TAPD 开放平台](https://www.tapd.cn/my_access_token/) → 创建个人访问令牌 |

### 可选配置

```json
{
  "bot_url": "企业微信机器人webhook地址",
  "filter_members": ["张三", "李四"]
}
```

- `bot_url`: 如需推送通知到企业微信群，配置群机器人 Webhook
- `filter_members`: 只推送指定成员相关的 Bug，留空则推送所有

---

## 三、触发流程

### 第一步：批量收集 Bug

**对 AI 说：**
> "收集一下项目里所有 new 的 Bug"

**AI 自动做：**
1. 读取 `config.json` 中的 `workspace_id` 和 `tapd_token`
2. 查询 TAPD 中 `status=new` 的 Bug 列表
3. 发通知（如果配置了 `bot_url`）：

```
【Bug 批量收集结果】
共 15 个 new Bug

1. 【任务系统】子任务未置顶排序
2. 【背包】出售装备后未获取收益
3. 【安卓/充值】沙盒充值完成后未到账券
...

请回复要处理的 Bug 编号
```

### 第二步：人工确认

**对 AI 说：**
> "修 1、3、5" → 处理指定编号
> "修客户端相关的" → AI 根据标题人工判断
> "全部修" → 处理所有
> "跳过第 3 个" → 跳过指定编号

### 第三步：逐个自动处理（全自动）

AI 对每个确认的 Bug 自动执行：

```
【开始处理 Bug 1/5】
标题：【任务系统】子任务未置顶排序
━━━━━━━━━━━━━━━━━━━━

→ 获取详情 + 截图... ✅
→ 解析截图内容... ✅
→ 分析代码... ✅（定位到 TaskView.ts 第 163 行）
→ 修改代码... ✅（新增 guestDatas.sort()）
→ 发群通知... ✅
→ 更新 TAPD... ✅（状态：resolved）
━━━━━━━━━━━━━━━━━━━━
【Bug 1/5 完成】

【开始处理 Bug 2/5】...
```

**每步都自动发群通知**，不需要人工干预。

---

## 四、配置检查（自动）

启动流程前，AI 会自动检查：

| 检查项 | 是否强制 | 缺失时行为 |
|--------|---------|-----------|
| `tapd_token` | ✅ 必须 | 提示配置令牌 |
| `workspace_id` | ✅ 必须 | 提示填写项目 ID |
| `bot_url` | ❌ 可选 | 仅不发群通知 |
| `filter_members` | ❌ 可选 | 默认推送所有 Bug |

---

## 五、快速验证

配置完成后，输入：

> "查一下我参与的项目"

如果返回项目列表，说明配置成功。

---

## 六、文件说明

| 文件 | 作用 |
|------|------|
| `config.json` | 核心配置文件（token、项目ID、机器人） |
| `SETUP.md` | 详细的首次配置指南 |
| `SKILL.md` | Skill 主文档（API 说明、请求规范） |
| `bug_workflow.md` | Bug 批量处理工作流详细说明 |
| `scripts/tapd_client_stdlib.py` | Python 标准库客户端脚本（可选） |

---

## 七、注意事项

1. **不要提交含真实 token 的 config.json 到 GitHub**
   - GitHub 版本使用 `YOUR_TAPD_ACCESS_TOKEN` 占位符
   - 本地使用真实值

2. **代码修改风险**
   - 当前阶段：AI 分析代码 → 给出修复建议 → **不改代码**（B 阶段）
   - 确认后阶段：AI 修改代码 → 发群通知 → 更新 TAPD（C 阶段）
   - 建议先在测试分支验证修改

3. **项目类型自动识别**
   - 不绑定特定游戏品类
   - `type`、`tech_stack`、`code_path` 由 AI 根据当前项目自动识别
   - 配置中只需填写 `workspace_id` 和 `name`
