# Voice AI Customer Agent

An AI-powered voice assistant for customer support: speech-to-text, text-to-speech,
conversational memory, and tool calling — wired up to real phone calls via Twilio,
with a FastAPI backend and a LangGraph agent brain.



---
![Project Home](images/va1.png)
---

---
![Project Home](images/va2.png)
---

---
![Project Home](images/va3.png)
---

```

Caller ──(phone call)──> Twilio ──(Media Stream WS)──> FastAPI
                                                          │
                                                          ▼
                                              ┌─── STT (Whisper) ───┐
                                              │                     │
                                     LangGraph Agent (LLM + Tools + Memory)
                                              │                     │
                                              └─── TTS (OpenAI) ────┘
                                                          │
                                                          ▼
Caller <──(audio back)── Twilio <──(base64 μ-law)── FastAPI
```

## Features

- **Speech-to-Text**: streaming-friendly buffered transcription via OpenAI Whisper
- **Text-to-Speech**: OpenAI TTS (swap in ElevenLabs by implementing `TTSProvider`)
- **Conversational memory**: LangGraph `MemorySaver` checkpointer, keyed per call/session
- **Tool calling**: order lookup, FAQ/policy search, human escalation, callback scheduling
- **Telephony**: real Twilio phone call handling via Media Streams (real-time, bidirectional audio)
- **Text/chat API**: test the agent brain over plain HTTP without any audio infra
- **Browser test client**: record your mic, talk to the agent, hear it reply — no phone needed

## Project layout

```
app/
  main.py               FastAPI app + router wiring
  config.py              Settings (env vars)
  models/schemas.py      Pydantic request/response models
  agent/
    graph.py             LangGraph state graph: agent node + tool node + memory
    tools.py             Callable tools the LLM can invoke
    prompts.py           System prompt
  services/
    stt.py                Speech-to-text provider (OpenAI Whisper)
    tts.py                Text-to-speech provider (OpenAI TTS)
    memory_store.py       Mock "backend" data (orders, FAQs) the tools query
  telephony/
    twilio_routes.py      /telephony/incoming-call (TwiML) + /telephony/media-stream (WS)
    audio_utils.py        μ-law/PCM conversion, chunking, simple silence-based VAD
  routes/
    chat_routes.py        POST /chat  (text in/out, for quick testing)
    voice_routes.py        POST /voice/chat (audio in -> audio out, for browser client)
static/
  index.html              Minimal browser mic-to-agent test page
scripts/
  demo_cli.py             Terminal chat loop against the agent (text only)
tests/
  test_agent.py           Basic tool + graph sanity tests
```

## Setup

```bash
python -m venv venv
source venv/bin/activate         # Windows: venv\Scripts\activate
pip install -r requirements.txt
cp .env.example .env
# edit .env and add your keys
```

### Required environment variables (`.env`)

| Variable | Purpose |
|---|---|
| `OPENAI_API_KEY` | LLM (agent brain), Whisper STT, OpenAI TTS |
| `TWILIO_ACCOUNT_SID` | Only needed for real phone calls |
| `TWILIO_AUTH_TOKEN` | Only needed for real phone calls |
| `PUBLIC_BASE_URL` | Public HTTPS URL Twilio can reach (e.g. an ngrok URL) |

You can run and fully test the agent (text + browser mic) with **only** `OPENAI_API_KEY` set.
Twilio is only needed for real phone calls.

## Running it

```bash
uvicorn app.main:app --reload --port 8000
```

Open `http://localhost:8000/static/index.html` to talk to the agent from your browser mic,
or:

```bash
curl -X POST http://localhost:8000/chat \
  -H "Content-Type: application/json" \
  -d '{"session_id": "demo-1", "message": "Where is my order 1001?"}'
```

Or run the terminal demo:

```bash
python scripts/demo_cli.py
```

## Wiring up a real phone number (Twilio)

1. Expose your local server: `ngrok http 8000`, copy the `https://...ngrok...` URL into
   `PUBLIC_BASE_URL` in `.env`.
2. In the Twilio console, buy/select a phone number → Voice & Fax → "A call comes in" →
   Webhook → `https://<PUBLIC_BASE_URL>/telephony/incoming-call` (HTTP POST).
3. Call the number. Twilio opens a Media Stream WebSocket back to
   `/telephony/media-stream`; audio is buffered, transcribed on pauses, run through the
   LangGraph agent, synthesized, and streamed back — a live phone conversation with the agent.

## How the agent works (LangGraph)

`app/agent/graph.py` builds a small state graph:

```
START -> agent (LLM decides: answer directly OR call a tool) -> [tool? -> agent] -> END
```

- State carries the running message list (`add_messages` reducer) so context persists.
- `MemorySaver` checkpoints state per `thread_id` (== call/session id), giving the agent
  memory across turns of the same conversation without any extra plumbing.
- Tools are plain Python functions decorated with `@tool`; the LLM sees their docstrings
  as its "API docs" and decides when to call them (native OpenAI tool calling under the hood).

## Extending it

- **Swap the LLM**: change the model string in `app/config.py` (works with any
  `langchain-openai`/`langchain-anthropic` chat model).
- **Swap TTS provider**: implement `TTSProvider.synthesize()` in `app/services/tts.py`
  for ElevenLabs/Azure/etc., point `get_tts_provider()` at it.
- **Add tools**: add a new `@tool`-decorated function in `app/agent/tools.py` and add it
  to the `TOOLS` list — the agent picks it up automatically.
- **Persistent memory**: swap `MemorySaver()` for `SqliteSaver`/`PostgresSaver` in
  `graph.py` to survive process restarts.

## Notes on scope

This is a portfolio-grade reference implementation, not a production system. For
production you'd add: authentication on the API, proper VAD (e.g. `webrtcvad` or Silero
VAD) instead of the simple energy-threshold VAD here, call recording/consent handling,
rate limiting, structured logging/tracing, and a real ticketing/CRM integration behind
the tools instead of the in-memory mock data in `memory_store.py`.
