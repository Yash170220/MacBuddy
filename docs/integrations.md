# Integrations

MacBuddy connects to a number of external services. Each integration is
listed here with its purpose, scope, and the credential type required.

## AI / LLM Services

### Anthropic Claude

- **Model:** `claude-sonnet-4-6`
- **Tool:** `computer_20251124` (Computer Use)
- **Purpose:** Multi-step GUI control of the Mac
- **Where:** FaceTime lane
- **Credentials:** API key

Sonnet 4.6 was selected for its leading OSWorld benchmark score, which
translates to reliable multi-step task execution.

### Google Gemma

- **Model:** `gemma3:4b` via Ollama
- **Purpose:** Local intent classification (which lane handles a request)
- **Where:** Router
- **Credentials:** None - runs locally
- **Fallback:** Regex keyword matcher if Ollama is unreachable

### Groq

- **Models:** `whisper-large-v3-turbo`, `llama-3.1-8b-instant`,
  `llama-4-scout-17b`
- **Purpose:** Fast STT (Whisper), structured intent parsing in JSON mode
  (Llama 3.1), optional vision backend (Llama 4 Scout)
- **Where:** Voice daemon, Orbit lane intent parser
- **Credentials:** API key

### ElevenLabs

- **Model:** `eleven_turbo_v2_5`
- **Voice:** Rachel (`21m00Tcm4TlvDq8ikWAM`) by default
- **Purpose:** Text-to-speech for FaceTime replies and Mac speaker output
- **Where:** Voice daemon, iMessage bridge
- **Credentials:** API key

## Workspace APIs

### Google Drive

- **Scopes:** `drive.readonly`, `drive.metadata.readonly`
- **Operations:** search, list recent, read file content, download
- **Where:** Orbit lane

### Gmail

- **Scopes:** `gmail.send`, `gmail.readonly`, `gmail.compose`
- **Operations:** send messages with attachments, list unread, search,
  summarize inbox
- **Where:** Orbit lane

### Google Calendar

- **Scopes:** `calendar.readonly`, `calendar.events`
- **Operations:** list events, create events, check availability
- **Where:** Orbit lane

OAuth flow uses standard authorization code with PKCE disabled for
confidential web client compatibility.

## Agent Ecosystem

### Fetch.ai Agentverse

- **Library:** `uagents` + `uagents-core`
- **Protocol:** `chat_protocol_spec` from
  `uagents_core.contrib.protocols.chat`
- **Mode:** Direct endpoint (via ngrok tunnel)
- **Manifest:** Published with `publish_manifest=True`
- **Where:** Agentverse wrapper
- **Credentials:** API key + agent seed phrase

The agent is registered with Innovation Lab tag and discoverable through
ASI:One.

### ASI:One

- Discovery is automatic once the agent is registered + Active on Agentverse
- No additional configuration on the MacBuddy side
- Users invoke it from `https://asi1.ai` chat

## Storage

### MongoDB Atlas

- **Purpose:** Audit log of every command + response
- **Collections:** `interactions` (timestamp, sender, text, lane, action,
  result)
- **Where:** Router
- **Credentials:** Connection URI
- **Optional:** if `MONGO_URI` is unset, logs go to stdout instead

## Tunneling

### ngrok

- **Purpose:** Public HTTPS tunnel to local Agentverse wrapper port
- **Where:** External tunnel for `agentverse_wrapper.py:8005`
- **Credentials:** Authtoken

## macOS System Integrations

### iMessage / chat.db

- Direct SQLite read access to `~/Library/Messages/chat.db`
- Read-only access; sends use AppleScript instead
- Requires Full Disk Access permission

### AppleScript

- `tell application "Messages"` for sending
- `tell application "FaceTime"` for call initiation
- No user-supplied strings ever reach AppleScript - all parameters are
  validated and quoted

### macOS Native Tools

- `screencapture -x` for screenshots
- `mdfind` for Spotlight file search
- `afplay` for audio playback
- `say` as fallback TTS

## UI

### Tauri 2

- Native frameless transparent window
- Embedded WebView pointing at `http://localhost:8000/ui`
- WebSocket connection to `ws://localhost:8000/events`
- Requires Rust toolchain
