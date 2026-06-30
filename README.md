# 🩺 AI Medical Scribe

### Ambient clinical documentation pipeline — Deepgram + Claude AI + Make.com

[![Live Demo](https://img.shields.io/badge/Live%20Demo-GitHub%20Pages-4A90D9?style=for-the-badge&logo=github)](https://hahmedsid.github.io/ai-scriber-recorder)
[![Make.com](https://img.shields.io/badge/Automation-Make.com-6D00CC?style=for-the-badge)](https://make.com)
[![Deepgram](https://img.shields.io/badge/Transcription-Deepgram%20Nova--2%20Medical-13EF93?style=for-the-badge)](https://deepgram.com)
[![Claude AI](https://img.shields.io/badge/AI-Claude%20Sonnet-D4A017?style=for-the-badge)](https://anthropic.com)
[![License: MIT](https://img.shields.io/badge/License-MIT-lightgrey?style=for-the-badge)](LICENSE)

> A physician hits record, speaks naturally with their patient, and receives a finalized SOAP note inside their EHR in under 30 seconds — no typing, no copy-paste, no separate scribe.

---

## Screenshots

| Recorder UI | Generated SOAP Note |
|---|---|
| <img width="424" height="326" alt="RecorderUI" src="https://github.com/user-attachments/assets/bec3c9f4-8ba0-4891-861c-480c473d6d19" />
 | *(add screenshot)* |

---

## What It Does

Physicians spend an average of **2–3 hours per day** on clinical documentation — time stolen from patients and from the rest of their lives. This pipeline eliminates that friction entirely.

**You hit record and speak.**

The pipeline:

1. Captures audio directly from the browser microphone
2. Streams it to **Deepgram Nova-2 Medical** for real-time transcription with clinical vocabulary
3. Sends the transcript to a **Make.com** scenario via webhook
4. **Claude Sonnet 4.6** converts the transcript into a structured SOAP note (pure JSON, sub-2-second response)
5. Writes the note directly to **OpenEMR**
6. Logs the encounter to **Google Sheets** as an audit trail

---

## The Clinical Problem

- Physicians spend **2–3 hrs/day** on EHR documentation (AMA, 2023)
- **62%** of physicians report burnout — EHR burden is the #1 cited cause
- Traditional human scribes cost **$15–20/hr** per physician and don't scale across a practice
- Most AI scribe tools produce copy-paste output — the physician still has to open the EHR and paste manually

This pipeline writes directly to the EHR. That last step is the one that actually saves time.

---

## Pipeline Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                     PHYSICIAN'S BROWSER                     │
│                                                             │
│   [Record Button] → Microphone → MediaRecorder API         │
│                          │                                  │
│                    Audio chunks (WebM)                      │
└──────────────────────────┼──────────────────────────────────┘
                           │
                           ▼
              ┌────────────────────────┐
              │   Railway Backend      │
              │   (Node.js)            │
              │                        │
              │  Receives audio stream │
              │  Forwards to Deepgram  │
              └────────────┬───────────┘
                           │
                           ▼
              ┌────────────────────────┐
              │  Deepgram Nova-2       │
              │  Medical               │
              │                        │
              │  Real-time ASR with    │
              │  medical vocabulary    │
              │  Smart formatting ON   │
              └────────────┬───────────┘
                           │
                  Transcript JSON
                  + patient demographics
                           │
                           ▼
┌─────────────────────────────────────────────────────────────┐
│                    MAKE.COM SCENARIO                        │
│                                                             │
│  Step 1 │ Webhook trigger                                   │
│         │ Receives: transcript, firstName, lastName, DOB    │
│         │                                                   │
│  Step 2 │ Set Variable — extract clean transcript           │
│         │ Path: results.channels[].alternatives[]           │
│         │       .paragraphs.transcript                      │
│         │                                                   │
│  Step 3 │ Claude Sonnet API call                             │
│         │ Input:  clean transcript + patient name           │
│         │ Output: { subjective, objective,                  │
│         │           assessment, plan }                      │
│         │                                                   │
│  Step 4 │ Parse JSON — extract SOAP fields                  │
│         │                                                   │
│  Step 5 │ HTTP POST → OpenEMR                               │
│         │ Creates clinical note on patient record           │
│         │                                                   │
│  Step 6 │ Google Sheets append                              │
│         │ Logs: timestamp, patient, SOAP fields,            │
│         │       detected language                           │
└─────────────────────────────────────────────────────────────┘
```

---

## Tech Stack

| Component | Technology |
|---|---|
| AI / LLM | Claude Sonnet (`claude-Sonnet-4.6`) |
| Transcription | Deepgram Nova-2 Medical |
| Automation | Make.com |
| Recorder Frontend | HTML / CSS / JS (GitHub Pages) |
| Backend | Node.js (Railway) |
| EHR | OpenEMR |
| Audit Log | Google Sheets |

---

### Import Steps

1. Open Make.com → **Create a new scenario**
2. Click the **three-dot menu** → **Import Blueprint**
3. Upload `ai_scribe_blueprint.json`
4. In the **Webhook** module, create a new webhook — copy the generated URL
5. In the **HTTP (Claude API)** module, add your `Authorization: Bearer YOUR_ANTHROPIC_API_KEY` header
6. In the **HTTP (OpenEMR)** module, configure your OpenEMR base URL and credentials
7. In the **Google Sheets** module, connect your Google account and select your target sheet
8. **Activate** the scenario

Set your webhook URL as `MAKE_WEBHOOK_URL` in your Railway environment variables.

---

## How to Run Locally

### Prerequisites

- [Railway](https://railway.app) account (or any Node.js host)
- [Deepgram](https://console.deepgram.com) API key
- [Anthropic](https://console.anthropic.com) API key
- Make.com account
- OpenEMR instance (or swap for any EHR/database HTTP endpoint)

---

## Project Structure

```
ai-medical-scribe/
├── recorder/
│   ├── index.html                  # Browser-based recorder UI (GitHub Pages)
│   └── server.js                   # Node.js backend (Railway)
├── make-blueprint/
│   └── ai_scribe_blueprint.json    # Import directly into Make.com
├── docs/
│   └── system-prompt.md            # Claude system prompt reference + rationale
├── screenshots/
│   ├── recorder_ui.png
│   └── soap_output.png
├── .env.example                    # Environment variable template
├── .gitignore                      # Excludes secrets and audio recordings
└── README.md
```

---

## Key Design Decisions

**Claude Sonnet over Sonnet** — At the point of care, response speed matters. Sonnet generates a complete SOAP note in ~1.5 seconds. Sonnet would be 4–6× slower and 15× more expensive per note. For structured JSON output on a well-constrained task, Sonnet is the right tool.

**JSON-only system prompt enforcement** — Make.com's JSON parse module requires clean input. If Claude wraps its output in markdown code fences (` ```json ... ``` `), the downstream parse step fails silently and the note is lost. The system prompt's explicit prohibition on backticks is load-bearing — not defensive styling.

**Deepgram Nova-2 Medical** — General-purpose ASR models mishandle clinical vocabulary: drug names, anatomical terms, ICD-style shorthand. Nova-2 Medical's vocabulary significantly reduces transcription errors that would otherwise propagate into the SOAP note.

**Make.com for orchestration** — The entire AI pipeline runs without a custom backend. All logic — variable extraction, API calls, JSON parsing, EHR write, audit log — lives in the visual Make scenario. This makes the pipeline maintainable and debuggable by non-engineers.

**Deepgram paragraph transcript path** — The clean transcript is pulled from `results.channels[].alternatives[].paragraphs.transcript` rather than the top-level transcript field. This uses Deepgram's smart formatting layer, which produces better-punctuated, paragraph-structured text — which in turn produces better-structured SOAP notes from Claude.

**Audio never touches Make.com** — Audio is a large binary. Routing it through Make would consume data operations and slow the scenario. Only the text transcript is sent to the webhook. Deepgram handles the audio-to-text step upstream.

---

## Security Notes

- API keys stored as environment variables — never in source code or committed to the repo
- `.gitignore` explicitly excludes audio files (`.mp3`, `.wav`, `.webm`) to prevent accidental PHI commits
- This demo stack uses **synthetic patient data only** — it is not HIPAA-compliant
- For production use with real patient data, a signed BAA is required with every vendor in the pipeline (Deepgram, Anthropic, Make.com, your hosting provider)

---

## About the Author

**Hassaan Ahmed Siddiqui** building AI-powered ambient documentation tools for clinical practice. This demo stack was the proof-of-concept that validated the pipeline before production hardening.

- 🔗 [LinkedIn](https://linkedin.com/in/hassaan-ahmed-siddiqui)

---

## License

MIT License — free to use, modify, and distribute.

---

*Built with Deepgram Nova-2 Medical · Powered by Claude Sonnet · Orchestrated by Make.com*

