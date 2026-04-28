# Architecture

## System Overview

MacBuddy is a five-service Python stack with a Tauri 2 desktop UI. Each service runs as an independent process, coordinated through a single `start.sh` launcher. Inter-service communication happens over local HTTP and WebSocket.

```
                 +-----------------------+
                 |  iPhone (input layer) |
                 +-----------------------+
                  |        |          |
              iMessage   FaceTime   ASI:One
                  |        |          |
        +---------v---+ +--v-------+ +v-------------+
        | Bridge      | | Voice    | | Agentverse   |
        | (chat.db    | | Daemon   | | Wrapper      |
        |  poller)    | | (STT/TTS)| | (uAgent)     |
        +-----+-------+ +----+-----+ +-----+--------+
              |              |             |
              +--------------+-------------+
                             |
                       +-----v-----+
                       |  Router   |   <- Gemma 4 classifier
                       | (FastAPI) |
                       +-----+-----+
                             |
              +--------------+-------------+
              |                            |
        +-----v-------+              +-----v---------+
        | FaceTime    |              | Orbit Lane    |
        | Lane        |              | (Google APIs) |
        | Claude 4.6  |              | + Screenshot  |
        | Computer    |              | + File Send   |
        | Use         |              |               |
        +-------------+              +---------------+
              |                            |
              +--------------+-------------+
                             |
                       +-----v-----+
                       | ElevenLabs|
                       |    TTS    |
                       +-----+-----+
                             |
                +------------v------------+
                | Output: text + voice    |
                | back through any input  |
                | channel                 |
                +-------------------------+
```

## Service Responsibilities

### Router (Port 8000)

The central coordinator. Every command flows through here.

- **Endpoints:**
  - `POST /text` — text command from any source
  - `POST /voice` — voice transcript from the daemon
  - `WebSocket /events` — live event stream for the UI
  - `GET /ui` — serves the Tauri-embedded dashboard
- **Classifier:** Local Gemma 4 via Ollama with regex fallback. Decides whether a command goes to the FaceTime, Orbit, or Agentverse lane.
- **Dispatcher:** Forwards to the chosen lane and waits for a response.
- **Event bus:** Broadcasts `transcript`, `dispatch`, `response`, `error` events to the UI.

### FaceTime Lane (Port 8003)

GUI control. Uses Claude Sonnet 4.6 with the `computer_20251124` tool.

- Takes a screenshot
- Sends to Claude with the user instruction
- Claude returns an action (click, type, scroll, etc.)
- An executor module runs the action via pyautogui
- Loop continues for up to 12 iterations or until the task completes

### Orbit Lane (Port 8001)

Structured Google Workspace operations. Uses OAuth (PKCE disabled for confidential web clients).

- **Drive:** search, list recent, read files
- **Gmail:** send, list unread, search, summarize
- **Calendar:** list events, create events, check availability
- **Files:** delivers from Drive or local Mac to email or iMessage
- **Screenshot:** captures and emails the Mac screen

### Voice Daemon (Port 8004)

Real-time STT/TTS over FaceTime audio.

- Captures audio via sounddevice
- Transcribes with Groq Whisper-large-v3-turbo
- Forwards transcript to the Router
- Synthesizes Router's reply with ElevenLabs Turbo v2.5
- Plays reply back through the FaceTime call

### iMessage Bridge

Polls `~/Library/Messages/chat.db` every 2 seconds.

- Deduplicates messages via processed-rowid set
- Forwards to Router
- Sends replies via AppleScript (`tell application "Messages"`)
- Optionally speaks replies aloud through Mac speakers via ElevenLabs

### Agentverse Wrapper (Port 8005)

Exposes the system on Fetch.ai's Agentverse marketplace as a uAgent.

- Implements the standard `chat_protocol_spec`
- Direct-endpoint mode via ngrok tunnel
- Two-layer allowlist (sender + action) before any execution
- Forwards approved commands to the Router

## Data Flow

### iMessage command path

```
iPhone Messages
  -> chat.db (SQLite)
  -> Bridge (poll)
  -> Router (POST /text)
  -> Classifier (Gemma 4)
  -> Lane (FaceTime or Orbit)
  -> Result
  -> Router
  -> Bridge
  -> AppleScript -> iMessage reply
  -> ElevenLabs -> Mac speaker (optional)
```

### Voice command path (FaceTime)

```
iPhone FaceTime call
  -> Mac mic
  -> Voice Daemon
  -> Whisper transcribe
  -> Router (POST /voice)
  -> Classifier
  -> Lane
  -> Result
  -> Router
  -> Voice Daemon
  -> ElevenLabs
  -> FaceTime audio output
```

### ASI:One command path

```
ASI:One chat
  -> Agentverse routing
  -> ngrok tunnel
  -> Agentverse Wrapper
  -> Sender allowlist check
  -> Action allowlist check
  -> Router (POST /text)
  -> Lane
  -> Result
  -> Wrapper
  -> ChatMessage reply
  -> Agentverse
  -> ASI:One chat
```

## UI Architecture

The dashboard is a Tauri 2 native window with a frameless, transparent shell and an animated mint conic-gradient glow border.

- **Connection:** WebSocket to `ws://localhost:8000/events`
- **Display states:** Idle / Thinking / Executing / Done
- **Visible content:** "You said: ..." and "MacBuddy: ..." only — internal routing chatter stays in terminal logs
- **Auto-trim:** keeps the most recent 3 turns in view

## Environment Configuration

Each service has its own `.env` file. See `.env.example` for the full list of configurable values, including:

- API keys (Anthropic, Groq, ElevenLabs, Agentverse, Google OAuth)
- Allowlist values (sender addresses, allowed phone numbers)
- Behavior toggles (SPEAK_REPLIES, ANTHROPIC_MAX_ITERATIONS, BACKEND)
- Service URLs (router, voice daemon, ngrok endpoint)
