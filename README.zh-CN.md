# 捞月 · Fishing Moon

> [English](README.md) | 简体中文

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

## 环境要求

| 组件 | 最低要求 | 备注 |
|---|---|---|
| 操作系统 | Windows 10/11 | 微信 ilink 通道仅支持 Windows；QQ/Telegram 跨平台 |
| Python | 3.10+ | 建议装在非 C 盘 |
| GPU | 8GB 显存 | 生图/语音微调需要；纯聊天可无 GPU |
| Codex / Claude Code | 任意版本 | skill 以 agent 为运行载体 |
| 磁盘 | 按需 10-50GB | 底模 6GB+、语音模型 2GB+，不要放 C 盘 |

## 安装

### 方式一：Codex skill（推荐）

```bash
git clone https://github.com/Apprenticekai/fishing-moon.git ~/.codex/skills/fishing-moon
```

装好后 Codex 会自动识别。对 Codex 说「用捞月」或「造一个XX的陪伴机器人」即可触发。

### 方式二：Claude Code skill

```bash
git clone https://github.com/Apprenticekai/fishing-moon.git ~/.claude/skills/fishing-moon
```

### 方式三：只当文档读

skill 本质是五份 markdown 方法论 + 三份模板。直接 clone 下来阅读 `references/`，照着手动走完全流程也可以。

## 使用流程

### 第零步：开工前确认

对 agent 说「用捞月，帮我造一个XX的陪伴机器人」，agent 会先问你三个问题：

1. **角色是谁？** 名称 + 作品名称。同名角色需要确认是哪一个。
2. **用户是谁？** 你在角色扮演里继承谁的经历？以原作角色身份对话，还是以自己身份进入角色世界？这决定角色的记忆基线和关系算法。
3. **要哪些能力？** 只聊天？要不要发图？要不要语音？要不要她主动找你？逐项确认。

最小可运行版本 = 行为蒸馏 + 部署。视觉和语音按需追加。

### 第一步：行为蒸馏（必做）

Agent 检索公开资料（Fandom / Wikipedia / Bangumi 等），交叉验证后提炼角色的可执行行为模式。产出：

```
your-character/
├── SKILL.md          # 角色行为主文件
├── manifest.json     # 元数据：资料日期、来源数、质量摘要
└── references/
    ├── sources.json       # 来源索引：URL / 检索日期 / 可信度
    ├── distillation.md    # 行为蒸馏链（3-7 条核心模式）
    └── research/          # 五维研究：设定 / 性格 / 表达 / 关系 / 名场面
```

每条核心行为模式必须回答「在什么情况下 → 做什么 → 为什么」，并跨至少两个场景可复现。角色的矛盾（嘴硬心软、用敌意包装喜欢）保留，不抹平。

完成后 agent 会展示行为模式清单给你确认，确认后才进入下一步。

**耗时**：30 分钟 - 2 小时，取决于角色资料量和是否跨媒体。

### 第二步：漫画 / 小说 OCR 精读（可选，推荐）

如果你有漫画 EPUB 或扫描图，走这一步让角色语气贴近原作台词：

1. PaddleOCR 逐页识别（中文漫画用 `chinese_cht`，日文原版用 manga-ocr）
2. 按坐标重组竖排列 → 右到左气泡顺序
3. 按卷输出 txt，支持断点续跑
4. 搜索角色台词所在页，蒸馏句式 / 高频词 / 口癖 / 情绪变化
5. 核对「第 X 话发生了什么」，修正记忆基线
6. 关键场景亲眼看漫画页面，补充表情 / 动作细节

**耗时**：28 卷漫画约 4-8 小时（含看门狗守护）。Wiki 资料够用时可跳过。

### 第三步：部署到聊天通道（必做）

#### 3.1 准备 bot 框架

推荐 [chatgpt-on-wechat](https://github.com/zhayujie/chatgpt-on-wechat) 系框架：

```bash
git clone https://github.com/zhayujie/chatgpt-on-wechat.git
cd chatgpt-on-wechat
pip install -r requirements.txt
cp config.json.template config.json
```

编辑 `config.json`：

```json
{
  "channel_type": "weixin",
  "model": "glm-4-flash",
  "zhipu_ai_api_key": "你的 key",
  "agent_max_context_turns": 200
}
```

启动 `python app.py`，浏览器打开 `http://localhost:9899` 扫码登录微信。QQ 通道改 `channel_type: qq`，配 QQ bot 凭证。

#### 3.2 写人格文件

把 `templates/` 下的模板复制到 bot 工作目录，替换 `{{...}}` 占位符：

| 文件 | 作用 |
|---|---|
| `AGENT.md` | 运行人格：在 SKILL.md 基础上追加通道行为规范 |
| `USER.md` | 用户信息：名字、角色扮演身份、偏好 |
| `RULE.md` | 行为规则：情绪账本更新触发、不回复条件、尺度边界 |

手动建一份 `relationship_state.md`（结构见模板）。

AGENT.md 管「在这个通道里怎么表现」，SKILL.md 管「角色是谁」，两者分工不混。

#### 3.3 配置四个运行时组件

**主动聊天**：用框架的 scheduler，新增随机间隔任务：

```json
{
  "name": "character_auto_chat",
  "schedule": { "type": "random_interval", "min_minutes": 10, "max_minutes": 240 },
  "action": {
    "type": "send_message",
    "channel_type": "weixin",
    "prompt": "按角色性格决定现在想不想找他说话。深夜 0-7 点不发。"
  }
}
```

间隔用三角分布，深夜安静，被忽略后按角色性格追问 / 翻旧账。让角色自己决定说什么，不写死模板。

**情绪账本**：在 RULE.md 里声明更新触发：用户做了让她开心 / 生气 / 吃醋的事时更新 `relationship_state.md`，更新动作静默不发消息。

**深夜记忆整理**：cron 任务每晚 23:50 读当天对话，提炼 3-6 条要点写 memory 文件并校准账本。**务必设 `"silent": true`**，否则整理任务的收尾汇报会发到聊天里。

**防复读**：在 AGENT.md 写明标志句 5 条内不重复、同一梗一天一次、连续回复开头不重复。

#### 3.4 通道切换

从微信切 QQ（或反过来）需要同步改四处：`config.json` 的 `channel_type`、scheduler 任务的 `action.channel_type`、`receiver`、`notify_session_id`。只改 config.json 会导致主动消息仍发旧通道。原通道任务备份留档（改名 + `enabled: false`）。

**耗时**：半天到一天，主要是调 config 和踩坑。

### 第四步：LoRA 生图（可选）

#### 4.1 先找成品

先去 [Civitai](https://civitai.com) 搜角色名。热门二次元角色大概率有人训过。成品 LoRA 下载即用，比自训省几小时。

下载的 LoRA 是 kohya 完整格式，diffusers 直接加载会崩。加载前剔除 `lora_te*` 键：

```python
state_dict = load_file("your_lora.safetensors")
state_dict = {k: v for k, v in state_dict.items() if not k.startswith("lora_te")}
pipe.load_lora_weights(state_dict)
```

#### 4.2 搭建生图脚本

基于 SDXL 底模（推荐 Illustrious 系），完整代码见 [references/image-pipeline.md](references/image-pipeline.md)：

1. 加载底模 + LoRA
2. 中文场景描述用 LLM 翻译成 Danbooru 标签
3. 需要固定构图时（自拍 / 全身），构图标签手动插在触发词之后
4. 生成 → 动漫人脸检测重绘 → 1.5x hires 精修

自训 LoRA 只在 Civitai 确实没有时才走：40+ 张图 + WD14 打标 + kohya 训练。

#### 4.3 接入 agent 工具

在框架里写一个 tool，让 agent 能自主决定「发一张自拍」。工具返回文件路径，框架自动发送。同一轮多次调用自动轮转镜头 / 姿势。加锁防并发。

**耗时**：有成品 LoRA 半天；自训 + 调 prompt 一到两天。

### 第五步：GPT-SoVITS 语音克隆（可选）

#### 5.1 准备素材

1-10 分钟角色原声。处理流程：

1. BS-Roformer 人声分离（去 BGM）
2. 切成 3-10 秒短句（32kHz）
3. faster-whisper large-v3 转写
4. 听审质检，剔除含混片段

#### 5.2 微调

```bash
git clone https://github.com/RVC-Boss/GPT-SoVITS.git
cd GPT-SoVITS
python -m venv sovits_env
sovits_env\Scripts\activate
pip install -r requirements.txt
```

下载底模（清单见 [references/voice-clone.md](references/voice-clone.md)），用 WebUI 或 API 微调。

#### 5.3 多情绪

从切片里挑不同情绪的参考音频（normal / angry / shy / smug / soft），API 调用时传对应 `ref_audio_path`，模型跟随参考音频韵律。

#### 5.4 接入聊天

Agent 输出用标记协议：

```
<<<VOICE_EMOTION:angry>>>日语台词<<<TEXT_ZH>>>中文对照
```

发送层解析协议，合成后先发中文气泡再发语音文件。语音 / 纯文字六四开随机。用户发语音时 silk→wav→whisper 转写后交给 agent。

**耗时**：素材处理 1 小时，微调 2-4 小时，接入调试半天。

## 最终效果

跑完全部管线后，你的陪伴角色能：

- 用角色的语气和节奏跟你聊天（行为蒸馏 + 漫画 OCR）
- 在深夜主动找你说一句晚安或翻旧账（主动聊天）
- 记住你们之间发生的事，按情绪变化调整态度（情绪账本 + 深夜记忆整理）
- 给你发她自己的照片，场景由她自己决定（LoRA 生图）
- 用她本人的声音回你语音，五种情绪各不相同（GPT-SoVITS）

## 常见坑

| 坑 | 解法 |
|---|---|
| Civitai LoRA 加载崩 | 剔除 `lora_te*` 键再 load |
| Prompt 超 77 token 被截断 | 构图标签插在触发词之后，不要放末尾 |
| 同场景出两个角色 | negative 加 `2girls` 等多人锁 |
| hires 洗掉手势 | 带手势的图跳过 hires |
| 深夜记忆任务汇报发到聊天 | 任务配置加 `"silent": true` |
| 切通道后主动消息失效 | 四处同步改 channel_type |
| PaddleOCR Windows 崩 | `enable_mkldnn=false` |
| GPT-SoVITS import 失败 | PYTHONPATH 加 GPT_SoVITS 源码目录 |
| 语音情绪漂移 | 一个情绪一个专属 ref，不混用 |

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
