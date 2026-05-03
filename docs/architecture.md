# Technical Architecture: `replay-agent`

**Status:** Draft
**Last updated:** 2026-05-02
**Companion to:** [replay-agent-PRD.md](./replay-agent-PRD.md)

---

## 1. Package Structure

```
replay_agent/
├── __init__.py              # Public API: record(), replay(), tool(), ReplayMismatch
├── _context.py              # RecordContext / ReplayContext (context managers + thread-local state)
├── _transport.py            # Custom httpx.BaseTransport that intercepts send()
├── _journal.py              # JSONL reader/writer + event schema
├── _scrub.py                # Header/field scrubbing (secrets removal)
├── _hash.py                 # Canonical JSON hashing
├── _diff.py                 # Structured diff logic (used by CLI + ReplayMismatch)
├── _tool.py                 # @replay.tool decorator implementation
├── _stream.py               # SSE stream recording/replaying
├── cli.py                   # `replay diff` CLI entry point
├── py.typed                 # PEP 561 marker
│
pytest_replay_agent/
├── __init__.py
├── plugin.py                # pytest plugin: `replay` fixture
│
tests/
├── conftest.py
├── test_record.py
├── test_replay.py
├── test_tool.py
├── test_stream.py
├── test_diff.py
├── test_scrub.py
├── test_pytest_plugin.py
├── fixtures/                # Sample .replay.jsonl files for testing
│   └── ...
```

Two packages, one repo:
- `replay-agent` — the core library (`pip install replay-agent`)
- `pytest-replay-agent` — the pytest plugin (`pip install pytest-replay-agent`)

The plugin depends on the core. The core has zero dependencies beyond `httpx` (which all four SDKs already bring).

---

## 2. Interception Strategy

### Decision: httpx transport monkeypatching (single implementation)

All four target SDKs (Anthropic, OpenAI, Google GenAI, Ollama) use `httpx` under the hood. Rather than writing per-SDK adapters that hook SDK internals (fragile, four implementations to maintain), we intercept at the `httpx` transport layer.

### How it works

```
User code
  │
  ▼
SDK client (anthropic.Anthropic / openai.OpenAI / google.genai.Client / ollama.Client)
  │
  ▼
httpx.Client / httpx.AsyncClient
  │
  ▼
┌─────────────────────────────────┐
│  ReplayTransport (our code)     │  ← wraps the original transport
│  ┌───────────────────────────┐  │
│  │  Original httpx transport │  │  ← only called in record mode
│  └───────────────────────────┘  │
└─────────────────────────────────┘
```

**Record mode:** `ReplayTransport.handle_request()` calls the real transport, captures the request + response, writes an event to the JSONL journal, and returns the response unchanged.

**Replay mode:** `ReplayTransport.handle_request()` reads the next expected event from the journal, validates the request matches (or raises `ReplayMismatch`), and returns the recorded response — no network call.

### Patching mechanism

We wrap the SDK client's internal `httpx.Client` instance:

```python
def _patch_client(httpx_client: httpx.Client, journal: Journal, mode: Mode):
    original_transport = httpx_client._transport
    httpx_client._transport = ReplayTransport(
        wrapped=original_transport,
        journal=journal,
        mode=mode,
    )
    return httpx_client
```

Discovery of the httpx client varies by SDK:
- **Anthropic:** `client._client` (httpx.Client) / `client._async_client`
- **OpenAI:** `client._client` (httpx.Client) / `client._async_client`
- **Google GenAI:** `client._api_client._http_client` (needs verification)
- **Ollama:** `client._client` (httpx.Client) / `client._async_client` — same pattern as Anthropic/OpenAI. Uses `httpx.Client` created in `BaseClient.__init__()`. Requests go through `self._client.request()` (sync) and `self._client.stream()` for streaming.

We also support the user passing any `httpx.Client` directly for non-SDK use cases.

### Async support

Both `httpx.Client` and `httpx.AsyncClient` use the same transport interface pattern (`handle_request` / `handle_async_request`). `ReplayTransport` implements both.

---

## 3. JSONL Event Schema (v1)

Each line in a `.replay.jsonl` file is one JSON object:

```jsonc
{
  "v": 1,                           // schema version — always 1 for now
  "seq": 0,                         // 0-indexed sequence number
  "ts": "2026-05-02T14:30:00.123Z", // ISO 8601 timestamp
  "kind": "model_call",             // "model_call" | "tool_call" | "http_request"
  "request": {
    "method": "POST",
    "url": "https://api.anthropic.com/v1/messages",
    "headers": { ... },             // scrubbed
    "body": { ... }                 // parsed JSON body
  },
  "response": {
    "status_code": 200,
    "headers": { ... },
    "body": { ... },                // parsed JSON body
    "is_stream": false              // true if this was an SSE stream
  },
  "hash": "sha256:abcdef..."        // hash of canonicalized request+response
}
```

For `tool_call` events:

```jsonc
{
  "v": 1,
  "seq": 3,
  "ts": "...",
  "kind": "tool_call",
  "request": {
    "tool_name": "get_weather",
    "args": { "city": "SF" }
  },
  "response": {
    "return_value": { "temp": 65 },
    "error": null,                   // or error string if tool raised
    "duration_ms": 142
  },
  "hash": "sha256:..."
}
```

### Kind classification

How we distinguish `model_call` from `http_request`:
- If the URL matches a known LLM API endpoint (Anthropic, OpenAI, Google, Ollama), `kind = "model_call"`
- All other HTTP traffic through the patched transport: `kind = "http_request"`
- Tool decorator calls: `kind = "tool_call"`

Known endpoint patterns:
- `api.anthropic.com/v1/messages`
- `api.openai.com/v1/chat/completions`
- `generativelanguage.googleapis.com/v1beta/models/*/generateContent`
- `localhost:11434/api/chat` and `localhost:11434/api/generate` (Ollama — also matches custom `OLLAMA_HOST`)

### First line: file header

The very first line of every `.replay.jsonl` is a header (not an event):

```jsonc
{
  "v": 1,
  "type": "header",
  "created_at": "2026-05-02T14:30:00Z",
  "replay_agent_version": "0.1.0",
  "python_version": "3.12.3"
}
```

This enables forward-compatibility checks.

---

## 4. Context Management & Thread-Local State

### Public API

```python
import replay_agent as replay

# Record mode
with replay.record("session.replay.jsonl") as ctx:
    client = anthropic.Anthropic()
    ctx.patch(client)                # patches the client's httpx transport
    response = client.messages.create(...)

# Replay mode
with replay.replay("session.replay.jsonl") as ctx:
    client = anthropic.Anthropic()
    ctx.patch(client)
    response = client.messages.create(...)  # no network call
```

### Alternative: auto-discovery

For convenience, the context can auto-discover SDK clients if the user opts in:

```python
with replay.record("session.replay.jsonl", auto_patch=True):
    # Automatically patches any Anthropic/OpenAI/Google/Ollama client created inside
    client = anthropic.Anthropic()
    response = client.messages.create(...)
```

`auto_patch=True` works by monkeypatching the SDK constructors (`anthropic.Anthropic.__init__`, `ollama.Client.__init__`, etc.) for the duration of the context manager. More magical but much better DX for the common case.

### Thread-local state

Active context is stored in a `contextvars.ContextVar`, not `threading.local()`. This is important because:
- Works with `asyncio` (each task gets its own context)
- Works with `threading` (each thread can have its own context)
- No global mutable state

```python
_active_context: ContextVar[Optional[BaseContext]] = ContextVar('replay_context', default=None)
```

The `@replay.tool` decorator checks this context var to decide whether to record/replay.

---

## 5. Tool Decorator

```python
@replay.tool
def get_weather(city: str) -> dict:
    return requests.get(f"https://weather.api/{city}").json()
```

Behavior depends on active context:
- **No context:** passes through, tool runs normally
- **Record context:** runs the tool, writes a `tool_call` event to the journal
- **Replay context:** reads the next event from the journal, validates args match, returns recorded value without calling the function

### Implementation

```python
def tool(fn):
    @functools.wraps(fn)
    def wrapper(*args, **kwargs):
        ctx = _active_context.get()
        if ctx is None:
            return fn(*args, **kwargs)
        elif ctx.mode == Mode.RECORD:
            result = fn(*args, **kwargs)
            ctx.journal.write_tool_call(fn.__name__, kwargs, result)
            return result
        elif ctx.mode == Mode.REPLAY:
            return ctx.journal.read_tool_call(fn.__name__, kwargs)
    return wrapper
```

Async variant (`@replay.tool` on an `async def`) is detected and wraps accordingly.

---

## 6. Streaming (SSE) Handling

### The problem

Most real agent calls use streaming (`stream=True`). The SDK yields chunks over an SSE connection. We need to:
1. **Record:** collect all chunks, write them as a single event, and still yield them to the caller in real-time
2. **Replay:** yield the recorded chunks back to the caller, optionally with simulated timing

### Record path

```
Real SSE stream → TeeStream → user code (yields chunks normally)
                           → buffer (collects all chunks)
                           → on stream end: write event to journal
```

`TeeStream` wraps the httpx response stream. It yields each chunk to the caller and appends it to an internal buffer. When the stream is exhausted, it writes the complete event.

The recorded event stores the full list of chunks:

```jsonc
{
  "response": {
    "is_stream": true,
    "chunks": [
      {"type": "content_block_start", ...},
      {"type": "content_block_delta", "delta": {"text": "Hello"}},
      ...
      {"type": "message_stop"}
    ]
  }
}
```

### Replay path

The replay transport returns a synthetic httpx.Response whose stream yields the recorded chunks. Default: yield all chunks immediately (fast tests). With `simulate_timing=True`: insert delays based on recorded `ts` deltas between chunks.

---

## 7. Request Matching & Hashing

### Canonical JSON

Before hashing or comparing, we canonicalize JSON:
1. Sort all object keys recursively
2. Normalize numbers (no trailing zeros, consistent exponent notation)
3. Normalize strings (NFC unicode normalization)
4. Serialize with `json.dumps(separators=(',', ':'), sort_keys=True, ensure_ascii=False)`

### Hash computation

```python
def compute_hash(request: dict, response: dict) -> str:
    canonical = canonical_json({"request": request, "response": response})
    return "sha256:" + hashlib.sha256(canonical.encode()).hexdigest()[:16]
```

### Matching during replay (strict mode)

When `ReplayTransport` receives a request during replay:
1. Read the next event from the journal (by sequence number)
2. Compare the request body (canonicalized) against the recorded request body
3. If match: return the recorded response
4. If mismatch: raise `ReplayMismatch` with a structured diff

Fields excluded from matching even in strict mode:
- `ts` (timestamps always differ)
- Request headers that are session-specific: `X-Request-Id`, `User-Agent` version suffix
- `seq` (sequence is positional, not content-based)

### ReplayMismatch error

```python
class ReplayMismatch(AssertionError):
    def __init__(self, expected, actual, diff_text):
        self.expected = expected
        self.actual = actual
        self.diff = diff_text
        super().__init__(f"Replay mismatch at event #{expected['seq']}:\n{diff_text}")
```

Inherits from `AssertionError` so pytest treats it as a test failure with clean output.

---

## 8. Secrets Scrubbing

### Default scrub list

Headers scrubbed before writing to JSONL:
- `Authorization`
- `X-API-Key`
- `anthropic-api-key`
- `OpenAI-Organization`
- `api-key` (Azure OpenAI)

Body fields scrubbed:
- Any field named `api_key`, `secret`, `token`, `password`

Scrubbed values are replaced with `"[SCRUBBED]"`.

### Implementation

```python
SCRUB_HEADERS = {"authorization", "x-api-key", "anthropic-api-key", "openai-organization", "api-key"}
SCRUB_BODY_KEYS = {"api_key", "secret", "token", "password"}

def scrub_event(event: dict) -> dict:
    event = copy.deepcopy(event)
    # Scrub headers (case-insensitive)
    for section in ("request", "response"):
        headers = event.get(section, {}).get("headers", {})
        for key in list(headers):
            if key.lower() in SCRUB_HEADERS:
                headers[key] = "[SCRUBBED]"
    # Scrub body fields (recursive)
    _scrub_dict(event.get("request", {}).get("body", {}))
    return event
```

Users can extend the scrub list:

```python
with replay.record("session.jsonl", scrub_headers=["X-Custom-Secret"]):
    ...
```

---

## 9. Diff CLI

### Usage

```bash
replay diff old.replay.jsonl new.replay.jsonl
```

### Output

Colorized unified diff, event-by-event:

```diff
Event #2 (model_call):
  request.body.messages[1].content:
-   "Summarize this document in 3 bullet points"
+   "Summarize this document in 5 bullet points"

Event #4 (tool_call):
  [ADDED] — no corresponding event in old recording

Event #5 (model_call):
  response.body.content[0].text:
-   "Here are 3 key points..."
+   "Here are 5 key points..."
```

### Implementation

Uses `_diff.py` which:
1. Aligns events by `seq` (primary) and `hash` (fallback for insertions/deletions)
2. For each pair of events, deep-diffs the JSON trees
3. Formats as colorized text (click library for terminal colors)

CLI entry point via `pyproject.toml`:

```toml
[project.scripts]
replay = "replay_agent.cli:main"
```

---

## 10. Pytest Plugin

### Fixture: `replay`

```python
# pytest_replay_agent/plugin.py

@pytest.fixture
def replay(request):
    """Fixture that provides record/replay for a test."""
    def _replay(path: str, mode: str = "replay", **kwargs):
        jsonl_path = Path(request.fspath).parent / path
        if mode == "replay":
            ctx = ReplayContext(jsonl_path, **kwargs)
        elif mode == "record":
            ctx = RecordContext(jsonl_path, **kwargs)
        ctx.__enter__()
        request.addfinalizer(lambda: ctx.__exit__(None, None, None))
        return ctx
    return _replay
```

### Usage in tests

```python
def test_my_agent(replay):
    ctx = replay("fixtures/session-1.replay.jsonl")
    client = anthropic.Anthropic()
    ctx.patch(client)

    result = my_agent(client)
    assert "expected output" in result
```

Or with auto-patch:

```python
def test_my_agent(replay):
    replay("fixtures/session-1.replay.jsonl", auto_patch=True)
    result = my_agent()  # agent creates its own client internally
    assert "expected output" in result
```

### Plugin registration

```toml
# pytest-replay-agent/pyproject.toml
[project.entry-points.pytest11]
replay_agent = "pytest_replay_agent.plugin"
```

### Failure output

When `ReplayMismatch` fires in a test, the pytest output shows:

```
FAILED test_my_agent - ReplayMismatch: Mismatch at event #3 (model_call):
  request.body.messages[2].content:
-   "old prompt text"
+   "new prompt text"
```

---

## 11. Dependency Graph

```
replay-agent
├── httpx (already in all four SDK dep trees)
└── (stdlib only otherwise)

pytest-replay-agent
├── replay-agent
└── pytest
```

No other dependencies. This is deliberate — the library's value prop is "5KB import, not a platform."

---

## 12. Key Design Decisions & Rationale

| Decision | Choice | Rationale |
|---|---|---|
| Interception layer | httpx transport | Single implementation covers all 4 SDKs (Anthropic, OpenAI, Google, Ollama). Lower risk of breakage from SDK internal changes than hooking SDK-specific layers. |
| State management | `contextvars.ContextVar` | Works with async, threads, and nested contexts. No global mutable state. |
| Event format | JSONL (one event per line) | Diff-able in git, streamable, appendable. No need to parse full file to read one event. |
| Hash algorithm | SHA-256 (truncated to 16 hex chars) | Good enough for content-addressing events. Short enough to be readable in diffs. |
| `ReplayMismatch` base class | `AssertionError` | Pytest natively understands assertion errors — gives clean failure output without custom plugins. |
| Streaming replay | Immediate yield (default) | Fast tests. `simulate_timing=True` opt-in for demos/debugging. |
| Scrubbing | Opt-in extension, safe defaults | Scrub known secret headers by default. Users add more, never remove defaults. |
| Package split | Core + pytest plugin | Users who don't use pytest don't pay for the dependency. Standard pattern (cf. `pytest-httpx`, `pytest-asyncio`). |

---

## 13. Open Design Questions (to resolve during W1)

1. **Auto-patch vs explicit patch:** Should `auto_patch=True` be the default? It's more magical but dramatically better DX. Leaning: make it the default, let users opt out with `auto_patch=False`.

2. **Google GenAI client internals:** Need to verify the httpx client path. If Google uses a different HTTP library, we'd need a separate adapter for that SDK only.

3. **Async-first or sync-first?** The transport interface is the same either way, but test infrastructure differs. Plan: implement sync first (W1), add async in W2.

4. **Recording file location convention:** Should the pytest fixture default to looking for recordings in a `__recordings__/` directory next to the test file? Or require explicit paths? Leaning: explicit paths, document the convention.

5. **Event alignment in diff:** When events are inserted/removed between two recordings, how to align? Options: (a) longest common subsequence on hashes, (b) align by `kind` + request URL. Leaning: LCS on (kind, url/tool_name) tuples.

6. **Ollama endpoint detection:** Ollama runs on `localhost:11434` by default but users can set `OLLAMA_HOST` to any address. For `kind` classification, we match on the URL path (`/api/chat`, `/api/generate`) rather than the host. This avoids false negatives when users run Ollama on a custom host/port. We should also detect the OpenAI-compatible endpoint (`/v1/chat/completions` on Ollama) which would already be caught by the OpenAI pattern.

---

## 14. W1 Implementation Plan

Goal: Record one Claude API call and one Ollama call, replay both, with strict matching. No tools, no streaming, no pytest plugin yet. Ollama is included from day one because it enables local development and testing without API keys.

### Files to create:

1. **`pyproject.toml`** — package metadata, dependencies (httpx)
2. **`replay_agent/__init__.py`** — public API surface: `record()`, `replay()`, `ReplayMismatch`
3. **`replay_agent/_context.py`** — `RecordContext`, `ReplayContext`, context var management
4. **`replay_agent/_transport.py`** — `ReplayTransport(httpx.BaseTransport)`
5. **`replay_agent/_journal.py`** — `Journal` class: write/read JSONL events
6. **`replay_agent/_scrub.py`** — header/body scrubbing
7. **`replay_agent/_hash.py`** — canonical JSON + SHA-256
8. **`tests/test_record.py`** — record an Anthropic call, verify JSONL output
9. **`tests/test_replay.py`** — replay from a fixture JSONL, verify no network call
10. **`tests/test_ollama.py`** — record/replay an Ollama call (requires local Ollama instance for recording; replay is offline)

### End-of-W1 demo (Anthropic):

```python
import anthropic
import replay_agent as replay

# Record
with replay.record("demo.replay.jsonl", auto_patch=True):
    client = anthropic.Anthropic()
    resp = client.messages.create(
        model="claude-sonnet-4-20250514",
        max_tokens=100,
        messages=[{"role": "user", "content": "Say hello"}]
    )
    print(resp.content[0].text)

# Replay (no network, no API key needed)
with replay.replay("demo.replay.jsonl", auto_patch=True):
    client = anthropic.Anthropic(api_key="not-needed")
    resp = client.messages.create(
        model="claude-sonnet-4-20250514",
        max_tokens=100,
        messages=[{"role": "user", "content": "Say hello"}]
    )
    print(resp.content[0].text)  # same output as above
```

### End-of-W1 demo (Ollama):

```python
import ollama
import replay_agent as replay

# Record (requires local Ollama running)
with replay.record("ollama-demo.replay.jsonl", auto_patch=True):
    client = ollama.Client()
    resp = client.chat(
        model="llama3",
        messages=[{"role": "user", "content": "Say hello"}]
    )
    print(resp["message"]["content"])

# Replay (no Ollama instance needed)
with replay.replay("ollama-demo.replay.jsonl", auto_patch=True):
    client = ollama.Client()
    resp = client.chat(
        model="llama3",
        messages=[{"role": "user", "content": "Say hello"}]
    )
    print(resp["message"]["content"])  # same output as above
```
