# Changelog

All notable changes to the `video-bgmusic` Agentino skill are documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/) and this skill adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [1.0.0] — 2026-04-23

First public release for the Agentino marketplace.

### Added

- `agentino skill exec video-bgmusic` overlays a music bed on a narrated video with automatic ducking:
  - `sidechaincompress` triggered by the voice track drops the music `duck_depth_db` when dialogue is present.
  - Configurable attack / release / threshold / ratio — defaults tuned for broadcast-style 200 ms attack + 500 ms release + 12 dB depth.
  - Music auto-loops (`loop_music=true`) when shorter than the video; baseline volume in dBFS; fade-in + fade-out at start/end.
  - Video track stream-copied (`-c:v copy`) — zero re-encode quality loss on the original picture.
- Single `filter_complex` pass, AAC 192 kbps + `+faststart`.
- `sandbox_level: none` (ffmpeg discovery), `network_access: false`, `file_access: read-write`.

### Tested

- 13.57 s Italian narrated MP4 + 30 s synth music → 13.57 s scored output, voice intelligible throughout with music audibly ducking on every phrase.
- Zero findings from `agentino skill exec skill-security-check -i path=skill.yaml` (fail-on = high).

### Requires

- Agentino ≥ 1.2.0-rc.1
- `ffmpeg` (and `ffprobe`) on `PATH`
