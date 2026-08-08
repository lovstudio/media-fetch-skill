---
name: lovstudio-media-verification
description: >
  使用 ffprobe 验证已下载视频的可读性、时长、分辨率、编码、音轨、字幕和剧集完整性；适用于“检查下载是否完整”、"verify the downloaded media"，输出结构化验收报告。
license: MIT
metadata:
  author: lovstudio
  version: "0.1.0"
  tags:
    - ffprobe
    - media-validation
    - subtitle-audit
  compatibility: "Python 3.9+ and ffprobe from FFmpeg."
  dependencies:
    - python>=3.9
    - ffprobe
---

# Media Verification

Confirm the local artifact, selected edition, and language coverage before declaring
the request complete.

## Triggers

### Activate when

- 用户说“检查下载是否完整”“确认是不是导演剪辑版和中英字幕”。
- 下载客户端显示完成，需要验证实际本地文件。
- The user asks to verify the downloaded media or inspect its audio and subtitles.

### Do not activate when

- 用户只想搜索候选或测速，尚无完成文件。
- 用户要重新压制、转码、剪辑或翻译字幕。
- The user asks only for a plot summary or release recommendation.

## Workflow (MANDATORY)

### Step 0: Resolve expected truth

Load the acquisition report, selected candidate, expected edition runtime, requested
episode set, and preferred languages. Locate the exact final path; do not scan unrelated
user directories broadly.

### Step 1: Inspect the container

Run:

```bash
python3 "$KIT_DIR/scripts/verify_media.py" \
  --path FINAL_PATH \
  --output VERIFICATION_JSON
```

Require at least one readable video stream and positive duration. Capture container,
resolution, video codec, HDR hints, audio codecs/languages/channels, subtitle codecs
and languages, total size, and remaining partial suffixes.

### Step 2: Verify edition and completeness

Compare duration with reliable edition runtime using the configured tolerance. For
episodes, match the requested season/episode set rather than accepting a folder name.
Record conflicts between filename claims and observed duration.

### Step 3: Repair subtitle gaps

If preferred subtitle streams are absent, look for synchronized external subtitles
matching the exact release or runtime. Store beside the media using player-compatible
naming and inspect its declared language. Do not label unsynchronized text as complete.

### Step 4: Issue the verdict

Use `passed`, `passed_with_warnings`, or `failed`. Report exact local path, edition
confidence, technical summary, subtitle coverage, size, and any open evidence. A client
state of 100% alone is insufficient evidence.

## Dependencies

Python 3.9+ and `ffprobe`. External subtitle repair additionally needs an available
subtitle discovery path.
