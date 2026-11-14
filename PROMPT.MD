# Codex Implementation Brief: VOICE STUDIO for QWEN
# VOICE STUDIO for QWEN 

# Disclaimer
# This is an independent, unofficial project built using the open-source Qwen3-TTS models.
# This project is not affiliated with, endorsed by, or sponsored by the Qwen team or Alibaba Cloud.
# Qwen3-TTS and its associated components remain the property of their respective owners and are subject to # their applicable licenses.

> Usage: place this file in a new project and tell Codex: **“Read QWEN_VOICE_STUDIO_PROMPT.MD and build the application according to the entire specification. Start implementing and continue through testing and documentation.”**
>
> This specification is self-contained. Neither the original project nor the previous conversation is required. It describes the functionality and appearance of VOICE STUDIO for QWEN; it is not a copy of its source code or a guarantee of a pixel-identical result.

---

## 1. Task and Working Approach

You are an experienced Python, AI/ML, and frontend developer. In the current project, build a complete, functional local web application called **VOICE STUDIO for QWEN** for creating character voiceovers for a commercial game.

I want a working implementation, not a proposal, pseudocode, or screens with nonfunctional buttons. Work in stages, verify the results as you go, and fix any issues you find. Make routine implementation decisions independently. If the project already contains files, inspect them first and preserve user data.

Communicate and write documentation in Czech. Build the user interface using the English section names and control labels specified in this document. Voice delivery instructions must be in English.

The application must run on **Windows 11**, locally at **http://127.0.0.1:8765**. Do not publish it online or use cloud inference services. Do not require an account for normal use.

## 2. Main User Scenario

I create a character named Robert, define his personality and voice, and save him. Later, I select Robert, enter `Hello! Are you ready?`, and receive a WAV file using his saved voice. I can choose an emotion, adjust delivery, and compare several variants of the same line according to the capabilities of the selected model.

After restarting the application, Robert must continue using the same saved speaker ID or reference recording. **Do not design a new voice for every dialogue request.** Character identity takes priority over changes in emotion. Do not present preserved identity as a guarantee of bit-identical audio output.

## 3. Technology and Structure

Backend:

- Python 3.12, 64-bit.
- FastAPI, Uvicorn, Pydantic.
- The official `qwen-tts` library and PyTorch, with a compatible `torchaudio` version.
- SQLite through `sqlite3`, with transactions and WAL.
- NumPy and SoundFile for WAV handling and audio validation.
- Hugging Face Hub for explicitly requested model downloads.
- `psutil` for memory diagnostics.
- A shared REST API used by the frontend, CLI, and external automation.

Frontend:

- HTML5, CSS3, Vanilla JavaScript.
- No React, Vue, or Vite, and no Node.js requirement for installation or operation.
- Static files served by the same FastAPI server.
- No mandatory remote fonts, images, CDNs, or other runtime network dependencies.
- Locally stored Swagger UI assets for `/docs`, including their license.

The reference implementation used `qwen-tts==0.1.1`, `torch==2.8.0`, `torchaudio==2.8.0`, `transformers==4.57.3`, `accelerate==1.12.0`, `fastapi==0.115.12`, `uvicorn==0.34.2`, and `soundfile==0.13.1`. Treat these as a reference combination, not a claim that they are the latest versions. During implementation, verify the official API and compatibility of the selected versions. Separate runtime dependencies from test dependencies.

Use approximately this structure:

```text
backend/
  __init__.py
  app.py                # REST API, HTTP validation, static frontend
  config.py             # project-local paths and cache configuration
  catalog.py            # supported models, languages, speakers, emotions
  schemas.py            # Pydantic input schemas
  db.py                 # SQLite
  models.py             # model manager and official Qwen adapter
  service.py            # shared application logic
  jobs.py               # single sequential queue
  audio.py              # WAV validation, filenames, OGG encoder
frontend/
  index.html
  style.css
  app.js
  icon.svg
  vendor/               # local Swagger assets and license
scripts/
  __init__.py
  init_data.py
  check_environment.py
  launch.py
  model_info.py
  smoke_inference.py
  browser_smoke.py
tests/
  conftest.py
  test_api.py
  test_models.py
data/
  database/
  characters/
  outputs/
  references/
  models/
  exports/
cli.py
setup.ps1
start.bat
requirements.txt
requirements-web.txt
requirements-dev.txt
pytest.ini
.gitignore
README.md
TESTING.md
LICENSE
THIRD_PARTY_NOTICES.md
```

SQLite is the authoritative profile store; `data/characters/` may be reserved for future use. Keep production data separate from test data. `.gitignore` must exclude `data/`, `.venv/`, `.runtime/`, `.cache/`, `__pycache__/`, `.pytest_cache/`, `test-results/`, logs, and `.env`.

## 4. Hardware and Model Manager

Target computer: Automatically detect actual hardware and available resources at runtime. Do not assume a fixed hardware configuration.

- The default, fully functional inference path is **CPU, float32, eager attention**.
- FlashAttention must not be a mandatory dependency.
- Offer CPU / Auto / CUDA. Auto may select the GPU only after checking a compatible PyTorch build, the driver, an actual small CUDA operation, and sufficient free VRAM.
- Use conservative application thresholds: 8 GiB of free VRAM for 1.7B and 6 GiB for 0.6B. Label these as application safety margins, not official Qwen minimum requirements.
- If available GPU memory is below the required threshold, Auto must fall back to CPU.
- Explicitly selecting unusable CUDA must produce a clear error without crashing the entire application.
- Before loading on CPU, check available RAM: approximately 12 GiB for 1.7B and 7 GiB for 0.6B.
- Load models on demand. Keep at most one large model in memory. When switching models, release the previous model, its prompt cache, and any CUDA cache.
- Add manual **Load model** and **Release active model** actions; execute both through the same queue as inference.
- Changes to the CPU thread count must also apply when reusing an already loaded model.

Support these models:

| Model ID | Role | Direct delivery instructions in this application version |
|---|---|---|
| `Qwen/Qwen3-TTS-12Hz-1.7B-CustomVoice` | Recommended first model; preset voices | Yes |
| `Qwen/Qwen3-TTS-12Hz-0.6B-CustomVoice` | Smaller preset model | No |
| `Qwen/Qwen3-TTS-12Hz-1.7B-VoiceDesign` | Voice candidate creation | Yes, during voice design |
| `Qwen/Qwen3-TTS-12Hz-1.7B-Base` | Cloning a saved reference | No |
| `Qwen/Qwen3-TTS-12Hz-0.6B-Base` | More economical cloning; default for custom voices | No |

Model download flow:

1. **Review download** retrieves metadata only: revision, file list, and file sizes.
2. The UI shows the exact total, available disk space, and required reserve, such as 15% + 512 MiB.
3. Only an explicit **Download this model** action queues the download.
4. Download the displayed revision and all required files, including the speech tokenizer, into the project. Do not download the other models.
5. Verify file presence and sizes. Only then write the manifest atomically and display **Downloaded & verified**.
6. An interrupted or incomplete download must not count as complete. Allow the user to restart or finish it.
7. Show an error if exact metadata is unavailable or disk space is insufficient. Clearly label catalog estimates as estimates.
8. Actual inference must use the complete local model directory and work without internet access. Never automatically download a missing model when Generate is clicked.

Do not download large models during implementation without explicit user approval. If a complete model already exists, you may use it for a necessary short test without downloading another model.

## 5. Character Profiles and Management

Each profile must include:

```text
id
name
description
gender_age
language
voice_type                # preset / designed / imported
voice_model               # model used for subsequent dialogue
voice_prompt
base_speaker
reference_audio_path
reference_transcript
reference_sha256
default_emotion
default_instructions
created_at
updated_at
```

For imports, retain the rights confirmation. Reference paths must be created by the server; do not allow clients to supply arbitrary system paths.

Characters must support creation, editing, searching, duplication, selection for the Studio, playback of an existing reference, and deletion after confirmation. Names must be unique regardless of case, and IDs must remain stable. Duplicating a character preserves its voice while assigning a new ID and unique name. Changing voice type must resolve incompatible settings or references.

Deleting a character must not delete previous recordings. Preserve old immutable references because another character or historical metadata may use them. Make it clear to the user what a deletion removes.

## 6. Three Voice Modes

### A. Preset Voice

The user names a character Robert, selects CustomVoice, and chooses a built-in speaker. Save the actual speaker ID and use `generate_custom_voice()` for subsequent lines.

Include speakers according to the official catalog for the selected version: Ryan, Aiden, Vivian, Serena, Uncle_Fu, Dylan, Eric, Ono_Anna, Sohee. Validate identifiers; the character's name is not a speaker ID.

Pass `instruct` only for 1.7B CustomVoice. Do not present 0.6B as supporting direct emotion control.

### B. Custom Designed Voice

The profile has the type `designed`, a voice description, and a Base model for future dialogue. VoiceDesign is used only to create candidates.

Example description:

```text
Male, 35 years old, deep slightly raspy voice,
American English, calm military veteran.
```

The user provides a short reference text, a language, and a count of 1–5 candidates. Use `generate_voice_design()`. Save each candidate with its text and metadata and provide playback.

When **Use this voice** is clicked, require confirmation that the user has listened to the candidate and that the actual spoken words exactly match the reference text. Copy the selected WAV into a separate immutable reference file. Save the text and SHA-256. Deleting the candidate must not delete this reference.

Generate subsequent dialogue exclusively through Base `generate_voice_clone()`. For repeated use, create `create_voice_clone_prompt(ref_audio=..., ref_text=..., x_vector_only_mode=False)` and limit the feature cache, for example to eight references. The cache is not the persistent identity; after a restart, recompute it from the saved WAV and transcript.

### C. Import Reference Voice

The profile has the type `imported` and uses a Base model. In Voice Designer, the user uploads a WAV, supplies an exact transcript, and confirms the right to use both the recording and the voice.

Validate the actual file contents, not just the extension: WAV, mono/stereo, 8–96 kHz, 1–30 seconds, maximum 20 MiB. Reject empty transcripts, silence, non-finite or invalid samples, other formats, and invalid durations. Save the original recording, transcript, and SHA-256. Recommend clean speech without music.

## 7. Emotions and Languages

Offer these emotions with editable English instruction mappings:

| Emotion | Default instruction |
|---|---|
| Neutral | Speak naturally, with a calm, neutral tone. |
| Happy | Speak warmly, with happiness and enthusiasm. |
| Sad | Speak quietly, with deep sadness. |
| Angry | Speak with strong anger and urgency. |
| Afraid | Speak nervously, with fear in the voice. |
| Nervous | Speak with hesitant, nervous energy. |
| Whispering | Whisper softly and intimately. |
| Shouting | Shout loudly and clearly, projecting your voice. |
| Sarcastic | Speak with a dry, sarcastic tone. |
| Confident | Speak firmly and confidently, with steady authority. |
| Exhausted | Speak wearily, with a tired and breathless voice. |
| Crying | Speak tearfully, with a trembling voice. |
| Urgent | Speak quickly and clearly, with immediate urgency. |
| Custom | An empty field for a custom instruction. |

Allow persistent editing of the mappings in Settings and manual overrides for individual dialogue requests. Append the character's default delivery instructions to the dialogue instruction and retain the effective instruction in metadata.

**Do not invent nonexistent Qwen API parameters.** Base does not have the same `instruct` interface as 1.7B CustomVoice. For Base and 0.6B CustomVoice, disable the instruction field, visibly indicate limited control, and explain that the selected emotion is only a library label. Do not pass it to inference. Suggest experimenting with punctuation and variants; do not automatically insert tags that might be spoken aloud.

Languages in the reference interface: English, Chinese, Japanese, Korean, German, French, Russian, Portuguese, Spanish, Italian, Auto. Verify them against the model's official capabilities. Do not offer Czech as a guaranteed supported language unless the selected model officially supports it.

## 8. Dialogue Studio and Asynchronous Jobs

The Studio must include:

- A Character dropdown and a card for the selected character.
- A Dialogue textarea, maximum 1500 characters, with a character counter.
- Language, Emotion, and an editable Delivery instruction.
- Variants from 1 to 5.
- Expandable Generation settings: optional seed, temperature (0.1–1.5; default 0.9), and max codec tokens (128–4096; default 2048).
- Generate dialogue.
- An Activity panel and Session takes with audio players, Download WAV, optional OGG export, Save to library, Metadata, Rename, and Delete.

The HTTP request must not wait for the entire inference run. `POST /api/generate` validates the input, character, model, and reference, creates a job, and returns HTTP 202 with `job_id`.

Persist jobs in SQLite using the states `queued`, `running`, `completed`, and `failed`. Store creation, start, and finish timestamps, job type, input, partial results, a status message, and any error.

Display the actual state, variant number, and elapsed time. **No invented progress percentages.** The UI may poll approximately every 2.5 seconds. Refreshing the page must not make jobs or completed results impossible to find.

A single bounded queue, for example at most 50 waiting jobs, and one worker handle inference, loading/unloading, downloads, and exports. Prevent multiple large models from running simultaneously. Run a single Uvicorn process.

When queuing a job, capture a snapshot of the profile, settings, and reference. Later character edits must not retroactively change queued generation. Save a seed for each variant, such as base seed + index. Save completed variants incrementally; write WAV to a temporary file first, then rename it atomically.

After a restart, mark unfinished jobs as failed with an explanation. Preserve completed files and results. Do not automatically repeat expensive inference. If immediate cancellation is not implemented, do not claim it is supported.

## 9. Audio Library and Exports

Automatically save every generated file and record:

```text
id, job_id, character_id, character_name
text, emotion, voice_model, kind (dialogue/design)
created_at, name, audio_filename, audio_path
duration, sample_rate, saved, mock_audio, warning
settings:
  request and effective instruction
  seed for this variant
  character_snapshot including reference and SHA-256
  audio precision used
  model_revision
  instruction_applied
  library versions, device, and dtype
```

Filename examples: `robert_angry_001.wav`, `robert_angry_002.wav`, `alex_happy_001.wav`. Use a safe slug and a persistent counter that will not overwrite existing recordings after a restart.

WAV is mandatory, default PCM_16, optionally PCM_24. Preserve the model's native sample rate. The audio endpoint must support playback and HTTP Range.

The library must support playback, renaming, deletion after confirmation, filtering by character and emotion, a Saved takes only filter, and pagination with 30 items per page. **Save to library** marks a favorite or approved take; other takes remain stored.

Renaming may preserve the original physical file path, but downloads and exports must use the new name. Show metadata for the individual take.

Asynchronously export one or more selected files as a ZIP containing audio and a `manifest.json` with metadata for a game engine. Handle duplicate filenames. Offer the ZIP only after successful completion.

Offer OGG only when FFmpeg and the libvorbis encoder are actually available. Without the encoder, keep WAV fully functional and explain the unavailable option. Invoke FFmpeg with an argument list, without shell interpolation, with a timeout, and without a popup console window on Windows. Do not silently install it system-wide.

## 10. REST API

Implement at least the following endpoints. In these examples, `{id}` represents an ID; for characters, also support an unambiguous name where appropriate.

```text
GET    /api/health
GET    /api/catalog
GET    /api/models
POST   /api/models/plan
POST   /api/models/download
POST   /api/models/load
POST   /api/models/unload
GET    /api/diagnostics/hardware
GET    /api/settings
PUT    /api/settings

GET    /api/characters
POST   /api/characters
GET    /api/characters/{id}
PUT    /api/characters/{id}
DELETE /api/characters/{id}
POST   /api/characters/{id}/duplicate
GET    /api/characters/{id}/reference/audio
POST   /api/characters/{id}/reference/design
POST   /api/characters/{id}/reference/upload
POST   /api/characters/{id}/reference/select

POST   /api/generate
GET    /api/jobs
GET    /api/jobs/{id}
GET    /api/generations
GET    /api/generations/{id}
PATCH  /api/generations/{id}
DELETE /api/generations/{id}
GET    /api/generations/{id}/audio
POST   /api/exports
GET    /api/exports/{job_id}/download

GET    /docs
GET    /openapi.json
```

Generation input:

```json
{
  "character": "Robert",
  "text": "Hello! Are you ready?",
  "emotion": "Happy",
  "language": "English",
  "variants": 2,
  "seed": 42
}
```

Additional optional parameters: `instruction`, `temperature`, `max_new_tokens`. When language or emotion is omitted, use the character profile defaults. Reject empty text, invalid values, and unknown fields rather than silently ignoring them.

Response:

```json
{"job_id": "UUID", "status": "queued", "warning": null}
```

A completed generation includes `{"generation_id":"UUID"}` entries in the job's `results`. WAV is available at `/api/generations/{id}/audio`; `?download=true` forces a download.

Additional contracts:

- Model plan: `{ "model_id": "Qwen/..." }`; returns a plan with its ID, revision, sizes, and free disk space.
- Download: `{ "plan_id": "UUID", "confirmed": true }`. Reject missing confirmation or a missing plan.
- Load: `{ "model_id": "Qwen/..." }`.
- VoiceDesign: `{ "text": "...", "prompt": "...", "language": "English", "variants": 3 }`.
- Reference selection: `{ "generation_id": "UUID", "transcript_confirmed": true }`; the candidate must belong to the same character and have the design type.
- Upload: multipart `file`, `transcript`, `rights_confirmed=true`.
- Generation PATCH: `{ "name": "squad_intro", "saved": true }`.
- Export: `{ "generation_ids": ["UUID"], "format": "wav" }`, or `ogg`.
- Generation filters: `character_id`, `emotion`, `kind`, `saved`, `limit`, `offset`.

Return errors with a human-readable `detail`: 404 for a missing record, 409 for conflicts or missing models/references, 422 for invalid input, 507 for insufficient disk space or memory, and 503 for an unavailable service or queue. An error during an accepted inference job must appear as a failed job, never as a fabricated success.

The API and UI must use **one shared inference implementation**. Do not add a runtime option to enable fake AI audio in production.

## 11. Python CLI

Create `cli.py` with argparse and a local REST client. Prefer the standard library; do not perform inference directly in the CLI.

Support:

```powershell
python cli.py health
python cli.py characters
python cli.py generate --character Robert --text "Hello!" --emotion Happy
python cli.py generate --character Robert --text "Hold your position." --variants 2 --seed 42 --output data\downloads
python cli.py generate --character Robert --text "Hello!" --no-wait
python cli.py job JOB_ID
python cli.py export GENERATION_ID --format wav --output data\exports\game.zip
```

The default server is `http://127.0.0.1:8765`; support an alternative through `--url` before the command name. The CLI waits for job status, reports clear errors, and downloads WAV plus JSON metadata when `--output` is provided. A client timeout must not claim that server-side inference has stopped.

## 12. Interface Appearance and Layout

Create a minimalist, professional dark audio studio with a subtle olive accent. Do not use a marketing landing page, large photos, emoji icons, or dependencies on internet-hosted images.

Use approximately these design values:

```css
--bg: #101212;
--panel: #191c1c;
--panel2: #1e2221;
--line: #2c302e;
--muted: #8c9590;
--text: #ecefe7;
--accent: #d6e79a;
--danger: #f0a196;
```

- A local system font, such as `"Segoe UI", Arial, sans-serif`.
- A fixed left sidebar approximately 232 px wide, with the QWEN / VOICE STUDIO brand and a simple SVG waveform icon at the top.
- Navigation: **Dashboard, Characters, Voice Designer, Dialogue Studio, Audio Library, Settings**. The active section has an olive background and green text.
- At the bottom of the sidebar: local server status, “Your voices. Your machine.”, an API documentation link, and the version.
- A top bar approximately 75 px high with breadcrumbs and brief device/model status.
- Content width up to approximately 1500 px, margins of 24–40 px, panels with roughly 12 px corner radii and subtle borders.
- Headings around 30–34 px, understated uppercase labels, sufficient contrast, and generous spacing.
- Light olive primary buttons with dark text; dark secondary buttons with a border.
- Character cards use initials and a stylized avatar rather than random portraits.
- Use simple inline SVG icons. A decorative waveform must not pretend to represent measured audio data.
- Responsive layout: a narrower sidebar with icons on tablets, and a single column on phones. Preserve accessible navigation names and keyboard controls.
- Visible focus, form labels, accessible dialogs, `aria-live` announcements, and respect for reduced motion preferences.
- Status notifications must not overwhelm the screen; limit simultaneous toasts.

Screens:

**Dashboard**: counts of saved characters, downloaded models, and active/queued jobs; three steps, Prepare your model → Build your cast → Record your dialogue; recent recordings.

**Characters**: search, New character, and a card grid. A modal editor for all editable fields. Open studio, Edit, Duplicate, Delete. Show a clear Needs reference badge when appropriate.

**Voice Designer**: a left panel with character selection, voice description, reference text, language, and candidate count; for imported characters, show the upload form instead. On the right, Saved reference with a player and transcript. Below, a list of candidates with Use this voice. Explain the Design → Select → Reuse workflow.

**Dialogue Studio** (default page): a wide left editor with “01 Compose dialogue” and “02 Direct the performance”, followed by Session takes. A right column approximately 290 px wide contains Active character, Activity, and brief guidance. Indicate direct/limited emotion control and model/reference availability. Remember the last selected character on reload if localStorage is available.

**Audio Library**: filters, file selection, players, metadata, actions, and pagination. Export selected with a choice of WAV/OGG + metadata.

**Settings**: device, CPU threads, PCM_16/PCM_24, editing of all emotion instructions, model manager, and diagnostics. Show Python/PyTorch/qwen-tts, available RAM, encoder, active model, cache path, output path, and database path. Paths may be fixed within the project and informational only. Check GPU must perform a real check, not show an invented result.

Empty states must offer a specific next step. Do not populate the production database with fake completed recordings.

## 13. Windows Installation and Launch

`setup.ps1` must:

- Verify Python 3.12 x64 and support `-PythonPath`.
- Recognize an existing project-local runtime in `.runtime/`.
- Create `.venv`; never install packages globally.
- Validate an existing `.venv` and avoid using an incompatible version without explanation.
- Bootstrap pip if necessary, install runtime dependencies, and install CPU PyTorch by default.
- Verify imports of PyTorch and the official Qwen wrapper, check CPU/GPU availability and disk space, and initialize the database.
- Offer `-WebOnly` for UI/API development and `-WithTests` for test dependencies.
- Offer optional CUDA installation only through a clear explicit choice and compatibility checks. Do not silently retain a CPU build while claiming CUDA was installed.
- Exit with a nonzero code on failure and explain how to resolve it.
- Avoid downloading voice models or making hidden system changes.

If Python or a system tool is missing, provide exact installation instructions. Do not automatically install system tools. Prepare a project-local runtime only transparently and within the environment's permissions.

`start.bat` must switch to the project directory, use its `.venv`, start the server, open the browser after checking readiness, and keep a visible terminal with logs. If the Studio is already running, open the existing instance. If `.venv` is missing, display the setup command. Correctly handle paths containing spaces and accented characters.

Also support:

```powershell
.\.venv\Scripts\python.exe -m scripts.launch --no-browser
```

## 14. Security and Persistence

- Listen only on loopback, not `0.0.0.0`.
- Restrict Host and foreign browser Origins for write operations. Keep the API usable by a local CLI without a browser Origin.
- Do not insert user-supplied names or text into HTML without escaping.
- Verify that resolved server paths remain within the project. Prevent path traversal through the API, filenames, and exports.
- Use parameterized SQL and transactions.
- Tests must not modify production profiles, references, or models.
- Do not modify reference files in place; give each new reference a new file. This preserves consistency for old jobs and duplicated characters.
- Do not remove existing user data during setup or restart.
- Document backing up the entire `data/` directory while the server is stopped, and how old references and exports are retained.

## 15. Testing and Completion Criteria

Create automated tests covering at least:

1. Character CRUD, unique names, duplication, and stable speaker IDs.
2. Saving the complete profile and loading it after creating a fresh application instance.
3. Persistent reference WAV, transcript, hash, and preservation after deleting the candidate.
4. All emotion mappings, custom instructions, and their persistence.
5. Validation of generation requests, profiles, languages, speakers, and model compatibility.
6. A single sequential queue and HTTP responsiveness during generation.
7. Immutable character snapshots for queued jobs.
8. Variants, seeds, metadata, WAV headers, duration, and sample rate.
9. Audio Library, saved filtering, renaming, deletion, and ZIP with a manifest.
10. Missing models without automatic downloads.
11. Invalid WAV files, audio that is too short, silence, and missing confirmation or transcripts.
12. Insufficient disk space, unusable GPUs, and readable errors.
13. Metadata-only download plans, explicit confirmation, pinned revisions, file verification, and incomplete manifests.
14. Jobs interrupted by restart and preservation of completed variants after partial failure.
15. Correct calls to official Qwen methods: no `instruct` for Base or 0.6B CustomVoice.
16. Clone prompt cache reuse and release, local float32/eager loading, and at most one loaded model.
17. REST/OpenAPI, local static files, Host/Origin protection, and path traversal prevention.
18. Unavailable OGG encoders while preserving WAV functionality.

Mock inference belongs **only in tests**. Its synthetic tone must have `mock_audio: true`, the label **TEST FIXTURE — NOT AI SPEECH**, and the test UI label **TEST ENGINE · NO AI AUDIO**. Never present mock output as actual Qwen synthesis.

If a browser is available, run an end-to-end test covering character creation/editing, two variants, saved library entries, ZIP downloads, settings changes, VoiceDesign candidates, reference selection, clone requests, WAV imports, and reloads. Check all six sections on desktop and phone without horizontal overflow or JavaScript errors. Browser automation is a development dependency only; production users must not need it.

Test the CLI against an actual running local test server, and test Swagger with external requests blocked.

When a complete local model is available, perform a short **real inference smoke test**:

```text
Character: Robert
Speaker: Ryan
Model: Qwen/Qwen3-TTS-12Hz-1.7B-CustomVoice
Text: Hello, my friend.
Language: English
Device: CPU
```

Block network access during the test, verify that a playable WAV is created, and release the model afterward. A valid WAV does not automatically prove correct pronunciation; distinguish technical validation from listening. Do not load models into several processes simultaneously.

If the model is missing, the smoke test must transparently finish as **SKIPPED** with instructions, not as PASS and not with mock audio. Likewise, clearly identify untested CUDA and OGG when no encoder is available.

**Do not copy test results or test counts from another project.** Record only what you actually ran in the new project in TESTING.md, with the current date, versions, results, and remaining limitations.

## 16. Documentation and Licensing

README must include:

- Windows 11 installation and the exact launch command.
- An explanation of `completed` for downloads, the Downloaded & verified label, and the next steps to generate the first line.
- Creating a preset character, a persistent designed voice, and an imported reference.
- Differences between model capabilities and accurate limits of emotion control.
- CPU/GPU behavior, memory thresholds, sizes/cache, and explicit model downloads.
- Generation, job timing, seeds/reproduction metadata, and long-dialogue limitations.
- REST API examples for another agent, the Python CLI, and offline documentation.
- WAV/OGG/ZIP export to a game engine and instructions for installing optional FFmpeg.
- Data locations, backups, and deletion/restart behavior.
- Test commands and an honest distinction between mock tests and real inference.
- Licensing information and third-party notices. Use Apache 2.0 for new application code; preserve licenses for libraries and vendored assets. Rights to an imported voice are a separate matter.

Verify Qwen against primary sources and the actual code of the installed version:

- https://github.com/QwenLM/Qwen3-TTS
- https://github.com/QwenLM/Qwen3-TTS/blob/main/qwen_tts/inference/qwen3_tts_model.py
- https://huggingface.co/Qwen/Qwen3-TTS-12Hz-1.7B-CustomVoice
- https://huggingface.co/Qwen/Qwen3-TTS-12Hz-1.7B-VoiceDesign
- https://huggingface.co/Qwen/Qwen3-TTS-12Hz-1.7B-Base
- https://huggingface.co/Qwen/Qwen3-TTS-12Hz-0.6B-Base
- https://huggingface.co/Qwen/Qwen3-TTS-12Hz-0.6B-CustomVoice
- https://pypi.org/project/qwen-tts/

Do not infer nonexistent parameters from unofficial wrappers. If the upstream API has changed, verify the differences, choose a compatible implementation, and document it.

## 17. Implementation Order and Handoff

1. Inspect the project, check the environment, and create the architecture and files.
2. Implement SQLite, profiles, validation, and shared application logic.
3. Implement the model manager, real Qwen adapters, and sequential jobs.
4. Implement all six screens and connect them to the same REST API.
5. Finish references, the audio library, exports, and CLI.
6. Create setup, the launcher, and local OpenAPI documentation.
7. Install required project dependencies within the environment's permissions and run available tests.
8. Fix identified issues and verify the affected functionality.
9. Test actual local server startup, API responses, and the frontend. Stop any test instance after verification.
10. Finish README, TESTING, and licenses.

At the end, clearly state:

- What is complete.
- What was actually tested and with what results.
- What requires an explicit model download.
- What does not work or remains unverified.
- The exact launch command, normally `.\start.bat`, and the first next step in the application.

**Start implementing now and continue beyond a proposal through completion.**
