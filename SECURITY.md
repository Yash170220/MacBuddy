# Security Model

MacBuddy executes real commands on a real Mac. This document describes the
defense-in-depth measures we implemented to make the system safe to expose to
public agent ecosystems like Agentverse.

## Threat Model

The system has three input channels:

1. **iMessage** - locked to specific phone numbers/Apple IDs via `ALLOWED_SENDERS`
2. **FaceTime voice** - implicit allowlist via FaceTime contact graph
3. **Agentverse / ASI:One** - public agent ecosystem, anyone can attempt to
   send a `ChatMessage` to the agent's address

The third channel is the highest-risk surface. The other two are mitigated by
Apple's identity model. Our defenses focus primarily on the Agentverse
channel.

## Layer 1 - Sender Allowlist

Every `ChatMessage` received by the Agentverse wrapper is checked against
`ALLOWED_SENDERS` before any processing.

```
ALLOWED_SENDERS=agent1q...,agent1q...
```

If the sender is not on the list, the request is logged and a polite refusal
is sent back. The local Concierge router is never called. The action
allowlist is never checked. No state is mutated.

This is implemented as the first thing inside the message handler, before
text parsing, classification, or dispatch.

## Layer 2 - Action Allowlist

Even authorized senders are restricted to a curated set of safe verbs:

| Action | Triggers |
|--------|----------|
| `open_app` | "open ", "launch ", "start " |
| `close_app` | "close ", "quit " |
| `browse_url` | "go to ", "open url", "navigate to " |
| `search` | "search ", "look up ", "google " |
| `send_email` | "email ", "send email", "send a mail" |
| `send_imessage` | "text ", "imessage ", "send a message" |
| `drive_file` | "from drive", "from google drive" |
| `mac_file` | "from mac", "from my mac" |
| `calendar` | "calendar", "what's on my", "schedule" |
| `ask_status` | "status", "are you", "hello", "hi" |
| `type_text` | "type ", "write " |
| `screenshot` | "screenshot", "capture screen" |

Any request that doesn't match a verb prefix gets a "supported actions" reply
without ever calling the executor or Claude.

The action allowlist is intentionally broader than the sender allowlist. The
sender allowlist is the hard security boundary; the action allowlist is a
secondary safety net that limits damage from any sender that does pass.

## Layer 3 - No Raw Shell Execution

There is **no subprocess call that takes a user-supplied string as an
argument** anywhere in the codebase. All shell-like operations use:

- Fixed argument arrays for `subprocess.run` (e.g., `["screencapture", "-x", path]`)
- Path validation via `os.path.exists` and `mdfind` queries with hardcoded
  flags
- AppleScript templates with parameterized values, not string concatenation
- Environment variables for credentials, never inlined

## Layer 4 - Capability Scoping

Each lane has limited capability:

- **FaceTime lane** — pyautogui actions only. No file system writes outside
  of `/tmp`. No network calls except to Anthropic.
- **Orbit lane** — only Google APIs the user has OAuth-granted. No arbitrary
  HTTP. File reads scoped to Drive results or `mdfind` matches.
- **Voice daemon** — audio I/O only. No file system access except to write
  TTS output to `/tmp`.

## Layer 5 - Request Logging

Every sender + action attempt is logged to MongoDB Atlas (or stdout if
`MONGO_URI` is unset) with:

- timestamp
- sender address
- raw input text
- classified lane
- action taken
- result status
- response text

This provides an audit trail for any incident review.

## What an Attacker Can and Can't Do

**Can do (with valid sender allowlist entry):**
- Trigger any of the 12 allowlisted actions
- Open or close any Mac application
- Read public web pages
- Read the user's Google Workspace data via the connected OAuth scopes
- Cause the Mac to play audio at the user's set volume

**Can't do:**
- Run arbitrary shell commands
- Read or write arbitrary files outside the orbit lane's scoped paths
- Modify system settings, install software, or change passwords
- Make payments (no Payment Protocol integration)
- Persist state across sessions outside the audit log
- Access camera, location, contacts, or any Apple system service not
  explicitly wired

## Recommended Production Hardening

The current config is hackathon-grade. For real deployment:

1. **Generate a fresh seed** per deployment, never reuse the example seed
2. **Set `ALLOWED_SENDERS`** to the user's specific ASI:One agent address
3. **Run behind a managed reverse proxy** with rate limiting, not raw ngrok
4. **Add Anthropic budget caps** to prevent runaway Claude iteration costs
5. **Rotate all API keys** after any logs or screenshots are shared
6. **Enable MongoDB write** so the audit log persists
