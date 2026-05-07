# DeepSeek-OCR Client — CLAUDE.md

## Project Overview

Electron desktop GUI for DeepSeek-OCR. Supports both **local inference** (HuggingFace model via transformers) and **API mode** (SiliconFlow OpenAI-compatible API).

## Architecture

```
start-client.bat → start.py (venv setup + deps) → npm start → Electron (main.js)
                                                              │
                                                              │ spawns
                                                              ▼
                                                     backend/ocr_server.py (Flask on :5000)
```

- **Electron main process** (`main.js`): spawns Python Flask server, creates window, proxies IPC calls to Flask HTTP endpoints
- **Flask backend** (`backend/ocr_server.py`): model loading/inference OR API-based OCR
- **Renderer** (`renderer.js`): UI logic, canvas-based box visualization, progress polling

## Key Files

| File | Role |
|------|------|
| `main.js` | Electron main process — window creation, Python server lifecycle, IPC handlers |
| `renderer.js` | Frontend logic — image handling, OCR params, progress polling, box rendering |
| `index.html` | HTML structure with header controls, results panels, lightbox |
| `styles.css` | All styling |
| `backend/ocr_server.py` | Flask server — model loading, OCR inference, API mode, config persistence |
| `start.py` | Launcher — venv management, GPU detection, dependency installation |
| `api_config.json` | Persisted API config (env vars take priority over file) |

## Running

```cmd
start-client.bat         # Windows（推荐 — 处理所有设置）
```
或
```
python start.py          # 同上
```
或
```
.\venv\Scripts\activate  # 激活 venv
npm start                # 如果已配置 venv
npm run dev              # 打开 DevTools 运行
```

## Two Operation Modes

### Local Mode (default)
- Loads DeepSeek-OCR model from HuggingFace into local GPU/CPU
- `OCR_MODE=local` or unset
- Model must be loaded via "Load Model" button before OCR

### API Mode
- Calls SiliconFlow API instead of local model
- `OCR_MODE=api` + valid `SILICONFLOW_API_KEY`
- No model loading needed — "Load Model" button is disabled
- Config persisted in `api_config.json`; env vars take priority
- **Flow**: upload → base64 encode → POST to SiliconFlow SSE endpoint → stream tokens → /progress polling → frontend renders

### Switching Modes
- UI: "API Settings" `<details>` panel → Fill API Key/URL/Model → Click **Apply** (saves settings only) → Click **API Mode**/**Local Mode** toggle button
- Buttons: **Apply** saves config without changing mode; **API/Local Mode** toggles mode without changing saved settings


## OCR Box System

The SVG overlay system renders bounding boxes on the right preview panel:

### Coordinate System
- Model outputs normalized coordinates in range `[0, 999]` (`DEEPSEEK_COORD_MAX = 999`)
- Token format: `<|ref|>CONTENT<|/ref|><|det|>[[x1,y1,x2,y2]]<|/det|>`
- `parseBoxesFromTokens()` (renderer.js:482) uses regex to extract boxes
- `renderBoxes()` (renderer.js:554) scales from normalized space to image pixels and creates SVG elements

### SVG Layout
- SVG overlay (`#ocr-boxes-overlay`) is `position: absolute` inside `#ocr-preview-container` (inline-block)
- viewBox set to the image's natural pixel dimensions, `preserveAspectRatio="xMidYMid meet"`
- Container wraps image naturally; SVG fills container at 100% × 100%
- Boxes align with image regardless of scroll state

### CSS Layout Chain (critical for overlay alignment)
```
.ocr-preview-wrapper (flex: 1; align-items: safe center; overflow: auto)
  └─ .ocr-preview-container (inline-block; position: relative; max-width: 100%)
       ├─ img#ocr-preview-image (max-width: 100%; height: auto)
       └─ svg#ocr-boxes-overlay (absolute; top/left/width/height: 100%)
```

### DO NOT:
- Do NOT set `preserveAspectRatio="none"` on the SVG — causes coordinate distortion when container resizes
- Do NOT set `max-height` with viewport-relative values like `calc(100vh - 300px)` — breaks when header height changes
- Do NOT use `align-items: center` without the `safe` keyword — cuts off scrollable content at top

## Backend Config Priority

1. Environment variables (highest priority)
2. `api_config.json` (persisted from UI, fallback)
3. Default values in code (lowest priority)

Key variables: `OCR_MODE`, `SILICONFLOW_API_KEY`, `SILICONFLOW_API_URL`, `SILICONFLOW_MODEL`

## Key Flask Endpoints

| Endpoint | Method | Purpose |
|----------|--------|---------|
| `/health` | GET | Server status, `api_mode`, `model_loaded` |
| `/model_info` | GET | Model/device info |
| `/load_model` | POST | Load HuggingFace model (local mode only) |
| `/ocr` | POST | Process image (local or API mode) |
| `/progress` | GET | Streaming token progress (polled every 200ms) |
| `/config` | GET/POST | Get/set API config |

## Known CSS Gotchas

- The header uses flex layout with `align-items: center` and has NO fixed height — any added content (like `<details>` elements) will grow the header and shrink the main panel
- `.ocr-preview-image` must NOT have a `max-height` constraint — let flex layout and `overflow: auto` on the wrapper handle sizing
- The `<details>` element for API Settings is native HTML collapse — its open/close state changes header height at runtime
