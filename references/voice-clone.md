# 语音克隆

用 GPT-SoVITS 克隆角色音色，接入聊天机器人实现多情绪语音回复。

## 素材准备

角色需要至少 1-10 分钟的高质量语音素材。来源：

- 动画/游戏原声台词剪辑
- 用户提供的音频
- 角色语音包

素材处理流程：

1. 人声分离：BS-Roformer 或 Demucs，去掉背景音乐和音效
2. 切片：切成 3-10 秒的短句片段（32kHz）
3. 转写：faster-whisper large-v3 按原语言转写
4. 质检：听审若干条，剔除含混、BGM 残留、气声过重的片段

## GPT-SoVITS 微调

### 版本选择

v2ProPlus 音色相似度优于 v4（v4 的 48kHz 带宽反而会放大素材噪音），优先选 v2ProPlus。

### 环境配置

```
GPT-SoVITS 仓库 + 独立 venv
torch 2.x + cu126
numpy 锁 1.26.4 + scipy 1.13.1
venv 内 torchvision 版本注意不要与主环境冲突
jieba_fast 用 shim 包替代
PYTHONPATH 需包含 GPT_SoVITS 目录
```

### 底模清单

| 模型 | 用途 |
|---|---|
| chinese-hubert-base | 特征提取 |
| chinese-roberta-wwm-ext-large | 文本编码 |
| v2Pro s2Gv2ProPlus + s2Dv2ProPlus | 声码器 |
| sv 说话人验证 | 训练质量检查 |
| s1v3.ckpt | 基础模型 |
| G2PWModel | 中文前端注音 |

## 情绪参考音频

GPT-SoVITS 的 `ref_audio_path` 决定韵律。同一个音色想要不同情绪，用不同的参考音频：

```python
EMOTION_REFS = {
    "normal": (ref_wav_normal, ref_prompt_normal),
    "angry":  (ref_wav_angry,  ref_prompt_angry),
    "shy":    (ref_wav_shy,    ref_prompt_shy),
    "smug":   (ref_wav_smug,   ref_prompt_smug),
    "soft":   (ref_wav_soft,   ref_prompt_soft),
}
```

从切片库里挑出最能代表各情绪的片段做参考音频。模型会跟随参考音频的语调，所以选片段时注意情绪纯度。

## API 接入

GPT-SoVITS 提供 RESTful API（默认 `127.0.0.1:9880`）：

```python
payload = {
    "text": ja_text,
    "text_lang": "ja",
    "ref_audio_path": ref_path,
    "prompt_text": ref_prompt,
    "prompt_lang": "ja",
    "text_split_method": "cut5",
    "batch_size": 1,
    "media_type": "wav",
    "streaming_mode": False,
}
resp = requests.post(api_url, json=payload)
```

## 协议设计

Agent 输出用标记协议区分语音和文字：

```
<<<VOICE_EMOTION:angry>>>日语台词<<<TEXT_ZH>>>中文对照
```

发送层解析协议：合成语音 → 先发中文气泡 → 再发语音文件。多条消息用 `<<<NEXT_MESSAGE>>>` 分隔。

### 语音/文字混合策略

- 语音/纯文字按比例随机（如六四开），不连续三次同形式
- 纯文字带 emoji + 动作
- 语音后可跟纯文字段
- emoji 按情绪映射（正常/生气/害羞/得意/温柔各有不同 emoji）

## 语音输入（ASR）

用户发语音消息时：

1. silk → wav（pysilk-mod，微信语音格式）
2. faster-whisper large-v3 本地转写
3. 加前缀 `<<<VOICE_INPUT>>>` 交给 agent，agent 知道这是语音消息

## 踩坑记录

- numpy 版本和 torch/torchvision 兼容性极其敏感，严格锁定版本
- GPT-SoVITS 的 PYTHONPATH 必须包含源码目录，否则 import 失败
- 情绪参考音频选错了会导致音色漂移，一个情绪一个专属 ref，不要混用
- 切通道后 `import re` NameError 之类的错误通常是进程未重启加载旧代码

