# 生图管线

用 LoRA + SDXL 底模生成角色形象图，接入 agent 工具让角色自主发图。

## 先搜成品，再自训

这是最重要的教训：先去 Civitai 搜有没有现成的角色 LoRA。很多热门角色已经有人训过并发布了。成品 LoRA 5 分钟到手，自训 LoRA 要花几小时并踩一堆环境坑。

只有确实没有成品时才走自训路线。

## 成品 LoRA 加载

### 关键兼容性问题

Civitai 的 LoRA 是完整 kohya 格式（包含 text encoder 键 `lora_te*`），但 diffusers 加载时会因为 `rank_dict` 为空而崩。解法：加载前剔除 `lora_te*` 键，按纯 UNet 字典加载。

```python
from safetensors.torch import load_file
import diffusers.loaders.lora_pipeline as lora_pipeline

state_dict = load_file(LORA_PATH)
state_dict = {k: v for k, v in state_dict.items() if not k.startswith("lora_te")}

original_te_loader = lora_pipeline._load_lora_into_text_encoder
def te_loader_or_skip(state_dict, *args, **kwargs):
    if not any("lora_te" in k or "text_encoder" in k for k in state_dict):
        return None
    return original_te_loader(state_dict, *args, **kwargs)
lora_pipeline._load_lora_into_text_encoder = te_loader_or_skip

pipe.load_lora_weights(state_dict)
```

### 底模选择

Illustrious 系底模（JANKU、NoobAI 等）对二次元角色效果好。加载方式用 `from_single_file`，需要传 config：

```python
pipe = StableDiffusionXLPipeline.from_single_file(
    BASE_MODEL_PATH, torch_dtype=torch.float16,
    config="stabilityai/stable-diffusion-xl-base-1.0"
).to("cuda")
```

VAE 推荐 `madebyollin/sdxl-vae-fp16-fix`。

## Prompt 架构

### 提示词翻译

场景描述用 LLM 翻译成 Danbooru 标签。翻译提示词保持朴素（「翻译成 Danbooru 标签，只输出标签」），不要加过多规则——LLM 会自作主张加 standing/full body 之类的标签，覆盖掉你要的构图。

### 构图兜底

需要固定构图时（如自拍），把构图标签手动插在触发词之后、LLM 标签之前：

```python
if "自拍" in scene:
    tags = "close-up, face focus, pov, from above, " + tags
```

插在末尾会被 CLIP 的 77-token 截断吃掉——这是最常踩的坑。

### 触发词

角色触发词是画风锚点，裁掉它整批图崩坏。核心触发词永不裁。

### 服装切换

默认校服装束写成常量。场景含非校服关键词（运动服/便服/居家等）时跳过默认校服，让 LLM 翻译的服装标签生效。这样同一个角色可以出不同服装。

### 负面词

必备：质量词 + 多人锁（`2girls` 等）+ 排除词。

多人锁是必备：同一角色同场景会出「两个花对打」，`1girl, solo` + negative 多人锁能锁死。

## 脸部增强

生成后用动漫人脸检测（ONNX）找最大脸 → 裁 768 重绘（strength 0.35）→ 羽化贴回。

```python
# face detection: deepghs/anime_face_detection (ONNX, CPU)
# crop face region, img2img at strength 0.35, paste back with Gaussian blur mask
```

## 高清精修

1.5x hires 二次精修（img2img strength 0.3，22 步）无条件开启。但注意：hires 会洗掉手势和 pose 指令，带手势的图应跳过 hires 或加权 pose。

## 接入 Agent 工具

写一个 agent tool，让角色能自主调用：

- 工具描述写清楚「生成你的照片并发给用户」，场景由角色自己理解
- 同一轮多次调用时自动轮转镜头/姿势（特写 → 全身 → 侧面 → 动态）
- 场景灵感池从角色性格/作息/原作设定提炼，降低同质场景概率
- 工具返回文件路径，框架自动发送；描述里明令禁止对返回路径再调 send（防重复发图）
- 生成是阻塞的（单张约 2 分钟），加锁防止并发

## 踩坑记录

- Civitai LoRA 断点续传需要校验文件完整性，多出几 MB 就会报 file not fully covered
- prompt 超 77 token 被静默截断，触发词表内词冲突（如 pants 与 short blue pants）会导致错误服装
- 自训 LoRA 前先搜 Civitai 成品——43 张图、3 次训练失败后弃案，成品 5 分钟到手
- 需求确认先行：「裁脸」「底模优缺点分析」两次被用户纠偏——动手前先问清用户要什么

