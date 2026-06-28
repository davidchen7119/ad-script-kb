---
name: ad-script-kb
description: |
  This skill manages a local knowledge base for advertising/promotional video script analysis. 
  It handles the complete pipeline: downloading videos from platforms like xinpianchang.com, 
  extracting audio, transcribing via OpenAI Whisper, performing director-level copy analysis, 
  and archiving into a searchable markdown knowledge base with 11-style classification.
  
  Use this skill when the user:
  - Drops video files or video links into an inbox folder for processing
  - Asks to extract narration/voiceover from a promotional video
  - Requests director-level analysis of ad copy structure, style, narrative flow, and creative elements
  - Wants to build or query a knowledge base of advertising scripts and analysis
  - Provides links from xinpianchang.com, B站, or similar video platforms
  - Mentions setting up a "文案知识库" (script knowledge base) or "解说词分析" (narration analysis)
agent_created: true
---

# 广告解说词知识库 · 完整工作流

## Overview

This skill manages an end-to-end pipeline for analyzing promotional video scripts: downloading/extracting video → transcribing narration → performing director-level copy analysis → archiving into a structured, searchable knowledge base. The knowledge base uses an 11-style classification system across 5 categories (政务城市/企业商业/文旅人文/科教医疗公益/小众特色).

## Prerequisites

Before any processing, verify these tools are available:
- `yt-dlp` — for downloading videos from video platforms
- `ffmpeg` + `ffprobe` — for audio extraction and video metadata
- `whisper` (OpenAI Whisper, medium model) — for speech-to-text transcription

## Knowledge Base Initial Setup (Phase 0)

If the knowledge base does not yet exist at the configured location (default: `~/Downloads/文案知识库/`), create the full directory structure:

```
文案知识库/
├── inbox/
├── archive/
│   ├── 政务城市/
│   ├── 企业商业/
│   ├── 文旅人文/
│   ├── 科教医疗公益/
│   └── 小众特色/
├── index.md
└── templates/
    └── style_reference.md
```

The `index.md` and `templates/style_reference.md` templates are documented in `references/kb-structure.md` and `references/style-classification.md`. Initialize them with the full 11-style classification table, empty search indexes, and the frontmatter template.

## Core Workflow

When the user drops a video file into `inbox/` or provides a video link, execute the following pipeline in order. Do NOT skip steps.

### Step 1: Detect Input Type

Check `inbox/` for new files:
- **Video link** (e.g., `https://www.xinpianchang.com/aXXXXX`) → go to Step 2a
- **Local video file** (e.g., `inbox/some_video.mp4`) → go to Step 2b
- **No file** → inform user and wait

### Step 2a: Download Video (from Link)

Use `yt-dlp` to download at 360p resolution. Example:

```bash
yt-dlp -f "bestvideo[height<=360]+bestaudio/best[height<=360]" --merge-output-format mp4 -o "inbox/{output_filename}.mp4" "{URL}"
```

- Use `ffprobe` to get video duration: `ffprobe -v quiet -show_entries format=duration -of default=noprint_wrappers=1:nokey=1 {file}`
- Fetch the video's webpage for title/brand/industry context using WebFetch.

### Step 2b: Process Local Video File

- Use `ffprobe` to get duration
- Proceed directly to audio extraction

### Step 3: Extract Audio

Extract monaural 16kHz WAV for Whisper:

```bash
ffmpeg -i "inbox/{video_file}" -ar 16000 -ac 1 -vn /tmp/{id}_audio.wav -y
```

### Step 4: Transcribe with Whisper

Use the `medium` model with Chinese language:

```python
import whisper
model = whisper.load_model('medium')
result = model.transcribe('/tmp/{id}_audio.wav', language='zh', fp16=False)
with open('/tmp/{id}.txt', 'w', encoding='utf-8') as f:
    for seg in result['segments']:
        f.write(f"[{seg['start']:.1f}-{seg['end']:.1f}] {seg['text'].strip()}\n")
```

Run via Python with `PYTHONPATH` set correctly. Use background execution (`run_in_background=true`).

**🚨 不可打断协议（CRITICAL — 两次违规后强化）**：

Step 4 启动后，**必须**在同一轮 tool call batch 中用 `sleep 180`（3 分钟）作为首次等待，之后检查文件是否生成。转写完成后，**禁止**向用户输出任何确认消息或停顿，直接在同一条消息中无缝衔接执行 Step 5–9。

具体规则：
1. **同一 batch 启动 + 分段等待**：`Bash(run_in_background=true)` 启动 Whisper 后，在同一组 tool calls 中发出 `Bash("sleep 180 && test -f /tmp/{id}.txt || echo NOT_READY")`，3 分钟后检查文件
2. **分段重试**：若首次返回 `NOT_READY`，再 `sleep 60` → 检查；仍未完成则 `sleep 60` 再试。最多重试 3 次（总等待 ≤ 6 分钟），超过则改用 `TaskOutput(block=true, timeout=120000)` 兜底
3. **不中断**：转录文件生成后，立即 Read 并在同一回复中完成 Step 5 校正 + Step 6 分类 + Step 7 写入分析文件 + Step 8 归档索引 + Step 9 日志更新
4. **禁止的行为**：
   - ❌ 转写启动后输出「正在转写中，稍后通知你」然后停止
   - ❌ 转写完成后输出「转写完成，现在来写分析」然后等待下一轮
   - ❌ 在任何步骤之间停下来等待用户推动
   - ❌ 整个 Step 4–9 链条中，任何单个 step 完成后输出聊天消息但不继续下一步
5. **允许的例外**：仅当分析文件写入工具报错（如 InternalServerError）时，可以在下一轮重试写入，但仍需从断点继续而非从头开始

**⚠️ 分段等待节奏说明**：

绝大多数宣传片时长在 2–5 分钟，CPU Whisper 转写耗时约 3–5 分钟。以 3 分钟作为首次检查点，覆盖 80% 以上的场景：
- **首次检查**（3 min）：大部分视频此时已完成转写，直接进入 Step 5
- **二次检查**（4 min）：少量较长的视频需要额外 1 分钟
- **三次检查**（5 min）：极少超 5 分钟的视频
- **兜底**（> 5 min）：`TaskOutput(block=true)` 阻塞等待最长 2 分钟

等待期间只用极简短句（如「转写中，3 分钟后检查」「再等 1 分钟」）更新状态，不展开解释、不提问、不求反馈。

### Step 5: Manual Correction of Transcription

Whisper produces errors — especially with:
- Chinese technical terms (出版 → 初版, 赋能 → 富能, 元宇宙 → 云宇宙)
- Place names and proper nouns
- Financial/government terminology
- Four-character idioms
- Policy and political terms
- Homophonic characters

**Correction rules:**
1. Fix obvious homophonic errors by semantic context
2. Cross-reference against the video's webpage/description for brand/industry terms
3. Mark genuinely unclear sections as `【听不清】` — never fabricate content
4. Document all corrections in a table at the end of the analysis file
5. If uncertain about a highly specific term (e.g., corporate strategy names like "四横七纵"), note it for user verification

### Step 6: Determine Classification

Load `references/style-classification.md` (if not already in context). Based on the corrected transcript:

1. **Determine primary style (主风格)**: Which of the 11 styles dominates >50% of the content?
2. **Determine secondary style(s) (次风格)**: Are there clearly distinct sections (≥20%) using a different style? Max 2.
3. **Determine category (分类)**: Which of the 5 categories does the content belong to?
4. **Determine industry tags**: Extract 2-4 industry/labels (e.g., `银行 / 金融服务`, `文化出版 / 国有文化企业`)

### Step 7: Write Director-Level Analysis

Load `references/analysis-template.md`. Write the complete markdown analysis file following the exact structure:

1. **解说词转写** — Clean, role-segmented transcription (no timestamps)
2. **文案分析** — Six sub-sections:
   - 2.1 文案风格 (tone, rhythm, emotion, narrative sense, language features)
   - 2.2 文案结构 (hook, unfolding, pacing, emotional peak, closure)
   - 2.3 主题与主旨 (theme, information structure, value points, implicit claims)
   - 2.4 文案脉络与串线方式 (narrative thread, key point arrangement, path to ending)
   - 2.5 导演视角解读 (visual expression, rhythm & lens language, music, emotion→camera transitions)
   - 2.6 风险与可优化点 (risks, blind spots, structure/rhythm issues with optimization suggestions)
3. **可借鉴的文案创意元素** — 8-12 items, each with: name/feature + why effective + applicable scenarios
4. **行业 × 主题拓展** — 3 industries × 3 themes each (9 directions), each with: 1-2 sentence creative brief + 15-30 character sample script
5. **附：转写校正说明** — Correction table (original → corrected → basis)

Write the file to `inbox/{filename}_解说词导演视角深度分析.md`.

### Step 8: Archive & Index

Load `references/kb-structure.md` for the index structure.

1. **Move analysis file** to the appropriate archive subfolder:
   ```bash
   cp "inbox/{file}" "archive/{category}/{file}" && rm "inbox/{file}"
   ```

2. **Update `index.md`** in four places:
   - **已入库文档** table: Add new row with sequential number, doc name, category, primary/secondary styles, industry/brand, creative elements summary, date
   - **按风格检索速查**: Add entry under the primary style and each secondary style section
   - **按行业检索速查**: Add or update the industry row
   - **按创意元素检索速查**: Add each creative element as a new row
   - If the style combination is new: add row to **风格混合使用参考** table

3. **Delete the video file** from inbox:
   ```bash
   rm "inbox/{video_file}"
   ```

4. **Clean up temp files**:
   ```bash
   rm -f /tmp/{id}_audio.wav /tmp/{id}.txt
   ```

### Step 9: Update Memory Log

Append a log entry to `.workbuddy/memory/YYYY-MM-DD.md` with:
- Video source, file size, duration
- Transcription quality (segment count, correction count)
- Style classification and rationale
- Number of creative elements identified
- Industry × theme expansion directions
- Any notable issues or observations

Update `.workbuddy/memory/MEMORY.md` if any workflow refinements or user preferences emerged.

## Key Conventions

- **User's inbox**: The user ONLY drops files into `inbox/`. They never manually edit `index.md` or move files between archive folders.
- **Style classification**: Always justify style assignments with specific evidence from the transcript. Never assign a style without reasoning.
- **Correction integrity**: Never fabricate content for unclear audio. Always mark with `【听不清】` and note it for user verification.
- **Creative element extraction**: Each element must be concrete and reusable — not vague observations. Include why it works (psychological/narrative basis) and what types of ads it suits.
- **File naming**: Use `{brand}_{title keyword}_解说词导演视角深度分析.md` pattern. Keep hyphens/underscores consistent.
