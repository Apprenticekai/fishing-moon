# 部署运行时

把蒸馏好的角色 SKILL.md 部署到聊天机器人框架，变成可用的陪伴系统。

## 框架选择

| 框架 | 通道 | 优势 | 劣势 |
|---|---|---|---|
| cowagent（chatgpt-on-wechat fork） | 微信 ilink / QQ / Telegram / DingTalk 等 | 插件式、自带 scheduler/memory/agent 工具链、Windows 本机 | 定制需改源码 |
| 自建 FastAPI + 官方 bot API | Telegram / Discord / Slack | 灵活、云部署友好 | 主动推送/凭证管理自己写 |

微信陪伴场景推荐 cowagent：ilink 协议扫码登录、无封号风险、凭证自动重连。

## 人格文件结构

角色的人格不是只有 SKILL.md，还需要运行时的灵魂文件：

| 文件 | 作用 |
|---|---|
| `AGENT.md` | 运行人格：在 SKILL.md 基础上追加通道行为规范（语音/纯文字比例、emoji 映射、防复读、发送格式） |
| `USER.md` | 用户信息：用户名、角色扮演中继承的身份、用户偏好 |
| `relationship_state.md` | 情绪账本：好感度、吃醋等级、未平账本、约定 |
| `RULE.md` | 行为规则：何时更新情绪账本、何时不回复、尺度边界 |
| `world_state.md` | 世界状态：角色的当前场景、时间、天气 |
| `STYLE.md` | 语言风格笔记（从 OCR 或原文蒸馏的语气校准文件，按角色命名） |

AGENT.md 和 SKILL.md 的分工：SKILL.md 管「角色是谁」，AGENT.md 管「在这个通道里怎么表现」。

## 四个运行时组件

### 1. 主动聊天

用 scheduler 引擎，随机间隔发消息：

- 间隔用三角分布（最短 10 分钟，众数偏短，最长 4 小时），避免机械感
- 深夜 0-7 点安静（不发主动消息）
- 被用户忽略后不等于话题结束：追问、翻旧账、宣布惩罚
- 任务描述里让角色自己决定说什么，不写死模板

### 2. 情绪账本

`relationship_state.md` 记录：

```
## 好感度：X/5
## 吃醋等级：X/5
## 未平账本
- 日期 | 事件 | 她的反应 | 是否已扯平
## 约定
- 日期 | 内容 | 状态
```

RULE.md 声明更新触发：用户做了让她开心/生气/吃醋的事时，agent 主动更新账本。更新动作静默，不发消息。

### 3. 深夜记忆整理

cron 任务（如每晚 23:50）：

1. 读取当天所有对话
2. 提炼 3-6 条要点写入当天的 memory 文件
3. 校准情绪账本
4. 静默运行：输出永不投递到聊天

关键踩坑：框架默认对非空输出投递结果，必须在任务配置中显式设 `"silent": true`，否则整理任务的收尾汇报会发到用户聊天里。

### 4. 防复读

AGENT.md 加规则：

- 标志句在最近 5 条内用过必须换说法
- 同一个梗一天最多一次
- 连续回复的开头不重复

## 通道切换

从微信切到 QQ（或反过来）时，需要同步修改四处：

1. `config.json` 的 `channel_type`
2. scheduler 任务里的 `action.channel_type`
3. scheduler 任务里的 `receiver`（不同通道的接收者 ID 不同）
4. `notify_session_id` / `instance_id`

只改 config.json 里的 channel_type 会导致 scheduler 主动消息仍然发到旧通道，报「No context_token」错误。

原通道的 scheduler 任务备份留档（改名 + `enabled: false`），方便切回。

## 微信发送链注意事项

- 语音合成后如果还带纯文字尾巴，需要拆分成多个气泡（分隔符正则边界注意别咬碎内容）
- 语音 + 中文对照是两个独立发送动作，先发中文气泡再发语音文件
- 多段消息用 `<<<NEXT_MESSAGE>>>` 分隔符，发送层拆开逐条发
- 清理死代码：不要留无条件 TEXT 分支或 elif 死代码，容易漏出标记符号

## 重启方式

Windows 部署推荐用计划任务（`schtasks`）管理 bot 进程，不要直接 `python app.py`：

- 计划任务与终端会话解耦，关终端不会杀掉 bot
- 设 LogonTrigger 开机自启
- 日志重定向到文件
- 修改计划任务需要管理员 PowerShell

