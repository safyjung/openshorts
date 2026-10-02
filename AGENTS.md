# AGENTS.md — OpenShorts

## Quick Start
```bash
# Full stack (Docker)
docker compose up --build
# Backend: http://localhost:8000  |  Frontend: http://localhost:5175

# Frontend only
cd dashboard && npm install && npm run dev   # port 5173

# Backend only
pip install -r requirements.txt && uvicorn app:app --host 0.0.0.0 --port 8000
```

## Test / Lint
```bash
# Backend tests (self-host mode, no ML stack)
pytest tests/ -v

# Frontend build + lint
cd dashboard && npm ci && npm run build && npm run lint
```

## Architecture Highlights
- **Backend**: FastAPI + async job queue (`app.py`), core pipeline in `main.py`
- **Frontend**: React 18 + Vite + Tailwind, SPA with hash routing (`dashboard/`)
- **MCP Server**: `/mcp` (Streamable-HTTP JSON-RPC) + stdio transport (`mcp_stdio.py`)
- **Cloud mode** gated by `BILLING_ENABLED=1` (Stripe, managed keys, autopilot)
- **Self-host** = BYOK, no `cloud/` imports unless `BILLING_ENABLED`
- **License**: Core is MIT; `cloud/` is source-available (OpenShorts Commercial License) — cannot offer as paid/hosted service to third parties

## Critical Conventions
- **Tests run in self-host mode**: `conftest.py` sets `BILLING_ENABLED=0` before imports
- **Layout picker**: sends 12 frames @ 1024px to Gemini, NOT the video (token ceiling)
- **Watermark**: free plan serves `wm_<file>` copy; clean original preserved for upgrade
- **Partial minutes**: 402 returns `partial_minutes`; dashboard resubmits with `max_minutes`
- **Deploy handover**: rolling update shares `output/`; old instance drains, heartbeats `.resume.json`
- **GPU limits**: VRAM is the bottleneck; `MAX_CONCURRENT_JOBS` sized to free VRAM, not CPU
- **Proxy accounting**: downloads go direct → static proxies → DataImpulse; never `PROXY_URL` for non-YouTube
- **SEO**: `vite-plugin-seo.js` injects `seo/landing-fallback.js` into `#root` at build; emits flat `.html` pages, `sitemap.xml`, `llms.txt` from `seo/pages.js` — keep in sync with `Landing.jsx`
- **Silent videos**: vision fallback sends whole video to Gemini (token ceiling!) — requires `GEMINI_API_KEY`

## Key Files
| File | Purpose |
|------|---------|
| `main.py` | Video pipeline: transcribe, scenes, clips, reframe, hooks, watermark |
| `app.py` | FastAPI, job queue, resume/drain, auth, webhooks, partial minutes |
| `editor.py` | Gemini → FFmpeg filters for effects |
| `hooks.py` | Hook overlay text + font rendering |
| `subtitles.py` | SRT/ASS generation, burning, dubbed video transcription |
| `translate.py` | ElevenLabs dubbing |
| `thumbnail.py` | AI thumbnail studio (titles + images) |
| `layout_picker.py` | Gemini chooses layout from 12 frames |
| `cloud/metering.py` | Quota, probe, proxy ledger, reservation |
| `cloud/autopilot.py` | Background YouTube channel clipping |
| `dashboard/vite-plugin-seo.js` | Build-time SEO injection + static page emission |

## Env Vars (Backend)
| Var | Default | Notes |
|-----|---------|-------|
| `MAX_CONCURRENT_JOBS` | 5 | Size to VRAM |
| `GPU_MIN_FREE_MB` | 4500 | Queue admission guard |
| `ASR_HOST_SLOTS` | 2 | Host-wide transcription flock |
| `LLM_BASE_URL` | — | OpenAI-compatible endpoint for moment picker (Ollama, vLLM…) |
| `BILLING_ENABLED` | 0 | Enables `cloud/` imports, Stripe, managed keys |
| `PARTIAL_MIN_MINUTES` | 5 | Min minutes for partial-offer wall |
| `FIRST_VIDEO_MAX_MINUTES` | 60 | Free account first-video grant |
| `WHISPER_MODEL` | small | `large-v3-turbo` for GPU |
| `WHISPER_DEVICE` | cpu | `cuda` for GPU |
| `TRANSCRIBE_BACKEND` | whisper | `parakeet` for ~2x faster GPU transcription |
| `FFMPEG_ENCODER` | x264 | `auto` probes nvenc |

## Common Pitfalls
- **Don't add `PROXY_URL` to local `.env`** — bills DataImpulse on every `main.py` run
- **Layout picker needs frames, not video** — 1 hr video = 1M+ tokens, 12 frames = ~3k
- **Watermark is a served copy** — never burned into canonical; upgrade moves to clean twin
- **Tests must not import `app` before `conftest.py` sets `BILLING_ENABLED=0`**
- **SEO fallback content must match `Landing.jsx`** — React replaces it on mount
- **Clip count is derived, not a setting** — see `clip_selection.py` for floor logic
- **Silent videos use vision fallback** — sends whole video to Gemini (token ceiling!)
- **`cloud/` imports are gated by `BILLING_ENABLED`** — self-host never loads them

## CI Order
```bash
# 1. Backend tests (no heavy deps)
pytest tests/ -v

# 2. Frontend build + lint
cd dashboard && npm ci && npm run build && npm run lint
```