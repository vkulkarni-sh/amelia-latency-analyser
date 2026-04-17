# Amelia Latency Analyser

A browser-based diagnostic tool for analyzing latency in Amelia AI voice conversations. Upload Engine and EVG log files, provide a Conversation ID and/or Call ID, and the tool maps the full call cycle turn-by-turn — pinpointing exactly where time is spent across TTS, STT, Engine processing, LLM calls, and API integrations.

![Single-file HTML — no build step required](https://img.shields.io/badge/single--file-HTML-blue)
![React 18](https://img.shields.io/badge/React-18-61dafb)
![No server required](https://img.shields.io/badge/server-none-green)

---

## Features

### Page 1 — Conversation Troubleshooter

- **Full Call Cycle Mapping**: Traces each turn through the complete path — Amelia → EVG → TTS → User → STT → Amelia → Integrations → Amelia
- **Turn-by-Turn Timeline**: Visual waterfall chart showing system latency per turn with color-coded segment bars (TTS, Engine, LLM, API)
- **STT Sub-Phase Breakdown**: Splits user input into 4 distinct phases — STT Activation, Speech Detection, STT Recognition, and ASR Finalization (timer wait)
- **Engine / LLM Split**: Separates LLM call latency from general Engine processing (NLU, Dialog/BPN, routing) so you can see exactly how much time the model consumes
- **API Call Pairing**: Matches `Submitting requestId` / `Returned requestId` pairs to compute individual API integration latencies with service names
- **Stage Breakdown**: Aggregated latency by pipeline stage (STT, NLU, Dialog, API, TTS, EVG) with computed averages from the conversation flow
- **AI-Powered Analysis**: Sends a compact summary (never raw logs) to OpenAI GPT-4.1 for root cause analysis and actionable recommendations
- **Drill-Down Logs**: Click any turn or stage to see the correlated Engine and EVG log entries with timestamps
- **Multi-File Support**: Upload multiple log files (.log, .txt, .json, .jsonl, .csv, .ndjson) — the tool searches across all of them
- **Auto-Discovery**: Automatically discovers jambonz callIds from SBC Call IDs and correlates Engine logs by conversation ID

### Page 2 — Client Monitoring Dashboard (WIP)

- P50/P95/P99 latency monitoring across Amelia clients
- Sortable table with status indicators
- Currently shows simulated data — requires Service Account token for live data

---

## Quick Start

### Option 1 — Open Locally

Just double-click `amelia-latency-analyser.html` in your browser. That's it. No server, no install, no build.

### Option 2 — GitHub Pages

1. Push the file to a GitHub repository
2. Go to **Settings → Pages → Source** and select your branch
3. Access at `https://<username>.github.io/<repo>/amelia-latency-analyser.html`

### Option 3 — Any Static Host

The file works on Netlify, Vercel, Cloudflare Pages, S3, or any static hosting. Just upload the single HTML file.

---

## How to Use

1. **Upload Logs** — Drag and drop your Engine Logs and/or EVG (jambonz) Logs into the upload zone
2. **Enter IDs** — Provide the Conversation ID (searches Engine Logs) and/or Call ID (searches EVG Logs)
3. **Click Analyze** — The tool parses all files, filters by ID, classifies entries, builds the conversation flow, and runs AI analysis

### What You'll See

| Card | What It Shows |
|------|---------------|
| **Avg System Latency** | Average user-perceived wait time (from end of speech to Amelia's next response) |
| **Slowest Turn** | The worst turn with its latency and turn number |
| **Conversation** | Total duration, turn count, TTS/LLM averages, P95 |
| **Conversation Flow Timeline** | Per-turn waterfall with segmented bars — click to expand |
| **Stage Breakdown** | Aggregated latency by pipeline stage |
| **AI Analysis** | Root cause diagnosis with severity rating and recommendations |

### Expanded Turn Detail

Clicking a turn in the timeline reveals the full call cycle breakdown:

| Segment | What It Measures |
|---------|-----------------|
| **TTS Synthesis** | Time for text-to-speech audio generation |
| **Amelia Speaking** | Audio playback duration |
| **STT Activation** | Gather:exec → startTranscribing (STT engine startup) |
| **Speech Detection** | Waiting for user to begin speaking |
| **STT Recognition** | Partial → Final transcript (actual recognition time) |
| **ASR Finalization** | Timer wait after final transcript (typically 2000ms) |
| **Webhook** | HTTP round-trip from EVG to Engine |
| **Engine (non-LLM)** | NLU, Dialog/BPN execution, routing |
| **LLM Calls** | Individual LLM/ChatCompletion calls with model + latency |
| **API Calls** | External integration Submit→Return pairs with service names |
| **System Wait** | Total user-perceived delay (resolve → next Amelia response) |

---

## Supported Log Formats

| Format | Source | Notes |
|--------|--------|-------|
| `.json` / `.jsonl` / `.ndjson` | EVG (jambonz) | Pino/Bunyan JSON with structured fields (callId, speechTranscript, etc.) |
| `.csv` | Engine (Axiom export) | CSV with `_time`, `message` columns; Java Spring log format in message |
| `.log` / `.txt` | Either | Auto-detected from content patterns |

The tool auto-detects whether a file contains Engine or EVG logs based on filename keywords and content patterns. Files are tagged with a colored badge (Engine = blue, EVG = purple) after upload.

---

## Architecture

```
Single HTML file (~2000 lines)
├── Log Parsing Engine
│   ├── parseLogFile()          — Format detection + routing
│   ├── parseCSVLogFile()       — Axiom CSV (Engine)
│   ├── parseSingleLine()       — ISO/bracket/level-first formats
│   └── parseEngineLogMessage() — Java Spring format with UTC fix
├── Classification
│   ├── classifyLogEntry()      — Pattern + structured field matching
│   └── extractLatencyMs()      — Explicit latency value extraction
├── Flow Builder
│   ├── buildConversationFlow() — Full call cycle mapper
│   └── computeSubmitReturnLatencies() — API pair matching
├── AI Integration
│   ├── buildLLMPayload()       — Compact summary builder
│   └── analyzeWithOpenAI()     — GPT-4.1 analysis
└── React UI (Babel standalone)
    ├── ConversationFlowTimeline — Waterfall + drill-down
    ├── StageBreakdownChart      — Stage latency bars
    ├── LogViewer                — Raw log browser
    └── AIAnalysis               — Formatted AI output
```

All processing happens client-side in the browser. No data leaves the browser except the compact summary sent to OpenAI (if an API key is configured).

---

## Configuration

Click **Settings** in the sidebar to configure:

| Setting | Purpose | Required? |
|---------|---------|-----------|
| **OpenAI API Key** | Powers the AI-driven latency diagnosis | Optional — falls back to template analysis |
| **OpenAI Model** | Model to use (default: `gpt-4.1`) | Optional |
| **Service Account Token** | For Page 2 client monitoring (WIP) | Not yet functional |

The API key is stored only in browser memory for the session. It is never persisted, logged, or sent anywhere except directly to OpenAI's API.

---

## Key Technical Details

### Timestamp Correlation

Engine logs use server-local timestamps without timezone suffix. The tool appends `Z` to force UTC interpretation, which is critical for correlating with EVG timestamps (already in UTC). Without this fix, a 5.5-hour offset occurs in Asia/Calcutta timezone containers.

### Turn Detection

Turns are anchored on `TaskGather:resolve with reason speech` events — the moment the user finishes speaking. The tool specifically excludes `TaskGather:_resolve` sub-events (5 fire per turn as internal state machine transitions) to avoid inflating the turn count.

### ASR Timer

The ASR finalization timer is a fixed 2000ms wait after the final transcript, configured via `_startAsrTimer: set for 2000ms`. This is a known constant delay in every turn.

### LLM Detection Patterns

LLM call latencies are extracted from Engine logs matching:
- `Chat completion call in X s` / `X ms`
- `elapsedTime=N` in LlmMetrics/CognitiveProvider context
- `DefaultChatCompletionService` / `AbstractAgenticLlmService` with `took Nms`

---

## Privacy

- All log parsing and classification runs **entirely in the browser**
- Raw log files are **never uploaded** to any server
- Only a compact summary (~5-15KB) is sent to OpenAI if an API key is provided
- The summary contains aggregated latencies, turn counts, and a small selection of key log entries — not the full log files
- No API key? The tool still works with template-based analysis

---

## Browser Compatibility

Tested in modern Chromium-based browsers (Chrome, Edge) and Firefox. Requires JavaScript enabled. Uses React 18 with Babel standalone for in-browser JSX compilation — no build step needed.

---

## License

Internal tool — SoundHound / Amelia AI.
