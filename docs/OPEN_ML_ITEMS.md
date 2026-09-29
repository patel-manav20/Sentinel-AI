# Known limitations and future work

Sentinel AI was built and demoed at Edge AI SJSUHack 2026. This page lists what the finished system does **not** do yet and how each piece could be added. None of it is needed to run the demo.

## Known limitations

| Area | Limitation |
|---|---|
| **Weapon detection** | Weapon boxes in the demo are pre-computed. Small handguns in 960×540 footage were not detected reliably live: per-person CLIP missed a visible handgun, YOLOE-26 never scored a weapon above 0.35, full-frame Qwen3-VL takes 1–2.7 s per frame, and Qwen on per-person crops hallucinated knives. So `data/feeds/seville_weapon_timeline.json` replays the dataset's hand-drawn weapon boxes by clip time. People, tracking, escalation and Qwen classification stay live. |
| **Accuracy** | Detection accuracy has not been measured. The severity thresholds in `bench/thresholds.json` are demo settings, not tuned on labelled data. |
| **Cross-camera matching** | Handoffs follow the camera map (Camera 1 → 2 → 3), not what the person looks like. |
| **Cameras** | The demo reads recorded MP4 clips. Real RTSP cameras are not wired in. |
| **Voice** | The ElevenLabs free tier allows 10,000 characters per month. A local backup voice covers slow or failed requests. |

## Future work

### 1. Live weapon detector
Replace the pre-computed timeline with a model that can see small handguns in low-resolution CCTV. Examples are a detector fine-tuned on weapon datasets, or running at a higher input resolution on the armed person's crop. `services/vision/weapon_timeline.py` shows exactly where its output plugs in: mark the tracked person holding the weapon, and the existing `weapon` rule escalates.

### 2. Appearance re-identification (OSNet)
**Today:** `services/vision/osnet.py` is a stub. It hashes camera, track ID and box geometry into a digest, and nothing calls it.

**To make it real:**
1. **Model code:** vendor torchreid's `osnet.py` (MIT, about 600 lines) or add `torchreid` as a dependency.
2. **Weights:** add `osnet_x0_25_msmt17` (about 1 MB) or `osnet_x1_0_msmt17` (about 9 MB) to `services/vision/pull_weights.py`.
3. **Crop path:** in `pipeline.py`, crop each tracked person (256×128), embed once every N frames, and store a normalised 512-d vector on the track.
4. **Consumer:** gate handoffs in `services/api/vision_bridge.py` on cosine similarity between the last track on camera N and new tracks on camera N+1 (start around 0.6, then tune).
5. **GPU budget:** keep it on CPU or batch it after YOLO. Don't add another resident GPU model next to Qwen without checking [`STABILITY.md`](./STABILITY.md).

### 3. Confidence calibration
**Today:** Qwen's class confidence comes from token logprobs, used raw. The calibration plumbing is built but off by default:
- `CS_CALIB_LOG=<file>.jsonl` logs one row per live classification.
- `scripts/fit_temperature.py <file>.jsonl` fits a temperature T and prints NLL and ECE before and after. `--selftest` recovers a known T on synthetic data.
- `CS_VLM_TEMPERATURE=<T>` applies it in `services/brain/adjudicate.py` (1.0 means no change).

**To finish:** run live classification over labelled clips with logging on (100+ rows across all classes, including BENIGN), add a `"label"` to each row, fit T, and re-check the thresholds. Caveat: Qwen may split a class name like `WEAPON` into several tokens, so the first-token logprob is only a proxy.

### 4. Real cameras through MediaMTX
**Today:** `services/mediamtx/mediamtx.yml` defines RTSP paths `cam01`–`cam06` on `:8554` (`docker compose --profile full up mediamtx`), but nothing publishes to it or reads from it. `FileSource` in `services/vision/decode.py` reads the demo MP4s and handles demo sync and Reset.

**To adopt:**
1. **Publishers:** real cameras push to those paths, or for testing, `ffmpeg -re -stream_loop -1 -i <clip> -c copy -f rtsp rtsp://127.0.0.1:8554/camNN`.
2. **Reader:** add an `RtspSource` to `decode.py` with the same `next_frames()` / `rewind()` / `close()` shape. A live stream can't be rewound, so Reset would work from the live edge.
3. **Hardening:** pin the image tag (currently `latest`) and keep the RTSP port on the local network only.
4. **Browser playback:** the dashboard still needs MJPEG, HLS or WebRTC. HLS and WebRTC are off in the current config.

### 5. More incident types
The pipeline already classifies `WEAPON | FIGHT | THEFT | RUN | MEDICAL | BENIGN`. Adding a type such as fire and smoke means a new class token, a matching rule, and a severity entry. The rest of the pipeline stays the same.
