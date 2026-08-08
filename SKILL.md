---
name: lovstudio-media-fetch
description: >
  自动检索、比较并下载电影、剧集或其他长视频，优先高质量版本、合适体积与中英字幕，完成磁盘预检、并行测速、慢源切换和媒体验收；适用于“帮我找并下载这部电影”、"find and download the best release"。
license: MIT
metadata:
  author: lovstudio
  version: "0.1.0"
  tags:
    - media-discovery
    - release-selection
    - download-orchestration
    - quality-control
  compatibility: "Portable Agent Skills format; Python 3.9+, qBittorrent 5.x Web API, and ffprobe recommended."
  dependencies:
    - python>=3.9
    - pyyaml
    - qbittorrent>=5.0
    - ffprobe
---

# Media Fetch

Turn one natural-language request into a verified local media file. Search broadly,
identify the actual cut, balance picture quality against size, check storage before
transfer, race viable sources, monitor the winner, recover from stalls, and inspect
the completed file.

## Triggers

### Activate when

- 用户说“帮我找并下载这部电影”“下载导演剪辑版，画质好一点但别太大”“找带中英字幕的完整版”。
- 用户给出片名、年份、版本偏好，或已有 Magnet/Torrent，希望自动完成选择、下载和验收。
- The user asks to find and download the best release, fetch an extended cut, or download a compact high-quality copy with Chinese and English subtitles.

### Do not activate when

- 用户只想了解影片资料、比较不同剪辑或获得观看建议，不要求取得本地文件。
- 用户只要下载普通网页文件、软件安装包、网页视频片段或直播流。
- 用户明确要求发布、上传、转码压制或制作字幕；这些是下载完成后的独立任务。

## Product contract

- One request should normally run end to end without repeated confirmation.
- Prefer an edition whose identity is supported by runtime, release metadata, or
  file-level evidence. A filename alone is weak evidence.
- “Best” means the highest useful viewing quality inside the user's size and disk
  budget, not the largest file or highest advertised resolution.
- Default to embedded Simplified Chinese plus English subtitles. Treat filename
  claims as hints until streams or synchronized external subtitles are inspected.
- Ask the user only when title identity is ambiguous, editions contain materially
  different content, or the top candidates are close enough that taste decides.
- Never begin payload transfer before the destination capacity check passes.
- Keep observing an active job. A task added to a client is not a completed result.

## User configuration

Resolve defaults through `$KIT_DIR/references/user-config.md`. On first use, show the
resolved output directory and preferences before persisting them. Keep credentials in
environment variables or the operating system credential store.

## Skill Kit modules

Load the selected module completely before acting:

- `$SKILL_DIR/skills/media-discovery/SKILL.md` — identify the title and collect normalized candidates from several independent discovery paths.
- `$SKILL_DIR/skills/media-selection/SKILL.md` — verify editions and rank picture, codec, size, audio, subtitles, health, and evidence.
- `$SKILL_DIR/skills/media-acquisition/SKILL.md` — capacity preflight, parallel swarm probing, winner selection, progress observation, stall recovery, and cleanup.
- `$SKILL_DIR/skills/media-verification/SKILL.md` — inspect the downloaded files, edition runtime, streams, subtitle coverage, completeness, and final path.

`kit.yaml` defines the available pipelines. Shared schemas and decision rules live in
`$KIT_DIR/references/`.

## Workflow (MANDATORY)

**You MUST follow these steps in order.**

### Step 0: Resolve runtime and select a pipeline

1. Resolve `SKILL_DIR`, `KIT_DIR`, configuration, qBittorrent connectivity, and
   `ffprobe` availability.
2. On first use, bootstrap missing stable dependencies through the platform's native
   package manager, bind the qBittorrent WebUI to loopback only, generate a strong
   credential, and store it in the operating system credential store. Reuse an
   existing compatible client instead when present.
3. Preserve existing client tasks. Every task created by this Skill must receive a
   unique job tag and an isolated probe directory.
4. Select `full` for a title request, `choose` for comparison only, `download-known`
   for supplied links, `resume` for an existing job, or `verify` for a local file.
5. Read `$KIT_DIR/references/candidate-schema.md`, then validate all handoff JSON.

### Step 1: Resolve the requested work

Capture title, year, media type, season/episode when relevant, edition preference,
maximum size, destination override, audio/subtitle preference, and urgency. Infer
omitted values from the portable profile. Do not ask the user to choose tooling.

### Step 2: Discover independent candidates

Run the discovery module. Use at least two independent discovery paths when possible:
the qBittorrent search API, a local DHT index such as Rats Search, direct web research,
or user-supplied links. Deduplicate by info hash and canonical release identity.

For title and edition truth, prefer distributor, studio, disc, catalog, or reliable
release metadata. Keep search-result claims separate from verified facts.

### Step 3: Rank releases and resolve genuine ambiguity

Run the selection module and `scripts/rank_candidates.py`. Apply
`$KIT_DIR/references/quality-policy.md`.

- Auto-select when one candidate clearly leads and its edition is supported.
- Show at most three concise choices when the output says `choice_required=true`.
- Explain only the user-facing tradeoff: edition/content, picture, size, subtitles,
  and current health. Do not expose internal scoring mechanics unless asked.

### Step 4: Preflight destination capacity

Resolve the destination, candidate size, probe concurrency, temporary probe budget,
and free-space reserve. Run `scripts/storage_preflight.py` before starting transfers.

If capacity is short, report available, required, and shortfall immediately. Offer the
best smaller candidate or a different destination, then wait for that user-facing
decision. Never silently consume the reserve.

### Step 5: Probe, select, and download

Run the acquisition module and `$KIT_DIR/references/acquisition-policy.md`.

1. Probe up to the configured concurrency in an isolated job directory.
2. Observe warm speed, sustained speed, availability, peers, metadata readiness, and
   ETA; a short burst alone does not win.
3. Pause non-winners, move the winner to the final destination, and continue polling.
4. If the winner stalls beyond the configured threshold, resume the next proven
   candidate. If all candidates are slow, return to discovery for another wave.
5. Keep the terminal session alive and poll at intervals short enough to provide the
   user a progress update at least once per minute during active work.
6. Clean only exact job-tagged losing tasks and their isolated probe files after the
   final candidate is complete. Leave pre-existing client tasks untouched.

### Step 6: Verify the completed media

Run the verification module and `scripts/verify_media.py`.

- Confirm a readable video stream, non-zero duration, expected resolution and codec,
  audio tracks, subtitle streams, and duration close to the selected edition.
- Inspect every episode for episodic requests; a season folder is complete only when
  the requested episode set is present.
- When preferred subtitles are missing, search for a matching release or synchronized
  subtitle, add it beside the media, and verify naming plus duration alignment.
- Recheck final free space and ensure no partial suffix remains on the primary file.

### Step 7: Report the result

Lead with completion status and the exact local path. Include title/edition, video and
audio summary, subtitle coverage, final size, source health observed, elapsed time,
and any remaining evidence gap. Distinguish “download complete” from “quality and
edition verified”.

## References

- `$KIT_DIR/references/candidate-schema.md` — normalized candidate and decision JSON.
- `$KIT_DIR/references/quality-policy.md` — edition, picture, codec, size, language, and ambiguity rules.
- `$KIT_DIR/references/acquisition-policy.md` — capacity, probing, monitoring, switching, and cleanup rules.
- `$KIT_DIR/references/user-config.md` — portable defaults and secrets handling.

## Dependencies

- Python 3.9+ for deterministic helpers.
- qBittorrent 5.x with WebUI enabled for integrated search and acquisition.
- Search plugins or another discovery adapter for title search.
- `ffprobe` from FFmpeg for final stream and duration inspection.
- Optional Rats Search for independent DHT discovery.
