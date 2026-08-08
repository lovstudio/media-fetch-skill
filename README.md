# lovstudio-media-fetch

![Version](https://img.shields.io/badge/version-0.1.0-CC785C)

一键完成长视频的检索、版本选择、磁盘预检、多源测速下载、慢源切换和文件验收。

## 本地安装

在本仓库根目录执行：

```bash
export SKILL_SOURCE_DIR="$(pwd)"
mkdir -p "${LOVSTUDIO_SKILLS_INSTALL_DIR:?请设置本地 Skills 目录}"
ln -s "$SKILL_SOURCE_DIR" \
  "$LOVSTUDIO_SKILLS_INSTALL_DIR/lovstudio-media-fetch"
```

安装链接必须解析到当前源码目录；打包文件不等于本地安装。

## 用户配置

默认下载到 `$HOME/Downloads/Media`。首次运行先查看解析结果：

```bash
python3 scripts/media_config.py show
```

确认后写入共享配置：

```bash
python3 scripts/media_config.py init --write
```

显式请求和环境变量始终覆盖配置。qBittorrent 密码只通过
`QBITTORRENT_PASSWORD` 或系统凭据提供。

## 使用

- “帮我找并下载《影片名》的导演剪辑版，优先中英字幕，体积控制在 20GB 内。”
- “Find and download the best compact 4K release of TITLE, then verify the English and Chinese subtitles.”
- “这个 Magnet 帮我下载；先确认磁盘够用，太慢就自动换另一个版本。”

自然语言调用默认运行 `full` 流水线。已有链接使用 `download-known`；本地文件验收使用 `verify`。

## qBittorrent 连接

```bash
export QBITTORRENT_URL="http://127.0.0.1:8080"
export QBITTORRENT_USERNAME="admin"
export QBITTORRENT_PASSWORD="从安全凭据读取"
```

依次可运行：

```bash
python3 scripts/qbittorrent_search.py --query "TITLE YEAR" --output candidates.json
python3 scripts/rank_candidates.py --input candidates.json --output decision.json
python3 scripts/storage_preflight.py --decision decision.json
python3 scripts/qbittorrent_acquire.py --decision decision.json --wait-complete --result acquisition.json
python3 scripts/verify_media.py --path "$HOME/Downloads/Media" --output verification.json
```

## 质量门

```bash
python3 scripts/validate_skill.py .
python3 scripts/media_config.py show --json
python3 scripts/rank_candidates.py \
  --input assets/example-candidates.json \
  --output /tmp/media-fetch-decision.json
python3 scripts/storage_preflight.py \
  --decision /tmp/media-fetch-decision.json \
  --output-dir /tmp
python3 scripts/qbittorrent_acquire.py \
  --decision /tmp/media-fetch-decision.json \
  --dry-run --result /tmp/media-fetch-acquisition.json
```

## 依赖

- Python 3.9+
- PyYAML（Skill 源码校验）
- qBittorrent 5.x WebUI（搜索、测速和下载）
- FFmpeg / `ffprobe`（媒体验收）
- 可选：Rats Search（独立 DHT 检索）

## License

MIT
