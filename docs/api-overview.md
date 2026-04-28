# API Overview

This document describes the HTTP and WebSocket surface area of MacBuddy.
All endpoints are local to the host running the stack.

## Router (port 8000)

### `POST /text`

Submit a text command for processing.

**Form fields:**
- `message` — natural language command
- `sender` — identifier of the calling channel (phone number, agent
  address, or "ui")

**Response:**
```json
{
  "ok": true,
  "result": {
    "lane": "facetime",
    "response": "Opened Calculator.",
    "data": { ... }
  }
}
```

### `POST /voice`

Submit a voice transcript. Same shape as `/text`, used by the voice daemon.

### `WebSocket /events`

Live event stream consumed by the UI.

Event types:
- `transcript` — user input received
- `classify` — classifier decided which lane
- `dispatch` — request sent to lane
- `action` — lane is performing a step
- `response` — final reply ready
- `error` — failure during processing

### `GET /ui`

Serves the embedded dashboard HTML.

## FaceTime Lane (port 8003)

### `POST /execute`

Run a multi-step GUI task using Claude Computer Use.

**JSON body:**
- `instruction` — plain English task

Returns the lane's final response after Claude completes the task or hits
the iteration limit.

## Orbit Lane (port 8001)

### `POST /run`

Run a Google Workspace operation.

**JSON body:**
- `instruction` — plain English request
- `sender` — identifier of the requester (used for iMessage delivery)

Returns the result of the parsed service call.

### `GET /oauth/callback`

Standard OAuth 2.0 callback for the Google integration setup flow.

## Voice Daemon (port 8004)

### `POST /start`

Begin listening on the configured input device. Used after a FaceTime call
is established.

### `POST /stop`

Stop listening. Used to pause the daemon during silent demos or to break
feedback loops.

### `POST /speak`

Synthesize and play a text reply through the configured output device.

**JSON body:**
- `text` — what to say
- `voice_id` — optional ElevenLabs voice override

## Agentverse Wrapper (port 8005)

### `POST /submit`

uAgents-managed endpoint for incoming `ChatMessage` envelopes from
Agentverse / ASI:One. Not called directly by user code.

## Internal helpers

- `services/files.send_from_drive` — find a Drive file by name, deliver
- `services/files.send_from_mac` — find a local file, deliver
- `services/screenshot.capture_and_email` — capture and email screen
- `services/delivery.send_via_email` — Gmail attachment send
- `services/delivery.send_via_imessage` — iMessage attachment send
