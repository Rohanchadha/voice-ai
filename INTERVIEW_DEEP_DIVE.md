# 🧠 Deep Dive: How This AI Voice Agent Works (Interview Prep)

> **Purpose**: This document is a complete, interview-ready breakdown of the entire AI Voice Agent platform — every component, every interaction, every protocol. Use this to explain the system confidently at any level of detail.

---

## Table of Contents

1. [30-Second Elevator Pitch](#1-30-second-elevator-pitch)
2. [System Overview — The 10,000ft View](#2-system-overview--the-10000ft-view)
3. [The Cast of Characters (Every Component)](#3-the-cast-of-characters-every-component)
4. [Inbound Call Flow — Step by Step](#4-inbound-call-flow--step-by-step)
5. [Outbound Call Flow — Step by Step](#5-outbound-call-flow--step-by-step)
6. [The Real-Time Voice Pipeline (STT → LLM → TTS)](#6-the-real-time-voice-pipeline-stt--llm--tts)
7. [Tool Calling — How the AI Takes Actions Mid-Call](#7-tool-calling--how-the-ai-takes-actions-mid-call)
8. [Post-Call Shutdown — What Happens After Hangup](#8-post-call-shutdown--what-happens-after-hangup)
9. [The Web Dashboard (ui_server.py)](#9-the-web-dashboard-ui_serverpy)
10. [Configuration System — How Settings Flow](#10-configuration-system--how-settings-flow)
11. [Key Protocols & Concepts Explained](#11-key-protocols--concepts-explained)
12. [File-by-File Breakdown](#12-file-by-file-breakdown)
13. [Data Flow Diagrams (ASCII)](#13-data-flow-diagrams-ascii)
14. [Common Interview Questions & Answers](#14-common-interview-questions--answers)

---

## 1. 30-Second Elevator Pitch

> "This is a production AI phone agent. When someone calls a phone number (or we call them), LiveKit's real-time infrastructure bridges the phone network to a WebRTC room. Inside that room, a Python agent listens to the caller's speech, transcribes it with Sarvam AI, sends the text to GPT-4o-mini for a response, converts that response back to speech with Sarvam TTS, and streams it to the caller — all in real-time, under 1 second latency. The agent can book appointments, transfer to humans, and speaks 11 Indian languages. Post-call, it logs everything to Supabase, does sentiment analysis, and sends Telegram notifications."

---

## 2. System Overview — The 10,000ft View

```
┌──────────────────────────────────────────────────────────────────────┐
│                        EXTERNAL SERVICES                             │
│                                                                      │
│  ┌─────────┐  ┌──────────┐  ┌─────────┐  ┌─────────┐  ┌─────────┐ │
│  │ Vobiz   │  │ Sarvam   │  │ OpenAI  │  │ Cal.com │  │Supabase │ │
│  │ (Phone) │  │ (STT+TTS)│  │ (LLM)   │  │(Calendar│  │ (DB+S3) │ │
│  └────┬────┘  └────┬─────┘  └────┬────┘  └────┬────┘  └────┬────┘ │
│       │             │             │             │             │       │
└───────┼─────────────┼─────────────┼─────────────┼─────────────┼──────┘
        │             │             │             │             │
        │ SIP         │ HTTPS       │ HTTPS       │ HTTPS       │ HTTPS
        │             │             │             │             │
┌───────┼─────────────┼─────────────┼─────────────┼─────────────┼──────┐
│       ▼             ▼             ▼             ▼             ▼       │
│  ┌─────────────────────────────────────────────────────────────────┐ │
│  │                    LiveKit Cloud                                │ │
│  │  ┌───────────┐  ┌──────────┐  ┌────────────┐  ┌────────────┐  │ │
│  │  │SIP Gateway│  │  Rooms   │  │ Job        │  │  Egress    │  │ │
│  │  │(Inbound & │  │(WebRTC   │  │ Dispatcher │  │ (Recording)│  │ │
│  │  │ Outbound) │  │ audio)   │  │            │  │            │  │ │
│  │  └───────────┘  └──────────┘  └────────────┘  └────────────┘  │ │
│  └──────────────────────────┬─────────────────────────────────────┘ │
│                             │ WebSocket                              │
│                             ▼                                        │
│  ┌─────────────────────────────────────────────────────────────────┐ │
│  │              YOUR PYTHON PROCESS (agent.py)                     │ │
│  │                                                                 │ │
│  │  ┌─────────┐  ┌─────────────┐  ┌────────────┐  ┌───────────┐  │ │
│  │  │ Worker  │  │ AgentSession│  │  Agent     │  │AgentTools │  │ │
│  │  │(listens │→│(STT→LLM→TTS │→│(personality│→│(book,call, │  │ │
│  │  │for jobs)│  │  pipeline)  │  │ + prompt)  │  │ transfer) │  │ │
│  │  └─────────┘  └─────────────┘  └────────────┘  └───────────┘  │ │
│  └─────────────────────────────────────────────────────────────────┘ │
│                                                                      │
│  YOUR SERVER                                                         │
└──────────────────────────────────────────────────────────────────────┘
```

---

## 3. The Cast of Characters (Every Component)

### 3.1 LiveKit Cloud — The Real-Time Backbone

LiveKit is an **open-source WebRTC infrastructure** (like Twilio Video, but open). In this project, LiveKit Cloud (hosted version) does several things:

| Sub-component | What it does |
|---------------|-------------|
| **SIP Gateway** | Speaks the SIP protocol. Accepts incoming SIP calls from Vobiz (inbound) and places outgoing SIP calls through Vobiz (outbound). Converts SIP↔WebRTC. |
| **Rooms** | Virtual spaces where participants exchange real-time audio/video. When a call happens, a Room is created. The phone caller is one participant (via SIP), the AI agent is another (via WebSocket). |
| **Job Dispatcher** | When a room needs an agent, LiveKit dispatches a "job" to a registered worker. It load-balances across multiple workers if you have them. |
| **Egress** | Recording service. Can capture all audio in a room, encode it (OGG), and upload to S3-compatible storage (Supabase S3 in our case). |

**Key insight**: LiveKit is the **switchboard**. It doesn't know anything about AI, booking, or Telegram. It just moves audio between participants and dispatches jobs to your code.

### 3.2 Vobiz — The Phone Network Bridge

Vobiz is an **Indian VoIP carrier** (SIP trunk provider). It does one thing: bridges the **PSTN** (Public Switched Telephone Network — regular phone calls) and the **internet** (SIP protocol).

- **Inbound**: Someone dials your Vobiz DID number → Vobiz converts the PSTN call into a SIP INVITE → sends it to LiveKit's SIP gateway.
- **Outbound**: LiveKit sends a SIP INVITE to Vobiz → Vobiz converts it to a PSTN call → a real phone rings.

**Credentials needed**: SIP domain, username, password, DID number (your purchased phone number).

### 3.3 The Python Agent (agent.py) — The Brain

This is YOUR code. It's a **long-running Python process** that:

1. **Registers as a Worker** with LiveKit Cloud via persistent WebSocket
2. **Waits for jobs** (incoming/outgoing calls)
3. **Joins the room** when a job arrives
4. **Runs the voice pipeline**: STT → LLM → TTS in real-time
5. **Handles tool calls**: booking, availability checks, transfers, hangups
6. **Runs post-call hooks**: sentiment analysis, logging, notifications

#### The Four Key Classes in agent.py:

```
┌─────────────────────────────────────────────────────────────────┐
│                         agent.py                                 │
│                                                                  │
│  ┌──────────────┐    ┌───────────────────┐                      │
│  │ WorkerOptions│    │ entrypoint(ctx)   │                      │
│  │              │    │                   │                      │
│  │ agent_name=  │    │ • Connects to room│                      │
│  │ "outbound-   │    │ • Extracts caller │                      │
│  │  caller"     │    │   phone number    │                      │
│  │              │    │ • Loads config     │                      │
│  │ entrypoint=  │───▶│ • Builds STT,LLM, │                      │
│  │ entrypoint   │    │   TTS instances   │                      │
│  └──────────────┘    │ • Creates Agent   │                      │
│                      │ • Starts Session  │                      │
│                      │ • Sets up hooks   │                      │
│                      └───────────────────┘                      │
│                              │                                   │
│                      creates │                                   │
│                              ▼                                   │
│  ┌──────────────────────────────────────────────────┐           │
│  │            OutboundAssistant (Agent)              │           │
│  │                                                   │           │
│  │  • instructions = system prompt + IST context     │           │
│  │                    + language directive            │           │
│  │                    + caller history               │           │
│  │  • tools = [check_availability, save_booking,     │           │
│  │             transfer_call, end_call,              │           │
│  │             get_business_hours]                   │           │
│  │  • on_enter() → speaks the greeting               │           │
│  └──────────────────────────────────────────────────┘           │
│                              │                                   │
│                      uses    │                                   │
│                              ▼                                   │
│  ┌──────────────────────────────────────────────────┐           │
│  │            AgentTools (ToolContext)                │           │
│  │                                                   │           │
│  │  Holds state:                                     │           │
│  │  • caller_phone, caller_name                      │           │
│  │  • booking_intent (filled by save_booking tool)   │           │
│  │  • sip_domain, ctx_api, room_name                 │           │
│  │                                                   │           │
│  │  Methods (AI can call these mid-conversation):    │           │
│  │  • transfer_call()    → SIP transfer to human     │           │
│  │  • end_call()         → SIP hangup                │           │
│  │  • save_booking_intent() → store booking data     │           │
│  │  • check_availability()  → query Cal.com          │           │
│  │  • get_business_hours()  → return open/closed     │           │
│  └──────────────────────────────────────────────────┘           │
│                                                                  │
│  ┌──────────────────────────────────────────────────┐           │
│  │              AgentSession                         │           │
│  │  (from livekit.agents — NOT your code)            │           │
│  │                                                   │           │
│  │  Wires together:                                  │           │
│  │  • STT (Sarvam/Deepgram)                          │           │
│  │  • LLM (OpenAI/Groq/Claude)                       │           │
│  │  • TTS (Sarvam/ElevenLabs)                        │           │
│  │  • VAD (Silero — voice activity detection)        │           │
│  │  • Turn detection (knows when user stopped)       │           │
│  │  • Interruption handling                          │           │
│  │                                                   │           │
│  │  This is the "audio loop" — it continuously:      │           │
│  │  1. Receives audio frames from the room           │           │
│  │  2. Feeds them to STT                             │           │
│  │  3. When a complete utterance is detected,        │           │
│  │     sends to LLM                                  │           │
│  │  4. Streams LLM response to TTS                   │           │
│  │  5. Sends TTS audio frames back to the room       │           │
│  └──────────────────────────────────────────────────┘           │
└─────────────────────────────────────────────────────────────────┘
```

### 3.4 Supporting Modules

| File | Role | Talks to |
|------|------|----------|
| `calendar_tools.py` | Checks availability & creates bookings | Cal.com REST API, Google Calendar API |
| `notify.py` | Sends notifications after calls | Telegram Bot API, Twilio WhatsApp API |
| `db.py` | Reads/writes call logs and analytics | Supabase (PostgreSQL + S3) |
| `ui_server.py` | Web dashboard + API endpoints | Supabase (for data), LiveKit API (for dispatching calls) |
| `make_call.py` | CLI tool to trigger an outbound call | LiveKit API (creates agent dispatch) |
| `setup_trunk.py` | One-time setup of outbound SIP trunk | LiveKit SIP API |
| `setup_inbound.py` | One-time setup of inbound trunk + dispatch rule | LiveKit SIP API |
| `config.json` | Runtime configuration (prompt, model, voice, etc.) | Read by agent.py and ui_server.py |

---

## 4. Inbound Call Flow — Step by Step

This is the most important flow to understand. Here's what happens when someone dials your Vobiz number:

```
  STEP 1          STEP 2            STEP 3           STEP 4
  Caller          Vobiz             LiveKit           LiveKit
  dials           converts          SIP Gateway       Job Dispatcher
  +91-XXX         PSTN→SIP          receives          matches
                                    SIP INVITE        dispatch rule
    📱 ──────▶  📞──────▶  🌐──────▶  📋
    PSTN          SIP INVITE         Inbound           Dispatch Rule:
    call          to LiveKit's       Trunk             "agent_name:
                  SIP endpoint       matches!          outbound-caller"
                                                       │
  STEP 8          STEP 7            STEP 6           STEP 5
  Real-time       AgentSession      Worker            LiveKit
  conversation    pipeline          joins             creates Room
  begins          starts            the room          + notifies Worker
                                                       │
    🗣️ ◀──────▶ 🔄──────▶  🤖──────▶  🏠              ▼
    Caller         STT→LLM→TTS      entrypoint()      Room: "call-xyz"
    hears AI       audio loop        fires             Worker: AW_xxx
```

### Detailed breakdown:

**Step 1: Caller dials** → Their carrier routes the call through the PSTN to Vobiz (because Vobiz owns the DID number).

**Step 2: Vobiz converts** → Vobiz's SIP switch creates a **SIP INVITE** message (this is like an HTTP request but for phone calls) and sends it over the internet to LiveKit's SIP endpoint.

The SIP INVITE contains:
- `From`: the caller's phone number
- `To`: your Vobiz DID number
- `Via`: routing information
- Media negotiation (codec, ports for RTP audio)

**Step 3: LiveKit SIP Gateway receives** → LiveKit checks: "Do I have an **Inbound SIP Trunk** that matches this call?" It looks at the trunk's `numbers` field (your DID) and `allowed_addresses`. If it matches → accepts the SIP INVITE and sends back `200 OK`.

**Step 4: Dispatch Rule fires** → LiveKit finds the dispatch rule associated with this trunk. The rule says: "Create a room with prefix `call-` and dispatch to agent named `outbound-caller`."

**Step 5: Room created, Worker notified** → LiveKit creates a Room (e.g., `call-abc123`). The caller's SIP audio stream becomes a participant in this room. LiveKit then sends a **job notification** over the persistent WebSocket to your registered Worker.

**Step 6: Worker accepts, entrypoint fires** → Your `agent.py` process receives the job. The `entrypoint(ctx: JobContext)` function executes:
```python
async def entrypoint(ctx: JobContext):
    await ctx.connect()  # Join the room
    # Extract caller phone from SIP participant attributes
    # Load config, build STT/LLM/TTS, create Agent
    # Start AgentSession
```

**Step 7: AgentSession starts** → The voice pipeline begins:
- `on_enter()` fires → Agent speaks the greeting
- Audio loop: Room audio → STT → LLM → TTS → Room audio

**Step 8: Real-time conversation** → Caller and AI talk naturally. The LLM can call tools (booking, transfer, etc.) mid-conversation.

---

## 5. Outbound Call Flow — Step by Step

```
  STEP 1           STEP 2            STEP 3           STEP 4
  Trigger          LiveKit           Worker            Agent reads
  (make_call.py    creates Room      picks up          metadata,
  or Dashboard)    + dispatches      the job           dials via SIP
                   job
    💻 ──────▶   🏠──────▶   🤖──────▶   📞
    create_        Room:             entrypoint()      SIP INVITE
    dispatch()     "call-91xx-1234"  fires             → Vobiz → PSTN
                   metadata:                            Phone rings!
                   {phone: +91xxx}
                                                        │
  STEP 5           STEP 6                               │
  Callee           Conversation                         ▼
  answers          begins                             📱
                                                      Callee's
    🗣️ ◀──────▶  🔄                                  phone
    Callee          STT→LLM→TTS
    hears AI        audio loop
```

### Key difference from inbound:

In **inbound**, Vobiz initiates the SIP INVITE → LiveKit. In **outbound**, LiveKit initiates the SIP INVITE → Vobiz. Everything downstream (room, agent, voice pipeline) is identical.

The trigger for outbound is `make_call.py` or the dashboard's `/api/call/single` endpoint, which calls:
```python
api.agent_dispatch.create_dispatch(
    agent_name="outbound-caller",
    room=room_name,
    metadata=json.dumps({"phone_number": "+91XXXXXXXXXX"})
)
```

The agent reads `ctx.job.metadata` to get the phone number, then LiveKit's SIP infrastructure dials it via the **Outbound SIP Trunk** (configured by `setup_trunk.py`).

---

## 6. The Real-Time Voice Pipeline (STT → LLM → TTS)

This is the core "magic" — how the AI actually converses in real-time.

```
┌─────────────────────────────────────────────────────────────────────┐
│                    AgentSession Audio Loop                           │
│                                                                     │
│  ┌─────────┐    ┌──────────┐    ┌─────────┐    ┌──────────┐       │
│  │  Room   │    │   VAD    │    │  STT    │    │   LLM    │       │
│  │  Audio  │───▶│ (Silero) │───▶│(Sarvam) │───▶│(GPT-4o-  │       │
│  │  Input  │    │          │    │         │    │  mini)   │       │
│  │(16kHz)  │    │ Detects  │    │ Audio   │    │          │       │
│  │         │    │ speech   │    │ → Text  │    │ Text     │       │
│  │         │    │ start/   │    │         │    │ → Text   │       │
│  │         │    │ end      │    │ Hindi,  │    │ response │       │
│  └─────────┘    └──────────┘    │ English │    │ (≤120    │       │
│                                 │ Tamil.. │    │  tokens) │       │
│                                 └─────────┘    └────┬─────┘       │
│                                                     │              │
│                                                     │ may call     │
│                                                     │ tools here   │
│                                                     ▼              │
│  ┌─────────┐    ┌──────────┐                  ┌──────────┐        │
│  │  Room   │    │   TTS    │                  │  Tool    │        │
│  │  Audio  │◀───│ (Sarvam) │◀── text ─────────│  Results │        │
│  │  Output │    │          │                  │          │        │
│  │ (24kHz) │    │ Text     │                  │check_avl │        │
│  │         │    │ → Audio  │                  │save_book │        │
│  │ Streamed│    │          │                  │transfer  │        │
│  │ back to │    │ kavya,   │                  │end_call  │        │
│  │ caller  │    │ ritu,dev │                  │biz_hours │        │
│  └─────────┘    └──────────┘                  └──────────┘        │
│                                                                     │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │  Turn Detection & Interruption Handling                      │  │
│  │                                                              │  │
│  │  • min_endpointing_delay = 0.05s (how long to wait after    │  │
│  │    user stops speaking before treating it as "done")         │  │
│  │                                                              │  │
│  │  • allow_interruptions = True (if user speaks while AI is   │  │
│  │    talking, AI stops and listens)                            │  │
│  │                                                              │  │
│  │  • Filler word filter: drops "ok", "hmm", "haan", "accha"  │  │
│  │    so they don't trigger a full LLM response                │  │
│  │                                                              │  │
│  │  • Echo filter: drops transcripts detected while AI is      │  │
│  │    speaking (prevents AI hearing its own voice)              │  │
│  └──────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────┘
```

### Latency Breakdown (approximate):

| Stage | Time | Notes |
|-------|------|-------|
| Audio capture + VAD | ~50ms | Silero processes in chunks |
| STT (Sarvam Saaras v3) | ~200-400ms | Streaming, 16kHz input |
| LLM (GPT-4o-mini) | ~200-500ms | 120 token cap helps |
| TTS (Sarvam Bulbul v3) | ~150-300ms | Streaming, 24kHz output |
| Network round-trips | ~50-100ms | India South region |
| **Total first-byte** | **~650ms-1.3s** | **Feels natural for phone** |

### The System Prompt Assembly

The agent's personality is built from multiple pieces concatenated together:

```
┌──────────────────────────────────────────────────┐
│              Final System Prompt                  │
│                                                   │
│  1. agent_instructions (from config.json)         │
│     "You are Aryan from RapidX AI..."             │
│                                                   │
│  2. IST Time Context (auto-generated)             │
│     "Current date: Monday, April 21, 2026         │
│      Today: Monday 21 April 2026 → 2026-04-21     │
│      Tomorrow: Tuesday 22 April..."               │
│                                                   │
│  3. Language Directive (from lang_preset)          │
│     "Detect the caller's language from their      │
│      first message and reply in that SAME          │
│      language..."                                  │
│                                                   │
│  4. Caller History (from Supabase, if exists)     │
│     "[CALLER HISTORY: Last call 2026-04-15.        │
│      Summary: Booking Confirmed: abc123]"          │
└──────────────────────────────────────────────────┘
```

---

## 7. Tool Calling — How the AI Takes Actions Mid-Call

The LLM doesn't just generate text — it can **call functions**. This is OpenAI's "function calling" / "tool use" feature. Here's how it works in a real conversation:

```
Caller: "Kal ke liye koi slot available hai?"
         (Is any slot available for tomorrow?)

    ┌─────────────────────────────────────────────┐
    │  LLM receives:                               │
    │  - System prompt (with date table)           │
    │  - Conversation history                      │
    │  - User message: "Kal ke liye koi slot..."   │
    │                                              │
    │  LLM decides: I need to call a tool!         │
    │  Output: {                                   │
    │    "tool_call": "check_availability",        │
    │    "arguments": {"date": "2026-04-22"}       │
    │  }                                           │
    └───────────────────┬─────────────────────────┘
                        │
                        ▼
    ┌─────────────────────────────────────────────┐
    │  AgentSession intercepts the tool call       │
    │  → calls check_availability("2026-04-22")    │
    │  → which calls Cal.com API                   │
    │  → returns: "Available slots: 10:00, 11:00,  │
    │              14:00, 15:00 IST"               │
    └───────────────────┬─────────────────────────┘
                        │
                        ▼
    ┌─────────────────────────────────────────────┐
    │  LLM receives the tool result                │
    │  → generates natural response:               │
    │  "Haan, kal ke liye slots available hain —   │
    │   10 baje, 11 baje, 2 baje, aur 3 baje.     │
    │   Kaunsa time aapko suit karega?"            │
    └───────────────────┬─────────────────────────┘
                        │
                        ▼
    ┌─────────────────────────────────────────────┐
    │  TTS converts response → audio → caller      │
    └─────────────────────────────────────────────┘
```

### All 5 Tools:

| Tool | When LLM calls it | What it does internally |
|------|-------------------|------------------------|
| `check_availability(date)` | User asks about available times | Calls Cal.com `/v1/slots` API or Google Calendar FreeBusy API |
| `save_booking_intent(start_time, name, phone, notes)` | User confirms they want to book | Saves intent to `agent_tools.booking_intent` dict (actual booking happens post-call) |
| `transfer_call()` | User asks for human / is angry | Calls LiveKit `sip.transfer_sip_participant()` → SIP REFER → Vobiz transfers the call |
| `end_call()` | User says goodbye / booking confirmed | Calls SIP transfer to dummy number `tel:+00000000` (hangup trick) |
| `get_business_hours()` | User asks if open | Returns current IST time vs hardcoded schedule |

### Why booking is "intent-based":

The agent saves a **booking intent** during the call but doesn't actually create the booking until the **post-call shutdown hook**. This is intentional:
- The caller might change their mind before hanging up
- Creating a booking mid-call adds latency to the conversation
- If the call drops, the intent is still processed in shutdown

---

## 8. Post-Call Shutdown — What Happens After Hangup

When a call ends (caller hangs up, or `end_call()` is triggered), the `unified_shutdown_hook` fires:

```
┌─────────────────────────────────────────────────────────────────┐
│                 unified_shutdown_hook()                          │
│                                                                  │
│  ① Calculate duration                                           │
│     duration = now - call_start_time                             │
│                                                                  │
│  ② Process booking (if intent was saved)                        │
│     │                                                            │
│     ├─ YES: call async_create_booking() → Cal.com/Google Cal    │
│     │       → notify_booking_confirmed() → Telegram + WhatsApp  │
│     │                                                            │
│     └─ NO:  notify_call_no_booking() → Telegram                 │
│             "Consider a manual follow-up call 📲"               │
│                                                                  │
│  ③ Build transcript from chat history                           │
│     agent.chat_ctx.messages → "[USER] ...\n[ASSISTANT] ..."     │
│                                                                  │
│  ④ Sentiment analysis                                           │
│     Send transcript[:800] to GPT-4o-mini:                       │
│     "Classify as: positive/neutral/negative/frustrated"          │
│                                                                  │
│  ⑤ Cost estimation                                              │
│     STT cost + LLM cost + TTS cost ≈ $0.002-0.01 per call      │
│                                                                  │
│  ⑥ Stop recording egress                                        │
│     → LiveKit stops recording → uploads .ogg to Supabase S3     │
│                                                                  │
│  ⑦ Update active_calls table → status: "completed"              │
│                                                                  │
│  ⑧ Fire n8n webhook (if configured)                             │
│     POST {phone, duration, booked, sentiment, recording_url}    │
│                                                                  │
│  ⑨ Save to Supabase call_logs table                             │
│     {phone, duration, transcript, summary, sentiment,            │
│      cost, recording_url, call_date, call_hour, was_booked,     │
│      interrupt_count, caller_name, day_of_week}                  │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## 9. The Web Dashboard (ui_server.py)

`ui_server.py` is a **FastAPI** application that provides:

| Endpoint | Purpose |
|----------|---------|
| `GET /` | Main dashboard HTML (call logs, stats, config editor, CRM) |
| `GET /demo` | Browser-based voice demo (uses LiveKit JS SDK + microphone) |
| `GET /api/config` | Read current config.json |
| `POST /api/config` | Update config.json (prompt, model, voice, API keys) |
| `GET /api/logs` | Fetch last 50 call logs from Supabase |
| `GET /api/logs/{id}/transcript` | Download full transcript as .txt |
| `GET /api/bookings` | Fetch bookings |
| `GET /api/stats` | Aggregate stats (total calls, booking rate, avg duration) |
| `GET /api/contacts` | CRM view — deduplicated contacts from call_logs |
| `POST /api/call/single` | Trigger a single outbound call |
| `POST /api/call/bulk` | Trigger bulk outbound calls |
| `GET /api/demo-token` | Generate LiveKit token for browser demo |
| `GET /metrics` | Prometheus metrics (calls total, booking rate, duration histogram) |

### How the Demo Works:

```
Browser                    LiveKit Cloud              Agent Worker
  │                            │                          │
  │ GET /api/demo-token        │                          │
  │───────────────────▶        │                          │
  │ {token, room, url}         │                          │
  │◀───────────────────        │                          │
  │                            │                          │
  │ Connect via LiveKit JS SDK │                          │
  │ + enable microphone        │                          │
  │────────────────────────────▶                          │
  │                            │  dispatch job            │
  │                            │─────────────────────────▶│
  │                            │                          │
  │                            │  agent joins room        │
  │                            │◀─────────────────────────│
  │                            │                          │
  │◀═══════════ real-time audio ═══════════════════════▶│
  │    (browser mic → STT → LLM → TTS → browser speaker) │
```

---

## 10. Configuration System — How Settings Flow

```
┌────────────────┐     ┌──────────────────┐     ┌─────────────────┐
│ .env file      │     │ config.json      │     │ configs/        │
│ (secrets)      │     │ (runtime config) │     │ per-phone .json │
│                │     │                  │     │                 │
│ API keys       │     │ agent prompt     │     │ 919876543210    │
│ Supabase creds │     │ LLM model        │     │   .json         │
│ LiveKit creds  │     │ TTS voice         │     │                 │
│ Telegram token │     │ language preset   │     │ Custom prompt   │
│ Vobiz creds    │     │ first_line        │     │ for specific    │
└───────┬────────┘     │ max_turns         │     │ caller          │
        │              └────────┬─────────┘     └────────┬────────┘
        │                       │                         │
        ▼                       ▼                         ▼
┌──────────────────────────────────────────────────────────────────┐
│                    get_live_config(phone_number)                  │
│                                                                   │
│  Priority:                                                        │
│  1. configs/{phone}.json  (per-caller config)                    │
│  2. configs/default.json  (default override)                     │
│  3. config.json           (main config)                          │
│  4. Hardcoded defaults    (gpt-4o-mini, kavya, hi-IN, etc.)     │
│                                                                   │
│  Returns merged config dict used throughout entrypoint()          │
└──────────────────────────────────────────────────────────────────┘
```

---

## 11. Key Protocols & Concepts Explained

### SIP (Session Initiation Protocol)
- **What**: The protocol for setting up, managing, and tearing down voice/video calls over the internet.
- **Think of it as**: HTTP but for phone calls. `INVITE` = start a call, `BYE` = end a call, `200 OK` = accepted.
- **In this project**: Vobiz ↔ LiveKit communicate via SIP. Your Python code never touches SIP directly — LiveKit handles it.

### WebRTC (Web Real-Time Communication)
- **What**: A browser/application protocol for real-time audio/video with very low latency.
- **Think of it as**: The actual audio pipe. SIP sets up the call, WebRTC carries the audio.
- **In this project**: LiveKit rooms use WebRTC internally. The agent joins via WebSocket (LiveKit's SDK handles the WebRTC part).

### SIP Trunk
- **What**: A virtual "phone line" between two SIP endpoints. Like a network cable for calls.
- **Inbound trunk**: Tells LiveKit which phone numbers to accept calls for.
- **Outbound trunk**: Tells LiveKit how to authenticate to Vobiz when placing calls (domain, username, password).

### Dispatch Rule
- **What**: LiveKit's routing table. Maps incoming SIP calls to agent actions.
- **Think of it as**: An `if-then` rule. "IF a call comes in on trunk X, THEN create room with prefix Y and summon agent Z."

### Worker vs Agent
- **Worker**: The OS process (`python agent.py dev`). Maintains a persistent WebSocket to LiveKit. Can handle multiple concurrent calls.
- **Agent**: The logical AI entity (personality, prompt, tools) that runs inside a Worker when a job is assigned.
- **Analogy**: Worker = an employee sitting at their desk. Agent = the role they play when a customer calls.

### Egress
- **What**: LiveKit's recording/streaming service. "Egress" = data leaving the system.
- **In this project**: Records the room audio as .ogg, uploads to Supabase S3 via the S3Upload API.

### VAD (Voice Activity Detection)
- **What**: Algorithm that detects when someone is speaking vs. when it's silence/background noise.
- **In this project**: Silero VAD helps determine when the user has finished speaking, so the AI knows when to respond.

---

## 12. File-by-File Breakdown

| File | Lines | What it does | Key functions/classes |
|------|-------|-------------|----------------------|
| `agent.py` | ~850 | Main brain — Worker, Agent, voice pipeline, tools, shutdown hooks | `entrypoint()`, `OutboundAssistant`, `AgentTools`, `unified_shutdown_hook()` |
| `ui_server.py` | ~1370 | FastAPI web dashboard with config editor, call logs, CRM, demo | `read_config()`, `/api/config`, `/api/logs`, `/api/call/single`, `/demo` |
| `calendar_tools.py` | ~260 | Cal.com + Google Calendar integration | `get_available_slots()`, `async_create_booking()`, `cancel_booking()` |
| `notify.py` | ~200 | Telegram + WhatsApp + webhook notifications | `send_telegram()`, `send_whatsapp()`, `notify_booking_confirmed()` |
| `db.py` | ~200 | Supabase client + call log persistence | `get_supabase()`, `save_call_log()` (with retry + schema fallback) |
| `make_call.py` | ~70 | CLI to trigger outbound calls | `main()` — creates agent dispatch via LiveKit API |
| `setup_trunk.py` | ~55 | One-time outbound trunk configuration | Updates SIP trunk credentials on LiveKit |
| `setup_inbound.py` | ~80 | One-time inbound trunk + dispatch rule setup | Creates inbound trunk + dispatch rule |
| `config.json` | — | Runtime config (prompt, model, voice, keys) | Read by `get_live_config()` |
| `.env` | — | Secrets (API keys, credentials) | Read by `dotenv` at startup |

---

## 13. Data Flow Diagrams (ASCII)

### Complete Inbound Call — Data Flow

```
                    PHONE NETWORK                    INTERNET
                    ─────────────                    ────────
Caller's Phone ──PSTN──▶ Vobiz ──SIP INVITE──▶ LiveKit SIP Gateway
                                                      │
                                                 Match Trunk
                                                 Match Dispatch Rule
                                                      │
                                                 Create Room
                                                 Notify Worker
                                                      │
                                              ┌───────▼────────┐
                                              │   agent.py      │
                                              │   entrypoint()  │
                                              └───────┬────────┘
                                                      │
                                    ┌─────────────────┼──────────────────┐
                                    │                 │                  │
                              Load Config      Extract Phone      Check Rate Limit
                              (config.json)    (SIP attributes)   (5 calls/hr)
                                    │                 │                  │
                                    └─────────────────┼──────────────────┘
                                                      │
                                              Build Pipeline:
                                    ┌─────────────────┼──────────────────┐
                                    │                 │                  │
                              Sarvam STT        GPT-4o-mini        Sarvam TTS
                              (16kHz)           (120 tokens)       (24kHz, kavya)
                                    │                 │                  │
                                    └─────────────────┼──────────────────┘
                                                      │
                                              Start AgentSession
                                              Start Recording (Egress → Supabase S3)
                                              Speak Greeting
                                                      │
                                              ╔═══════╧══════════╗
                                              ║  CONVERSATION    ║
                                              ║  LOOP            ║
                                              ║                  ║
                                              ║  Audio In ──────▶║──▶ Sarvam STT
                                              ║                  ║         │
                                              ║                  ║    GPT-4o-mini
                                              ║                  ║    (+ tool calls)
                                              ║                  ║         │
                                              ║  Audio Out ◀────║◀── Sarvam TTS
                                              ║                  ║
                                              ║  Events:         ║
                                              ║  • user_speech   ║──▶ Supabase
                                              ║    _committed    ║    (real-time
                                              ║                  ║     transcript)
                                              ║  • agent_speech  ║
                                              ║    _interrupted  ║──▶ interrupt
                                              ║                  ║    counter
                                              ╚═══════╤══════════╝
                                                      │
                                              CALL ENDS (hangup)
                                                      │
                                              ┌───────▼────────┐
                                              │ Shutdown Hook   │
                                              └───────┬────────┘
                                    ┌─────────────────┼──────────────────┐
                                    │                 │                  │
                              Create Booking    Sentiment        Stop Recording
                              (Cal.com)         Analysis         (Egress)
                                    │           (GPT-4o-mini)         │
                                    │                 │               │
                              Telegram +         Save to          n8n Webhook
                              WhatsApp          Supabase
```

### Data Storage Schema

```
┌─────────────────────────────────────────────────────────┐
│                    Supabase                              │
│                                                          │
│  ┌─────────────────────────────────────────────────┐    │
│  │ TABLE: call_logs                                 │    │
│  │                                                  │    │
│  │ id                    UUID (auto)                │    │
│  │ phone_number          "+91XXXXXXXXXX"            │    │
│  │ caller_name           "Rohan Chadha"             │    │
│  │ duration_seconds      180                        │    │
│  │ transcript            "[USER] Hello...\n..."     │    │
│  │ summary               "Booking Confirmed: abc"   │    │
│  │ recording_url         "https://.../.ogg"         │    │
│  │ sentiment             "positive"                 │    │
│  │ estimated_cost_usd    0.0034                     │    │
│  │ was_booked            true                       │    │
│  │ interrupt_count       2                          │    │
│  │ call_date             "2026-04-21"               │    │
│  │ call_hour             14                         │    │
│  │ call_day_of_week      "Monday"                   │    │
│  │ created_at            auto timestamp             │    │
│  └─────────────────────────────────────────────────┘    │
│                                                          │
│  ┌─────────────────────────────────────────────────┐    │
│  │ TABLE: call_transcripts (real-time stream)       │    │
│  │                                                  │    │
│  │ call_room_id          "call-abc123"              │    │
│  │ phone                 "+91XXXXXXXXXX"            │    │
│  │ role                  "user" / "assistant"       │    │
│  │ content               "Kal ka slot batao"        │    │
│  └─────────────────────────────────────────────────┘    │
│                                                          │
│  ┌─────────────────────────────────────────────────┐    │
│  │ TABLE: active_calls                              │    │
│  │                                                  │    │
│  │ room_id               "call-abc123"              │    │
│  │ phone                 "+91XXXXXXXXXX"            │    │
│  │ caller_name           "Rohan"                    │    │
│  │ status                "active" / "completed"     │    │
│  │ last_updated          ISO timestamp              │    │
│  └─────────────────────────────────────────────────┘    │
│                                                          │
│  ┌─────────────────────────────────────────────────┐    │
│  │ STORAGE: call-recordings (S3 bucket)             │    │
│  │                                                  │    │
│  │ recordings/call-abc123.ogg                       │    │
│  └─────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────┘
```

---

## 14. Common Interview Questions & Answers

### Q: "Walk me through what happens when someone calls your number."
**A**: Use Section 4 above. Hit these points: PSTN → Vobiz → SIP INVITE → LiveKit SIP Gateway → Inbound Trunk match → Dispatch Rule fires → Room created → Worker notified → entrypoint() runs → Agent joins room → STT/LLM/TTS pipeline starts → real-time conversation → post-call hooks fire.

### Q: "How do you handle latency in voice AI?"
**A**: Multiple strategies:
- **Streaming everything**: STT, LLM, and TTS all stream (don't wait for complete output)
- **Token cap**: LLM limited to 120 tokens per response — keeps replies short
- **Min endpointing delay**: 0.05s — agent responds quickly after user stops
- **16kHz STT / 24kHz TTS**: Optimized audio quality vs bandwidth
- **Sentence chunking**: Only the first sentence is spoken, rest is discarded
- **TTS pre-warming**: TTS engine is warmed up at session start
- **India South region**: Worker registered in closest region to reduce RTT

### Q: "How does the AI know when to stop listening and start responding?"
**A**: **Turn detection** using STT-based endpointing. The `min_endpointing_delay` (0.05s) defines how long to wait after the last detected speech before treating it as "turn complete." Silero VAD assists by detecting speech boundaries. Filler words ("ok", "hmm", "haan") are filtered out so they don't trigger responses.

### Q: "How do you handle the AI hearing its own voice?"
**A**: An `agent_is_speaking` global flag. When `agent_speech_started` fires, the flag is set to True. Any `user_speech_committed` events that arrive while the flag is True are dropped (echo filtering). The flag resets on `agent_speech_finished`.

### Q: "What happens if the call drops unexpectedly?"
**A**: The `participant_disconnected` event fires → triggers `unified_shutdown_hook()` → all post-call processing (booking, logging, notifications) still happens. The LiveKit Worker also has auto-reconnect — if the WebSocket to LiveKit drops, it reconnects within seconds.

### Q: "How do you support multiple languages?"
**A**: Three-layer approach:
1. **STT**: Sarvam Saaras v3 with `language="unknown"` (auto-detect mode) recognizes 11 Indian languages
2. **LLM**: System prompt includes `[LANGUAGE DIRECTIVE]` telling GPT-4o-mini to match the caller's language
3. **TTS**: Sarvam Bulbul v3 with appropriate voice (kavya for Hindi, dev for English, priya for Tamil, etc.)

The `lang_preset` config switches between `multilingual` (auto-detect), `hinglish`, `hindi`, `english`, `tamil`, etc.

### Q: "Why is the booking done post-call instead of during the call?"
**A**: Three reasons:
1. **Latency**: Making a Cal.com API call mid-conversation adds 1-2s delay
2. **Caller might change mind**: They might pick a slot then say "actually, never mind"
3. **Reliability**: If the call drops, the shutdown hook still processes the intent

### Q: "How would you scale this to handle 1000 concurrent calls?"
**A**: 
- **Horizontal scaling**: Run multiple `agent.py` workers (LiveKit load-balances jobs across them)
- **Each worker gets a unique ID** (`AW_xxx`) — LiveKit tracks availability
- **Containerize with Docker** (Dockerfile already exists)
- **Deploy on Coolify/K8s** (COOLIFY_DEPLOYMENT.md exists)
- **Supervisord** manages processes (supervisord.conf exists)
- **Prometheus metrics** for monitoring (`/metrics` endpoint)

### Q: "What's the cost per call?"
**A**: The `estimate_cost()` function calculates:
- STT: ~$0.002/min (Sarvam)
- LLM: ~$0.006/min (GPT-4o-mini at 120 tokens/turn)
- TTS: ~$0.003/1000 chars (Sarvam)
- Total: **~$0.01-0.02 for a 3-minute call**

### Q: "What security measures are in place?"
**A**:
- **Rate limiting**: 5 calls per phone number per hour (in-memory)
- **SIP trunk authentication**: Username/password for Vobiz
- **LiveKit API key/secret**: All API calls are authenticated
- **Supabase Row Level Security**: Database access is controlled
- **SSL/TLS**: All connections are encrypted
- **Sentry**: Error tracking in production
- **Max turns**: Auto-wraps up after 25 turns (prevents abuse)

---

## Appendix: The Technology Stack

| Layer | Technology | Why chosen |
|-------|-----------|-----------|
| Real-time infra | LiveKit Cloud | Open-source WebRTC + SIP + agent framework |
| Phone gateway | Vobiz | Indian SIP trunk provider with DIDs |
| STT | Sarvam Saaras v3 | Best-in-class for Indian languages |
| LLM | OpenAI GPT-4o-mini | Fast, cheap, good at function calling |
| TTS | Sarvam Bulbul v3 | Natural Indian language voices |
| VAD | Silero | Lightweight, accurate voice activity detection |
| Database | Supabase (PostgreSQL) | Free tier, real-time, S3-compatible storage |
| Calendar | Cal.com / Google Calendar | Open-source scheduling with API |
| Notifications | Telegram Bot + Twilio WhatsApp | Instant alerts to business owner + caller |
| Dashboard | FastAPI + vanilla HTML/JS | Simple, no build step, serves from one file |
| Monitoring | Prometheus + Sentry + OpenTelemetry | Production observability |
| Deployment | Docker + Coolify + Supervisord | Self-hosted, managed restarts |
