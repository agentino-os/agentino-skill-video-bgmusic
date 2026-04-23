# video-bgmusic

> **Mix a music bed under an existing video with automatic ducking — the music drops when narration is present and comes back when the voice pauses. Pure ffmpeg sidechain compression.**

An Agentino skill that takes a video already carrying a voice track and a music file, and produces a rendered MP4 where:
- the music plays at a configurable baseline volume,
- a sidechain compressor automatically lowers the music whenever the voice exceeds a threshold (so dialogue stays intelligible),
- the music fades in at the start and out at the end of the video,
- the music loops if it's shorter than the video,
- the video track is stream-copied (no re-encode) so quality is preserved.

Designed to chain after [`video-voiceover`](https://github.com/dagoSte/agentino-skill-video-voiceover) + [`video-slideshow`](https://github.com/dagoSte/agentino-skill-video-slideshow) + [`video-captions-burn`](https://github.com/dagoSte/agentino-skill-video-captions-burn) — add a music bed to the captioned narrated MP4 in one pass.

## Install

Requires [Agentino](https://github.com/dagoSte/agentino) ≥ `1.2.0-rc.1` and `ffmpeg`.

```bash
brew install ffmpeg                                                    # macOS
# sudo apt install ffmpeg                                              # Debian / Ubuntu

agentino marketplace install dagoSte/agentino-skill-video-bgmusic
agentino skill show video-bgmusic
```

## Use

```bash
agentino skill exec video-bgmusic \
  -i video_path=/tmp/narrated.mp4 \
  -i music_path=/tmp/lounge.mp3 \
  -i output_path=/tmp/scored.mp4 \
  -i duck_depth_db=14 \
  -i music_volume_db=-20
```

### Example output

```json
{
  "output_path": "/tmp/scored.mp4",
  "input_duration_s": 13.567,
  "output_duration_s": 13.567,
  "music_duration_s": 180.5,
  "music_looped": false,
  "music_volume_db": -20.0,
  "duck_depth_db": 14.0,
  "duck_attack_ms": 200,
  "duck_release_ms": 500
}
```

## Use with agentino run

```bash
agentino run "score /tmp/narrated.mp4 with /tmp/lounge.mp3 as a music bed that ducks under the voice, save to /tmp/scored.mp4"
```

`agentino run` lets the LLM planner pick this skill automatically from the task description — the paths and optional knobs (`duck_depth_db`, `music_volume_db`) are parsed from the prose itself.

## Inputs

| Input | Type | Default | Description |
|---|---|---|---|
| `video_path` | string | **required** | Source video with an existing voice track on stream 0:a. |
| `music_path` | string | **required** | Music file (WAV / MP3 / AAC / FLAC / OGG). |
| `output_path` | string | `<video>.scored.<ext>` | Destination MP4. |
| `music_volume_db` | number | `-18.0` | Baseline music level in dBFS. `-12` = prominent, `-24` = barely-there. |
| `duck_depth_db` | number | `12.0` | Extra dB the music drops when the voice is active. 12 is broadcast-standard; 18+ is aggressive. |
| `duck_attack_ms` | integer | `200` | How fast the music ducks when the voice starts (ms). |
| `duck_release_ms` | integer | `500` | How fast the music returns when the voice pauses (ms). |
| `duck_threshold` | number | `0.03` | Sidechain trigger amplitude. 0.03 catches conversational speech. |
| `loop_music` | boolean | `true` | Loop the music if it's shorter than the video. |
| `fade_in_s` | number | `1.0` | Music fade-in at the start. |
| `fade_out_s` | number | `2.0` | Music fade-out at the end. |

## Outputs

- **`output_path`** — absolute path to the scored MP4.
- **`input_duration_s`**, **`output_duration_s`** — should match; the video track is stream-copied.
- **`music_duration_s`** — source music duration from `ffprobe`.
- **`music_looped`** — whether the music was looped to cover the video length.
- **`music_volume_db`**, **`duck_depth_db`**, **`duck_attack_ms`**, **`duck_release_ms`** — resolved settings for audit.

## How it works

1. **Probe** both inputs for duration; derive whether the music needs to loop.
2. **Convert knobs** — `music_volume_db` → linear scalar, `duck_depth_db` → compressor ratio via `R = 10^(depth/20)`.
3. **Build the music leg** — `[1:a]` → optional `aloop=loop=-1` → `volume=<linear>` → `afade in/out` → `atrim` to video length → `[bed]`.
4. **Split the voice** — `[0:a]asplit=2[vox_main][vox_sc]`. One copy becomes the sidechain trigger, the other stays at full level.
5. **Duck** — `[bed][vox_sc]sidechaincompress=threshold=…:ratio=…:attack=…:release=…[ducked]`.
6. **Final mix** — `[vox_main][ducked]amix=inputs=2:duration=first[aout]`.
7. **Render** with `-c:v copy` (no video re-encode) + AAC 192 kbps + `+faststart`.

## Safety

- `sandbox_level: none` (ffmpeg discovery).
- `network_access: false`.
- `file_access: read-write`.
- `agentino skill exec skill-security-check -i path=skill.yaml` → zero findings.

## License

MIT — see [`LICENSE`](./LICENSE).
