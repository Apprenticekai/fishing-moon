# Fishing Moon

> English | [简体中文](README.zh-CN.md)

> Lift a character off the page, like fishing the moon out of water. What you catch isn't the real thing, but it can stay by your side.

Fishing Moon is a methodology and toolchain for turning anime/manga characters into deployable companion bots.

Give it a character name and a series title, and it produces a companion bot for WeChat or QQ - one that talks in the character's voice, reaches out to you late at night, sends you images of herself, and replies with her own voice.

## What It Does

- **Behavior distillation** - Cross-reference public sources to extract executable behavior patterns
- **Manga OCR reading** - Page-by-page OCR of original manga text to calibrate the character's speech patterns
- **WeChat/QQ deployment** - Chat bot framework integration with proactive messaging, emotional ledger, nightly memory distillation
- **LoRA image generation** - Character image generation; the agent decides when to send a selfie
- **GPT-SoVITS voice cloning** - Clone the character's voice from a small audio sample, with multi-emotion voice replies

## What It Is Not

- Not a character card: not a single prompt, but a full behavior system with a research chain
- Not a cloud service: everything runs locally, no data leaves your machine
- Not a specific character port: the methodology is generic, use it to build a companion for any character

## Requirements

| Component | Minimum | Notes |
|---|---|---|
| OS | Windows 10/11 | WeChat ilink channel is Windows-only; QQ/Telegram cross-platform |
| Python | 3.10+ | Recommended on non-C drive |
| GPU | 8GB VRAM | Needed for image gen / voice fine-tuning; pure chat works without |
| Codex / Claude Code | Any | Skills run inside an agent |
| Disk | 10-50GB as needed | Base model 6GB+, voice models 2GB+, do not install to C drive |

## Installation

### Option 1: Codex skill (recommended)

```bash
git clone https://github.com/Apprenticekai/fishing-moon.git ~/.codex/skills/fishing-moon
```

Codex will auto-detect the skill. Say "use fishing moon" or "build a companion bot for [character]" to trigger it.

### Option 2: Claude Code skill

```bash
git clone https://github.com/Apprenticekai/fishing-moon.git ~/.claude/skills/fishing-moon
```

### Option 3: Read as documentation

The skill is essentially five markdown methodologies plus three templates. You can also clone it and follow the `references/` manually.

## Usage

### Step 0: Initial confirmation

Tell your agent "use fishing moon to build a companion bot for [character]". The agent will ask three questions:

1. **Who is the character?** Name + series title. Disambiguate if there are multiple matches.
2. **Who is the user?** Which character's experiences does the user inherit? This determines the memory baseline and relationship algorithm.
3. **What capabilities?** Chat only? Images? Voice? Proactive messaging? Confirm each one.

Minimum viable version = behavior distillation + deployment. Vision and voice are optional add-ons.

### Step 1: Behavior distillation (required)

The agent searches public sources (Fandom / Wikipedia / Bangumi etc.), cross-validates, and extracts executable behavior patterns. Output:

```
your-character/
├── SKILL.md          # Character behavior file (agent loads and speaks as the character)
├── manifest.json     # Metadata: research dates, source counts, quality summary
└── references/
    ├── sources.json       # Source index: URL / retrieved date / confidence
    ├── distillation.md    # Distillation chain (3-7 core patterns)
    └── research/          # Five-dimension research: setting / personality / expression / relationships / key scenes
```

Each core pattern must answer "in what situation -> does what -> why", and be reproducible across at least two scenes. Character contradictions (soft-hearted beneath a tough shell, disguising affection as hostility) are preserved, not smoothed over.

The agent presents the behavior pattern list for your confirmation before moving to the next step.

**Time**: 30 min - 2 hours, depending on source availability and cross-media complexity.

### Step 2: Manga / novel OCR reading (optional, recommended)

If you have manga EPUB or scans, run this step to calibrate the character's speech against original dialogue:

1. PaddleOCR page-by-page recognition (`chinese_cht` for Chinese manga, manga-ocr for Japanese)
2. Reassemble vertical text columns by coordinate, then order right-to-left bubbles
3. Per-volume txt output with resume support
4. Search for character dialogue pages, distill speech patterns / high-frequency words / tics / emotional shifts
5. Verify "what happened in chapter X", correct the memory baseline
6. Visually inspect key scenes for facial expressions / body language details

**Time**: about 4-8 hours for 28 volumes (with watchdog). Skip if wiki sources are sufficient.

### Step 3: Deploy to chat channel (required)

#### 3.1 Set up bot framework

Recommended: [chatgpt-on-wechat](https://github.com/zhayujie/chatgpt-on-wechat) family:

```bash
git clone https://github.com/zhayujie/chatgpt-on-wechat.git
cd chatgpt-on-wechat
pip install -r requirements.txt
cp config.json.template config.json
```

Edit `config.json`:

```json
{
  "channel_type": "weixin",
  "model": "glm-4-flash",
  "zhipu_ai_api_key": "your key",
  "agent_max_context_turns": 200
}
```

Run `python app.py`, open `http://localhost:9899` in browser, scan the QR code to log in WeChat. For QQ, change `channel_type` to `qq` and configure QQ bot credentials.

#### 3.2 Write persona files

Copy templates from `templates/` to the bot working directory, replace `{{...}}` placeholders:

| File | Purpose |
|---|---|
| `AGENT.md` | Runtime persona: channel behavior rules layered on top of SKILL.md |
| `USER.md` | User info: name, roleplay identity, preferences |
| `RULE.md` | Behavior rules: ledger update triggers, no-reply conditions, content boundaries |

Manually create a `relationship_state.md` (structure in templates).

AGENT.md governs "how to behave in this channel"; SKILL.md governs "who the character is". Keep the separation clean.

#### 3.3 Configure four runtime components

**Proactive chat**: use the framework's scheduler with a random interval task:

```json
{
  "name": "character_auto_chat",
  "schedule": { "type": "random_interval", "min_minutes": 10, "max_minutes": 240 },
  "action": {
    "type": "send_message",
    "channel_type": "weixin",
    "prompt": "Decide whether to reach out based on your character's personality. Stay quiet 0-7 AM."
  }
}
```

Use triangular distribution for intervals, stay quiet late night, follow up / bring up old accounts when ignored. Let the character decide what to say; don't hardcode templates.

**Emotional ledger**: declare update triggers in RULE.md: update `relationship_state.md` when the user does something that makes the character happy / angry / jealous. Updates are silent, no message sent.

**Nightly memory distillation**: cron task at 23:50 reads today's conversations, extracts 3-6 key points to memory files and calibrates the ledger. **Must set `"silent": true`**, or the distillation task's closing report leaks into the chat.

**Anti-repetition**: in AGENT.md, require signature phrases not repeat within the last 5 messages, same gag max once per day, consecutive reply openings differ.

#### 3.4 Channel switching

Switching WeChat to QQ (or reverse) requires changing four places: `config.json` `channel_type`, scheduler task's `action.channel_type`, `receiver`, and `notify_session_id`. Changing only config.json leaves proactive messages going to the old channel. Keep a backup of the old channel task (rename + `enabled: false`).

**Time**: half a day to a day, mostly config tuning and pitfall debugging.

### Step 4: LoRA image generation (optional)

#### 4.1 Search existing LoRA first

Search the character name on [Civitai](https://civitai.com). Popular characters likely have a ready-made LoRA. Download and use, saves hours versus training.

Downloaded LoRA is full kohya format; diffusers will crash on load. Strip `lora_te*` keys first:

```python
state_dict = load_file("your_lora.safetensors")
state_dict = {k: v for k, v in state_dict.items() if not k.startswith("lora_te")}
pipe.load_lora_weights(state_dict)
```

#### 4.2 Build the generation script

Based on SDXL base model (Illustrious family recommended), full code in [references/image-pipeline.md](references/image-pipeline.md):

1. Load base model + LoRA
2. Translate Chinese scene description to Danbooru tags via LLM
3. When fixed composition is needed (selfie / full body), insert composition tags manually after the trigger
4. Generate, then anime face detection and refinement, then 1.5x hires polish

Train your own LoRA only when Civitai truly has nothing: 40+ images + WD14 tagging + kohya training.

#### 4.3 Integrate as agent tool

Write a tool so the agent can autonomously decide to "send a selfie". Tool returns a file path, framework sends it automatically. Auto-rotate camera angles / poses across multiple calls in one turn. Add a lock to prevent concurrency.

**Time**: half a day with an existing LoRA; 1-2 days for self-training plus prompt tuning.

### Step 5: GPT-SoVITS voice cloning (optional)

#### 5.1 Prepare audio

1-10 minutes of character voice. Processing:

1. BS-Roformer vocal separation (remove BGM)
2. Slice into 3-10 second clips (32kHz)
3. faster-whisper large-v3 transcription
4. Manual QA, discard muddy clips

#### 5.2 Fine-tune

```bash
git clone https://github.com/RVC-Boss/GPT-SoVITS.git
cd GPT-SoVITS
python -m venv sovits_env
sovits_env\Scripts\activate
pip install -r requirements.txt
```

Download base models (list in [references/voice-clone.md](references/voice-clone.md)), fine-tune via WebUI or API.

#### 5.3 Multi-emotion

Pick reference clips of different emotions (normal / angry / shy / smug / soft), pass the corresponding `ref_audio_path` in the API call; the model follows the reference prosody.

#### 5.4 Integrate into chat

Agent output uses a marker protocol:

```
<<<VOICE_EMOTION:angry>>>Japanese text<<<TEXT_ZH>>>Chinese text
```

The send layer parses the protocol: synthesize voice, send Chinese text bubble first, then the voice file. Voice/text ratio around 40/60 randomized. User voice messages: silk -> wav -> whisper transcription, then hand to agent.

**Time**: 1 hour audio prep, 2-4 hours fine-tuning, half a day integration.

## Final Result

After running all pipelines, your companion character can:

- Talk in the character's voice and rhythm (behavior distillation + manga OCR)
- Reach out late at night to say goodnight or bring up old accounts (proactive chat)
- Remember what happened between you, adjust attitude by emotional state (ledger + nightly distillation)
- Send you images of herself, choosing scenes on her own (LoRA image gen)
- Reply with her own voice, five distinct emotions (GPT-SoVITS)

## Common Pitfalls

| Pitfall | Fix |
|---|---|
| Civitai LoRA crashes on load | Strip `lora_te*` keys before loading |
| Prompt silently truncated past 77 tokens | Insert composition tags after the trigger, not at the end |
| Two copies of the character in one scene | Add `2girls` etc. multi-person lock to negative prompt |
| hires washes out gestures | Skip hires for gesture-heavy images |
| Memory distillation report leaks to chat | Set `"silent": true` in task config |
| Proactive messages fail after channel switch | Update channel_type in four places |
| PaddleOCR crashes on Windows | `enable_mkldnn=false` |
| GPT-SoVITS import fails | Add GPT_SoVITS source dir to PYTHONPATH |
| Voice emotion drifts | One dedicated ref per emotion, don't mix |

## Pipeline Overview

```
Character name + series title
    |
    v
Behavior distillation (CSP methodology)
    |         Manga OCR reading (optional)
    v
SKILL.md + manifest.json + references/
    |
    v
Deploy to WeChat/QQ (bot framework)
    |         Proactive chat + ledger + nightly distillation
    v
LoRA image gen (optional)    GPT-SoVITS voice (optional)
    |                             |
    v                             v
Send selfies                  Send voice replies
```

## Acknowledgments

- [CSP - Character Skill Producer](https://github.com/qian-gugugaga/Character_Skill_Producer): foundation for the behavior distillation methodology
- [chatgpt-on-wechat](https://github.com/zhayujie/chatgpt-on-wechat): WeChat bot framework
- [GPT-SoVITS](https://github.com/RVC-Boss/GPT-SoVITS): voice cloning
- [kohya-ss/sd-scripts](https://github.com/kohya-ss/sd-scripts): LoRA training (this project ultimately used a pre-trained Civitai LoRA)

## License

MIT
