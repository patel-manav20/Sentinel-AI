# Sentinel AI · Security Console (`web/`)

The officer dashboard for Sentinel AI. Plain HTML, CSS and JavaScript modules:
**no build step, no npm dependencies, no framework**, and no internet needed for
the UI itself.

<p align="center">
  <img src="../docs/assets/screenshots/live-operations.png" alt="Live Operations page: six camera feeds, a weapon alert, the campus map and the call panel" width="820">
</p>

## Run it

From the repo root:

```bash
python3 -m http.server 8090 --bind 127.0.0.1 --directory web
# open http://localhost:8090/
```

- **Mock mode (default):** with no `?ws=`, the console plays a built-in demo timeline. You don't need a GPU, backend or keys.
- **Live mode:** point it at the API's WebSocket:
  `http://localhost:8090/?ws=ws://<box>:8080/ws`
- **From a laptop over SSH:** `ssh -L 8090:localhost:8090 <user>@<box>`, then open `http://localhost:8090/`.

### URL options

| Option | What it does |
|---|---|
| `?ws=<websocket url>` | Live events from the API. REST calls use the same host over HTTP(S). |
| `?api=<http url>` | Overrides the REST base URL (otherwise it comes from `?ws=`) |
| `?map=plan` or `?map=image` | Forces the simple floor plan or the SJSU map image for one load |
| `?mapedit=1` | Placement tool for moving cameras and walkway bends on the map |

`?nosplash=1` is now ignored; the branded intro always plays.

## Pages

| Page | Route | What's on it |
|---|---|---|
| **Live Operations** | `#/live` | Six-camera wall with auto focus and split focus on active incidents, detection boxes (red for an armed person), live pipeline strip, site map with tracked path, and the incidents and call panels |
| **Incidents** | `#/incidents` | Incident queue and details: confirm, dismiss, dispatch, broadcast |
| **Call Console** | `#/call` | **Active call** (live transcript, incident facts, actions) and **Call history** |
| **System** | `#/system` | Camera status grid, health (GPU, router p95), AI models in use, AI usage chart, and the "Cloud cost avoided" card |

There is also a **splash** intro, a **severe banner** when a severe incident is open and you're on another page, and a full-screen **Expand** map with live camera thumbnails around it.

## Keyboard shortcuts

| Key | Action |
|---|---|
| `I` | Report an incident (Live Operations only) |
| `Shift` + `D` | Open or close Demo control |
| `Shift` + `R` | Reset the demo (a fresh run on the backend in live mode) |
| `Esc` | Close the expanded map, a dialog, Demo control or the sidebar |

## Demo control (`Shift` + `D`)

- **Scenario:** the full armed-chase scenario (mock mode)
- **Playback:** play or pause, speed, and start time (mock mode)
- **Camera sources:** load a local video for any camera. It is stored **in this browser only** (IndexedDB), never uploaded or committed.

## How it connects to the backend

`js/transport.js` runs either the mock player or a WebSocket from `?ws=`. Every event is checked by `js/validate.js` and reduced into `js/store.js`. Field names follow `contracts/events.py` exactly, and unknown event types are ignored.

| Event | Used for |
|---|---|
| `incident.upsert` | Create or update an incident |
| `incident.state_change` | Incident state transitions |
| `overlay.boxes` | Detection boxes on a camera (normalised 0–1) |
| `camera.online` | Camera online or offline |
| `health.strip` | System health tiles |
| `call.transcript_delta` | Call transcript lines |
| `tool.call_live` | Facts shared during a call |
| `demo.control` | Reset, scenario and prewarm |
| `usage.tick`, `usage.total` | AI usage counters for the System page |
| `activity.tick` | Which pipeline stage is busy (live pipeline strip) |

**REST calls** (built in `js/config.js`):
- `POST /api/incidents/manual`
- `POST /api/incidents/{id}/dispatch`
- `POST /api/incidents/{id}/confirm`
- `POST /api/incidents/{id}/dismiss`
- `POST /api/broadcast`
- `GET /voice/status`, for the voice provider and readiness

In live mode, an action only shows as done once the API confirms it.

**Video:** cameras 1–3 stream from `/mjpeg/{camera}`, the exact frames the vision loop analysed, so boxes line up. Cameras 4–6 play byte-range MP4 from `/media/{camera}`.

## What stays in the browser

- **Camera sources:** local videos you load in Demo control (IndexedDB)
- **Call history:** the last 50 calls, saved while each call runs. Place names and phone-like numbers are masked before saving (IndexedDB).

Nothing from these is sent to the backend.

## Rules this UI keeps

- Every call and dispatch surface is labelled **SIMULATED**. The UI never dials an emergency number.
- Cameras are numbered 1 to 6 and show their canonical MacQuarrie Hall locations.
- Event text is rendered with `textContent`, never as HTML.
- A strict Content Security Policy: scripts and styles only from this folder, no inline script or style.
- When the backend is unreachable, the UI never claims an action succeeded.

## Developing

- **Cache busting:** modules are loaded with one shared `?v=` tag. Bump it on every change so browsers pick up new files.
- **Check:** `node web/check.mjs` runs quick store, transport and auto-follow checks. It needs Node, which is not installed on the ZGX.
- **Colors:** colors and spacing live in `css/tokens.css`. Canvas and SVG drawing read them through `js/theme.js`.

## Files

```text
web/
├── index.html              Page shell, CSP, splash markup
├── check.mjs               Node checks for store, transport and auto-follow
├── assets/                 logo.png, logo.svg, logo-mark.png, logo-inverse.png, site-map.png
├── css/
│   ├── tokens.css          Brand colors, spacing, light/dark tokens
│   ├── app.css             Layout and components
│   └── activity.css        Live pipeline strip
└── js/
    ├── boot.js             Runs before first paint
    ├── main.js             Starts the app
    ├── router.js           Hash routes: live, incidents, call, system
    ├── transport.js        Mock player or WebSocket
    ├── mock.js             Deterministic demo timeline
    ├── validate.js         Event shape checks
    ├── store.js            State and reducers
    ├── actions.js          Report, dispatch, confirm, dismiss, broadcast, reset
    ├── config.js           REST routes and cloud-price table
    ├── site.js             Cameras, adjacency, map mode, site label
    ├── tracking.js         Followed incident and its camera path
    ├── cameraSources.js    Local videos per camera (browser only)
    ├── cameraStatus.js     One online/offline rule for tiles, pins and lists
    ├── callHistory.js      Browser-only call history
    ├── models.js           AI models the pipeline runs
    ├── modelStatus.js      Voice provider and model readiness
    ├── modelActivity.js    Model busy state for the pipeline strip
    ├── usage.js            AI usage and cloud-cost math
    ├── clock.js            One shared clock
    ├── format.js           Labels, times and numbers
    ├── theme.js            Reads CSS tokens for canvas and SVG
    ├── dom.js, icons.js, logo.js   Small helpers
    └── ui/                 Pages and components
        ├── live.js, cameras.js, cameraLayout.js, liveExpand.js, activity.js
        ├── incidents.js, incidentDetail.js, incidentRow.js, incidentActions.js, incidentClip.js, dismiss.js
        ├── call.js, callpage.js, callConsole.js, conversation.js
        ├── map.js, sitePlanExtras.js, sidebar.js, autofollow.js
        ├── system.js, modelsTable.js, usageCard.js
        └── topbar.js, nav.js, banner.js, splash.js, operator.js, demo.js, demoSources.js
```
