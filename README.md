# Puter AI Proxy

An Anthropic-compatible proxy server that routes requests through the **Puter.js** AI service using the **GLM-5.1** model. This allows tools that speak the Anthropic Messages API protocol (such as Claude Code) to use Puter's free AI infrastructure instead of Anthropic's paid API.

## How It Works

```
Client (Claude Code / Browser)  ──▶  Express Proxy (/v1/messages)  ──▶  Puter.js SDK  ──▶  Puter Cloud AI (GLM-5.1)
```

The proxy exposes an Anthropic-compatible `POST /v1/messages` endpoint. Incoming Anthropic-format requests are translated into Puter.js SDK calls, and the responses are converted back into Anthropic's expected format — including streaming support via Server-Sent Events (SSE).

## Features

- **Anthropic Messages API compatibility** — Drop-in replacement for the `/v1/messages` endpoint
- **Streaming support** — Real-time token-by-token output via SSE (`content_block_delta` events)
- **Non-streaming mode** — Full JSON response for simple request/response workflows
- **Multi-turn conversations** — Maintains full message history across user/assistant turns
- **Browser chat UI** — Built-in dark-themed chat interface for interactive testing
- **Zero auth configuration** — Works with a dummy auth token; Puter.js handles authentication internally

## Prerequisites

- **Node.js** 18+ (ES modules are used via `"type": "module"`)
- **npm** 8+

## Installation

```bash
git clone https://github.com/JaveedIshaq/puter-ai-proxy.git
cd puter-ai-proxy
npm install
```

## Usage

### Starting the Proxy Server

```bash
node puter-proxy.js
```

By default, the server starts on port **3000**. You can override this with the `PORT` environment variable:

```bash
PORT=8080 node puter-proxy.js
```

On startup, the server prints:
```
✅ Puter proxy running on http://localhost:3000

Configure Claude Code with:
  export ANTHROPIC_BASE_URL="http://localhost:3000"
  export ANTHROPIC_AUTH_TOKEN="dummy"

Then run `claude` normally. All requests will route through Puter.js GLM-5.1
```

### Using with Claude Code

Set the environment variables to redirect Claude Code's API calls through the proxy:

```bash
export ANTHROPIC_BASE_URL="http://localhost:3000"
export ANTHROPIC_AUTH_TOKEN="dummy"
claude
```

All messages will now be routed through Puter's GLM-5.1 model instead of Anthropic's Claude models.

### Using the Browser Chat UI

Open `index.html` in any modern browser. The chat interface connects to `http://localhost:3000/v1/messages` and sends messages in streaming mode. Type your message and press **Enter** or click **Send**.

### Making API Requests Directly

You can send requests to the proxy using `curl` or any HTTP client:

**Streaming:**

```bash
curl -X POST http://localhost:3000/v1/messages \
  -H "Content-Type: application/json" \
  -d '{
    "messages": [{"role": "user", "content": "Hello, how are you?"}],
    "stream": true
  }'
```

**Non-streaming:**

```bash
curl -X POST http://localhost:3000/v1/messages \
  -H "Content-Type: application/json" \
  -d '{
    "messages": [{"role": "user", "content": "Hello, how are you?"}],
    "stream": false
  }'
```

## API Reference

### `POST /v1/messages`

Anthropic Messages API-compatible endpoint.

#### Request Body

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `messages` | `array` | Yes | Array of message objects with `role` (`user`/`assistant`) and `content` (string) |
| `stream` | `boolean` | No | Enable streaming SSE response (default: `false`) |
| `model` | `string` | No | Ignored — always uses GLM-5.1 |

#### Streaming Response Format

When `stream: true`, the response uses Server-Sent Events with two event types:

**`content_block_delta`** — Emitted for each text chunk:

```json
{
  "type": "content_block_delta",
  "index": 0,
  "delta": {
    "type": "text_delta",
    "text": "chunk of text"
  }
}
```

**`message_stop`** — Emitted once at the end with the full message:

```json
{
  "type": "message_stop",
  "message": {
    "id": "msg_puterglm_1712841600000",
    "type": "message",
    "role": "assistant",
    "content": [{ "type": "text", "text": "full response text" }],
    "model": "glm-5.1",
    "stop_reason": "end_turn",
    "usage": {
      "input_tokens": 42,
      "output_tokens": 128
    }
  }
}
```

#### Non-Streaming Response Format

When `stream: false`, returns a single JSON response:

```json
{
  "id": "msg_puterglm_1712841600000",
  "type": "message",
  "role": "assistant",
  "content": [{ "type": "text", "text": "response text" }],
  "model": "glm-5.1",
  "stop_reason": "end_turn",
  "usage": {
    "input_tokens": 42,
    "output_tokens": 128
  }
}
```

> **Note:** Token usage counts are estimates based on character length divided by 4, not actual BPE tokenization.

## Project Structure

```
puter/
├── puter-proxy.js      # Express proxy server (main entry point)
├── index.html          # Browser chat UI for testing
├── package.json        # Project manifest (ES modules)
├── package-lock.json   # Dependency lock file
└── node_modules/       # Installed dependencies
```

## Configuration

| Variable | Default | Description |
|----------|---------|-------------|
| `PORT` | `3000` | Port the proxy server listens on |
| `ANTHROPIC_BASE_URL` | — | Set to `http://localhost:<PORT>` for Claude Code integration |
| `ANTHROPIC_AUTH_TOKEN` | — | Set to any value (e.g. `dummy`) — not validated by the proxy |

The AI model is hardcoded to `z-ai/glm-5.1`. To change the model, modify the `model` parameter in `puter-proxy.js` (lines 38 and 80).

## Dependencies

| Package | Version | Purpose |
|---------|---------|---------|
| `express` | ^5.2.1 | HTTP server framework |
| `@heyputer/puter.js` | ^2.2.15 | Official Puter.js SDK for AI and cloud services |

## Limitations

- **Single model only** — The `model` field in requests is ignored; all requests use GLM-5.1
- **Text-only** — No support for images, files, or other multi-modal content types
- **No conversation persistence** — Message history is not stored server-side; clients must resend the full history with each request
- **Estimated token counts** — Token usage in responses is approximated (character count / 4), not from actual tokenization
- **No authentication** — The `Authorization` header is not validated; any token is accepted
- **No rate limiting** — No built-in rate limiting or usage quotas

## License

ISC
