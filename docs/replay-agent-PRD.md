# PRD: `replay-agent`

**A snapshot-and-replay library for LLM agents.**
*VCR.py, but for the Anthropic / OpenAI / Google SDKs.*

---

**Author:** Collin Neill
**Status:** Draft v1
**Last updated:** 2026-05-02
**Target ship:** v0.1 in 4 weekends

---

## TL;DR

`replay-agent` is a tiny Python library that intercepts every model call and tool call inside an agent run, writes the full request/response stream to a single JSONL file, and replays the agent deterministically against that file in tests or in CI. It exists because nondeterminism is the single biggest tooling problem in agent engineering today, and the existing answers — Langfuse, Phoenix, LangSmith — are observability platforms, not test harnesses. Developers want a 5KB import, not a self-hosted ClickHouse.

The library ships as `pip install replay-agent` and a companion `pytest-replay-agent` plugin. Together they let you record a real agent run once, commit the resulting `.replay.jsonl` file to your repo alongside the test, and snapshot-test the agent in CI without a network call or an API key.

---

## Problem Statement

LLM agents are nondeterministic by construction. The same input produces different outputs across runs, the same tool may return different results across invocations, and the same code path may take a different branch on the next call. This breaks the core development loop engineers rely on for every other kind of software: write a test, refactor, run the test, see what changed.

The people most affected are LLM application developers and agent researchers. Today their options are: pay for tokens on every CI run, mock every model call by hand, or skip integration testing entirely. The cost of skipping it is real — silent regressions in tool selection, prompt drift across model versions, and the inability to reproduce a bug a user reported yesterday. Anthropic's own engineering blog ("Demystifying evals for AI agents") frames this as a primary failure mode for production agent systems.

## Goals

The library must:

- **Record any agent run end-to-end** — capture every model API call, every tool call, every external HTTP request the agent makes, into a single human-readable JSONL artifact.
- **Replay that run deterministically** — given the same code path and the same recording, the agent produces byte-for-byte identical output, with zero network calls.
- **Work across the three major SDKs** — Anthropic, OpenAI, Google GenAI — without forcing the user to adopt a framework.
- **Integrate with pytest as a one-line fixture** — `def test_my_agent(replay): replay("session-1.replay.jsonl"); my_agent.run(...)`.
- **Produce diff-able artifacts** — when a recording goes stale, the developer sees a clean unified diff of what changed (which prompt, which tool, which token), not a 200-line stack trace.

## Non-Goals

- **Not an observability platform.** No web UI, no ClickHouse, no Postgres. Langfuse, Phoenix, and LangSmith already do that. Different problem, different shape, different audience.
- **Not a model evaluation framework.** No LLM-as-judge, no rubric scoring, no metric library. DeepEval and Inspect AI cover that. `replay-agent` is the layer underneath — it gives you the deterministic substrate those tools assume but rarely provide.
- **Not a framework.** It does not own the agent loop, define a node graph, or impose a state machine. Out of respect for the LangChain-exit trend, the library never replaces user code; it wraps SDK clients.
- **Not a load test or a benchmark runner.** Recording is for capturing real runs; we do not synthesize traffic.
- **Not a vector for vendor lock-in.** No SaaS tier, no telemetry, no "free with rate limits."

## User Stories

**As an LLM application developer building an agent at a startup, I want to record one good agent run and replay it as a regression test in CI**, so that I can refactor the orchestration loop without burning $50 of tokens every push.

**As a researcher prototyping experiments on Claude**, I want to replay yesterday's run with a tweaked prompt or a new tool, so that I can A/B my changes without re-paying for the full session.

**As an on-call engineer triaging a production agent failure**, I want to replay the exact session a user complained about — with the exact tool responses they saw — so that I can step through the failure locally instead of guessing.

**As an open-source maintainer of an agent library**, I want my users to send me `.replay.jsonl` files with bug reports, so that I can reproduce their issue without their API keys or their data.

**As a CI bot reviewing a pull request that touched the agent loop**, I want a one-line annotation on the PR showing which recordings now diverge and where in the trace the divergence happens, so that the reviewer can decide if the change is intentional.

## Requirements

### Must-Have (P0) — required for v0.1

**SDK interception.**
- Wraps `anthropic.Anthropic`, `openai.OpenAI`, and `google.genai.Client` at the request layer. Implementation: monkeypatch the underlying HTTPX transport.
- Acceptance: `with replay.record("foo.jsonl"): client.messages.create(...)` produces a JSONL file containing the request body, the response body, and a hash of both.

**Tool call capture.**
- The library exposes a `@replay.tool` decorator that, when applied to any callable, records inputs and outputs alongside the model calls in the same JSONL stream.
- Acceptance: a tool decorated with `@replay.tool` and called inside a recording context appears in the JSONL with its arguments, return value, and timing.

**Deterministic replay.**
- `with replay.replay("foo.jsonl"): client.messages.create(...)` returns the recorded response without making a network call. Tool calls return their recorded values.
- Acceptance: replay raises `ReplayMismatch` if the next observed call does not match the next recorded call. The error includes a diff of the request bodies.

**Pytest integration.**
- A `replay` fixture that takes a path and switches the test into replay mode automatically.
- Acceptance: `def test_x(replay): replay("x.jsonl"); my_agent()` passes when the agent matches the recording and fails with a structured diff when it doesn't.

**JSONL format.**
- One event per line. Each event is `{"ts", "kind", "request", "response", "hash"}`. `kind` is one of `model_call`, `tool_call`, `http_request`. Sensitive fields (Authorization headers, API keys) are scrubbed before write.
- Acceptance: the file opens in any text editor, diffs cleanly in Git, and is documented in a one-page schema.

**`replay diff` CLI.**
- `replay diff old.jsonl new.jsonl` produces a colorized unified diff showing which events changed, which were added, which were removed.
- Acceptance: when prompts drift between runs, the developer sees the exact prompt-text delta.

**Documentation.**
- README with: 30-second quickstart, 2-minute pytest example, "what this is and is not" framing, comparison table against Langfuse/LangSmith/DeepEval, and an animated GIF of the diff CLI.
- Acceptance: a developer who's never seen the project can record their first agent in under 5 minutes.

### Nice-to-Have (P1) — v0.2

**HTTP-level interception via `httpx` mounts.**
- For users not on a recognized SDK (custom REST clients, MCP servers).
- Acceptance: a raw `httpx.post()` to api.anthropic.com is captured and replayed without code changes.

**Match modes.**
- `strict` (default): exact byte equality. `lenient`: ignore timestamps, request IDs, and other known-volatile fields. `semantic`: optional LLM-judge match for response bodies (off by default; user supplies their own judge).
- Acceptance: `replay.replay("x.jsonl", match="lenient")` succeeds against a recording even if `request_id` changed.

**Recording filters.**
- Skip recording specific tools, redact specific fields, or sample at a rate. Useful for long-running agents.
- Acceptance: `@replay.tool(record=False)` excludes a tool from the recording without breaking replay.

**`replay redact` CLI.**
- Post-hoc redaction pass that scrubs PII or business-sensitive content from a recording before sharing.
- Acceptance: a recording produced inside a private app can be redacted to a publishable form for filing a bug report or building an open-source example.

**GitHub Actions reporter.**
- A small Action that runs replay tests and posts a PR comment with the diff. No SaaS, no auth — pure GitHub Actions YAML.
- Acceptance: a PR that breaks a recording gets a comment showing exactly which event diverged.

### Future Considerations (P2) — v0.3+

**Recording mutation.**
- A library API for taking an existing recording and patching one prompt, one tool response, or one model output, then re-running downstream events. Enables "what if" experiments.

**Cross-model translation.**
- Take a recording made against Claude and replay against GPT-5 (or vice versa) by translating the request format. Off by default; explicit opt-in.

**Trace export to OTLP.**
- For users who do want to feed recordings into Langfuse/Phoenix as OTel traces. Bridge, not replacement.

**Browser-based viewer.**
- A standalone `replay view` command that opens a self-contained HTML page rendering the recording as a tree. No server, just a file. Only ship this if v0.1 + v0.2 land well; the entire pitch of the library is "no UI required."

## Success Metrics

### Leading indicators (first 30 days post-launch)

- **GitHub stars: 200+** within 30 days. Validates that the framing resonates. Anchor: `inline-snapshot` reached this benchmark in a comparable window with a similar developer audience.
- **HN front page or `/r/LocalLLaMA` top post.** A single moment of distribution; if the framing doesn't land here it won't land anywhere.
- **5+ external contributors** with merged PRs. Signals that the API is small enough to be modifiable and the project is treated as community infrastructure.
- **At least one mention from a recognized voice** in the LLM-tooling space (Simon Willison, Hamel Husain, Eugene Yan, or an Anthropic engineer). Treat as a binary yes/no signal of credibility.

### Lagging indicators (90–180 days)

- **PyPI downloads: 5,000/month** by day 90, **20,000/month** by day 180. Comparable libraries (`vcrpy` at maturity, `inline-snapshot`) sit in this range.
- **At least 3 third-party blog posts or talks** referencing the library by name.
- **One inbound integration request** — someone building a higher-level tool (an eval framework, an agent IDE) asking to use `replay-agent` underneath. This is the strongest validation that the abstraction is the right shape.

### Personal-objective metric (the real one)

- **One inbound from an Anthropic recruiter or hiring manager** referencing the project, OR a successful interview where it comes up as evidence of full-stack judgment. The project is useful in its own right, but the ROI calculation for spending 4 weekends only closes if it materially advances the job application.

## Open Questions

- **Engineering — interception surface area.** Anthropic, OpenAI, and Google all expose Python SDKs that wrap `httpx`. Is monkeypatching `httpx.AsyncClient.send` clean enough, or should we ship per-SDK adapters that hook the SDK's own transport layer? The former is one implementation; the latter is three but more robust to SDK internals churning.
- **Engineering — streaming responses.** SSE token streams are the common case for agent UIs. Recording a stream is straightforward; replaying it as a stream (with realistic timing) vs. as a single chunk is a UX decision. Default to chunked replay with optional `simulate_timing=True`.
- **Engineering — tool-call hashing.** What's the canonical hash of a tool call? Stable across Python dict ordering, decimal precision, and timezone-aware datetimes. Likely answer: a custom JSON canonicalization pass before hashing.
- **Design — file format stability.** Once `.replay.jsonl` v1 is committed to a user's repo, we owe them forward-compatibility. Lock the format with a schema version field from day one.
- **Design — secrets handling.** Default scrubbing covers `Authorization`, `X-API-Key`, `OpenAI-Organization`, `anthropic-api-key`. Is opt-in *additional* scrubbing the right default, or opt-out? Lean opt-in (default to safe) and document the override.
- **Distribution — naming.** `replay-agent` describes the function but is generic. `vcrllm`, `agentvcr`, `recall`, `hindsight` are alternatives. Decide before publishing to PyPI; renaming after launch is expensive.
- **Distribution — license.** Apache 2.0 is the safe default for AI-tooling space (Inspect, DeepEval, LangChain Sandbox). MIT is more permissive but slightly less corporate-friendly. Pick Apache 2.0 unless there's a reason not to.

## Timeline & Phasing

| Phase | Scope | Calendar |
| --- | --- | --- |
| **W1: Core record/replay** | Anthropic SDK only. JSONL writer. Basic replay with strict matching. No tools yet. End-of-weekend demo: record one Claude call, replay it. | Weekend 1 |
| **W2: Tool capture + pytest plugin + diff CLI** | `@replay.tool` decorator. Pytest fixture. `replay diff` command. Basic README. | Weekend 2 |
| **W3: OpenAI + Google SDKs, polish, docs** | Add OpenAI and Google adapters. Write the README properly with the GIF, the comparison table, and the framing. Internal dogfood pass on a real agent (sanitized version of the Upstart PagerDuty triage agent). | Weekend 3 |
| **W4: Launch** | Cut v0.1.0 on PyPI. Publish a 1,500-word blog post: *"Snapshot testing for LLM agents: nondeterminism is a tooling problem, not a model problem."* Submit to HN on a Tuesday morning. Post on `/r/LocalLLaMA`, X, and the Anthropic developer Discord. Pin the repo on GitHub. | Weekend 4 |

**Hard deadlines.** Application to the Anthropic Senior+ Software Engineer, Research Tools role goes in by **end of W4** with the project linked from the resume and pinned on GitHub. The application is the actual deadline; the v0.1 launch is the gating dependency.

**Dependencies.** None external. The library deliberately depends only on the Python stdlib, `httpx`, and `pytest` (optional). All three are already in the dependency tree of any project that would use this.

**Risks and mitigations.**
- *Risk: someone ships this first.* Mitigation: 4 weekends, not 4 months. If a credible alternative ships during W1–W2, pivot to either (a) contribute to it or (b) sharpen the differentiation around pytest-native ergonomics.
- *Risk: SDK internals shift and break interception.* Mitigation: lock SDK versions in CI; ship a compatibility matrix in the README; prefer `httpx`-level interception over SDK-internal hooks where possible.
- *Risk: developer adoption requires a viral moment we don't get.* Mitigation: the project is portfolio-useful even at 50 stars. The success criterion that actually matters is the personal-objective metric, not GitHub vanity.
