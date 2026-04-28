# video-make — 主题到成片的故事视频生产线

**Status**: Draft, awaiting user review
**Date**: 2026-04-28
**Owner**: Yikuanzz
**Related skill**: `video-use` (existing, in this repo)

---

## 1. 目标与非目标

### 目标

在现有 `video-use` 仓库内新增一个 sibling skill `video-make`，把**「一个主题/想法」**自动转化为**一条 9:16 竖屏 60 秒的中文故事/叙事类短视频**，端到端覆盖：

- 脚本生成（LLM）
- 配音 TTS
- 视觉素材生成（文生图 + Ken Burns 推拉）
- 失败兜底（stock）
- 字幕（中文整句样式）
- BGM 选曲与 ducking 混音
- 成片自检

整个管线**复用 `video-use` 已有的 12 条生产硬规则、helpers、ASR 转写骨架、渲染骨架、session memory**，但**不修改 `video-use/SKILL.md` 本身**（Scribe 在 video-make 里被 driver 化为可选 ASR provider，默认改用 Whisper）。

### 非目标（MVP 不做）

- ❌ AI 文生**视频**（可灵 / Sora / Runway）— 留给 v2
- ❌ 数字人 / 虚拟主播
- ❌ 多语言（MVP 只做中文；架构保留 language 参数）
- ❌ 实时热点 / 联网拉素材（赛事快讯类型）
- ❌ Music AI 生成 BGM（用本地内置 royalty-free 库）
- ❌ 图片风格强一致性（不做 IP-Adapter / reference-only），每张独立调用
- ❌ 自动翻译用户的中文 prompt 给英文模型（即梦/通义吃中文；GPT Image 由 LLM 直接写英文 prompt 字段）
- ❌ 横屏 16:9 + 长片（架构支持参数化，但 MVP 默认值固定竖屏 60s）

### 后续扩展（写在路线图里，不在 MVP 范围）

- 横屏 / 中长片 strategy
- 「赛事快讯」「生活颜值切片」两个内容型作为独立的 prompts 包 + 数据源插件
- AI 文生视频 driver 接口（按现有 image driver 模式）
- BGM AI 生成 driver
- 多 shot 风格一致性（reference image / IP-Adapter）

---

## 2. 设计原则（继承自 video-use）

1. **LLM 不做基础设施决策。** Provider 选择、driver 切换、安全兜底由 helper 内部和 `.env` 决定，LLM 只做创作决策（脚本、分镜、prompt、情绪）。
2. **所有中间产物可缓存、可复算、可回滚。** 每个文件落盘时算 input hash，hash 不变则跳过。
3. **两道 gate 拦在烧钱前面。** 脚本 gate 后只是又一次 LLM 调用；分镜 gate 后才进入 TTS/生图等付费环节。
4. **复用胜过重写。** ASR（driver 化后） / EDL / render.py / 自检 / project.md 全部继承。
5. **失败显式、不静默兜底音色。** TTS 失败抛错，绝不偷换 driver；生图失败先重试同 driver，再走 stock，再标记 ⚠️。
6. **自检是最后一道闸。** Cap 3 次循环；3 次过不去就把问题列给用户决策，不偷偷标成功。

---

## 3. 目录布局

```
video-use/  (repo root, 现状不变)
├── SKILL.md                     ← video-use, 不动
├── helpers/                     ← 现有，复用
│   ├── transcribe.py            ← 现有：Scribe 单文件入口（沿用，但不再是 video-make 的默认 ASR）
│   ├── pack_transcripts.py
│   ├── render.py                ← 扩展：新增 image_shot segment 类型
│   ├── timeline_view.py
│   ├── grade.py
│   ├── (新) ken_burns.py        ← 图 → 推拉视频
│   ├── (新) tts.py              ← TTS dispatch 入口
│   ├── (新) asr.py              ← ASR dispatch 入口（新管线默认 Whisper API for zh）
│   ├── (新) image_gen.py        ← 图生图 dispatch 入口（含 provider 级 semaphore）
│   ├── (新) align.py            ← ASR 时间戳对齐 + 短句合并
│   ├── (新) cache.py            ← 通用 hash + sidecar 缓存
│   ├── (新) cost_estimator.py   ← gate 2 之前的成本估算
│   ├── (新) script_validator.py ← 校验 [Sxx] 编号在脚本和分镜间一致
│   └── (新) check_drivers.py    ← 启动健康检查
├── skills/
│   ├── manim-video/             ← 现有
│   └── video-make/              ← 新增
│       ├── SKILL.md
│       ├── drivers/
│       │   ├── tts/{base.py, doubao.py, minimax.py, cosyvoice.py}
│       │   ├── asr/{base.py, whisper.py, paraformer.py, scribe.py}
│       │   ├── image/{base.py, jimeng.py, tongyi.py, gpt_image.py}
│       │   └── stock/pexels.py
│       ├── bgm/
│       │   ├── narrative_calm_01.mp3 ... (6-10 首)
│       │   └── manifest.json
│       └── prompts/
│           ├── script_story_zh.md
│           └── storyboard_zh.md
└── docs/
    └── superpowers/specs/2026-04-28-video-make-design.md  ← 本文件
```

**用户工作流**：

```bash
mkdir my-story && cd my-story
claude
> 用 video-make 给我做一个 60 秒的故事视频，主题是「外婆藏起来的那本日记」
```

输出全部落到 `<work_dir>/edit/` 内，与 video-use 一致：

```
my-story/
└── edit/
    ├── script.md                ← gate 1 (确认后只读)
    ├── storyboard.json          ← gate 2 (确认后只读，不再回填字段)
    ├── pipeline_state.json      ← 各阶段动态回填的字段（actual/final duration、cost 实际值等）
    ├── narration.mp3
    ├── transcripts/asr.json     ← ASR 结果（默认 Whisper；Scribe 路径仍可选）
    ├── shots/
    │   └── shot_001/{prompt.txt, image.png, frames/, audio.aac, render.mp4, .hash}
    ├── bgm.mp3
    ├── master.srt
    ├── edl.json
    ├── concat.mp4
    ├── concat_bgm.mp4
    ├── verify/                  ← 自检截图 + 报告
    ├── preview.mp4
    ├── final.mp4
    └── project.md
```

---

## 4. 端到端管线

```
0. Inventory + 唤醒 (验 key、读 BGM manifest、读 project.md)
   ↓
1. 对话 + 主题确认
   ↓
2. LLM 写脚本 → script.md  ─────────────────────────┐
                                                     │
                            ╔════════════════════╗   │
                            ║ GATE 1 ── 脚本审阅 ║   │
                            ╚════════════════════╝   │
                                     ↓               │
2.5. script_validator.py 提取 [Sxx] 编号清单         │
     → pipeline_state.json.script_segments           │
                                     ↓               │
3. LLM 写分镜 → storyboard.json                      │
   (引用上一步的 [Sxx] 清单，确保不漏不错)            │
                                     ↓               │
3.5. cost_estimator.py 算预估成本                    │
     (TTS 字数 + 生图 N 张 + ASR 总秒数)             │
     → 在 gate 2 摘要里展示给用户                    │
                                     ↓               │
                            ╔════════════════════╗   │
                            ║ GATE 2 ── 分镜审阅 ║   │
                            ║ + 成本预估展示     ║   │
                            ║ + segment 一致性校验║  │
                            ╚════════════════════╝   │
                                     ↓               │
              ┌──────────────────────┼──────────────────────┐
              ↓ (并行)               ↓ (并行)              ↓ (并行)
         4a. TTS 合成            4b. 每 shot 生图       4c. LLM 选 BGM
         narration.mp3           shots/N/image.png     bgm.mp3
                                 (provider semaphore)
              ↓                       ↓
         5a. ASR 转写            5b. Ken Burns 推拉
         (默认 Whisper)          shots/N/render.mp4
         word-level timing
              └──────────┬────────────┘
                         ↓
              6. 时间轴对齐（短句合并、padding） → edl.json
                 + 把 actual/final duration 回填到 pipeline_state.json
                 (storyboard.json 不动)
                         ↓
              7. render.py 拼接 + BGM 静态 volume automation + 字幕 LAST → preview.mp4
                         ↓
              8. 自检循环 (≤3 次)，timeline_view 在 shot 边界采样
                         ↓
              9. 用户预览 → final.mp4 + 写 project.md

         [并行支线] 用户在 gate 2 之后说「shot 3 改成黄昏」：
              → 走 §4.4 的局部重做路径，不重走 gate 2
```

### 两道 gate 的本质区别

| | gate 1 脚本 | gate 2 分镜 |
|---|---|---|
| 拦的是什么 | 内容方向错（讲错故事） | 视觉方向错（画面想错） |
| 重做成本 | 1 次 LLM 调用，秒级、几乎 0 成本 | 1 次 LLM 调用，秒级、几乎 0 成本 |
| 过了之后的成本 | 进 gate 2，仍便宜 | 进并行付费阶段（TTS、生图 × N、ASR）— gate 2 之前会展示估算 |
| 用户能看到的额外信息 | 脚本全文 + 估算总时长 | 分镜表 + **预估 API 成本 + segment 编号一致性校验结果** |

### 4.4 Shot 级别的局部重做（gate 2 之后）

用户在看到分镜或预览后说「shot 3 的画面改成黄昏」时，**不重走 gate 2**——直接走最小化重做路径：

```
1. 用户指定: shot_id="shot_003", 修改: visual_prompt 或 camera_motion 或 image_provider
2. 更新 pipeline_state.json.shot_overrides[shot_003]
   (storyboard.json 仍然不动 — 它代表「初次确认的版本」，是 source of truth 的快照)
3. 失效 shots/shot_003/ 下的 image.png / frames/ / render.mp4 (改 .hash)
4. 局部重生图（一个 sub-agent，不是全部）
5. 局部 Ken Burns
6. 重 concat.mp4 → 重混 BGM → 重 burn 字幕（按 §8.3 最小化重做策略）
7. 自检只检该 shot ±1 个邻居
```

允许的修改类型：`visual_prompt` / `camera_motion` / `image_provider` / `mood`。
**不允许**的（会触发"建议重走 gate 2"提示）：`script_segment` 关联变更、`target_duration_s` 大幅变更、`shots[]` 数量增删。这两类是结构性改动，应当回到分镜重审。

> **target_duration_s 的"大幅变更"判定**：`abs(new - old) > max(old * 0.20, 0.6s)`。短 shot（target < 3s）单纯按百分比会被 ASR word-timestamp 抖动（量级 0.2-0.4s）误判为大改；0.6s 绝对下限保证 2s 短 shot 至少有 0.6s 余量。一行代码：`tolerance = max(shot.target_duration_s * 0.20, 0.6)`。

`pipeline_state.json` 里的 `shot_overrides` 字典是单一可信来源，自检和后续 session 都从这儿读叠加值。

---

## 5. 关键数据 artifact

### 5.1 `script.md` (gate 1)

```markdown
# 外婆藏起来的那本日记

**主题**：…
**情绪走向**：怀旧 → 惊讶 → 温柔
**目标时长**：60 秒（估算 252 字 ≈ 56 秒，留 4 秒呼吸）
**TTS 音色建议**：豆包女声叙事系（具体 voice_id 由 storyboard 阶段从 driver 可选音色清单中选；这里写的是音色风格描述，不是 ID）
**BGM 情绪**：narrative_calm → melancholy

## 脚本

[S01] 外婆走了之后，我回到那间老屋整理遗物。 *(情绪：平静开场，节奏慢)*
[S02] 衣柜的最底层，有一个上了锁的木盒。 *(情绪：好奇，节奏渐起)*
…
```

`[S01]`/`[S02]` 编号是后续所有 artifact 的串联键。

### 5.2 `storyboard.json` (gate 2)

```json
{
  "version": 1,
  "topic": "...",
  "aspect": "9:16",
  "target_total_duration_s": 60,
  "narration": {
    "tts_provider": "doubao",
    "voice_id": "zh_female_xueji_001",
    "speaking_rate": 0.95
  },
  "bgm": {
    "track": "bgm/melancholy_01.mp3",
    "duck_db": -12,
    "fade_in_s": 1.0,
    "fade_out_s": 2.0
  },
  "subtitle_style": "natural-sentence-zh",
  "shots": [
    {
      "id": "shot_001",
      "script_segment": "S01",
      "quote": "外婆走了之后，我回到那间老屋整理遗物。",
      "target_duration_s": 6.0,
      "visual_prompt": "黄昏的老式中式房间，斜阳从木窗格洒进来…",
      "mood": "calm_melancholy",
      "camera_motion": "slow_zoom_in",
      "image_provider": "jimeng",
      "fallback_stock_query": "old chinese house dusty afternoon nostalgic",
      "easing": "ease_in_out_cubic",
      "reason": "开场建立场景，缓慢推近暗示情绪沉浸"
    }
  ]
}
```

时间字段约定（**storyboard.json 在 gate 2 后只读，不再回填字段**——actual / final duration 全部写到 `pipeline_state.json`，避免并行阶段的 race condition）：
- `target_duration_s` — gate 2 时 LLM 估算（写在 storyboard.json，永远不变）
- `actual_duration_s` — ASR 转写后写到 `pipeline_state.json.shots[].actual_duration_s`
- `final_duration_s` — 短句合并 / padding 后写到 `pipeline_state.json.shots[].final_duration_s` + EDL

`camera_motion` 是有限枚举：`slow_zoom_in / push_in / pan_left / pan_right / hold / pull_back / ken_burns_diag`。Ken Burns helper 按枚举映射 ffmpeg/PIL 关键帧，**LLM 不能自由发挥**。

`easing` 是可选字段；不写则用 `helpers/ken_burns.py` 里每种 motion 的默认 easing（`hold` 默认 `linear`，其他默认 `ease_in_out_cubic`）。LLM 在 storyboard 阶段可以为某个 shot 显式覆盖。

### 5.3 `pipeline_state.json`（运行时回填，gate 2 后才创建）

`storyboard.json` 是**用户确认的 source-of-truth 快照**，gate 2 之后不变；任何流水线产生的动态字段都落到 `pipeline_state.json`。这样并行 sub-agent 读 storyboard 时永远拿到一致状态，不会有 race。

```json
{
  "version": 1,
  "storyboard_hash": "sha256:...",
  "estimated_cost": {
    "tts_yuan": 0.05,
    "image_gen_yuan": 0.80,
    "asr_yuan": 0.02,
    "total_yuan": 0.87,
    "computed_at": "2026-04-28T14:32:00+08:00"
  },
  "actual_cost": {
    "tts_yuan": 0.05,
    "image_gen_yuan": 1.20,
    "asr_yuan": 0.02,
    "total_yuan": 1.27
  },
  "shots": [
    {
      "id": "shot_001",
      "actual_duration_s": 6.12,
      "final_duration_s": 6.12,
      "merged_into": null,
      "image_attempts": 1,
      "image_seed": 12345,
      "fell_back_to_stock": false
    },
    {
      "id": "shot_004",
      "actual_duration_s": 0.84,
      "final_duration_s": null,
      "merged_into": "shot_003",
      "image_attempts": 0
    }
  ],
  "shot_overrides": {
    "shot_003": {
      "visual_prompt": "黄昏的老式中式房间，斜阳更强烈一些...",
      "modified_at": "2026-04-28T15:10:00+08:00",
      "modified_reason": "用户在预览后要求改"
    }
  },
  "asr_provider_used": "whisper",
  "self_eval_passes": 2
}
```

### 5.4 `edl.json` (向后兼容 video-use 格式)

**版本号从 1 升到 2**——新版引入 `kind` 字段、`narration_track`、`bgm_track`，但**旧 v1 EDL 仍能被新 render.py 读懂**（`kind` 缺省按 `video_segment` 处理；没有 `narration_track`/`bgm_track` 时走旧路径）。

```json
{
  "version": 2,
  "kind_extensions": ["image_shot"],
  "narration_track": "edit/narration.mp3",
  "bgm_track": {
    "file": "edit/bgm.mp3",
    "duck_db": -12,
    "fade_in_s": 1.0,
    "fade_out_s": 2.0
  },
  "ranges": [
    {
      "kind": "image_shot",
      "id": "shot_001",
      "image": "edit/shots/shot_001/image.png",
      "duration": 6.12,
      "camera_motion": "slow_zoom_in",
      "ken_burns": {
        "zoom_from": 1.00,
        "zoom_to": 1.10,
        "easing": "ease_in_out_cubic"
      },
      "audio_segment": {
        "source": "edit/narration.mp3",
        "start": 0.00,
        "end": 6.12
      },
      "script_segment": "S01",
      "quote": "外婆走了之后，我回到那间老屋整理遗物。"
    }
  ],
  "subtitles": "edit/master.srt",
  "total_duration_s": 60.4
}
```

**向后兼容路由**：`kind: "image_shot"` 由 render.py 走新分支；`kind` 缺省（旧 v1 EDL）继续走 `extract_video_segment` 老分支。两类 segment 可以混用——意味着未来「自拍 + AI 生成」混合内容是免费扩展。

### 5.5 `bgm/manifest.json`

```json
{
  "version": 1,
  "tracks": [
    {
      "file": "narrative_calm_01.mp3",
      "moods": ["calm", "narrative", "intro"],
      "tempo_bpm": 72,
      "duration_s": 180,
      "loop_safe": true,
      "license": "CC0",
      "source_url": "...",
      "description": "钢琴+弦乐淡淡铺底，空灵但不悲。"
    }
  ]
}
```

LLM 选 BGM 时**只能从 manifest 中选**，避免幻觉一个不存在的文件。

### 5.6 字幕样式 `natural-sentence-zh`

video-use 现有 `bold-overlay` 是英文短型；新增中文整句样式：

```
FontName=Source Han Serif SC,FontSize=14,Bold=0,
PrimaryColour=&H00FFFFFF,OutlineColour=&H80000000,BackColour=&H00000000,
BorderStyle=1,Outline=2,Shadow=0,
Alignment=2,MarginV=70
```

宋体、半透明描边、整句一行、底部 70px 安全区。`render.py` 的 `SUB_FORCE_STYLE` 改成由 `subtitle_style` 字段查表选取（旧路径 `bold-overlay` 仍工作）。

---

## 6. Provider driver 抽象

### 6.1 配置

```
# 已有
ELEVENLABS_API_KEY=...                 # video-use 现存的 ASR；video-make 把它降级为 zh 次选

# 新增
VOLCANO_AK=...                         # 豆包 TTS + 即梦
VOLCANO_SK=...
DASHSCOPE_API_KEY=...                  # 通义万相 + CosyVoice + paraformer ASR
MINIMAX_API_KEY=...
OPENAI_API_KEY=...                     # GPT Image + Whisper API（zh ASR 默认）
PEXELS_API_KEY=...                     # stock 兜底

# Driver 选择（不写则用 MVP 默认）
VIDEOMAKE_TTS_DRIVER=doubao
VIDEOMAKE_IMAGE_DRIVER=jimeng
VIDEOMAKE_ASR_DRIVER=whisper           # 中文字级时间戳精度比 Scribe 更稳
VIDEOMAKE_STOCK_FALLBACK=pexels

# 并发限速（防 429）
VIDEOMAKE_IMAGE_MAX_CONCURRENCY_JIMENG=3
VIDEOMAKE_IMAGE_MAX_CONCURRENCY_TONGYI=3
VIDEOMAKE_IMAGE_MAX_CONCURRENCY_GPT_IMAGE=2
```

启动时 `helpers/check_drivers.py` 验所有 key + ping 默认 driver。**绝不在管线中途发现 key 缺失。**

### 6.2 TTS driver 接口

```python
class TTSDriver(Protocol):
    def synthesize(
        self, text: str, voice_id: str,
        speaking_rate: float = 1.0, out_path: Path,
    ) -> TTSResult: ...

@dataclass
class TTSResult:
    out_path: Path                     # 写出的 mp3
    duration_s: float                  # ffprobe 回填
    word_timings: list[WordTiming] | None  # 可选；driver 能给就给
```

**关键约定**：
- 输出统一 mp3 / mono / 44.1kHz
- 即使 TTS driver 自带 word timing，**仍然走一遍独立 ASR**（§6.3）——避免混用两种来源
- driver 失败抛异常，**不静默切换其他 driver**（音色变了用户察觉不到，反而更糟）

### 6.3 ASR driver 接口（**新增**——这是中文管线的关键）

ElevenLabs Scribe 在中文字级时间戳上经常以 100-300ms 范围漂移、且分词颗粒度不稳（有时是字、有时是词、有时是短语）。新管线**默认走 OpenAI Whisper API**（`timestamp_granularities=["word"]`），把 Scribe 降为可选项。

```python
class ASRDriver(Protocol):
    def transcribe(
        self,
        audio_path: Path,
        language: str = "zh",
        out_json: Path,
    ) -> ASRResult: ...

@dataclass
class ASRResult:
    out_json: Path
    duration_s: float
    words: list[WordTiming]   # 必填；这是新管线全部对齐逻辑的基础
    text: str

@dataclass
class WordTiming:
    text: str
    start_s: float
    end_s: float
    confidence: float | None
```

**Driver 实现优先级**：
- `whisper.py` (OpenAI Whisper API) — MVP 默认，中文字级精度可接受
- `paraformer.py` (阿里 paraformer) — 中文 native，准确度可能更高，可选
- `scribe.py` (ElevenLabs Scribe) — 沿用，但只作 zh 次选

**关键约定**：
- 所有 driver 输出统一 schema（上面 `ASRResult`）
- 中文场景下 `words[].text` 是字级单位（确保对齐分辨率最小）
- driver 失败抛异常，不自动 fallback 到其他 ASR（时间戳准确度涉及对齐稳定性，跨 driver 切换会带来不一致风险）

L4 smoke 测试要在三个 driver 上各跑一遍，对比结果，选出 zh 实测最稳的作为锁定默认。

### 6.4 图生图 driver 接口

```python
class ImageDriver(Protocol):
    def generate(
        self, prompt: str, aspect: str, out_path: Path,
        seed: int | None = None, style_hint: str | None = None,
    ) -> ImageResult: ...

@dataclass
class ImageResult:
    out_path: Path
    width: int
    height: int
    provider_metadata: dict            # seed, model_version, request_id
```

**Provider 级 semaphore**（防 429）：

```python
# helpers/image_gen.py
_SEMAPHORES = {
    "jimeng": Semaphore(int(os.getenv("VIDEOMAKE_IMAGE_MAX_CONCURRENCY_JIMENG", "3"))),
    "tongyi": Semaphore(int(os.getenv("VIDEOMAKE_IMAGE_MAX_CONCURRENCY_TONGYI", "3"))),
    "gpt_image": Semaphore(int(os.getenv("VIDEOMAKE_IMAGE_MAX_CONCURRENCY_GPT_IMAGE", "2"))),
}

def generate(prompt, ..., provider="jimeng"):
    with _SEMAPHORES[provider]:   # 8+ sub-agent 并发会串行排队
        return _DRIVERS[provider]().generate(...)
```

每个 sub-agent 调用 `image_gen.generate(...)` 而不是直接调 driver——semaphore 透明限流。**LLM 看到的还是「全部并行」，底层在 helper 层面被排队**。Hard Rule 10 仍然成立（语义上是并行）。

**生图失败兜底链**：

```
即梦 1 次 → 同 prompt 换 seed 1 次 → LLM 改写 prompt 1 次 →
Pexels 用 fallback_stock_query 搜 → 仍失败：在 EDL 里标 ⚠️ 让用户人工补图
```

### 6.5 成本估算（gate 2 之前调用）

`helpers/cost_estimator.py` 在 storyboard 写完后、用户审阅前调用一次，把估算结果写到 `pipeline_state.json.estimated_cost`，并由 gate 2 摘要展示给用户：

```
预估 API 成本（gate 2 通过后会真实调用）：
  TTS 豆包:     约 252 字 × ¥0.00015 = ¥0.04
  生图 即梦:    约 10 张 × ¥0.08 = ¥0.80
  ASR Whisper:  约 60 秒 × ¥0.0006 = ¥0.04
  ─────────────────────────────────
  合计:                          ≈ ¥0.88
  (估算上下浮动 ±50%。重试和兜底会增加成本。)
```

每个 driver 在 `drivers/<kind>/<name>.py` 暴露 `estimate_cost(units)` 函数，cost_estimator 按 storyboard 结构汇总。

实际花费（含失败重试）写到 `pipeline_state.json.actual_cost`，自检通过后 final report 展示。

### 6.6 决策权分配

| 决策 | 谁决定 | 何时 |
|---|---|---|
| 用哪个 TTS provider | 用户（`.env`） | session 开始 |
| 哪个音色 | LLM 在 storyboard 写 `narration.voice_id` | gate 2 |
| 哪个 image provider | 用户（`.env`） | session 开始 |
| 哪个 ASR provider | 用户（`.env`） | session 开始 |
| 单张图的 prompt | LLM | gate 2 |
| 失败时降级 stock | helper 自动 | retry 后 |

**LLM 不切换 provider**——provider 是基础设施配置，不是创作决策。该规则写进 SKILL.md。

---

## 7. 渲染管线扩展

### 7.1 image_shot → 视频片段

每个 `image_shot` 的产物是一个 mp4，包含**带 Ken Burns 推拉的画面 + 对应那段 narration 音频**。

**Ken Burns 实现：PIL + PNG 序列法**（不用 ffmpeg `zoompan`），三个理由：

1. video-use 已有先例（动画 overlay 就是这种模式）
2. 缓动函数自由（cubic / quart / quint 都能写；`zoompan` 表达式不支持 `pow`）
3. 像素级可预期，无 zoompan 的 subpixel 漂移问题

新增 `helpers/ken_burns.py`：

```python
def render_ken_burns(
    image_path: Path,
    duration_s: float,
    motion: str,                        # 枚举：见 5.2
    output_size: tuple[int, int],       # (1080, 1920) for 9:16
    fps: int = 30,
    easing: str = "ease_in_out_cubic",
) -> Path: ...
```

**`motion` → 关键帧映射表**（写死）：

| motion | 起始裁框 | 终止裁框 | 默认 easing |
|---|---|---|---|
| `slow_zoom_in` | 100% 居中 | 110% 居中 | `ease_in_out_cubic` |
| `push_in` | 100% 居中 | 115% 居中 | `ease_out_cubic` |
| `pan_left` | 110% 偏右 | 110% 偏左 | `ease_in_out_cubic` |
| `pan_right` | 110% 偏左 | 110% 偏右 | `ease_in_out_cubic` |
| `hold` | 105% 居中 | 105% 居中（轻微 idle 漂移） | `linear`（cubic 对静止帧无意义） |
| `pull_back` | 115% 居中 | 100% 居中 | `ease_out_cubic` |
| `ken_burns_diag` | 110% 左上 | 110% 右下 | `ease_in_out_cubic` |

> 起止都不到 100%——避免下采到 1080×1920 后出现黑边。
> easing 列是 storyboard 不显式指定时的默认值；LLM 可在 storyboard 的 shot 字段里覆盖。

### 7.2 render.py 改动点

EDL 处理循环新增 `kind: "image_shot"` 分支：

```python
for r in edl.ranges:
    if r.kind == "image_shot":
        clip = render_image_shot(r, narration_path)
    else:
        clip = extract_video_segment(r)        # 现有路径
    clips.append(clip)
```

`render_image_shot` 4 步：
1. 从 `narration.mp3` 用 `-ss` `-to` 切出 `audio_segment` → 临时 aac
2. 调 `ken_burns.render_ken_burns(...)` 出无音轨 mp4
3. mux 1 和 2 → segment mp4
4. 加 30ms afade（Hard Rule 3）—— 在 segment 内就加好

合好后进入现有 concat（`-c copy` 不重编码，Hard Rule 2）。

### 7.3 BGM 混音 + 静态时间段 ducking（**deterministic，不用 sidechaincompress**）

**为什么不用 sidechaincompress**：sidechain 压缩比受 threshold、输入电平、makeup gain 共同决定，**不同响度的 BGM 曲目跑出来的实际 ducking 深度天差地别**——`ratio=8` 在某首曲目降 8dB，在另一首可能只降 3dB。这违背了 "中间产物可重复" 的原则。还有第二个问题：shot 之间有 50ms padding（Rule 7），padding 期间 narration 静音，sidechain 会瞬间释放，造成**每个 cut 处都有一个小 pump**——8-12 个 cut 的 60s 视频会被这个 pump 毁掉。

**改用：基于 ASR word-level 时间戳的静态 volume automation。**

ASR（§6.3）已经把 narration 切成 word_timings 列表。我们在编译时把词聚合成"句"，再为每句生成一个明确的 duck 窗口；多个窗口若重叠则合并；最后按时间段编一条确定性的 ffmpeg `volume` 滤镜表达式。

**窗口构造规则（写死）**：

```
sentence  = 连续 word，相邻 word 间隔 < 0.4s 视为同一句（gap_threshold=0.4）
duck_start = sentence_start - 80ms        # 提前 80ms 压下，避开元音突然爆出
duck_end   = sentence_end   + 200ms       # 延后 200ms 释放，覆盖句尾尾音 + 50ms cut padding
合并        = 若 windows[i].end >= windows[i+1].start，合并为一段
```

`-80ms / +200ms` 是写死常量。它们的存在不是 attack/release 这种动态包络，而是**静态边界外扩**——expression 一旦编出来，输出就是确定的台阶函数（在窗口边界用一阶 cubic 过渡 30ms 抹平瞬变即可，这点 30ms 也是写死的）。

```python
# helpers/audio_mix.py 伪代码
def build_bgm_automation(words, total_dur, duck_db,
                         pre_lead_s=0.08, post_tail_s=0.20,
                         edge_smooth_s=0.03):
    """
    把 word_timings 合并成「句子段」（连续词之间间隔 < 0.4s 视为同一句），
    每句构造 [start - 0.08, end + 0.20] 的 duck 窗口；重叠窗口合并。
    输出 ffmpeg volume filter 表达式：volume='if(in_window, duck_db, 0)' 形式，
    窗口边界用 30ms cubic 过渡抹平（避免离散跳变可听见）。
    """
    sentences = merge_words_into_sentences(words, gap_threshold=0.4)
    windows = [(s.start - pre_lead_s, s.end + post_tail_s) for s in sentences]
    windows = merge_overlapping(windows)
    return compile_volume_expr(windows, duck_db, edge_smooth_s)
```

ffmpeg 链：

```
[1:a]aloop=loop=-1:size=2e9,atrim=0:{total_dur}[bgm_loop];
[bgm_loop]volume='{automation_expr}',
  afade=t=in:st=0:d={fade_in},
  afade=t=out:st={total-fade_out}:d={fade_out}[bgm_ducked];
[0:a][bgm_ducked]amix=inputs=2:duration=longest:dropout_transition=0[mixed]
```

四件事：BGM 循环到总时长 / **基于 sentence 窗口的 duck 自动化**（替代 sidechain）/ 进出渐变 / 简单 amix 叠加。

`storyboard.bgm.duck_db` 现在是**目标 dB 值**（直接喂到 volume 表达式），不再是"映射到 ratio"的玄学。`-12dB` 就是真的降 12dB。

**关于 50ms cut padding**：cut 边界出现在某句话结束之后。`duck_end = sentence_end + 200ms` 已经把 sentence 后 200ms 全压在 duck 里——50ms padding 整段被这 200ms 覆盖，BGM **不会** 在 cut 边界浮起来，pump 问题从根上消除（不需要借任何动态 release 概念）。

**为什么这是更可靠的设计**：
- 完全确定性：同一份 ASR 输出 → 同一组 sentence 窗口 → 同一份 volume 表达式 → 同一种听感，跨 BGM 曲目一致
- 离线可计算：cost_estimator 阶段就能把窗口列表画出来给用户看（v1.1 可以做）
- 故障可定位：BGM 没 ducking 的 bug 一定在 word_timings、sentence 切分或 expression 生成，三处都是纯函数，可单元测试

### 7.4 字幕路径

不走 video-use 的"从 takes_packed.md 推 SRT"——那个路径假设有原始视频。新管线两步：

1. 构造 logical SRT：每条 EDL `image_shot` 的 `quote` + `audio_segment` start/end 直接成 SRT 条目
2. 用 ASR word-level（§6.3 driver，默认 Whisper）做精修（可选）：动效字幕拆到词级别

预设字幕样式 `natural-sentence-zh`（5.6）。Hard Rule 1 不动：字幕 burn 是最后一步。

### 7.5 12 条硬规则的对位

| Hard Rule | 在新管线里的体现 |
|---|---|
| 1. 字幕 LAST | 不变 |
| 2. 逐段抽取无损 concat | image_shot 也走 segment 模型 |
| 3. 30ms afade | render_image_shot 里就加上 |
| 4. overlay PTS shift | image_shot 没 overlay；动画 overlay 仍按规则 |
| 5. master SRT 输出时间轴偏移 | 不变 |
| 6. 不切词中间 | TTS 整句合成 → ASR word-level（§6.3）→ 边界一定在词上 |
| 7. cut padding 30-200ms | shot 边界自动加 50ms |
| 8. word-level verbatim ASR | 走 §6.3 ASRDriver（默认 Whisper），保持原义 |
| 9. cache transcripts per source | narration 当作 single source 缓存 |
| 10. 并行 sub-agents | image gen 一定并行（每 shot 一个 sub-agent） |
| 11. strategy confirmation | gate 1 + gate 2 |
| 12. 输出落 edit/ | 不变 |

**新增 1 条专属规则（Rule 13）**：**短句合并**——ASR 实测时长 < 1.2s 的 shot 合并到前一个，避免画面闪过。

---

## 8. 自检 + 错误处理

### 8.1 分阶段失败模式

| 故障类 | 阶段 | 处理 |
|---|---|---|
| 脚本超长 / 涉敏 | §2 | LLM 自审 → 重写 1 次 → 仍不过交用户 |
| 分镜 shot 数爆炸（>15） | §3 | LLM 自审 → 合并相邻 shot |
| target_duration 总和不对 | §3 | LLM 自审 + reflow |
| TTS API 限流/超时 | §4a | 指数退避 3 次 |
| TTS 音色错配 / 鉴权失败 | §4a | 立即抛错给用户 |
| TTS 输出长度异常 | §4a | ffprobe 验长度，异常重试 1 次后报错 |
| 单张图失败 | §4b | 同 prompt 换 seed 1 次 |
| 内容审核拒绝 | §4b | LLM 改写 prompt 再 1 次 |
| 仍失败 | §4b | Stock 兜底（用 fallback_stock_query） |
| Stock 也搜不到 | §4b | EDL 标 ⚠️，让用户人工补图 |
| ASR 转写失败 | §5a | 重试 1 次；仍失败拦下管线（不自动切换其他 ASR driver——跨 driver 时间戳不一致） |
| ASR 词时间戳不全 | §5a | 报警 + 让 LLM 看 narration 是否有问题；可在 `.env` 切换备用 ASR driver |
| shot duration 对不上脚本句 | §6 | 短句合并规则触发 |
| 总时长溢出 ±10% | §6 | 重新分配 padding 或合并 shot |
| ffmpeg 失败 | §7 | 报命令 + 退出码，让用户介入 |
| 渲完时长不对 | §7 | ffprobe 校验 → 重 render 1 次 |
| 自检发现问题 | §8 | 按层次最小化重做（≤3 次） |

### 8.2 成片自检检查项

`timeline_view` 在每个 shot 边界采样：

| 检查项 | 怎么检 | 失败标准 |
|---|---|---|
| 字幕被遮 | 边界 PNG 字幕区像素差 | 与上一帧差 < 5% |
| 边界跳变 | 边界前后帧像素直方图比对 | 整图 hue 突变 > 30% |
| 音频爆 | 边界 ±50ms 波形最大幅 | > -1 dBFS |
| BGM 没 ducking | duck 窗口 vs 静默期的 BGM RMS（500ms 滑窗） | 差值 < 6dB |
| 总时长 | ffprobe | 与 EDL 偏差 > 0.3s |
| 分辨率 | ffprobe | 不是 1080×1920 |
| 黑边 | shot 中点采样检测黑色边缘 | 检出黑边 |

> **BGM ducking 量测细节**（写进自检脚本注释，不进验收条文）：用 500ms RMS 滑窗对 mixed audio 在 BGM-only 通道做能量计算；duck 窗口取 §7.3 sentence 窗口列表的并集，静默期取 BGM 的 fade-in 之后、第一个 sentence 窗口之前的纯 BGM 段。500ms 窗避免单帧噪声 / 元音瞬态触发误报。

video-use 现有自检不检 BGM、不检黑边——这两条是新增的。

### 8.3 失败修补：按层次最小化重做

```
字幕被遮     → 字幕样式 MarginV 调高 → 只重 burn 字幕
边界跳变     → 转场前后加 200ms crossfade → 重 concat
音频爆       → afade 时长拉到 50ms → 重 render 那两 segment + 重 concat
BGM 没 ducking → 检查 word_timings 完整性 + 重生 volume automation 表达式 → 重混音
总时长偏差   → 找哪个 shot 漂了 → 局部重 render
画面黑边     → Ken Burns 起点 zoom 提到 1.10 → 重 render 那几个 shot
```

不要一遇问题就全盘重 render。

### 8.4 cap 3 次后

- `edit/verify/issues.md` 写清问题位置 + 现象 + 已尝试 + 建议人工动作
- 仍给用户 `preview.mp4` 看
- `project.md` 记下未解决问题
- **不偷偷标成功**

### 8.5 中间产物的幂等性

| 输入 | 中间产物 | 失效条件 |
|---|---|---|
| script.md + tts driver + voice_id | narration.mp3 | 任一变 → 重生成 |
| narration.mp3 + asr driver | transcripts/asr.json | 内容或 driver 变 → 重转写 |
| visual_prompt + image driver + seed | shots/N/image.png | 任一变 → 重生图 |
| image + duration + motion | shots/N/render.mp4 | 任一变 → 重 Ken Burns |
| EDL | concat.mp4 | EDL 变 → 重 concat |
| BGM 选择 + ducking 参数 | concat_bgm.mp4 | 任一变 → 重混音 |

每个产物落盘前算 input hash 写进 sidecar `.hash`。下次跑时比对，hash 一致跳过。

---

## 9. 测试策略

四层金字塔：

```
L4: 端到端 smoke   ← 真 API、真出片，发布前手动跑
L3: golden EDL → render ← 固定 EDL，校验结构属性，CI 跑
L2: 集成（mocked）  ← driver dispatch、流程，CI 跑
L1: 单元           ← 纯函数，CI 跑
```

### 9.1 L1 单元

只测纯函数。最有价值：
- Ken Burns 关键帧插值 (`compute_crop_at`)
- 短句合并 (`merge_short_shots`)
- EDL 时长校验 (`verify_edl_durations`)
- Hash 比对 (`cache.py`)
- `motion` → ffmpeg 表达式映射

### 9.2 L2 集成（mocked）

`responses` / `pytest-mock` mock 所有 driver。重点：
- Driver dispatch（环境变量切换 driver）
- 失败兜底链
- Gate 流程（没"通过"信号绝不进下一步）
- Cache 命中
- Hard Rule 11（用户没确认分镜，render 不被触发）

### 9.3 L3 golden EDL → render

固化 3 个预制 EDL（手工写死、配预制图片和 narration）：

```
tests/fixtures/golden/
├── 60s_vertical_5shots/
├── 30s_vertical_3shots/
└── 60s_horizontal_4shots/
```

测试：`render.py edl.json -o out.mp4` → ffprobe 取 duration / size / codec → 与 expected/ 比对。

**不做像素级 diff**，只验**结构性属性**。

### 9.4 L4 端到端 smoke

发布前手动 3 条预制 prompt：

```bash
./run_smoke.sh story_diary
./run_smoke.sh story_first_snow
./run_smoke.sh story_old_photo
```

产物：`final.mp4` + `smoke_report.md`（分阶段耗时、API 调用数、成本估算、自检结果）。会烧钱，发布前才跑。

### 9.5 LLM 提示词回归

跑 5 次同主题，断言 schema 形状（shot 数 8-15、target_duration 总和 [55,65]）。允许内容浮动、形状稳定。落 `tests/llm_regression/<date>.jsonl`，目视 diff 看提示词是否退化。

不进 CI，定期人工巡检。

### 9.6 启动健康检查

`helpers/check_drivers.py` 启动跑：
- ffmpeg / ffprobe / Python 版本
- 必需的 API key
- BGM library 加载
- 默认 driver ping（不烧 token，只算 ping）

---

## 10. 路线图

### MVP（本 spec）

故事/叙事、9:16 60s、中文、文生图+Ken Burns+stock 兜底、内置 BGM 库、两道 gate。

### v1.1

- 横屏 16:9 + 中长片 strategy
- 横屏字幕样式预设
- 多 shot 风格一致性（探索 reference image）

### v1.2

- 「生活颜值切片」内容型：竖屏、可无 narration、音乐节拍驱动剪辑
- 可选 prompts 包，独立 mini-spec

### v2

- AI 文生视频 driver（按现有 image driver 模式接）
- 「赛事快讯」内容型：需要联网拉数据、合法数据源（Sports Open Data 等）独立 mini-spec
- BGM AI 生成

---

## 11. 待回答 / 风险

- **BGM 库要谁去搜罗？** MVP 至少 6 首 CC0 / royalty-free 的中文叙事 BGM。可在 incompetech / Pixabay Music / YouTube Audio Library 搜，落到 `bgm/` 目录。这一步可能需要用户协助挑选。
- **豆包 / 即梦 / Whisper / paraformer API 申请门槛**：火山引擎、阿里云百炼、OpenAI 各自控制台开通对应模型权限。文档写进 `install.md`。
- **ASR 中文字级时间戳横评**：发布前 L4 smoke 在 Whisper / paraformer / Scribe 三个 driver 上各跑一遍同一段 narration，比 word boundary 稳定性，把横评结果回写到 `install.md` 里作为推荐配置依据；若发现 Whisper 不够稳，可在不改任何代码的情况下切到 paraformer。
- **TTS 音色 ID 风险**：MVP 默认 driver 的真实可用 voice_id 需要 driver 实现时枚举出来落到 `drivers/tts/<name>.py` 的 `AVAILABLE_VOICES`，避免 LLM 在 storyboard 随手写一个不存在的 ID 导致 gate 2 通过后才报错。

---

## 12. 验收标准

MVP 视为完成的硬指标：

- [ ] 一行 `cd <dir> && claude && "用 video-make 做..."` 能从主题端到端跑出 `final.mp4`
- [ ] 两道 gate 工作正常，不点头不进下一步
- [ ] 三个 smoke 案例全部通过自检（≤3 次）
- [ ] L1 + L2 + L3 测试全绿
- [ ] 用户改一句脚本只重跑相关 shot（cache 生效）
- [ ] 用户改一个 shot 的 visual_prompt 只重跑该 shot ±1（§4.4 局部重做生效，pipeline_state.json.shot_overrides 正确写入）
- [ ] 任一外部 driver 失败有清晰错误信息（不静默 swallow，不偷换 driver）
- [ ] gate 2 摘要展示 `pipeline_state.json.estimated_cost` 的预估金额
- [ ] BGM ducking 听感：narration 期间 BGM 比静默期低 ≥ 6dB，且每个 cut 边界无 pump（自检 §8.2）
- [ ] `final.mp4` 满足：1080×1920、24/30fps、字幕清晰、BGM ducking 听感正确、无音频爆、shot 边界平滑
- [ ] `install.md` 已更新：包含豆包 / 即梦 / Whisper / paraformer / Pexels 的 API 申请指引、各 driver 的 `.env` 切换示例、新 BGM library 的最小素材清单与替换方法

---

## 附录 A：与 video-use 的硬规则关系一览

video-use 12 条 + video-make 新增 1 条 = 13 条硬规则共同生效，全部写进 video-make 的 SKILL.md。video-use 自己的 SKILL.md 不动。

## 附录 B：未来扩展的接口稳定性承诺

- driver 接口（`TTSDriver.synthesize`、`ImageDriver.generate`、`StockDriver.search`）签名稳定，新 driver 只需实现接口即可接入
- EDL 格式向后兼容（`kind` 字段可扩展）
- `camera_motion` 枚举可扩展，新增的需要在 `helpers/ken_burns.py` 同步加映射

## 附录 C：v2 文生视频接入路径预览

文生视频时把 `kind: "image_shot"` 改造为 `kind: "ai_video_shot"`：
- 不再 Ken Burns，直接用 driver 返回的 mp4
- duration 由 driver 输出长度决定（与 storyboard 估算对齐）
- 失败兜底链：AI 视频 → 单帧抽出做 Ken Burns → stock 视频
- 主管线和 EDL 结构不变
