# 捞月 · Fishing Moon

> 把角色从纸面上捞起来，像从水里捞月亮。捞出来的不是本体，但能陪着你。

捞月是一套把二次元角色做成可运行陪伴机器人的方法论和工具链。

输入一个角色名和作品名，产出可以部署到微信/QQ 的陪伴机器人——她会用角色的语气跟你聊天、在深夜主动找你、给你发自己画的图、用她本人的声音回你语音。

## 它能做什么

- **行为蒸馏**：从公开资料交叉验证，提炼角色的可执行行为模式
- **漫画 OCR 精读**：逐页识别漫画原文，校准角色的台词语感
- **微信/QQ 部署**：接入聊天机器人框架，支持主动聊天、情绪账本、深夜记忆整理
- **LoRA 生图**：角色形象图生成，agent 可自主决定「发一张自拍」
- **GPT-SoVITS 语音克隆**：用少量素材克隆角色音色，支持多情绪语音回复

## 它不是什么

- 不是角色卡：不是一段 prompt，是一套带研究链的可执行行为系统
- 不是云服务：所有组件本地运行，数据不出你的电脑
- 不是特定角色的搬运：方法论通用，你用它造任何角色的陪伴体

## 快速开始

1. 把这个仓库作为 Codex skill 安装到 `~/.codex/skills/fishing-moon/`
2. 对 Codex 说：「用捞月，帮我造一个XX的陪伴机器人」
3. 回答三个问题：角色是谁？用户是谁？要哪些能力？
4. Codex 按管线逐阶段推进，每个阶段有完成标准

## 管线概览

```
角色名 + 作品名
    │
    ▼
行为蒸馏（CSP 方法论）
    │         漫画 OCR 精读（可选）
    ▼
SKILL.md + manifest.json + references/
    │
    ▼
部署到微信/QQ（cowagent 框架）
    │         主动聊天 + 情绪账本 + 深夜记忆整理
    ▼
LoRA 生图（可选）    GPT-SoVITS 语音（可选）
    │                     │
    ▼                     ▼
发自拍                 发语音
```

## 致谢

- [CSP · Character Skill Producer](https://github.com/qian-gugugaga/Character_Skill_Producer)：行为蒸馏方法论的基础
- [chatgpt-on-wechat](https://github.com/zhayujie/chatgpt-on-wechat)：微信机器人框架
- [GPT-SoVITS](https://github.com/RVC-Boss/GPT-SoVITS)：语音克隆
- [kohya-ss/sd-scripts](https://github.com/kohya-ss/sd-scripts)：LoRA 训练（本项目最终改用 Civitai 成品 LoRA）

## License

MIT

