# 🧠 Project Architecture — AI Voice Agent Platform

## What This Project Is

This is a **production-grade AI Voice Agent** platform — an **AI-powered phone calling system** built on top of **LiveKit's real-time infrastructure**. It can:

1. **Make outbound AI phone calls** — The system dials a real phone number, and when the person answers, they talk to an AI agent (not a human).
2. **Receive inbound calls** — Via a SIP trunk, callers dial in and are greeted by the AI.
3. **Hold natural multilingual conversations** — The AI listens (Speech-to-Text), thinks (LLM), and speaks (Text-to-Speech) in real-time, supporting **11 Indian languages** (Hindi, Hinglish, English, Tamil, Telugu, Gujarati, Bengali, Marathi, Kannada, Malayalam, + auto-detect).
4. **Book appointments** — During the call, the AI can check calendar availability (Cal.com or Google Calendar), save a booking intent, and confirm it after the call ends.
5. **Transfer calls** — If the caller asks for a human, the AI performs a live SIP transfer to a configured phone number.
6. **Record calls** — Audio is recorded and stored in Supabase S3 storage.
7. **Analyze sentiment** — Post-call, GPT-4o-mini classifies the conversation as positive/neutral/negative/frustrated.
8. **Log everything** — Full transcripts, duration, cost estimates, and analytics are saved to Supabase.
9. **Send notifications** — Telegram messages and WhatsApp confirmations (via Twilio) are sent for bookings, missed bookings, and errors.
10. **Trigger webhooks** — n8n/custom CRM webhooks fire on call completion for workflow automation.
11. **Provide a web dashboard** — A FastAPI-powered UI shows call logs, stats, bookings, a CRM contact list, config editor, and even an in-browser voice demo.

---

## 🔧 How It Works — End-to-End Flow

### Outbound Call Flow

1. **Trigger**: A human runs `make_call.py --to +91XXXXXXXXXX`. This uses the LiveKit API to create a **dispatch request** — it tells LiveKit: *"Send the agent named `outbound-caller` to a new room, and here's the phone number in the metadata."*

2. **Agent Picks Up Job**: `agent.py` is a long-running **LiveKit Worker**. It's constantly listening for dispatch jobs. When it receives one, the `entrypoint()` function fires.

3. **Room & SIP Setup**: The agent connects to the LiveKit room. LiveKit's SIP infrastructure dials the phone number via the **Vobiz SIP trunk**. The phone rings.

4. **Conversation Loop**: When the callee answers:
   - **STT (Sarvam Saaras v3 / Deepgram Nova-2)** transcribes audio → text in real-time.
   - **LLM (GPT-4o-mini / Groq / Claude)** processes the text + system prompt + conversation history → generates a response.
   - **TTS (Sarvam Bulbul v3 / ElevenLabs)** converts the AI text → speech audio streamed back to the caller.
   - This loop repeats with turn detection, interruption handling, and filler word filtering.

5. **Tool Calls During Conversation**: The LLM can invoke tools mid-conversation:
   - `check_availability(date)` — queries Cal.com or Google Calendar
   - `save_booking_intent(...)` — records the caller's intent to book
   - `transfer_call()` — SIP-transfers to a human
   - `end_call()` — hangs up the SIP call
   - `get_business_hours()` — reports current open/closed status

6. **Post-Call Shutdown**: When the call ends (hangup or disconnect):
   - If a booking intent was saved → actually creates the booking on Cal.com / Google Calendar
   - Sends Telegram notification + WhatsApp confirmation
   - GPT-4o-mini runs sentiment analysis on the transcript
   - Estimates call cost
   - Saves full call log to Supabase (transcript, duration, sentiment, cost, recording URL, etc.)
   - Fires n8n webhook
   - Stops the audio recording egress

---

## 🧱 The Pieces — Every Component Explained

### Core Infrastructure

| Component | Technology | Role |
|-----------|-----------|------|
| **LiveKit** | `livekit`, `livekit-agents`, `livekit-api`, `livekit-protocol` | The **real-time communication backbone**. LiveKit is a WebRTC/SIP server. It manages "rooms" where participants (AI agent + phone caller) exchange audio. The `livekit-agents` framework provides the `Agent`, `AgentSession`, `WorkerOptions`, and `JobContext` abstractions — the agent registers as a worker, LiveKit dispatches jobs to it, and the framework handles the audio pipeline (STT→LLM→TTS) automatically. |
| **Vobiz SIP Trunk** | Configured via `setup_trunk.py` | The **phone gateway**. SIP (Session Initiation Protocol) is how VoIP calls work. Vobiz provides a SIP trunk — a virtual phone line. LiveKit connects to Vobiz's SIP domain with credentials, and through it can dial any real phone number. The trunk is configured with `setup_trunk.py` which calls `lkapi.sip.update_outbound_trunk_fields()`. |

### AI Pipeline (The Voice Brain)

| Component | Technology | Role |
|-----------|-----------|------|
| **STT (Speech-to-Text)** | **Sarvam Saaras v3** (`livekit-plugins-sarvam`) or **Deepgram Nova-2** (`livekit-plugins-deepgram`) | Converts the caller's voice audio into text. Sarvam is optimized for **Indian languages** with auto-detection (`language="unknown"`). Deepgram's `nova-2-general` is used in multilingual mode as an alternative. Audio is processed at **16kHz** sample rate. The `min_endpointing_delay` (default 0.05s) controls how quickly the system decides the user has stopped talking. |
| **LLM (Large Language Model)** | **OpenAI GPT-4o-mini** (default), **Groq** (Llama 3.3 70B), or **Claude Haiku 3.5** | The "brain" — takes the system prompt + conversation history + user's latest utterance, and generates the next response. Capped at **120 tokens** per response to keep replies short and phone-appropriate. The system prompt includes: agent personality/instructions, IST time context (with a 7-day date table for relative date resolution), language directives, and caller history from past calls. |
| **TTS (Text-to-Speech)** | **Sarvam Bulbul v3** (`livekit-plugins-sarvam`) or **ElevenLabs Turbo v2.5** (`livekit-plugins-elevenlabs`) | Converts the AI's text response into spoken audio at **24kHz**. Sarvam supports Indian language voices (kavya, ritu, dev, priya, rohan, neha, shubh, rahul). A `before_tts_cb` sentence chunker splits responses so only the first sentence is spoken — keeping replies snappy. |
| **VAD (Voice Activity Detection)** | **Silero** (`livekit-plugins-silero`) | Detects when the caller is speaking vs. silent. Used for turn detection — knowing when the user finished their sentence so the AI can respond. |
| **Noise Cancellation** | LiveKit BVC (Background Voice Cancellation) | Attempts to clean up background noise from the phone call. Falls back gracefully if unavailable. |

### Data & Persistence

| Component | Technology | Role |
|-----------|-----------|------|
| **Supabase** | `supabase` Python client | The **database and storage layer**. Three tables: `call_logs` (full call records with transcript, sentiment, cost, duration), `call_transcripts` (real-time per-message transcript stream), `active_calls` (live call status tracking). Also uses **Supabase S3-compatible storage** for call recordings (`.ogg` files). The `db.py` module handles all DB operations with retry logic for transient SSL errors and graceful schema fallback if migration hasn't been run. |

### Calendar / Booking

| Component | Technology | Role |
|-----------|-----------|------|
| **Cal.com** | REST API (`api.cal.com/v1` and `/v2`) | Primary appointment scheduling. `get_available_slots()` fetches open time slots. `async_create_booking()` creates bookings with attendee info. `cancel_booking()` cancels by UID. |
| **Google Calendar** | `google-api-python-client`, `google-auth` | Alternative calendar backend. If `GOOGLE_CALENDAR_ID` is configured, it queries Google's FreeBusy API to compute available 30-minute windows between 10am–7pm IST, and creates events via the Calendar API. |

### Notifications

| Component | Technology | Role |
|-----------|-----------|------|
| **Telegram** | Bot API (`api.telegram.org`) | Sends rich Markdown notifications to a configured Telegram chat: booking confirmations, call-ended-no-booking alerts, and error notifications. The business owner monitors this. |
| **WhatsApp** | Twilio API | Sends booking confirmation messages **to the caller** via WhatsApp — "Your appointment is confirmed for Monday, 14 April at 10:00 AM IST." |
| **n8n Webhook** | HTTP POST | On call completion, fires a JSON payload to a configurable webhook URL containing phone, name, duration, booking status, sentiment, recording URL. This enables CRM integration, Zapier-style automations, etc. |

### Monitoring & Observability

| Component | Technology | Role |
|-----------|-----------|------|
| **Sentry** | `sentry-sdk` with `AsyncioIntegration` | Error tracking and crash reporting in production. Traces at 10% sample rate. |
| **OpenTelemetry** | `opentelemetry-*` packages | Distributed tracing and metrics export (OTLP protocol) for observability. |
| **Token Counter** | `tiktoken` | Counts system prompt tokens to warn if the prompt exceeds 600 tokens (impacts latency). |
| **Cost Estimator** | Custom formula | Estimates per-call cost: `(duration/60 × $0.002 STT) + (duration/60 × $0.006 TTS) + (chars/1000 × $0.003 LLM) + (chars/4000 × $0.0001)`. |

### Web Dashboard & Configuration

| Component | Technology | Role |
|-----------|-----------|------|
| **FastAPI** (`ui_server.py`) | `fastapi` + `uvicorn` | Serves a full admin dashboard with: config editor (change LLM model, TTS voice, system prompt live), call logs viewer, booking list, stats (total calls, booking rate, avg duration), CRM contacts page, and an **in-browser voice demo** (using LiveKit's browser SDK to let you talk to the AI directly from a web page). |
| **Config System** | `config.json` + per-client configs in `configs/` folder | Hot-reloadable configuration. Every call reads `config.json` fresh, so you can change the agent's personality, voice, language, or LLM model **without restarting**. Per-client configs (`configs/{phone}.json`) allow different behavior for different callers. |

### Safety & Rate Limiting

| Component | Role |
|-----------|------|
| **Rate Limiter** | Max 5 calls per phone number per hour. Prevents abuse. |
| **Turn Counter** | After `max_turns` (default 25) conversation turns, the agent politely wraps up. |
| **Filler Word Filter** | Drops "okay", "hmm", "haan", "ji" etc. so the LLM doesn't respond to noise. |
| **Echo Filter** | If the agent is speaking and STT picks up its own voice, the transcript is dropped. |
| **Interruption Tracking** | Counts how many times the caller interrupted the agent (logged for analytics). |

---

## 📊 Visual Flowchart

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                          SYSTEM ARCHITECTURE                                │
└─────────────────────────────────────────────────────────────────────────────┘

 ┌──────────────┐    dispatch request     ┌──────────────────┐
 │ make_call.py │ ──────────────────────► │   LiveKit Cloud   │
 │  (trigger)   │   room + phone meta     │    (SFU Server)   │
 └──────────────┘                         └────────┬─────────┘
                                                   │ assigns job
                                                   ▼
                                          ┌──────────────────┐
                                          │    agent.py       │
                                          │  (LiveKit Worker) │
                                          └────────┬─────────┘
                                                   │ connects to room
                                                   ▼
                              ┌─────────────────────────────────────────┐
                              │           LiveKit Room                   │
                              │  ┌─────────────┐   ┌────────────────┐  │
                              │  │  AI Agent    │◄─►│  Phone Caller  │  │
                              │  │ (participant)│   │  (SIP particip)│  │
                              │  └─────────────┘   └────────────────┘  │
                              └──────────┬──────────────────┬──────────┘
                                         │                  │
                            dials via SIP │                  │ audio stream
                                         ▼                  │
                              ┌──────────────────┐          │
                              │  Vobiz SIP Trunk  │          │
                              │  (Phone Gateway)  │──────────┘
                              └──────────────────┘
                                         │
                                    PSTN call
                                         ▼
                                  ┌──────────┐
                                  │  📱 Phone │
                                  │  (Callee) │
                                  └──────────┘


┌─────────────────────────────────────────────────────────────────────────────┐
│                     REAL-TIME CONVERSATION LOOP                             │
└─────────────────────────────────────────────────────────────────────────────┘

    📱 Caller speaks
         │
         ▼
  ┌──────────────┐    raw audio     ┌─────────────────────┐
  │   Silero VAD  │ ──────────────► │  STT (Sarvam/DG)    │
  │ (voice detect)│                 │  Audio → Text        │
  └──────────────┘                  └──────────┬──────────┘
                                               │ transcript text
                                               ▼
                                    ┌──────────────────────┐
                                    │  Filters:             │
                                    │  • Echo filter        │
                                    │  • Filler word filter │
                                    │  • Min length check   │
                                    └──────────┬───────────┘
                                               │ clean text
                                               ▼
                                    ┌──────────────────────┐
                                    │  LLM (GPT-4o-mini /  │
                                    │  Groq / Claude)       │
                                    │                       │
                                    │  System Prompt:       │
                                    │  • Agent personality  │
                                    │  • IST time context   │
                                    │  • Language directive  │
                                    │  • Caller history     │
                                    │                       │
                                    │  Tools available:     │
                                    │  • check_availability │
                                    │  • save_booking_intent│
                                    │  • transfer_call      │
                                    │  • end_call           │
                                    │  • get_business_hours │
                                    └──────────┬───────────┘
                                               │ response text
                                               ▼
                                    ┌──────────────────────┐
                                    │  TTS (Sarvam/11Labs) │
                                    │  Text → Speech Audio  │
                                    └──────────┬───────────┘
                                               │ audio stream
                                               ▼
                                          📱 Caller hears AI


┌─────────────────────────────────────────────────────────────────────────────┐
│                       POST-CALL SHUTDOWN PIPELINE                           │
└─────────────────────────────────────────────────────────────────────────────┘

    📱 Caller hangs up
         │
         ▼
  ┌────────────────────────────────────────────────────────┐
  │              unified_shutdown_hook()                     │
  │                                                         │
  │  1. Create booking (Cal.com / Google Calendar)          │
  │     └─► Telegram notification + WhatsApp confirmation   │
  │                                                         │
  │  2. Build full transcript from chat context              │
  │                                                         │
  │  3. Sentiment analysis (GPT-4o-mini)                    │
  │     └─► "positive" / "neutral" / "negative"             │
  │                                                         │
  │  4. Cost estimation                                      │
  │                                                         │
  │  5. Stop audio recording egress                          │
  │                                                         │
  │  6. Fire n8n webhook                                     │
  │                                                         │
  │  7. Save to Supabase:                                    │
  │     ┌─────────────────────────────────────┐             │
  │     │ call_logs table:                     │             │
  │     │  phone, duration, transcript,        │             │
  │     │  summary, sentiment, was_booked,     │             │
  │     │  interrupt_count, estimated_cost,     │             │
  │     │  recording_url, call_date/hour/day   │             │
  │     └─────────────────────────────────────┘             │
  └────────────────────────────────────────────────────────┘


┌─────────────────────────────────────────────────────────────────────────────┐
│                       WEB DASHBOARD (ui_server.py)                          │
└─────────────────────────────────────────────────────────────────────────────┘

  Browser ◄──► FastAPI (uvicorn)
                  │
                  ├── GET  /api/config     → Live agent configuration
                  ├── POST /api/config     → Update config (hot-reload)
                  ├── GET  /api/logs       → Call history from Supabase
                  ├── GET  /api/stats      → Aggregated analytics
                  ├── GET  /api/bookings   → Confirmed appointments
                  ├── GET  /api/contacts   → CRM: deduplicated contacts
                  ├── GET  /              → Full admin dashboard (HTML)
                  └── GET  /demo          → In-browser voice demo page
```

---

## Summary

This is essentially a **full-stack AI call center in a box** — you configure your phone line (Vobiz), your AI brain (OpenAI/Groq/Claude), your ears (Sarvam/Deepgram), your voice (Sarvam/ElevenLabs), your calendar (Cal.com/Google), and your notifications (Telegram/WhatsApp) — and you get an AI that can make and receive real phone calls, have intelligent multilingual conversations, book appointments, and report everything to a dashboard.
