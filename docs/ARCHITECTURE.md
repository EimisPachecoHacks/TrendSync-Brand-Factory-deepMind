# Architecture

TrendSync Brand Factory is a three-tier system: React frontend on Vercel, three FastAPI microservices on Google Cloud Run, and Supabase + Redis for persistence. Gemini 3 models power every AI decision.

## System Overview

```
┌─────────────────────────────────────────────────────────────────────────┐
│  Frontend (Vercel)                                                      │
│  React 18 + TypeScript + Vite + Tailwind (neumorphic pastel theme)     │
│  src/lib/api-client.ts → all backend calls                             │
│  src/contexts/AuthContext.tsx → Supabase Auth                           │
│  src/services/db-storage.ts → Supabase CRUD                            │
└──────────────┬──────────────────────────┬──────────────────┬────────────┘
               │ REST                     │ REST             │ WebSocket
               ▼                          ▼                  ▼
┌──────────────────────┐  ┌─────────────────────┐  ┌────────────────────┐
│  Main Backend        │  │  Video Gen Service  │  │  Voice Companion   │
│  Cloud Run :8080     │  │  Cloud Run :8080    │  │  Cloud Run :8080   │
│  FastAPI             │  │  FastAPI            │  │  FastAPI + WS      │
│  ADK Agent "Lux"     │  │  Veo 3.1 pipeline   │  │  ADK + Gemini Live │
│  60+ endpoints       │  │  Storyboard + video │  │  PCM audio stream  │
└──────┬───────────────┘  └──────┬──────────────┘  └──────┬─────────────┘
       │                         │                        │
       ▼                         ▼                        ▼
┌─────────────────────────────────────────────────────────────────────────┐
│  Shared Modules (trendsync-backend/shared/)                             │
│  trend_engine · collection_engine · image_generator · brand_guardian    │
│  techpack_generator · ad_video_engine · design_tools · foxit_service   │
│  pipeline_orchestrator · cache · image_utils                            │
└──────┬──────────────────┬───────────────────┬───────────────────────────┘
       │                  │                   │
       ▼                  ▼                   ▼
┌──────────────┐  ┌──────────────┐  ┌─────────────────────────────────────┐
│  Gemini 3    │  │  Veo 3.1     │  │  External Services                  │
│  Pro/Flash/  │  │  us-central1 │  │  Supabase (DB + Auth + Storage)     │
│  Image/Live  │  │              │  │  Redis (24h TTL cache)              │
│  global      │  │              │  │  Foxit Cloud (PDF conversion)       │
└──────────────┘  └──────────────┘  │  GCS (media storage)               │
                                    │  Resend (email notifications)       │
                                    └─────────────────────────────────────┘
```

## Deployment

All three backend services share a **single Docker image** (`python:3.11-slim` + ffmpeg). The `SERVICE` env var selects which service to start at runtime via `entrypoint.py`, which uses `importlib.util.spec_from_file_location()` to handle hyphenated directory names (`main-backend`, `video-gen-service`, `voice-companion`).

| Service | Cloud Run Name | `SERVICE` env | Memory | Min Instances | Timeout |
|---------|---------------|---------------|--------|---------------|---------|
| Main Backend | `trendsync-main` | `main-backend` | 2Gi | 1 (no cold starts) | 600s |
| Video Gen | `trendsync-video` | `video-gen-service` | 2Gi | 0 (scale to zero) | 600s |
| Voice Companion | `trendsync-voice` | `voice-companion` | 2Gi | 0 (scale to zero) | 3600s |

**Frontend**: Deployed on Vercel with `VITE_API_BASE_URL` and `VITE_VOICE_COMPANION_URL` env vars pointing to Cloud Run URLs. Static SPA build — env vars baked at build time.

**IAM**: Cloud Run default SA (`294722956004-compute@developer.gserviceaccount.com`) requires `roles/aiplatform.user` for Vertex AI access.

## Gemini Model Map

| Model | SDK Location | Purpose | Thinking |
|-------|-------------|---------|----------|
| `gemini-3-pro-preview` | `global` | Collection planning, tech packs, storyboards | `thinking_level=HIGH` |
| `gemini-3-pro-image-preview` | `global` | Product image generation and editing | — |
| `gemini-2.5-flash` | `global` | Trends (+ Google Search), art-direction, ADK design agent | — |
| `gemini-live-2.5-flash-native-audio` | `us-central1` | Voice companion (bidirectional streaming) | — |
| `veo-3.1-generate-preview` | `us-central1` | Ad video generation (5-scene cinematic) | — |

All models use `google-genai` SDK with `vertexai=True`. Gemini 3 requires `thinking_level` (HIGH/MEDIUM/LOW enum), **not** `thinking_budget` (that's Gemini 2.5).

## Frontend Architecture

### Tech Stack
- **React 18** + TypeScript + Vite (dev: `:5173`, prod: Vercel)
- **Tailwind CSS** with neumorphic pastel theme (`shadow-neumorphic`, `rounded-3xl`)
- **Supabase Auth** via `AuthContext.tsx` (JWT sessions, `user_profiles` on first login, `login_audit`)
- **DB abstraction** via `db-storage.ts` (all Supabase CRUD with RLS scoping)

### Component Tree

```
src/
├── App-v2.tsx                          # Root: view routing, collection state, progress
├── lib/
│   └── api-client.ts                   # Centralized REST client (60+ endpoints)
├── contexts/
│   └── AuthContext.tsx                  # Supabase Auth + user profiles + login audit
├── services/
│   └── db-storage.ts                   # CRUD: brands, styles, collections, items, trends, validations
├── components/
│   ├── auth/AuthPage.tsx               # Login / signup
│   ├── landing/LandingPage.tsx         # Welcome screen
│   ├── layout/Sidebar.tsx              # Navigation & view routing
│   ├── dashboard/
│   │   ├── Dashboard.tsx               # Main overview
│   │   └── RedisHealthCheck.tsx        # Cache status monitor
│   ├── brand-editor/
│   │   ├── BrandStyleView.tsx          # Brand editor container
│   │   ├── BrandStyleEditor.tsx        # Style JSON editor
│   │   ├── ColorPaletteEditor.tsx      # Color management
│   │   ├── CameraSettingsEditor.tsx    # Camera parameters
│   │   ├── LightingConfigEditor.tsx    # Lighting presets
│   │   ├── MaterialLibraryEditor.tsx   # Fabric/material database
│   │   └── NegativePromptsEditor.tsx   # Exclusion keywords
│   ├── brand-guardian/
│   │   ├── ValidationDemo.tsx          # Compliance UI
│   │   └── ValidationPanel.tsx         # Validation results display
│   ├── trends/
│   │   └── TrendInsightsView.tsx       # Trend analysis display
│   ├── collection/
│   │   ├── CollectionPlannerEnhanced.tsx   # Generation UI
│   │   ├── ProductGallery.tsx              # Product grid
│   │   ├── ProductDetailModal.tsx          # Product detail (6 tabs)
│   │   ├── DesignAdjustments.tsx           # Typing design companion panel
│   │   ├── AdvertisementVideo.tsx          # Video player
│   │   └── CollectionLibrary.tsx           # Saved collections browser
│   ├── voice/
│   │   └── VoiceCompanion.tsx          # Voice UI + WebSocket + audio capture/playback
│   ├── pipeline/
│   │   ├── PipelineRunner.tsx          # Full pipeline executor
│   │   └── PipelineStepCard.tsx        # Step progress display
│   ├── settings/Settings.tsx           # App configuration
│   └── ui/
│       ├── Modal.tsx                   # Modal wrapper
│       └── LoadingSkeleton.tsx         # Skeleton loaders
```

### API Client (`src/lib/api-client.ts`)

Centralized REST client — **zero direct Gemini calls from the browser**. All AI interactions go through the backend. Performance tracking logs color-coded by speed (green = Redis cached <200ms, amber = fresh API call).

Key endpoint groups:
- **Trends**: `POST /trends`, `GET /trends/celebrities`
- **Collections**: `POST /generate-collection`, `GET /collections/{id}`, `GET /collections`
- **Images**: `POST /generate-image`, `POST /edit-image`, `POST /direct-edit-image`
- **Design**: `POST /adk/design-companion`, `POST /design/chat`, `POST /save-design`
- **Tech Packs**: `POST /generate-techpack`, `POST /generate-techpack-pdf`, `POST /generate-lookbook`
- **Video**: `POST /generate-ad-video`, `GET /ad-videos/{id}`, `POST /generate-product-video`
- **Pipeline**: `POST /adk/pipeline`, `GET /adk/pipeline/{id}/status`
- **Voice**: `ws://{VOICE_URL}/ws/voice-companion/{sessionId}`

## Backend Architecture

### Main Backend (`services/main-backend/main.py`)

The API gateway — 60+ routes. Handles all Gemini calls, ADK agent orchestration, Foxit PDF generation, and proxies WebSocket connections to the voice service.

**Background tasks** (async via FastAPI `BackgroundTasks`):
- Collection generation: trends → plan → images
- Ad video generation: storyboard → Veo 3.1 scenes → ffmpeg stitch
- Product video generation: single product → Veo 3.1
- Full pipeline orchestration: all 4 steps chained

**ADK Design Companion ("Lux")**: Google ADK agent with `gemini-2.5-flash`, 7 tools from `shared/design_tools.py`, fresh session per request, external `_IMAGE_STORE` for base64 images. See [docs/DESIGN_AGENTS.md](DESIGN_AGENTS.md) for full details.

### Video Gen Service (`services/video-gen-service/main.py`)

Dedicated Veo 3.1 pipeline. Generates 5-scene storyboards (hook, hero, detail, lifestyle, CTA), renders each scene via Veo with product images as style references, stitches with ffmpeg.

- Model: `veo-3.1-generate-preview` (location: `us-central1`, **not** global)
- ffmpeg: auto-detected via `shutil.which()` with fallback to `/usr/bin/ffmpeg`

### Voice Companion (`services/voice-companion/main.py`)

ADK agent with `gemini-live-2.5-flash-native-audio` for bidirectional voice streaming.

- WebSocket endpoint: `/ws/voice-companion/{sessionId}`
- Audio: raw PCM at 16kHz as binary WebSocket frames (no JSON+base64 overhead)
- 10 tools: 7 shared design tools + video generation + page navigation + collection generation
- Tool deduplication: 30-second cache prevents duplicate execution on LLM retries
- Status feedback: `tool_status` messages sent via WebSocket so frontend shows "Editing image..."

See [docs/DESIGN_AGENTS.md](DESIGN_AGENTS.md) for the full agent architecture.

### Shared Modules (`shared/`)

| Module | Purpose | Model Used |
|--------|---------|------------|
| `trend_engine.py` | Gemini + Google Search grounding for fashion trends. Two-pass (grounded text → JSON parse). Truncated JSON repair. 429 retries. | `gemini-2.5-flash` |
| `collection_engine.py` | Two-phase collection planning (Phase A: structure, Phase B: expand). Validation + repair (3 retries). | `gemini-3-pro-preview` (HIGH) |
| `image_generator.py` | Two-step pipeline: art-direction prompt (Flash) → image gen (Image model). Compression for edits (>500KB → max 1024px JPEG). | `gemini-2.5-flash` + `gemini-3-pro-image-preview` |
| `brand_guardian.py` | Rule-based compliance scoring: color distance, camera range, lighting temp, negative prompts. Score = 100 - (critical×25) - (warning×10) - (suggestion×3). | None (pure math) |
| `techpack_generator.py` | Manufacturing specs: fabrics, measurements, construction notes, QC. Saved to Supabase as single source of truth. | `gemini-3-pro-preview` |
| `ad_video_engine.py` | 5-scene storyboard generation → Veo 3.1 video rendering → ffmpeg stitch. | `gemini-3-pro-preview` + `veo-3.1` |
| `design_tools.py` | 7 tools shared by typing + voice agents: edit_image, make_compliant, generate_variation, analyze_product, get_trends, check_compliance, save_design_signal. 429 retry with exponential backoff. | Various |
| `foxit_service.py` | python-docx builds DOCX → Foxit Cloud converts to PDF → compress → return. Async polling (2s interval, 120s timeout). | None |
| `pipeline_orchestrator.py` | Chains: trends → collection → images → video. Single endpoint orchestration. | All |
| `cache.py` | Redis caching with `@cached(prefix, ttl)` decorator. Falls back to in-memory dict. SHA256 cache keys. 24h default TTL. | None |
| `image_utils.py` | Image resizing and format conversion helpers. | None |

## Database (Supabase)

12 tables with Row-Level Security (all scoped to `auth.uid`):

| Table | Purpose | Key Columns |
|-------|---------|-------------|
| `brands` | Brand masters | `id`, `user_id`, `name`, `description` |
| `brand_styles` | Versioned JSONB style configs | `brand_id`, `style_json` (colorPalette, cameraSettings, lightingConfig, negativePrompts, materialLibrary, logoRules) |
| `collections` | Collection headers | `brand_id`, `name`, `season`, `description` |
| `collection_items` | Products within collections | `collection_id`, `name`, `category`, `image_url`, `compliance_score` |
| `trend_insights` | Cached trend analysis | `season`, `region`, `data_json`, `expires_at` |
| `validations` | Compliance scores per product | `item_id`, `score`, `violations_json` |
| `generated_images` | Image metadata + URLs | `item_id`, `url`, `prompt` |
| `tech_packs` | Manufacturing specs | `item_id`, `techpack_json`, `techpack_generated` |
| `generation_jobs` | Job queue (status, progress) | `type`, `status`, `progress`, `result_json` |
| `user_profiles` | User metadata + preferences | `user_id`, `display_name`, `role` |
| `login_audit` | Login events for security | `user_id`, `email`, `ip_address`, `timestamp` |

**Gotcha**: `auth.users.confirmed_at` is a generated column — only update `email_confirmed_at`.

**Connection**: Port **6543** (pooler), not 5432 (direct refuses connections).

## Caching Strategy

Redis with `@cached(prefix, ttl)` decorator in `shared/cache.py`.

| Prefix | TTL | What's Cached |
|--------|-----|---------------|
| `trends` | 86400s (24h) | Trend analysis results per season/region/demographic |
| `celebrities` | 86400s (24h) | Celebrity fashion influence data |
| `art_direction` | 86400s (24h) | Art-direction prompts per product description |

Falls back to in-memory dict if Redis is unavailable. Cache keys are SHA256 hashes of serialized function arguments (large blobs >10KB excluded from key generation).

**Clear cache**: `DELETE /cache/{prefix}` (e.g., `/cache/trends`).

**Stats**: `GET /cache/stats` returns hit/miss counters, memory usage, keys per prefix.

## Key Data Flows

### Full Pipeline (`POST /adk/pipeline`)
```
Request → Background Task
  1. Trends: Gemini 2.5 Flash + Google Search → trend JSON (cached 24h)
  2. Collection: Gemini 3 Pro (thinking_level=HIGH) → 2-phase plan → validate → repair
  3. Images: For each product: Flash art-direction → Image model generation → compress
  4. Video: Gemini 3 Pro storyboard → Veo 3.1 per scene → ffmpeg stitch
  ← Poll GET /adk/pipeline/{id}/status
```

### Tech Pack PDF (`POST /generate-techpack-pdf`)
```
Request (must have techpack_generated=true in DB)
  → Read saved tech pack from Supabase (single source of truth)
  → python-docx builds styled DOCX (navy headers, alternating tables)
  → Upload to Foxit Cloud → Convert DOCX→PDF → Compress → Download
  ← Return base64 PDF
```

### Design Companion Image Edit
```
User types "make the jacket terracotta"
  → POST /adk/design-companion (with product image as multimodal Part)
  → ADK Agent selects edit_image tool
  → design_tools.py: compress input (max 1024px JPEG) → call image_generator
  → image_generator.py: Gemini 3 Pro Image edits the image (with retries)
  → Store result in _IMAGE_STORE (NOT in ADK state)
  → Extract after run_async() → return to frontend
  → Frontend updates local state (NOT saved to DB until user clicks Save)
```

### Voice Companion Flow
```
User speaks → mic capture at 16kHz PCM
  → Binary WebSocket frame to /ws/voice-companion/{sessionId}
  → ADK Runner with Gemini Live processes audio
  → Agent selects tool → tool executes → status messages sent via WS
  → Agent speaks response → raw PCM audio frames sent via WS binary
  → If image edit: JSON { type: "image_updated", image_base64 } sent via WS
  → Frontend plays audio + updates image in local state
```

## Critical Architecture Patterns

### External Image Store (ADK Token Limit)
ADK serializes every `function_response` into conversation history. A single base64 image (~1.5MB text) would exhaust the token limit by the second edit. Solution:
- Typing agent: `_IMAGE_STORE` dict in `design_agent.py`
- Voice agent: `_pending_images` list in `voice-companion/main.py`
- Tools return text-only responses; images extracted after `run_async()`

### In-Memory State for Unsaved Edits
Image edits from both agents update **local React state only** — never written to DB until the user clicks "Save Design." No polling loops fetch from DB while the design panel is open (removed after discovering they reverted in-memory edits).

### Background Task + Polling
Long-running operations (collection gen, video gen, pipeline) use FastAPI `BackgroundTasks`:
1. `POST` creates a job, returns `{id, status: "pending"}`
2. Background task updates in-memory status dict
3. Frontend polls `GET /{id}` until `status: "completed"`

### Two-Pass Google Search Grounding
Google Search grounding is incompatible with `response_mime_type="application/json"`. Solution:
1. First call: grounding enabled, free-form text response
2. Second call: parse grounded text into structured JSON (or extract JSON from markdown fences)

### Retry & Resilience
- **429 RESOURCE_EXHAUSTED**: Two-layer retry — `design_tools.py` (8s/16s/24s) + `image_generator.py` (5s/10s/15s) = 9 total attempts
- **Truncated JSON**: `_repair_truncated_json()` closes open strings, removes trailing partials, terminates arrays/objects
- **Collection validation**: Up to 3 repair retries if generated collection fails structural validation
- **Foxit polling**: 2s interval, 120s timeout, fallback to uncompressed PDF if compression fails

## Dependencies

### Python (`trendsync-backend/requirements.txt`)
- `google-adk==1.18.0` — Google Agent Development Kit
- `google-cloud-storage>=2.14.0` — GCS media uploads
- `python-dotenv==1.0.1` — Environment variables
- `httpx>=0.27.0` — Async HTTP client
- `python-docx>=1.1.0` — DOCX generation for Foxit
- `redis>=5.0.0` — Caching layer
- `Pillow>=10.0.0` — Image compression/processing
- `requests>=2.31.0` — HTTP requests
- `websockets>=12.0` — WebSocket handling

### JavaScript (`package.json`)
- `react@18.3.1` / `react-dom@18.3.1` — Core UI
- `@supabase/supabase-js@2.57.4` — Database + Auth
- `tailwindcss@3.4.1` — Styling (neumorphic pastel theme)
- `typescript@5.5.3` — Type safety
- `lucide-react@0.344.0` — Icons
- `sonner@1.7.0` — Toast notifications
- `zod@3.24.1` — Data validation
- `ioredis@5.8.2` — Redis client (cache monitoring)
