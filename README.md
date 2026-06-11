# 📞 LiveCall.AI — AI-Powered Sales Call Evaluation Platform

> An end-to-end backend system that ingests sales call recordings, transcribes them using AWS Transcribe, evaluates the conversation against product-specific compliance and selling behaviour frameworks using GPT-4o, and delivers structured scoring reports — supporting both single-call and batch processing with real-time SSE progress streaming.

---

## 📌 Table of Contents

- [Project Overview](#project-overview)
- [Key Features](#key-features)
- [Architecture](#architecture)
- [Tech Stack](#tech-stack)
- [Project Structure](#project-structure)
- [API Endpoints](#api-endpoints)
- [Evaluation Pipeline](#evaluation-pipeline)
- [Evaluation Framework](#evaluation-framework)
- [Audio Processing](#audio-processing)
- [Batch Processing & SSE Streaming](#batch-processing--sse-streaming)
- [Database Schema](#database-schema)
- [Configuration & Environment Variables](#configuration--environment-variables)
- [Running the App](#running-the-app)
- [Challenges & Outcomes](#challenges--outcomes)

---

## Project Overview

**LiveCall.AI** is a backend API that automates the evaluation of pharmaceutical sales calls. It is designed for life sciences organisations where sales representatives (reps) conduct calls with healthcare professionals (HCPs/doctors), and those calls need to be systematically assessed for:

- **Compliance** — whether reps made unsubstantiated clinical claims or misrepresented reimbursement
- **Key Message Delivery** — whether key product messages were proactively delivered and actively discussed
- **Objection Handling** — whether HCP objections were raised and meaningfully addressed
- **Selling Behaviour** — whether the rep followed a needs-led, consultative selling approach
- **Commitment Outcomes** — whether clear next steps and prescribing commitments were reached
- **HCP Sentiment** — how the HCP reacted to each key message dimension

The system supports two pharmaceutical client products out of the box - with evaluations conducted in German.

---

## Key Features

| Feature | Description |
|---|---|
| 🎙️ Audio Transcription | AWS Transcribe with speaker diarisation, custom vocabularies, and retry logic |
| 🔊 Audio Refinement | Noise reduction, bandpass filtering, normalisation via FFmpeg + noisereduce |
| 🤖 GPT-4o Evaluation | 7 parallel prompt evaluations per call covering all selling dimensions |
| 📋 Consent Detection | Rule-based + keyword NLP consent verification before evaluation |
| 🔄 Batch Processing | Process multiple calls from Excel metadata + uploaded audio files |
| 📡 SSE Streaming | Real-time progress updates via Server-Sent Events for long-running batch jobs |
| 🏷️ Name Anonymization | GPT-4.1 powered PII detection and transcript anonymization |
| 📊 Scoring & Weighting | Weighted final score calculation with per-category breakdowns |
| 📥 S3 Multipart Upload | Presigned URL multipart upload flow for large audio files |
| 🗃️ MySQL Persistence | Call status tracking, transcript storage, evaluation results |

---

## Architecture

```
┌──────────────────────────────────────────────────────────────────────┐
│                          Client / Frontend                            │
│         Upload audio · Trigger evaluation · Poll status · View report│
└────────────────────────────┬─────────────────────────────────────────┘
                             │ HTTP / SSE
                             ▼
┌──────────────────────────────────────────────────────────────────────┐
│                       FastAPI Application                             │
│                                                                       │
│  POST /analyze/        POST /batch-analyze/sse    GET /report/{id}   │
│  POST /conversations/  GET /call_status/{id}      GET /transcription/ │
│  POST /delete/         POST /multipart/init       POST /multipart/complete│
│                                                                       │
└───────────┬─────────────────────┬────────────────────────────────────┘
            │                     │
  ┌─────────▼──────────┐   ┌─────▼────────────────────┐
  │  Single Call        │   │  Batch Call Pipeline      │
  │  analyze_services   │   │  batch_analyze_services   │
  │                     │   │  batch_analyze_sse_services│
  │  1. Check status    │   │                           │
  │  2. Transcribe      │   │  ThreadPoolExecutor       │
  │  3. Consent check   │   │  → Background thread      │
  │  4. Evaluate        │   │  → Own event loop         │
  │  5. Score & save    │   │  → SSE event stream       │
  └─────────┬──────────┘   └──────────────────────────┘
            │
  ┌─────────▼────────────────────────────────────────────────────┐
  │                    Core Processing Pipeline                    │
  │                                                               │
  │  ┌───────────────┐    ┌────────────────┐   ┌───────────────┐ │
  │  │ Audio          │    │  AWS Transcribe │   │  Consent      │ │
  │  │ Refinement     │───▶│  Speaker       │──▶│  Detection    │ │
  │  │ (FFmpeg +      │    │  Diarisation   │   │  (Rule-based) │ │
  │  │  noisereduce)  │    └────────────────┘   └───────┬───────┘ │
  │  └───────────────┘                                  │         │
  │                                              ┌──────▼───────┐ │
  │                                              │ 7× Parallel  │ │
  │                                              │ GPT-4o Evals │ │
  │                                              │ (asyncio)    │ │
  │                                              └──────┬───────┘ │
  │                                                     │         │
  │                                              ┌──────▼───────┐ │
  │                                              │ Score +       │ │
  │                                              │ Weight +      │ │
  │                                              │ Dashboard fmt │ │
  │                                              └──────────────┘ │
  └──────────────────────────────────────────────────────────────┘
            │                    │                    │
  ┌─────────▼────┐   ┌──────────▼──────┐   ┌────────▼──────┐
  │  AWS S3       │   │  AWS Transcribe  │   │  MySQL DB     │
  │  Audio files  │   │  Transcript JSON │   │  calls table  │
  │  Refined audio│   │  Output bucket   │   │  mr_task_reply│
  └──────────────┘   └─────────────────┘   └───────────────┘
                                                    │
                                           ┌────────▼──────┐
                                           │  OpenAI       │
                                           │  GPT-4o / 4.1 │
                                           └───────────────┘
```

---

## Tech Stack

| Layer | Technology |
|---|---|
| **Web Framework** | FastAPI (Python 3.10+) |
| **LLM Evaluation** | OpenAI GPT-4o (evaluation), GPT-4.1 (anonymization, summary) |
| **Transcription** | AWS Transcribe (speaker diarisation, custom vocabulary) |
| **Audio Processing** | FFmpeg, `noisereduce`, `scipy`, `soundfile`, `numpy` |
| **Async** | Python `asyncio`, `concurrent.futures.ThreadPoolExecutor` |
| **Streaming** | `sse-starlette` (Server-Sent Events) |
| **Storage** | AWS S3 (audio files, transcript JSON) |
| **Database** | MySQL (AWS Aurora) via `mysql-connector-python` |
| **Data Processing** | `pandas`, `openpyxl` |
| **HTTP Client** | `boto3` (AWS SDK) |
| **Config** | Environment variables / hardcoded (dev) |
| **Code Quality** | `black`, `isort`, `ruff`, `mypy` |

---

## Project Structure

```
livecall-backend/
├── app/
│   ├── app.py                          # FastAPI app factory, CORS, router registration
│   ├── env.py                          # Environment loader
│   │
│   ├── api/
│   │   ├── call/
│   │   │   ├── call_routers.py         # All API route definitions
│   │   │   ├── analyze_services.py     # Single-call full pipeline (transcribe + evaluate)
│   │   │   ├── batch_analyze_services.py     # Batch pipeline (parallel call processing)
│   │   │   ├── batch_analyze_sse_services.py # SSE batch with background thread
│   │   │   ├── call_recording_services.py    # S3 multipart upload, MR task reply insert
│   │   │   ├── call_status_services.py       # Call progress/status endpoint
│   │   │   ├── conversations_services.py     # Conversation list from mr_task_reply_data
│   │   │   ├── delete_services.py            # Call record deletion
│   │   │   ├── report_services.py            # Fetch scored evaluation report + transcription
│   │   │   └── audio_refinement.py           # Audio preprocessing (denoise, filter, normalise)
│   │   │
│   │   ├── user/
│   │   │   └── routers.py
│   │   │
│   │   ├── world_view/                 # (Legacy / commented-out module)
│   │   │   ├── world_view_router.py
│   │   │   └── world_view_service.py
│   │   │
│   │   └── deps.py                    # Shared FastAPI dependencies
│   │
│   ├── core/
│   │   └── config.py                  # App configuration
│   │
│   ├── crud/
│   │   └── world_view.py
│   │
│   ├── db/
│   │   ├── mysql_connector.py         # MySQL connection factory (AWS Aurora)
│   │   ├── session.py
│   │   ├── manager.py
│   │   ├── base.py / base_class.py
│   │   └── main.py                    # (Legacy SQLAlchemy async engine)
│   │
│   ├── models/
│   │   ├── call.py                    # Pydantic request/response models
│   │   └── world_view.py
│   │
│   ├── schemas/
│   │
│   ├── utils/
│   │   ├── transcribe.py              # AWS Transcribe wrapper (sync + async)
│   │   ├── s3_ops.py                  # S3 upload/download/list utilities
│   │   ├── json_ops.py                # JSON helpers
│   │   └── common_utils.py
│   │
│   └── tests/
│       └── test_users.py
│
├── alembic/                           # DB migrations
│   ├── env.py
│   └── script.py.mako
│
├── gunicorn-config.py                 # Production Gunicorn configuration
├── start.sh                           # Server startup script
├── pyproject.toml                     # Black, isort, ruff, mypy config
├── requirements.txt
└── README.md
```

---

## API Endpoints

### `POST /analyze/`
Triggers single-call evaluation pipeline asynchronously.

**Request:**
```json
{
  "call_id": "rep123_1_Survey_1",
  "user_id": "rep123",
  "s3_path": "s3://bucket/audio/call.m4a",
  "language": "de-DE",
  "projectCode": "4170DK0001",
  "customer": "ofa",
  "product": "Product",
  "region": "us-east-1",
  "is_consent": "Yes"
}
```

**Flow:** Checks if already evaluated → fires `analyze_calls()` as background `asyncio.Task` → returns `{"status": "success", "message": "Evaluation has started"}` immediately.

---

### `POST /batch-analyze/sse`
Batch evaluate multiple calls from Excel metadata + audio files. Streams real-time SSE progress events.

**Request:** `multipart/form-data`
- `metadata`: Excel file with columns `Filename`, `Product`, `User_role`, `Customer_role`
- `audio_files`: List of audio files
- `language`: e.g. `de-DE`
- `region`: e.g. `us-east-1`
- `project_code`: e.g. `4170DK0001`

**SSE Events:**
```
event: progress
data: {"status": "Transcribing & evaluating calls... (45%)", "progress": 45}

event: complete
data: {"status": "complete", "full_excel_b64": "...", "filtered_excel_b64": "...", "total": 10}
```

---

### `GET /call_status/{call_id}`
Returns current evaluation progress (0–100%) and status.

---

### `POST /report/`
Returns the full scored evaluation report for a completed call, including per-category scores, nested question details, justifications, quotes, and an AI-generated feedback summary.

---

### `GET /transcription/{call_id}`
Returns the diarised transcript as a structured JSON array with `start_time`, `end_time`, `speaker`, and `phrase` fields.

---

### `POST /conversations/`
Lists all available calls for a given `project_code` from the `mr_task_reply_data` table, joined with evaluation status from the `calls` table.

---

### `POST /multipart/init`
Initialises an S3 multipart upload. Returns `upload_id` and presigned URLs for each part.

### `POST /multipart/complete-and-mr`
Completes the S3 multipart upload and inserts respondent data into the `mr_task_reply_data` table atomically. On DB failure, deletes the orphan S3 object.

### `POST /multipart/abort`
Aborts an in-progress multipart upload.

---

### `POST /delete/`
Deletes a call record from the `calls` table.

---

## Evaluation Pipeline

```
Audio File (S3 URL)
       │
       ▼
┌─────────────────────────────────────────────────────┐
│  1. Audio Refinement (audio_refinement.py)           │
│     FFmpeg → mono WAV → noise reduction →           │
│     bandpass filter (70–7600 Hz) → normalise (-20dB)│
└──────────────────────────┬──────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────┐
│  2. AWS Transcribe (transcribe.py)                   │
│     Speaker diarisation (max 2 speakers)            │
│     Custom vocabulary · async polling every 10s    │
│     Retry with MAX_SPEAKERS+1 if one speaker = 0   │
└──────────────────────────┬──────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────┐
│  3. Transcript → DataFrame                           │
│     JSON → pandas DataFrame                         │
│     Columns: Start Time, End Time, Speaker, Phrase  │
└──────────────────────────┬──────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────┐
│  4. Consent Detection (rule-based)                   │
│     Checks for consent request phrases              │
│     Validates question intent + approval keywords   │
│     Maps speaker_label → "Sales Rep" / "HCP"        │
│     Status: "Transcription Done" or "No HCP Consent"│
└──────────────────────────┬──────────────────────────┘
                           │ (if consented)
                           ▼
┌─────────────────────────────────────────────────────┐
│  5. Optional: Name Anonymization (GPT-4.1)           │
│     Detects human names in transcript               │
│     Maps to generic roles (Doctor/Rep/Patient)      │
│     Replaces all occurrences in DataFrame           │
└──────────────────────────┬──────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────┐
│  6. Parallel GPT-4o Evaluation (7 prompts)           │
│     p1: Key Message Delivered (4 dimensions)        │
│     p2: Key Message Discussed (4 dimensions)        │
│     p3: Key Message Sentiment (4 dimensions)        │
│     p4: Commitment (3 items)                        │
│     p5: Objections Raised + Handling (14 items)     │
│     p6: Compliance (2 guardrails)                   │
│     p7: Selling Behaviour (4 items)                 │
│     ── All 7 run concurrently via asyncio.gather ── │
└──────────────────────────┬──────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────┐
│  7. Result Processing Pipeline                       │
│     enrich_by_type_structure() → Category/Sub/Topic │
│     build_nested_dict_by_category()                 │
│     postprocess_and_simplify() → conditional logic  │
│     add_score_metadata() → per-section scores       │
│     convert_to_dashboard_nested_format()            │
│     calculate_final_score_from_dashboard()          │
└──────────────────────────┬──────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────┐
│  8. AI Summary (GPT-4o)                              │
│     Evaluator-voice feedback paragraph              │
│     Multilingual output support                     │
│     Mentions scores, strengths, improvements        │
└──────────────────────────┬──────────────────────────┘
                           │
                           ▼
           Saved to MySQL `calls` table
           status: "Evaluation Done" · progress: 100
```

---

## Evaluation Framework

Each call is evaluated against 7 prompt categories producing ~31 scored items:

| Prompt | Category | Items | Scoring |
|---|---|---|---|
| p1 | Key Message — Delivered | Cost, Safety, Efficacy, Value Prop | 1=Yes, 0=No |
| p2 | Key Message — Discussed | Cost, Safety, Efficacy, Value Prop | 1=Yes, 0=No |
| p3 | Key Message — Sentiment | Cost, Safety, Efficacy, Value Prop | 1=Positive, 2=Neutral, 3=Negative |
| p4 | Commitment | Needs Validation, Commitment Agreed, Next Step Agreed | 1=Yes, 0=No |
| p5 | Objections Raised + Handling | Cost, Efficacy, Safety, Reputational, Patient Satisfaction, Patient Outcomes, Org Efficiency | 1=Yes, 0=No |
| p6 | Compliance | Unsubstantiated Claims, Reimbursement Misrepresentation | 1=Compliant, 0=Violation |
| p7 | Selling Behaviour | Needs Exploration, Balanced Positioning, Selling Progression, Needs-Led Conversation | 1=Yes, 0=No |

**Post-processing rules:**
- If Key Message Not Delivered → Discussed and Sentiment auto-set to N/A
- If Key Message Not Discussed → Sentiment auto-set to N/A
- If Objection Not Raised → Handling auto-set to N/A

**Final Score:** Weighted average across categories, with per-customer configurable `percentage_weight_map`.

---

## Audio Processing

`audio_refinement.py` implements a multi-step audio preprocessing pipeline:

1. **Format Conversion** — FFmpeg converts any format (m4a, mp3, etc.) to mono WAV at 16kHz
2. **Noise Reduction** — `noisereduce` library reduces background noise with configurable `noise_p` proportion (default: 0.5)
3. **Bandpass Filter** — Butterworth bandpass filter (default: 70–7600 Hz, order 4) isolates the voice frequency range
4. **Normalisation** — RMS normalisation to a target dB level (default: -20 dB)
5. **Clipping** — Hard clip to [-1.0, 1.0] to prevent distortion
6. **S3 Integration** — Downloads from S3, processes locally, uploads refined version back to S3

All parameters are configurable per project via the `env_config` JSON in the `project_code_mapping` table.

---

## Batch Processing & SSE Streaming

The batch pipeline (`batch_analyze_sse_services.py`) uses a background-thread + shared-dict pattern to avoid AWS ALB timeout limits:

```
POST /batch-analyze/sse
        │
        ▼
  Save audio files to tmpdir
  Create task_id (UUID)
  Init task_progress[task_id] = {progress: 0}
  executor.submit(run_batch_in_background)  ← returns immediately
        │
        ▼
  Return EventSourceResponse(event_stream(task_id))

Background Thread:
  Creates new event loop
  asyncio.run(_batch_pipeline())
    │
    ├─ Parse Excel metadata
    ├─ (Optional) Refine all audio files in parallel
    ├─ process_calls_parallel() with asyncio.Semaphore(10)
    ├─ Post-process results
    ├─ Build two Excel outputs (full + filtered)
    └─ task_progress[task_id] = {progress: 100, result: {...}}

SSE event_stream():
  Polls task_progress every 1 second
  Yields {"event": "progress", "data": {"progress": N}} as progress updates
  Yields {"event": "complete", "data": {...excel_b64...}} when done
  Yields {"event": "error", "data": {...}} on failure
```

---

## Database Schema

### MySQL Tables (AWS Aurora)

| Table | Description |
|---|---|
| `calls` | Core call records — `call_id`, `language`, `customer`, `user`, `transcripts` (JSON), `evaluation_results` (JSON), `durations` (JSON), `status`, `progress`, `evaluated_on` |
| `mr_task_reply_data` | Survey response records — `respondentID`, `level`, `email_id`, `projectCode`, `MR_QuestionID`, `MR_Select_Val`, `reply_message`, `fileName`, `time_stamp` |
| `project_code_mapping` | Per-project configuration — `project_code`, `env_config` (JSON with transcription settings, consent phrases, audio processing params) |
| `prompts` | Per-customer evaluation prompts — `customer`, `product`, `evaluation_prompts` (JSON array of 7 prompts), `customer_cfg` (JSON with weight map) |
| `customers` | Customer/doctor reference — `customer_id`, `customer_name` |

### `calls.status` State Machine

```
(new) → Transcription Started → Transcription Done → Evaluation Started → Evaluation Done
                                      └──────────────────────────────────→ No HCP Consent
```

### `calls.progress` Values

| Value | Stage |
|---|---|
| 15 | Transcription started |
| 50 | Transcription complete |
| 60 | Evaluation started |
| 100 | Evaluation complete |

---

## Configuration & Environment Variables

| Variable | Description | Default (dev) |
|---|---|---|
| `DB_HOST` | MySQL host (AWS Aurora endpoint) | Hardcoded in `mysql_connector.py` |
| `DB_NAME` | Database name | `dev_shrd_a0391_ffe_01` |
| `DB_USER` | Database username | Hardcoded |
| `DB_PASSWORD` | Database password | Hardcoded |
| `OPENAI_API_KEY` | OpenAI API key | Set via `os.environ` in each service |
| `AWS_REGION` | Default AWS region | `us-east-1` |
| `S3_BUCKET` | Default S3 bucket | `aws-a0391-use1-00-d-s3b-shrd-app-ffe01` |
| `PROJECT_CODE` | Default project code | `4170DK0001` |

> ⚠️ Note: The current codebase hardcodes credentials directly in source files. This should be refactored to use environment variables or AWS Secrets Manager before any public deployment.

---

## Running the App

### Prerequisites

- Python 3.10+
- MySQL database (AWS Aurora or local)
- AWS credentials configured (`~/.aws/credentials` or IAM role)
- FFmpeg installed (`apt install ffmpeg` or `brew install ffmpeg`)

### Install Dependencies

```bash
pip install -r requirements.txt
```

### Start Development Server

```bash
uvicorn app.app:app --reload --port 8000
```

### Start Production Server

```bash
bash start.sh
# or
gunicorn app.app:app -c gunicorn-config.py
```

Swagger docs available at: `http://localhost:8000/docs`

---

## Challenges & Outcomes

### Challenge 1: AWS ALB Timeout for Long-Running Batch Jobs

**Problem:** AWS Application Load Balancer enforces a 60-second idle timeout. Batch processing 10+ calls (each taking 3–5 minutes for transcription + evaluation) would cause the connection to drop before completion.

**Solution:** The SSE batch endpoint (`/batch-analyze/sse`) returns an `EventSourceResponse` immediately, while the actual pipeline runs in a `ThreadPoolExecutor` background thread with its own event loop. The SSE stream polls a shared `task_progress` dict every second and pushes incremental progress events — keeping the connection alive throughout the entire job.

**Outcome:** Batch jobs of any size run without timeout, with real-time progress visible to the client.

---

### Challenge 2: Speaker Diarisation Failures

**Problem:** AWS Transcribe's speaker diarisation occasionally assigns all audio to a single speaker, making it impossible to compute per-speaker metrics or identify the sales rep correctly.

**Solution:** After transcription, both speaker durations are calculated. If either `user_duration` or `customer_duration` is zero, the transcription is retried with `MAX_SPEAKERS + 1` (up to 2 retries). This forces AWS Transcribe to look harder for a second speaker.

**Outcome:** Significantly reduced failure rate for mono-recorded calls where speaker separation is ambiguous.

---

### Challenge 3: Multilingual Evaluation with German Audio

**Problem:** The evaluation prompts need to produce justifications and quotes in German (the language of the call), while the evaluation logic and code are in English. Maintaining consistency between the prompt language and the output expectations is error-prone.

**Solution:** All 7 prompts explicitly instruct GPT-4o that the transcript is in German, that justifications must be written in German, and that quotes must be verbatim extracts from the German transcript. The prompt role-filling (`{user_role}`, `{customer_role}`, `{product}`) uses English labels so the model correctly identifies entities regardless of language.

**Outcome:** German-language justifications and quotes in every evaluation response, maintaining linguistic accuracy for downstream reporting.

---

### Challenge 4: Conditional Post-Processing Logic

**Problem:** Raw GPT scores need business logic applied: if a key message was never delivered, it cannot have been discussed; if it was not discussed, no sentiment can exist; if an objection was not raised, handling cannot be scored.

**Solution:** `postprocess_and_simplify()` in `analyze_services.py` and `apply_post_processing_rules()` in `batch_analyze_services.py` both implement this conditional cascading logic — nullifying child scores when parent conditions are false, and setting `Justification` and `Quote` to `"N/A"` accordingly.

**Outcome:** Logically consistent evaluation reports with no contradictory scoring (e.g., "Handling: Yes" when "Objection Raised: No").

---

### Challenge 5: Safe JSON Parsing from LLM Output

**Problem:** GPT models sometimes wrap JSON responses in markdown code fences (` ```json `), include trailing commas, or return partial structures. Naive `json.loads()` fails on these, causing evaluation failures.

**Solution:** `safe_parse()` implements a multi-strategy parser: direct `json.loads()` → extract JSON array by `[` and `]` positions → `ast.literal_eval()` fallback. Each attempt independently validates that the resulting object is a list with proper `Answer` fields coercible to integers.

**Outcome:** Near-zero parsing failures across thousands of GPT evaluation calls.

---

### Challenge 6: Atomic S3 + DB Operations for Audio Upload

**Problem:** When a sales rep uploads a call recording, the S3 multipart upload needs to complete and the database record needs to be inserted atomically. If the DB insert fails after S3 completion, an orphan audio file is left in S3 with no associated record.

**Solution:** `complete_and_insert_service()` first completes the S3 multipart upload, then wraps the DB insert in a transaction. On DB failure, it immediately attempts to `delete_object` the completed S3 file before raising the HTTP exception — preventing orphaned objects.

**Outcome:** Data consistency between S3 and MySQL, with best-effort cleanup on partial failures.

---

## Outcomes

| Outcome | Impact |
|---|---|
| Automated call evaluation | Replaces manual listening and scoring; evaluates a call in ~4 minutes |
| 7-dimension scoring | Comprehensive view of selling quality, compliance, objection handling, and commitment |
| Multilingual support (German) | Production-ready for DACH market pharmaceutical reps |
| Real-time batch SSE streaming | No timeout failures for large batch jobs; instant client feedback |
| Conditional scoring logic | Logically consistent reports with no contradictory scores |
| Transcript anonymization | Optional PII-safe mode for privacy-compliant reporting |
| Audio refinement pipeline | Improves transcription accuracy on noisy field recordings |
| Weighted scoring | Configurable per-customer category weights for tailored performance tracking |
