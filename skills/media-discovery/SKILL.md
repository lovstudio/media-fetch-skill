---
name: lovstudio-media-discovery
description: >
  识别电影或剧集并从多个独立渠道收集、去重、归一化可下载候选；适用于“帮我找这部片的不同版本”、"search releases for this title"，输出可审计的候选清单。
license: MIT
metadata:
  author: lovstudio
  version: "0.1.0"
  tags:
    - media-search
    - dht
    - candidate-normalization
  compatibility: "Agent runtime with web research; optional qBittorrent search API or Rats Search."
  dependencies:
    - python>=3.9
---

# Media Discovery

Identify the requested work before collecting releases, then return normalized,
deduplicated candidates with evidence kept separate from search-result claims.

## Triggers

### Activate when

- 用户说“帮我找这部片的不同版本”“找导演剪辑版的下载候选”。
- 用户希望比较同一影片的多个 Magnet、Torrent 或发布版本。
- The user asks to search releases for a title or find several downloadable candidates.

### Do not activate when

- 用户已经给出唯一链接且只要求立即下载。
- 用户只询问剧情、影评、演员或上映日期。
- The user asks only to verify an existing local media file.

## Workflow (MANDATORY)

### Step 0: Load the shared contract

- Resolve `KIT_DIR` and read `$KIT_DIR/references/candidate-schema.md` plus
  `$KIT_DIR/references/user-config.md`.
- Establish the output JSON path before searching.

### Step 1: Disambiguate the work

Resolve canonical title, original title, year, media type, season/episode, and likely
alternate spellings. Ask only if two different works remain plausible after research.

### Step 2: Establish edition truth

Collect reliable evidence for known cuts and their runtimes: theatrical, original,
director's cut, extended, uncut, complete, restoration, or regional variants. Record
the source and confidence; do not treat a release filename as authoritative.

### Step 3: Search independent paths

Use at least two available paths:

1. `scripts/qbittorrent_search.py` across enabled, reviewed search plugins;
2. a local DHT index such as Rats Search;
3. direct web research or catalog/index queries;
4. user-supplied Magnet, Torrent, hash, or URL inputs.

Run independent searches in parallel when the environment supports it. Use title,
original title, year, edition terms, resolution, and subtitle markers as separate
queries; avoid one over-constrained query that hides viable candidates.

### Step 4: Normalize and deduplicate

- Emit UTF-8 JSON matching `$KIT_DIR/references/candidate-schema.md`.
- Deduplicate Magnet entries by normalized info hash, then by release identity.
- Preserve `source`, `source_url`, observed time, and edition evidence.
- Infer filename features conservatively; mark inferred values with
  `metadata_confidence: filename`.
- Retain at least three healthy candidates when available so acquisition can race
  independent swarms.

### Step 5: Validate handoff

Check that each candidate has `id`, `name`, `uri`, source, size when known, and current
health fields when available. Report discovery gaps instead of inventing metadata.

## Dependencies

No mandatory external search service. qBittorrent search plugins and Rats Search are
optional adapters; direct web research remains available when configured adapters are
absent.
