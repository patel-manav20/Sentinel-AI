# context.md

## Git rules
1. `git pull --rebase origin main` before every push.
2. Branch `feat/<name>/<thing>`, merge the same day.
3. Update this file in the same commit as the work.
4. Never commit weights, clips, or `.env`. Mock CCTV MP4s live only under `Documents/campus_sentinel_media/`, never in the repo.
5. `contracts/` changes are a team decision; log them below. Keep this file under ~150 lines.

## Team & how we work
Sentinel AI was built by the whole team working together, and everyone contributed across the project. The table shows who leads each part, not who is allowed to touch it: anyone can work anywhere, just tell the area lead and update this file.

| Person | Leads |
|--------|-------|
| **Ayush** | Brain (`services/brain/`), shared contracts (`contracts/`), tooling scripts (`scripts/`) |
| **Pratham** | Vision (`services/vision/`), models (Z Runtime serving, YOLO/CLIP weights) |
| **Manav** | Dashboard (`web/`), architecture planning, testing |
| **Naman** | API (`services/api/`), data (`data/`) |
| **Indraneel** | Voice alerts with ElevenLabs (`services/voice/`), benchmarks (`bench/`) |
| **Everyone** | Docker Compose, Makefile, integration, docs (`docs/`, README), demo & presentation |

## Status: complete
Sentinel AI was built and demoed at Edge AI SJSUHack 2026. Everything below describes the finished system.
- **Pipeline:** vision → Qwen3-VL → decision gate → console → phone call, end to end on the ZGX Nano.
- **Tracking:** cross-camera handoffs (Camera 1 → 2 → 3) with one call per incident, updated live.
- **Calls:** live SignalWire calls with ElevenLabs voice and local speech recognition.
- **Console:** Live Operations wall, Incidents, Call Console, System page, and the SJSU site map.
- **Operations:** fresh-run Reset, SQLite history, measured health strip.
- **Delivered:** demo video, README, and the Sentinel AI rebrand.

## Contract decision (2026-09-24)
- **IncidentClass:** `FALL` → **`WEAPON`**. Set: `WEAPON | FIGHT | THEFT | RUN | MEDICAL | BENIGN`. IncidentRecord **schema_version 1.1**.
- Hero demo = **Seville armed chase / WEAPON** (pose rule `fall` may still fire).
- `call.brief` event added to `EventType` (additive): `{type, incident_id, reason: "dispatch"|"handoff", brief, ts}`, sent after DISPATCHED and once per real camera handoff.

## Runtime (GB10): stability first
See **`docs/STABILITY.md`**. The box hard-crashed under Qwen+YOLO (NVRM OOM) before swap was added.
```bash
# Always: bash scripts/preflight_gb10.sh
zrt serve hf:Qwen/Qwen3-VL-30B-A3B-Instruct-FP8 --force --gpu-memory-fraction 0.40 \
  --extra "--max-model-len=8192"
# YOLO CUDA only after ZRT is healthy; or use CS_VISION_DEVICE=cpu for light tests
# .env sets CS_MEDIA_ROOT and CS_TTS_PROVIDER; add
# CS_SIGNALWIRE_ENABLED=1 CS_KILL_SWITCH=0 for real calls, CS_KILL_SWITCH=1 for testing.
set -a; . ./.env; set +a; export PYTHONPATH=.
CS_VISION_SEVILLE=1 CS_VISION_DEVICE=cuda:0 \
  CS_VISION_CONF=0.50 CS_VISION_COOLDOWN_S=600 CS_VISION_MAX_UPSERTS_MIN=1 \
  services/vision/.venv/bin/python -m services.api --host 0.0.0.0 --port 8080 2>&1 | tee -a logs/api_live.log
# Separate shell/tmux session:
python3 -m http.server 8090 --bind 0.0.0.0 --directory web
```
- Keep Qwen at **0.40** GPU fraction (0.55 hard-locked the box). **32G swap is on** (`/swapfile`). UI-only work: do **not** start ZRT.
- **Detector = YOLO26s-pose `.pt` only** (TensorRT removed). `CONF` is **0.50**.
- **One clock for video and boxes:** `/mjpeg/cam-01..03` streams the exact frames the vision loop decoded, so boxes match the picture and Reset rewinds every viewer. Repeated frames skip YOLO and reuse the last boxes.
- **Tracking:** every person is sent with a real track id. Tracker = IoU on a constant-velocity prediction, then centre-distance fallback; a track shows after 2 matches.
- Qwen runs once on cam-01; cam-02/03 reuse that result. cam-04..06 never enter YOLO/Qwen.
- Live dashboard: `http://<box>:8090/?ws=ws%3A%2F%2F<box>%3A8080%2Fws&nosplash=1` (drop `&nosplash=1` when recording so the intro plays).
- tmux sessions: `sentinel-api`, `sentinel-web`, `sentinel-asr`, `sentinel-tts`. ZRT serves Qwen on `127.0.0.1:8000`.

## Clips and media
- **cam-01..03:** locked Seville chase (WEAPON). **cam-04..06:** `feeds/naman/demo_clips`, VLM-assigned ambient (see `docs/NAMAN_CLIPS.md`).
- Seville clips are 5 fps holding ~2 fps of real motion. A 15 fps interpolated pack (`feeds/seville_option1_3cam_locked_15fps/`, via `CS_MEDIA_ROOT`) measured 13-14 fps on the wall (was ~3) but looked hazy, so the **recording uses the original 5 fps clips** (`CS_WALL_JPEG_QUALITY=85`, ~0.9 MB/s per viewer).
- Mac mount: `~/mnt/zgx-b505` = SSHFS of `/home/hp25` via **`zgx-up`** (must show in `mount`). If Mac writes never reach the ZGX: `zgx-down && zgx-up`.

## Canonical demo site
- `data/camera_map.json` is the only source of address and location facts: San Jose State University / MacQuarrie Hall. Cameras 1-3 = ground-floor lobby, east corridor, west corridor/stairwell.
- Address: One Washington Square, San Jose, CA 95192. Camera coordinates are demo map anchors, not surveyed emergency coordinates.

## Weapons (who is armed)
- Live per-frame weapon detection is not reliable here (tested 2026-09-26): per-person CLIP missed a visible handgun, YOLOE-26 never scored a weapon above 0.35, full-frame Qwen takes 1-2.7 s per frame, and Qwen on crops hallucinated knives.
- `data/feeds/seville_weapon_timeline.json` holds the dataset's **hand-drawn** weapon boxes (US Mock Attack Pascal VOC) matched to clip frames (`scripts/build_gt_weapon_timeline.py`, 640 of 649 frames matched). `services/vision/weapon_timeline.py` replays it and labels the armed person `weapon` for 4 s; the web draws it red. Known misses: extra handguns in some crowd frames; the CAM01 doorway rifle at ~321 s.
- With the timeline loaded, per-person CLIP is skipped; an armed person escalates through the `weapon` rule. People, tracking, escalation and Qwen classification stay live.

## Voice and calls
- **SignalWire** (Compatibility API) is the only phone provider. `/signalwire/voice` places the call, `/signalwire/media` carries audio both ways. Credentials live in the ZGX **`.env` only**. `CS_KILL_SWITCH=1` blocks all calls.
- **Voice = ElevenLabs** (`eleven_flash_v2_5`, voice Sarah, `ulaw_8000`). Turn it on with `CS_TTS_PROVIDER=elevenlabs` + `ELEVENLABS_API_KEY` (the code default is the local backup voice), then restart the API. Lines stream over one keep-alive HTTPS connection per call: 157-160 ms to first chunk on a warm connection. Free tier: 10,000 chars/month; short repeated lines are cached.
- **What leaves the box:** ElevenLabs is a cloud service and receives **only the text of each line Sentinel AI speaks**. Video, images, AI decisions, speech recognition and history stay local on the ZGX Nano. SignalWire carries the call audio and SOS broadcast text.
- **Fallback:** if ElevenLabs sends no audio within `CS_ELEVENLABS_FIRST_CHUNK_S` (1.5 s) or errors, that line uses the local backup voice (`scripts/serve_kokoro.py`, port 8092, `CS_KOKORO_URL`), never both. A bad key or quota error parks ElevenLabs for 10 min.
- **ASR:** faster-whisper `base.en` on CPU, live on port 8093 (script default 8091); the UI keeps the historical "Parakeet" label.
- **Call behaviour:** only a vision-detected WEAPON places a call; demo-panel scenarios never call. One call per incident: a second detection while a call is live is blocked as `voice_busy`, and Camera 1→2→3 handoffs update the same call. Facts advance only on observed handoffs.
- **Answers:** fixed questions (who/where calling from, address, how many, weapons, describe, injuries, where now, where they moved, status) are answered instantly from `services/voice/scene.py` at the current clip second; anything else gets an instant fallback. Qwen is no longer called during a call. Replies are one sentence; refusals (medical, identity, visibility) are short.
- **Turn-taking:** inbound audio is ignored while Sentinel speaks plus a 500 ms echo tail (no barge-in); turns close after 500 ms of silence; turns of about 1 s or more get one filler while ASR runs. End of speech to answer audio measured 0.75-1.20 s. <!-- TODO: confirm whether this was measured before ElevenLabs was switched on -->

## Demo recording mode (2026-09-26)
- `.env`: `CS_CALL_MODE=scripted` + `CS_DEMO_TOKEN_RATE=1200`; run the API with `CS_SIGNALWIRE_ENABLED=0 CS_KILL_SWITCH=0` (the kill switch blocks dispatch itself). Unset `CS_CALL_MODE` for real calls.
- Scripted call: `services/voice/demo_call.py`, 43 lines keyed to the clip second so every count, weapon and description matches the picture. Voice-over is added in editing.
- `data/feeds/seville_scene_script.json` (`scripts/build_scene_script.py`): per-second active camera, people, weapons and Qwen descriptions. `CS_DEMO_TOKEN_RATE` adds synthetic Qwen tokens to usage (0 = off).

## Console and API
- `DemoHub` is the in-memory authority for a demo run. Report, dispatch, confirm, dismiss, broadcast and reset go through REST and publish over the WebSocket; the UI never claims success when the API is down.
- **Reset = fresh run:** hangs up the call, clears incidents, replay, counters, voice and guardrail state, rewinds cam-01..03, drops in-flight Qwen results. Startup never creates an incident on its own.
- Reconnecting clients get camera state plus a bounded replay of incidents, dispatch state and transcript.
- Health strip is measured (`services/api/telemetry.py`: nvidia-smi GPU, router-step p95, real counters; `null` when unmeasured). `/voice/status` probes the ASR and TTS `/health` endpoints.
- History: `data/runtime/sentinel.db` (SQLite, gitignored, `CS_DB_PATH` overrides) with `runs`, `incidents`, `calls`, `transcripts`. Reset starts a new run; nothing is deleted.
- Web: plain HTML/JS, no build. Mock timeline without `?ws=`. Live wall: cam-01..03 via MJPEG, cam-04..06 via byte-range MP4. Real SJSU site map (OpenStreetMap export, "illustrative" note), `?map=plan` fallback, `?mapedit=1` placement tool. Bump the one `?v=` cache tag on every web change.

## Decisions
- Name is **Sentinel AI** (2026-09-28) in display text, docs and the Qwen call prompt. Repo folder, compose project, `CS_*` env vars, `campus_sentinel_media/` paths and `campus-sentinel-playbook.html` keep the old name for now.
- UI is branded **Sentinel**; cameras are numbered 1-6 and show canonical MacQuarrie locations from the camera map.
- Boxes are normalised 0-1; camera IDs are canonical and hyphenated (`cam-01`). WS overlays are `overlay.boxes` only.
- Call surfaces show the live provider state (SignalWire call / Simulated call); dispatch stays labelled **SIMULATED**. Demo calls go to a teammate, never 911.
- Brain temperature calibration is plumbed but off (`CS_CALIB_LOG`, `scripts/fit_temperature.py`, `CS_VLM_TEMPERATURE`); no labelled data yet.
- Deployment is Docker Compose. Playbook is the original plan (`PLAYBOOK.md`); `AUDIT.md` has the step table.

## Known limitations
| Area | Limitation |
|------|------------|
| Vision | Weapon boxes are pre-computed from the dataset's labels; no live detector we tried could see small handguns reliably |
| Voice | ElevenLabs free tier is 10,000 characters/month; a local backup voice covers failures |
| Web | Ambient cam-04..06 only rewind on the dashboard that pressed Reset; `site.js` duplicates the camera map instead of loading `GET /api/site` |
| Build | `make demo` and `make reset` only print steps, `make bench` is not wired, and `make demo` mentions web on `:8000` (use `:8090`; ZRT owns `:8000`) |
| Accuracy | Detection accuracy was not measured; thresholds are demo settings |

## Future ideas
- Live weapon detector that sees small handguns, replacing the pre-computed timeline.
- OSNet appearance re-id, temperature calibration and MediaMTX for real RTSP cameras (see `docs/OPEN_ML_ITEMS.md`).
- More incident types (e.g. fire and smoke) using the same pipeline.
- Pick a license and confirm the mock-attack dataset license before wider sharing.
