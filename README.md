# MacBuddy

![tag:innovationlab](https://img.shields.io/badge/innovationlab-3D8BD3)
![Claude](https://img.shields.io/badge/Claude_Sonnet_4.6-D97757?logo=anthropic&logoColor=white)
![uAgents](https://img.shields.io/badge/uAgents-Fetch.ai-3D8BD3)
![ElevenLabs](https://img.shields.io/badge/ElevenLabs-TTS-000)
![Gemma](https://img.shields.io/badge/Gemma_4-Google-4285F4)
![Python](https://img.shields.io/badge/python-3.12+-blue)
![License](https://img.shields.io/badge/license-Proprietary-red)

> Voice and text agent for your Mac. Send a message from your phone or call your Mac via FaceTime to control your computer with natural language.

## Demo

[![MacBuddy Demo](https://img.youtube.com/vi/8nKc-9fVM6w/maxresdefault.jpg)](https://youtu.be/8nKc-9fVM6w)

**Watch the full demo:** [https://youtu.be/8nKc-9fVM6w](https://youtu.be/8nKc-9fVM6w)

## What it does

MacBuddy turns your iPhone into a remote for your Mac. You can:

- **Text it** via iMessage — "open VS Code and add a comment to main.py"
- **Call it** via FaceTime — speak naturally, hear it speak back
- **Talk to it through ASI:One** — it's a registered agent on Fetch.ai's Agentverse marketplace
- Watch it execute multi-step GUI tasks in real time on a live dashboard
- Get files emailed from your Mac or Google Drive without touching a keyboard

Every reply comes back as text, voice through FaceTime, or audio through your Mac's speakers — the channel matches the input.

## How it works

```
iPhone (iMessage / FaceTime / ASI:One)
        |
        v
  [iMessage Bridge]   [Voice Daemon]   [Agentverse Wrapper]
        |                   |                   |
        +-------------------+-------------------+
                            |
                       [ Router ]  <-- Gemma 4 classifier (local Ollama)
                            |
        +-------------------+-------------------+
        |                                       |
   [FaceTime Lane]                       [Orbit Lane]
   Claude Sonnet 4.6                     Google Workspace
   Computer Use                          Drive / Gmail / Calendar
   (multi-step GUI)                      File delivery
                                         Screenshot capture
                            |
                            v
              [ ElevenLabs TTS ] -> back to user
```

For full architecture details, see [ARCHITECTURE.md](./ARCHITECTURE.md).

## Tech stack

| Component | Purpose | Tech |
|-----------|---------|------|
| Router | Classifies intent, dispatches to lane | Gemma 4 + FastAPI |
| FaceTime Lane | Multi-step Mac GUI control | Claude Sonnet 4.6 Computer Use |
| Orbit Lane | Structured Google Workspace API calls | Drive / Gmail / Calendar |
| Voice Daemon | Real-time STT + TTS over FaceTime audio | Groq Whisper + ElevenLabs |
| iMessage Bridge | Reads chat.db, replies, plays voice | sqlite3 + AppleScript |
| Agentverse Wrapper | Exposes the system as a uAgent | Fetch.ai uAgents Chat Protocol |
| UI Dashboard | Live event feed (Tauri native window) | HTML/CSS + WebSocket |

## Inputs / Outputs

### Chat Protocol (Agentverse / ASI:One)

**Input** - `ChatMessage` with one or more `TextContent` blocks containing natural language commands like:

- "Open Brave and go to YouTube"
- "What's on my calendar today?"
- "Send my resume from my Mac to alice@example.com"
- "Take a screenshot and email it to me"

**Output** - `ChatMessage` with `TextContent` confirming the action plus `EndSessionContent` to close the session.

## Security model

Two-layer allowlist before any execution:

1. **Sender allowlist** - only addresses in `ALLOWED_SENDERS` (env var) can issue commands
2. **Action allowlist** - 12 curated verbs (open_app, browse_url, send_email, screenshot, etc.). Anything else gets a list of supported actions and is not executed

No raw shell execution. No subprocess call ever takes user-supplied strings as arguments.

See [SECURITY.md](./SECURITY.md) for full details.

## Demo prompts

| Try saying | What happens |
|------------|--------------|
| "Open Brave and search YouTube" | Spotlight launches Brave, navigates to youtube.com |
| "What's on my calendar today" | Pulls events from Google Calendar, replies with summary |
| "Send asi.png from my Mac to friend@example.com" | mdfind locates the file, attaches it to a Gmail send |
| "Take a screenshot and email it to me" | screencapture -> email PNG to your authed Gmail |
| "Open Calculator and compute 2 plus 2" | Multi-step: opens Calculator, types operation, reads result |

## Sponsor / Track alignment

- **[Fetch.ai Innovation Lab](https://fetch.ai/innovation-lab)** - registered on Agentverse, ASI:One discoverable, Chat Protocol compliant
- **Anthropic** - Claude Sonnet 4.6 Computer Use as the GUI brain
- **ElevenLabs** - all Mac voice replies use ElevenLabs Turbo v2.5
- **Google Gemma** - local Gemma 4 powers the intent classifier

## Source code access

This is the public-facing repository. The full implementation is kept in a private repository for security reasons (iMessage and Mac control logic).

For evaluation access (judges, recruiters, etc.), please reach out through the contact info on the project's [Devpost page](https://devpost.com/software/macbuddy).

## Acknowledgments


Built at LA Hacks 2026.
