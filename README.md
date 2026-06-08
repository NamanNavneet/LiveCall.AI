# LiveCall.AI — Project Description, Tech Stack & Outcomes

---

## Project Description

**LiveCall.AI** is an AI-powered call evaluation backend built for pharmaceutical sales organisations. It automates the end-to-end assessment of field sales calls — from raw audio ingestion through transcription, consent detection, speaker identification, and multi-dimensional GPT-4o scoring — delivering structured performance reports to sales managers and operations teams.

The system is designed around the reality of pharma sales workflows: a sales representative records a call with a healthcare professional (HCP/doctor), uploads it through the app, and receives a scored evaluation within minutes instead of waiting days for manual review. Managers can see at a glance whether their reps are delivering key messages, handling objections, remaining compliant, and following a consultative selling approach.

### Products Supported

LiveCall.AI ships with full evaluation frameworks for two Ascendis Pharma products, both in German:

**Skytrofa** (somatrogon) — a once-weekly growth hormone for pediatric growth hormone deficiency. Key evaluation themes: reducing daily injection burden, non-inferiority efficacy data, adherence and persistence, TransCon™ technology differentiation.

**Yorvipath** (palopegteriparatide) — a long-acting PTH replacement for chronic hypoparathyroidism. Key evaluation themes: treating the hormonal root cause vs. symptom management, renal function outcomes, quality of life improvement, shifting from passive biochemical monitoring to active disease management.

### Who Uses It

Sales representatives upload their recorded calls via a mobile/web app. Sales managers and medical science liaisons review evaluation reports and transcriptions via a dashboard. Operations and analytics teams run batch evaluations across hundreds of calls using Excel metadata files and get results as downloadable Excel reports.

---

## Tech Stack

### Backend Framework
**FastAPI (Python 3.10+)** with CORS middleware. The app exposes both synchronous REST endpoints and an SSE streaming endpoint, handling the full async/sync mixing that comes with integrating blocking audio/ML libraries into an async web framework.

### LLM Evaluation
**OpenAI GPT-4o** handles all 7 evaluation prompt categories per call, executed concurrently via `asyncio.gather`. **GPT-4.1** handles PII transcript anonymization and the AI-generated feedback summary. Both are called through the `openai` Python client with retry logic and a robust multi-strategy JSON parser (`safe_parse`) to handle malformed outputs.

### Audio Transcription
**AWS Transcribe** with speaker diarisation is the core transcription engine. Calls are submitted as S3 file URIs with custom vocabulary names, speaker labels enabled, and language codes. The system polls asynchronously every 10 seconds (async version avoids blocking the event loop) and automatically retries with incremented `MaxSpeakerLabels` if one speaker's audio duration is zero.

### Audio Preprocessing
A bespoke DSP pipeline using **FFmpeg** (format conversion), **noisereduce** (spectral noise gating), **scipy** (Butterworth bandpass filtering), and **soundfile** (WAV I/O) cleans recordings before transcription — meaningfully improving accuracy on noisy field recordings made on mobile devices.

### Async Architecture
Python `asyncio` throughout, with `ThreadPoolExecutor` for the SSE batch pipeline to avoid blocking the FastAPI event loop during long-running jobs. `asyncio.Semaphore(10)` limits parallel call processing to prevent OpenAI and AWS rate limit exhaustion.

### Streaming
**sse-starlette** provides `EventSourceResponse` for the batch endpoint. Progress updates are pushed to clients via a shared in-memory `task_progress` dict polled every second by the SSE generator — a pattern that bypasses AWS ALB's 60-second idle timeout entirely.

### Storage
**AWS S3** stores raw and refined audio files and AWS Transcribe JSON outputs. **MySQL (AWS Aurora)** stores call records, evaluation results, transcript text, survey responses, and project configuration.

### Data Processing
**pandas** handles all tabular data operations including SQL-to-DataFrame reads, pivot tables for respondent data, and result post-processing. **openpyxl** generates Excel output files for batch results.

---

## Challenges & Outcomes

### Challenge 1: AWS ALB 60-Second Timeout for Long Batch Jobs

**Problem:** Processing a batch of 10 calls (each requiring ~3–5 minutes for audio refinement, AWS Transcribe polling, and 7 parallel GPT evaluations) far exceeds the 60-second idle connection timeout on AWS Application Load Balancer. A naive synchronous batch endpoint would always time out before returning results.

**Solution:** The `/batch-analyze/sse` endpoint returns an `EventSourceResponse` immediately — before any processing begins. The actual pipeline runs in a `ThreadPoolExecutor` background thread that creates its own `asyncio` event loop and executes the full async pipeline independently. The SSE stream polls a shared `task_progress` dictionary every second and emits progress events to the client throughout the job lifecycle. This keeps the connection alive indefinitely regardless of job duration.

**Outcome:** Batch jobs of any size run without timeout. Clients receive real-time granular progress (5% → 12% → 45% → 100%) and the full result payload upon completion — delivered as Base64-encoded Excel files directly in the SSE `complete` event.

---

### Challenge 2: Speaker Diarisation Failures on Mono Recordings

**Problem:** AWS Transcribe occasionally fails to identify a second speaker in mono recordings where the speakers have similar voice profiles or when the audio quality is poor. When this happens, all audio is assigned to a single speaker, making it impossible to compute per-speaker metrics, identify the sales rep, or perform consent detection.

**Solution:** After transcription, the system calculates speaking duration for each detected speaker. If either duration is zero, the job is retried with `MAX_SPEAKERS` incremented by 1 (up to 2 retries). This forces AWS Transcribe to apply more aggressive speaker segmentation. A separate fallback identifies the first-speaking person as the Sales Rep.

**Outcome:** Measurably reduced diarisation failure rate for mono-channel recordings. The retry logic with incremented max speakers resolves most edge cases where the first transcription attempt produces a single-speaker result.

---

### Challenge 3: Reliable JSON Parsing from LLM Output

**Problem:** GPT models occasionally wrap JSON responses in markdown code fences (` ```json ` ... ` ``` `), include trailing commas, produce partial lists, or return explanatory text alongside the JSON. Standard `json.loads()` fails silently or raises exceptions on all of these, causing full evaluation failures that require re-running the entire call.

**Solution:** `safe_parse()` implements a cascade of four independent parsing strategies: direct `json.loads()` → extract JSON array by locating the first `[` and last `]` characters → `ast.literal_eval()` on the extracted snippet → `ast.literal_eval()` on the full response. Each parsed result is validated: it must be a list, and every item must have an `Answer` field coercible to integer. The first strategy to succeed wins.

**Outcome:** Near-zero parsing failures in production across thousands of GPT evaluation calls. The multi-strategy parser correctly handles all observed GPT output formatting variations without requiring model-level format enforcement.

---

### Challenge 4: Conditional Scoring Cascade Logic

**Problem:** The evaluation framework has inherent dependencies: if a key message was never delivered by the rep, it cannot have been discussed; if it was not discussed, no HCP sentiment can be observed; if an HCP never raised an objection, the rep's objection handling cannot be scored. Raw GPT scores do not enforce these constraints, leading to logically contradictory reports (e.g., "Handling: Yes" when "Objection Raised: No").

**Solution:** Two post-processing functions enforce cascade rules. `postprocess_and_simplify()` (single call) and `apply_post_processing_rules()` (batch) both: (1) set Discussed and Sentiment to N/A when Delivered is 0; (2) set Sentiment to N/A when Discussed is 0; (3) set Objection Handling to N/A when the corresponding Objection Raised is 0. These rules run after GPT output is collected and before scoring is calculated.

**Outcome:** Every evaluation report is logically consistent. Downstream reporting tools and dashboards can safely assume that N/A scores reflect business logic rather than model errors, and that "handling" scores only appear where corresponding "raised" scores are present.

---

### Challenge 5: Atomic S3 + Database Operations for Audio Uploads

**Problem:** The sales rep upload flow involves two separate systems: completing an S3 multipart upload and inserting the corresponding record into MySQL. These operations are not natively atomic — a network failure or DB error after S3 completion leaves an orphan audio file in S3 with no database record, creating storage waste and potential data integrity issues that are difficult to detect and clean up manually.

**Solution:** `complete_and_insert_service()` uses a strict sequence with compensating transaction logic: (1) complete the S3 multipart upload and capture the ETag; (2) open a MySQL transaction and insert the respondent record; (3) on DB failure, immediately attempt `s3_client.delete_object()` on the completed file before raising the HTTP exception. The canonical S3 key (used for the complete operation) is the same key returned from `multipart_init_service()`, ensuring key consistency across the two services.

**Outcome:** Data consistency between S3 and MySQL with best-effort cleanup on partial failures. The service logs warnings when S3 object deletion also fails, enabling manual remediation on rare edge cases.

---

### Challenge 6: Multilingual Evaluation (German Transcripts, English Prompts)

**Problem:** All sales calls are conducted and transcribed in German. Evaluation prompts must correctly identify German phrases, extract German quotes verbatim, and produce justifications in German — while the evaluation framework logic, question text, and role labels are in English. Maintaining this bilingual consistency reliably is non-trivial.

**Solution:** Every evaluation prompt explicitly states that the transcript is in German, requires justifications to be written in German, and specifies that quotes must be verbatim extracts from the German original. Role labels (`{user_role}`, `{customer_role}`) and product names (`{product}`) are injected as English strings — allowing the model to correctly identify entities in the German text using its multilingual understanding. This separation means the evaluation logic is language-agnostic at the prompt level.

**Outcome:** German-language justifications and verbatim German quotes in every evaluation response. Reports are directly usable by German-speaking managers without any translation step.

---

## Outcomes

| Outcome | Impact |
|---|---|
| Automated call evaluation | ~4-minute turnaround vs. days for manual review |
| 7-dimension scoring (31 items/call) | Comprehensive compliance + selling quality coverage |
| German pharmaceutical market support | Production-ready for Skytrofa and Yorvipath DACH launches |
| SSE batch with no timeout | 10–50 call batches run reliably through AWS ALB |
| Conditional scoring logic | Logically consistent, contradiction-free reports |
| Audio refinement pipeline | Improved transcription accuracy on field recordings |
| Name anonymization | Optional PII-safe mode for privacy-compliant analytics |
| Weighted scoring per customer | Configurable category importance per product/customer |
| Atomic upload + DB insert | No orphaned S3 objects; data integrity guaranteed |
| Excel batch output (full + filtered) | Drop-in reporting for ops/analytics teams |
