# MIEL. Pâtisserie — scroll tour + AI voice counter

Vue 3 + Vite + Tailwind v4 frontend, FastAPI token service, and a LiveKit Agents worker.
**Author:** Lucky John Faderon, AI Solutions Architect.

Pastry variant of the HAVEN. demo: a fictional bakery page with
- **Scroll tour** (`web/src/components/tour/PastryTour.vue`): storefront → display window (pastries labelled) →
  camera pushes into the croissant → macro croissant anatomy. Pinned, scrubbed GSAP over illustrated SVG.
- **Exploded entremet** (`LayersSection.vue`): the signature cake separates into labelled layers on scroll.
- **Menu** (`MenuSection.vue`), ordering and visit sections. Light cream / raspberry / pistachio palette.
- **Voice counter**: the shared CONNECT. agent with persona `miel` (`AGENT_PERSONA=miel`); its menu, hours and
  cautious allergen rules live in `contact-center-demo/backend/agent/miel_prompt.py`.

Dev server: http://localhost:5399. Not deployed yet.

## Setup
Requires Node ≥ 20, `uv`, Python 3.12/3.13 (livekit-agents does not yet support 3.14 in this project's config).

```bash
cp .env.example .env        # fill LIVEKIT_URL / LIVEKIT_API_KEY / LIVEKIT_API_SECRET
cd web && npm install
cd ../backend && uv sync --python 3.13
```

Run three processes:

```bash
# 1. API (port 8000)
cd backend && uv run uvicorn app.main:create_app --factory --reload --port 8000
# 2. Agent worker
cd backend && uv run python -m agent.worker download-files   # first run only: fetch VAD/turn models
cd backend && uv run python -m agent.worker dev
# 3. Web (port 5199, proxies /api to :8000)
cd web && npm run dev
```

### Verify a live call
1. Open http://localhost:5199 and press **Start a conversation**; allow the microphone.
2. Expect: "Connecting" → "Agent is joining" → greeting audio, captions, waveform reacting.
3. Check: mute, interrupt the agent mid-sentence, end call, kill the API mid-call (reconnect state).
4. Agent logs show the job dispatched; if no agent joins, check the worker is registered under `AGENT_NAME`.

## Security
- API secret/key only in `.env` (git-ignored); the browser receives a JWT only.
- Token: unique room + identity per call, `roomJoin` for that room only, microphone-only publishing, no data
  publishing, 10-minute TTL, `Cache-Control: no-store`; secrets and tokens are never logged.
- Rate limiting: per-IP (5/min) and global (60/min) sliding windows, in memory — **single process only**; use
  Redis if you scale out. `TRUST_PROXY_HEADERS` must stay `false` unless behind a proxy you control.
- CORS restricted to `ALLOWED_ORIGINS`. Agent enforces `MAX_CALL_SECONDS` (default 60).

## Costs and limits (verify current pricing on livekit.io/pricing)
STT/LLM/TTS default to LiveKit Inference descriptors (`deepgram/nova-3`, `openai/gpt-4.1-mini`,
`cartesia/sonic-3`), billed through your LiveKit Cloud plan; the free tier has limited included usage and
concurrent sessions, after which usage is charged. Set `MAX_CALL_SECONDS` and rate limits conservatively.
Swap providers via `STT_MODEL` / `LLM_MODEL` / `TTS_MODEL` (any LiveKit Inference model string), or replace
the strings in `backend/agent/worker.py` with provider plugin instances using your own provider keys.

## Layout
```
web/src/components/{ui,layout,sections,graphics,voice}   web/src/stores/call.ts (LiveKit + state)
backend/app      FastAPI (config, tokens, ratelimit, main)
backend/agent    LiveKit worker + system prompt
design-system/   MASTER.md      PROGRESS.md   milestone tracker
```

## Testing results (what was run)
| Check | Result |
|---|---|
| `uv run pytest` | 5 passed (scoped JWT, per-call rooms, rate limit, window expiry, CORS) |
| `npm run build` (vue-tsc + vite) | passes; entry JS 274 kB (102 kB gzip), livekit-client lazy 154 kB gzip |
| Browser (built-in pane) 800px & 375px | renders; no horizontal overflow; no console errors |
| Mic denied → error + retry | works (pane blocks mic) |
| Live LiveKit call | **not run** (no credentials) |
| Lighthouse / screen-reader / real-device audio | not run |

## Future expansion (not implemented)
Seams: per-call agent selection via `agent_name` + room metadata (multi-agent, multi-tenant); prompt in
`agent/prompt.py` (prompt management); transcripts already stream over `lk.transcription` (persist server-side);
`RoomOptions`/dispatch rules accommodate SIP; tools can be added to `Agent` for CRM/RAG/escalation.
