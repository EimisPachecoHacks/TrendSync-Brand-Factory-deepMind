# TrendSync Brand Factory

AI-powered fashion design platform that analyzes real-time trends via Gemini + Google Search, generates brand-compliant collections with AI imagery, produces manufacturing-ready tech packs as PDFs via Foxit, creates ad videos with Veo 3.1, and offers both typing and voice AI design companions — all powered by Google's Gemini 3 ecosystem and deployed serverless on Google Cloud Run.

> **Live Demo:** [trendsync-brand-factory.vercel.app](https://trendsync-brand-factory.vercel.app)

## Inspiration

The fashion industry loses billions annually to trend misalignment — brands design collections months in advance, only to find they've missed what consumers actually want. I watched independent designers struggle with the same cycle: manually scrolling Instagram, guessing at trends, and hoping their collections would land. I thought — what if AI could close that gap entirely? What if a single platform could analyze real-time global fashion trends, generate brand-compliant product imagery, produce manufacturing-ready tech packs as professionally formatted PDFs, and even create ad-ready video content — all powered by Gemini and enhanced by Foxit's document automation?

The "aha" moment came when I imagined the Veo-powered feature: a designer uploads their brand, and Gemini generates a video of their future self presenting a completed collection on a runway. "This can be your future you." That emotional hook — seeing your vision materialized before you've even cut fabric — became the soul of TrendSync Brand Factory. And the practical backbone became Foxit — because a fashion collection isn't real until it's a PDF you can send to a manufacturer.

## What it does

TrendSync Brand Factory is an end-to-end AI fashion design platform built entirely on Google's Gemini 3 ecosystem with Foxit document automation for professional output:

- **Trend Intelligence Engine**: Uses `gemini-2.5-flash` with Google Search grounding to analyze real-time fashion trends across 6 global markets (LA, NYC, London, Tokyo, Paris, Seoul), broken down by colors, silhouettes, materials, themes, and celebrity influence — all sourced from live web data, not stale datasets. Includes automatic retry logic for rate limits and truncated JSON repair when Gemini responses are cut off mid-stream. Results are cached in Redis with 24-hour TTL to minimize API costs.

- **AI Collection Generator**: Takes trend insights + brand guidelines and uses `gemini-3-pro-preview` with `thinking_level=HIGH` for deep multi-step collection planning. A two-phase approach — Phase A plans the collection structure, Phase B expands each product with 200-300 word image prompts — followed by validation and automated repair (up to 3 retries) ensures every collection is structurally complete.

- **AI Image Generation**: A two-step pipeline — `gemini-2.5-flash` builds a detailed art-direction prompt incorporating brand style, lighting, camera settings, and trend data, then `gemini-3-pro-image-preview` (Gemini's native image generation model) generates the actual product image. Images are stored in Google Cloud Storage with signed URLs, falling back to base64 data URLs when needed.

- **Brand Guardian**: A rule-based compliance engine that scores every generated product against the brand's color palette, negative prompts, camera settings, and lighting configuration — automatically flagging violations. Compliance scores are stored per product and displayed in the gallery view.

- **Design Companion Chat ("Lux")**: A Google ADK agent powered by `gemini-2.5-flash` with 7 specialized tools — image analysis, image editing, brand compliance adjustment, trend data fetching, compliance validation, image variation generation, and design saving. Built on Vertex AI with a critical architectural innovation: large data (base64 images) is stored externally in `_IMAGE_STORE` to prevent ADK's history serialization from exceeding token limits. Includes intelligent image compression (images > 500KB are resized to max 1024px JPEG before editing) and automatic 429 retry logic with exponential backoff. Fresh sessions per request prevent history accumulation.

- **Tech Pack Generator + PDF Pipeline**: Uses `gemini-3-pro-preview` to produce manufacturing-ready technical specifications — fabric details, measurements, construction notes, quality control standards, packaging — from a single product description. Tech packs are persisted to Supabase as the **single source of truth**: the `techpack_json` column with a `techpack_generated` flag ensures that PDFs always reflect exactly what the designer approved in the UI, never a re-hallucinated variation. PDFs are generated via a python-docx → Foxit Cloud pipeline (see dedicated section below).

- **Voice Design Companion ("Lux")**: A Google ADK agent using `gemini-live-2.5-flash-native-audio` that lets designers talk through design decisions hands-free via bidirectional WebSocket streaming — real-time PCM audio at 16kHz as binary WebSocket frames (no JSON+base64 overhead) with live status feedback ("Editing image...", "Fetching trends..."). Both typing and voice agents share the same 7 tools (`shared/design_tools.py`) and image pipeline, ensuring identical capabilities regardless of input modality. The voice agent additionally has 3 exclusive tools: video generation, page navigation, and collection generation.

- **"Future You" Ad Video Generator**: The flagship feature — powered by Veo 3.1, it takes a completed collection and generates cinematic 5-scene ad videos (hook, hero, detail, lifestyle, CTA). Gemini 3 Pro writes the storyboard with structured thinking, then Veo 3.1 renders each scene with product images as style references, stitched together with ffmpeg. The concept: "this can be your future you" — turning a brand brief into an aspirational video that feels like looking into tomorrow.

- **Full Pipeline Orchestration**: A single endpoint (`POST /adk/pipeline`) chains the entire workflow: trends → collection plan → product image generation → ad video — executing all four steps as a coordinated pipeline with status polling.

- **Login Monitoring**: Every sign-in is double-logged — a `login_audit` record in Supabase (user ID, browser, timestamp) plus a real-time email notification via Resend API to the admin, providing instant awareness of platform activity.

## How Foxit Powers Professional Document Output

Foxit's document automation APIs are central to TrendSync's professional output pipeline. We use **Foxit PDF Services** for document conversion, compression, and merging — creating a complete "generate, process, deliver" workflow that turns AI-generated fashion data into manufacturer-ready documents.

### The Problem Foxit Solves

AI can generate brilliant fashion collections, but the fashion industry runs on PDFs. Manufacturers need tech packs. Buyers need lookbooks. Emails need attachments. Without professional document output, an AI platform is just a demo. Foxit bridges the gap between AI intelligence and industry-standard deliverables.

### Architecture: DOCX-to-PDF Pipeline

```
Designer clicks "Download PDF"
        |
   POST /generate-techpack-pdf
        |
        v
   Backend (foxit_service.py)
        |
        +-- Step 1: python-docx builds styled DOCX (local)
        |   - Navy header bar with brand name + product name
        |   - 4-column product info table (SKU, Category, Price, Persona...)
        |   - 7 styled sections with blue heading bars
        |   - Alternating-row measurements table (XS-XL)
        |   - Foxit-branded footer
        |
        +-- Step 2: Upload DOCX to Foxit PDF Services (cloud)
        |   POST /pdf-services/api/documents/upload
        |   -> returns documentId
        |
        +-- Step 3: Convert DOCX to PDF (cloud)
        |   POST /pdf-services/api/documents/create/pdf-from-word
        |   -> returns taskId -> poll until COMPLETED -> resultDocumentId
        |
        +-- Step 4: Compress PDF (cloud)
        |   POST /pdf-services/api/documents/modify/pdf-compress
        |   -> MEDIUM compression level -> poll -> download
        |
        +-- Step 5: Download final PDF
        |   GET /pdf-services/api/documents/{id}/download
        |
        v
   Return base64 PDF -> Frontend downloads to user's device
```

### Lookbook Generation (Collection Export)

The "Export Lookbook" feature demonstrates Foxit's **PDF merge capability** — combining multiple documents into a single professional deliverable:

```
Designer clicks "Export Lookbook"
        |
   POST /generate-lookbook
        |
        v
   For each product in collection:
        +-- Build individual DOCX (python-docx)
        +-- Upload to Foxit -> Convert to PDF
        |
   Foxit PDF Services: Merge all PDFs
        POST /pdf-services/api/documents/enhance/pdf-combine
        -> poll -> resultDocumentId
        |
   Foxit PDF Services: Compress merged PDF
        POST /pdf-services/api/documents/modify/pdf-compress
        -> poll -> download
        |
        v
   Return single lookbook PDF with all products
```

### Single Source of Truth Pattern

A critical design decision ensures document integrity: tech packs are **saved to the database before PDF generation is allowed**. The backend endpoint returns HTTP 400 if no saved tech pack exists. This prevents Gemini from hallucinating different values between what the designer sees in the UI and what appears in the PDF.

### Why Foxit Was Essential

1. **Professional Styling**: `python-docx` builds richly styled documents — navy header bars, alternating table rows, branded color palette, proper typography — that look like they came from a design agency, not a code generator.

2. **Cloud Conversion**: Foxit's PDF Services API handles DOCX-to-PDF conversion server-side with no local LibreOffice or headless browser dependency. The async task pattern (submit -> poll -> download) is reliable and production-ready.

3. **PDF Compression**: Fashion tech packs with detailed specs can be large. Foxit's compression (MEDIUM level) reduces file sizes for email attachments and faster downloads, with graceful fallback if compression fails.

4. **PDF Merging**: The lookbook feature — combining 5-20 individual tech pack PDFs into one document — would require complex PDF manipulation libraries locally. Foxit's `pdf-combine` endpoint handles this cleanly in the cloud.

5. **No Infrastructure Overhead**: Authentication is simple (client_id + client_secret headers, no OAuth), the API is RESTful, and the async polling pattern integrates naturally with Python's `httpx`. Zero DevOps burden.

## Brand Guardian — How Validation Works

The Brand Guardian is a **rule-based compliance engine** (not hardcoded scores). It validates every product's design specification against the brand's style configuration in real-time.

### Validation Checks

| Check | What It Does | Severity |
|-------|-------------|----------|
| **Color Palette** | Extracts hex colors from the product's `color_scheme` and measures Euclidean RGB distance against the brand palette. If distance > 30 (perceptually different), it flags a violation. | `suggestion` |
| **Camera Settings** | Checks focal length (converted to FOV) and camera angle against brand-defined min/max ranges. | `warning` |
| **Lighting** | Compares lighting temperature (warm vs cool) against the brand's configured color temperature (e.g., 5000K). | `suggestion` |
| **Negative Prompts** | Scans product description and object descriptions for forbidden terms defined in brand style (e.g., "blurry", "low quality"). | `critical` |

### Scoring Formula

```
compliance_score = 100 - (critical x 25) - (warning x 10) - (suggestion x 3)
```

- **100%** = No violations found — product fully matches brand guidelines
- **75-99%** = Minor suggestions (e.g., trend colors differ from brand palette)
- **50-74%** = Warnings present (e.g., camera angle out of range)
- **<50%** = Critical violations (e.g., forbidden terms in description)

### Where Brand Rules Are Stored

Brand style rules are stored in the **Supabase `brand_styles` table** as a JSONB column (`style_json`), configured via the Brand Style Editor page:

```json
{
  "colorPalette": [{ "name": "Brand Navy", "hex": "#1a237e", "designation": "primary" }],
  "cameraSettings": { "fovMin": 20, "fovMax": 80, "angleMin": 0, "angleMax": 90 },
  "lightingConfig": { "colorTemperature": 5000 },
  "negativePrompts": ["blurry", "low quality", "distorted"],
  "materialLibrary": [...],
  "logoRules": {...}
}
```

**Implementation:** `trendsync-backend/shared/brand_guardian.py` (`validate_prompt()` function)

## How I built it

The architecture is a three-tier system designed to keep Gemini at the center of every intelligent decision, deployed serverless on Google Cloud Run with Foxit handling the professional document output layer:

**Frontend** — React 18 + TypeScript + Vite with a custom neumorphic pastel design system, deployed on Vercel. Every AI interaction goes through a centralized API client (`api-client.ts`) — zero direct Gemini calls from the browser, keeping API keys secure server-side.

**Backend (3 FastAPI microservices on Google Cloud Run)**:
- **Main Backend**: The brain. All Gemini calls route through here using `google-genai` SDK with Vertex AI (`vertexai=True`). Hosts the Google ADK design companion agent ("Lux"), endpoints for trends (Google Search grounding), collection generation, image gen/edit, tech packs, PDF generation via Foxit, lookbook export, ad video orchestration, and a WebSocket proxy to the voice service. Includes Redis caching (24h TTL) to reduce API costs.
- **Video Generation Service**: Dedicated Veo 3.1 pipeline for the "Future You" ad video feature. Takes collection data + product images, generates cinematic fashion videos in `us-central1`.
- **Voice Companion**: WebSocket server running a Google ADK agent with `gemini-live-2.5-flash-native-audio` for real-time bidirectional voice interaction during design sessions. Features tool status feedback so designers see what's happening during long operations.

All three services share a single Docker image with an `entrypoint.py` that dynamically loads the correct service via the `SERVICE` env var — deployed to Cloud Run with `min-instances=1` on the main backend (eliminating cold starts) and auto-scale-to-zero on video and voice services.

**Document Layer** — Foxit PDF Services API for DOCX-to-PDF conversion, compression, and multi-document merging. `python-docx` generates styled DOCX files locally; Foxit's cloud handles the rest.

**Database & Auth** — Supabase (PostgreSQL + Auth) with 11 tables, Row-Level Security, a Redis caching layer for trend insights and structured prompts, and a `login_audit` table for security monitoring.

**Cloud Storage** — Google Cloud Storage for product images and generated videos, with signed URLs for secure access.

**Email** — Resend API for login notification emails (admin alerts on every sign-in) and tech pack email delivery with PDF attachments.

**Key Gemini integration points**:

| Task | Model | Notes |
|---|---|---|
| Trend analysis | `gemini-2.5-flash` | Google Search grounding, Redis cache, truncated JSON repair |
| Collection planning | `gemini-3-pro-preview` | `thinking_level=HIGH`, 2-phase + repair |
| Image prompt building | `gemini-2.5-flash` | Art direction with brand style |
| Product image generation | `gemini-3-pro-image-preview` | Two-step pipeline, compression |
| Tech pack generation | `gemini-3-pro-preview` | Structured output, Supabase persistence |
| Design Companion | `gemini-2.5-flash` | ADK agent, 7 tools, external image store |
| Voice Companion | `gemini-live-2.5-flash-native-audio` | ADK agent, BIDI streaming, 10 tools |
| Ad video storyboard | `gemini-3-pro-preview` | 5-scene storyboard, HIGH thinking |
| Video generation | `veo-3.1-generate-preview` | Cinematic clips, style reference images |

The critical architectural decision was using Google Search as a grounding tool for trend analysis — this gives TrendSync access to *live* fashion data rather than training cutoff knowledge, making every trend report current to the day.

## Architecture

```
React (Vercel) → Cloud Run: Main Backend (:8080) → Gemini 3 Pro / Flash / Image / Veo / Foxit / GCS
                                                  → Cloud Run: Video Service (:8080) for Veo 3.1
                                                  → Cloud Run: Voice Service (:8080) for Gemini Live audio
                          ↕
                     Supabase (PostgreSQL + Auth) + Redis Cache
```

### Gemini Models Used

| Model | Location | Purpose |
|-------|----------|---------|
| `gemini-3-pro-preview` | `global` | Collection planning, tech packs, storyboards |
| `gemini-3-pro-image-preview` | `global` | Product image generation and editing |
| `gemini-2.5-flash` | `global` | Trend engine, art-direction prompts, ADK design companion |
| `gemini-live-2.5-flash-native-audio` | — | Voice design companion (bidirectional streaming) |
| `veo-3.1-generate-preview` | `us-central1` | Ad video generation (5-scene cinematic) |

## Challenges I ran into

**Google Search grounding vs. structured JSON output**: I discovered that `google_search` grounding is incompatible with `response_mime_type="application/json"` in the Gemini API. I had to implement a two-pass approach — first call with grounding enabled (free-form text enriched with live web data), then parse the grounded response into structured trend data with robust JSON extraction.

**ADK token overflow from base64 images**: Google ADK serializes every `function_response` into the conversation history for subsequent model calls. When a tool returned a base64-encoded image (~1.5MB of text), the accumulated context exploded past Gemini's token limit by the second edit. The solution: store large data in an external `_IMAGE_STORE` dictionary (never in the tool response), and extract images after `run_async()` completes. This pattern is critical for any ADK agent handling binary data.

**Image edits reverting in the UI**: After the agent edited an image, it would flash the new version and immediately revert. Root cause: two independent `setInterval` polling loops were fetching from the database every 2 seconds and overwriting the in-memory edited image (which hadn't been saved to DB yet). Removed both polling loops — in-memory state is now the source of truth for unsaved edits.

**Gemini truncating JSON responses**: Long collection plans would get cut off mid-JSON, breaking the parser. Built a `_repair_truncated_json()` function that closes open strings, removes trailing partial entries, and properly terminates arrays/objects — recovering ~90% of otherwise-lost responses.

**429 rate limits not retrying**: Gemini's RESOURCE_EXHAUSTED errors were shown as user-facing messages instead of being retried. Added two layers of retry with exponential backoff: 3 retries at the `design_tools.py` layer (8s/16s/24s) and 3 retries at the `image_generator.py` layer (5s/10s/15s) — effectively 9 attempts before giving up.

**Slow image edits (15-25s to 8-12s)**: Images were 1.2MB PNGs being sent to Gemini for every edit, with each output slightly larger than the input. Added `_compress_for_edit()`: images > 500KB are resized to max 1024px and converted to JPEG (quality 85), giving a 6-8x size reduction. The 5th edit is now as fast as the 1st.

**Gemini 3 API differences**: Gemini 3 uses `thinking_level` (HIGH/MEDIUM/LOW enum), NOT `thinking_budget` (that's Gemini 2.5). Also requires `location=global` on Vertex AI, while Veo requires `us-central1` — mixing these up causes silent failures.

**Foxit async task pattern**: Foxit PDF Services operations are async — you submit a task, get a `taskId`, and poll until `COMPLETED`. Implementing reliable polling with timeouts (120 seconds), error handling, and graceful fallbacks (if compression fails, return uncompressed PDF) required careful engineering.

**PDF data consistency**: Gemini can hallucinate different values each time it's called. Solution: save the tech pack to Supabase once (with `techpack_generated: true` flag), and enforce that the PDF endpoint only accepts pre-saved data. The frontend blocks PDF download until the tech pack is persisted.

**Vertex AI authentication on macOS to Cloud Run**: The service account credential chain had subtle issues with the `google-genai` SDK's `vertexai=True` mode locally. Moving to Cloud Run solved this — the attached service account inherits permissions natively, but required adding `roles/aiplatform.user` to the Compute Engine default SA.

**Supabase RLS infinite recursion**: A Row-Level Security policy on `user_profiles` that checked admin status by querying `user_profiles` itself created an infinite loop. Had to drop it and rely on JWT-based role checks instead.

## Accomplishments that I'm proud of

**Full pipeline from trend to video in one click**: A single `POST /adk/pipeline` endpoint chains trend analysis → collection planning (Gemini 3 Pro with deep thinking) → product image generation (gemini-3-pro-image-preview) → 5-scene ad video (Veo 3.1). A designer can go from "what's trending?" to "here's my manufacturing-ready collection with professional tech pack PDFs and a promotional video" in a single session.

**Two AI design companions sharing one brain**: Both the typing agent and voice agent ("Lux") are Google ADK agents sharing the same 7 tools via `shared/design_tools.py`. A designer can type "make the jacket terracotta" or say it out loud — same tool executes, same image pipeline runs, same result. The voice agent is actually faster (~8-12s vs ~12-18s for image edits) because Gemini Live's persistent bidirectional session eliminates the routing roundtrip.

**Real-time trend intelligence that actually works**: The combination of `gemini-2.5-flash` + Google Search grounding produces trend reports that match what I see on Vogue and WGSN — colors, silhouettes, materials, themes — all grounded in current web data. This isn't hallucinated fashion advice; it's AI-synthesized market intelligence.

**Professional document output via Foxit**: The Foxit integration transforms AI-generated data into documents that look like they came from a professional design agency. Navy branded headers, structured measurement tables, compressed PDFs ready for email — this is the difference between a hackathon demo and a production tool. The lookbook merge feature (combining multiple tech packs into a single PDF) is something manufacturers actually need.

**Single source of truth architecture**: The tech pack persistence pattern — save to DB once, generate PDF from saved data only, clear on design changes — eliminates the #1 risk of AI-generated documents: inconsistency. What you see in the UI is exactly what appears in the PDF.

**Zero API keys in the browser**: Every Gemini call, every Foxit call, every video generation goes through the FastAPI backend on Cloud Run. The frontend is a pure presentation layer deployed on Vercel — secure by architecture, not by obscurity.

**The "Future You" concept**: Using Veo 3.1 to generate aspirational 5-scene runway videos from a brand brief creates an emotional moment that no competitor offers. It transforms AI from a tool into a creative partner that shows you what's possible.

## What I learned

**Gemini 3's native image generation changes everything**. Using `gemini-3-pro-image-preview` instead of external image APIs means the model that understands your design intent is the same model generating the image — no prompt translation layer, no API mismatch. The two-step pipeline (Flash writes art-direction prompt → Image model renders) gives precise brand-compliant results.

**Google ADK is the right abstraction for multi-tool agents**. Building the design companion with ADK's `Agent` + `Runner` + `ToolContext` pattern meant I could add tools incrementally, share them between typing and voice agents, and let Gemini handle tool selection — no manual routing logic. Fresh sessions per request prevent token accumulation.

**Gemini's Google Search grounding is a game-changer for real-time applications**. Most AI apps are limited by training cutoffs — TrendSync breaks that barrier entirely. Fashion trends shift weekly; grounded search means the platform is always current.

**`gemini-2.5-flash` is remarkably capable for production workloads**. I expected to need Pro for most tasks, but Flash handled trend analysis, conversational design chat, tech pack generation, and brand compliance reasoning with excellent quality at a fraction of the cost and latency. I reserved `gemini-3-pro-preview` only for the most complex multi-step reasoning tasks (collection planning, storyboards).

**The Gemini ecosystem is genuinely composable**. Gemini 3 Pro for deep reasoning, Gemini 3 Pro Image for generation, Flash for speed, native audio for voice, Veo 3.1 for video, Google Search for grounding, Google ADK for agent orchestration — these aren't separate products bolted together; they share the same SDK patterns (`google-genai` with `vertexai=True`) and work naturally as a unified AI backend.

**Foxit's APIs are production-ready with minimal friction**. Simple auth (client_id + client_secret headers), clean REST endpoints, and the async task pattern is straightforward to implement. The combination of DOCX-to-PDF conversion, compression, and merge covers the full document lifecycle without any local dependencies like LibreOffice or Puppeteer.

**Architecture matters more than model size**. The biggest improvements came not from switching models, but from designing the right prompts, caching strategy (Redis with 24h TTL), image compression pipeline, retry logic, and data flow. A well-structured `gemini-2.5-flash` call with proper context outperformed naive Pro calls every time. Similarly, the single source of truth pattern for tech packs solved a consistency problem that no amount of prompt engineering could fix.

## What's next for TrendSync Brand Factory

- **Multi-brand portfolio management**: Supporting agencies that manage multiple fashion brands, each with distinct guidelines, from one dashboard.
- **Runway video customization**: Letting designers control Veo 3.1 parameters — venue style, lighting mood, model demographics, music genre — to create truly personalized "Future You" videos.
- **Supplier matching**: Using Gemini to match tech pack specifications with a database of global manufacturers, completing the design-to-production pipeline.
- **Mobile companion app**: A voice-first mobile interface using `gemini-live-2.5-flash-native-audio` so designers can iterate on collections while away from their desk — "Hey TrendSync, swap the jacket color to the trending terracotta I saw in the Seoul report."
- **GCS storage integration**: Moving generated images and videos from base64 data URLs to Google Cloud Storage for better performance and persistence.
- **Foxit watermarking**: Adding brand watermarks to tech pack PDFs via Foxit's PDF Services watermark API for intellectual property protection during the vendor review process.
- **Interactive PDF lookbooks**: Using Foxit's advanced PDF features to add clickable navigation, embedded color swatches, and interactive measurement tables to collection lookbooks.

## Getting Started

### Prerequisites
- Node.js 22.x
- Python 3.11+
- Google Cloud project with Vertex AI enabled
- Supabase project (PostgreSQL + Auth)
- Redis instance (optional, for caching)
- Foxit PDF Services credentials

### Frontend
```bash
npm install
npm run dev          # http://localhost:5173
```

### Backend
```bash
cd trendsync-backend
pip install -r requirements.txt

# Main API (port 8000)
uvicorn services.main-backend.main:app --host 0.0.0.0 --port 8000 --reload

# Video service (port 8001)
uvicorn services.video-gen-service.main:app --host 0.0.0.0 --port 8001 --reload

# Voice companion (port 8002)
python -m services.voice-companion.main
```

### Environment Variables
Copy `.env.example` to `.env` and configure:
- `GOOGLE_CLOUD_PROJECT` / `GOOGLE_CLOUD_LOCATION=global`
- `GOOGLE_GENAI_USE_VERTEXAI=TRUE`
- `SUPABASE_URL` / `SUPABASE_ANON_KEY`
- `REDIS_URL` (optional)
- `FOXIT_CLIENT_ID` / `FOXIT_CLIENT_SECRET`

## Tech Stack

| Layer | Technology |
|---|---|
| Frontend | React 18, TypeScript, Vite, Tailwind CSS |
| Backend | Python, FastAPI, 3 microservices |
| AI Models | Gemini 3 Pro, Gemini 3 Pro Image, Gemini 2.5 Flash, Gemini Live Audio, Veo 3.1 |
| AI Framework | Google ADK (Agent Development Kit), Vertex AI |
| Documents | Foxit PDF Services API, python-docx |
| Database | Supabase (PostgreSQL), Redis cache |
| Storage | Google Cloud Storage |
| Auth | Supabase Auth (JWT + RLS) |
| Email | Resend API |
| Video | Veo 3.1 via Vertex AI, ffmpeg |
| Deployment | Google Cloud Run (backend), Vercel (frontend) |

## License

This project is open source and available under the [MIT License](LICENSE).

## Disclaimer

This is an open source hackathon project built for the Google Agent Development Kit (ADK) hackathon. It is provided **"as is"** without warranty of any kind, express or implied, including but not limited to the warranties of merchantability, fitness for a particular purpose, and noninfringement. In no event shall the authors or copyright holders be liable for any claim, damages, or other liability arising from the use of this software.

This software is intended for **demonstration and educational purposes**. AI-generated content (images, text, videos, tech packs) may contain inaccuracies and should not be used for production manufacturing without human review. Use of Google Cloud services (Vertex AI, Cloud Run, GCS), Supabase, Foxit PDF Services, Resend, and other third-party APIs is subject to their respective terms of service and pricing. The authors are not affiliated with or endorsed by any of these service providers.

**No credentials or API keys are included in this repository.** You must provide your own service credentials to run this project.
