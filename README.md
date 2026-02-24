# RustyClaw 🦀

> *"OpenClaw, AI-rewritten in Rust. Because our AI overlords demand rust!"*

A port of [nanobot](https://github.com/HKUDS/nanobot) (~8,300 LOC Python) to Rust — a lightweight personal AI agent that connects LLMs to chat channels (Telegram, Discord, Slack...) with tool use, persistent memory, and MCP.

**Dual purpose:**

1. **Real project** — Build a functional AI agent in Rust with concrete improvements over the Python version
2. **Content series** — Learn Rust from scratch in public, with honest benchmarks and RIIR satire

## What is nanobot?

A personal AI assistant derived from OpenClaw. It sits between LLMs and messaging platforms, giving the model hands: filesystem access, shell commands, web search, scheduled tasks, and extensible skills via Markdown + YAML.

**The problem:** It's Python. Single-threaded. Processes one message at a time. Uses ~50MB RAM to route JSON between APIs.

**The plan:** Port it to Rust. Keep the simplicity. Fix the concurrency. See if it matters.

## Stack

| Layer | Crate | Replaces (Python) |
|-------|-------|--------------------|
| Async runtime | `tokio` | `asyncio` |
| HTTP + SSE | `reqwest` + `reqwest-eventsource` | `httpx` + `litellm` internals |
| LLM (OpenAI-compat) | `async-openai` | `litellm` |
| LLM (Anthropic) | `reqwest` (direct) | `litellm` |
| Telegram | `teloxide` | `python-telegram-bot` |
| Discord | `tokio-tungstenite` (raw WS) | `websockets` (raw WS) |
| Config | `figment` | `pydantic` |
| Serialization | `serde` + `serde_json` | `json` + `pydantic` |
| CLI | `clap` | `typer` |
| Logging | `tracing` | `loguru` |
| Errors | `anyhow` + `thiserror` | `str(e)` everywhere |
| MCP | `rmcp` | `mcp` |
| Cron | `tokio-cron-scheduler` | `croniter` |
| Templates | `minijinja` | f-strings |

**Notable absence:** There is no Rust equivalent to LiteLLM (the Python multi-provider LLM router). We'll build a custom provider router in ~300 LOC. 80% of providers are OpenAI-compatible; Anthropic needs a custom implementation.

## Porting approach

Lessons learned from [vjeux's C++ port](https://blog.vjeux.com/) and the [Tokamak incident](https://blog.val.town/blog/codegen-hallucinations/):

1. **File by file.** One Python module per work session.
2. **Never port and improve at the same time.** Faithful port first → rustify later → improve last.
3. **The compiler is the TODO list.** Port → compile → fix errors → repeat.
4. **Differential tests first.** Same input → Python nanobot vs Rust rustyclaw → same output.
5. **Verify every crate manually.** Every crate the LLM suggests → check on crates.io/docs.rs.
6. **Short sessions.** One module, max 90 minutes, then summary and commit.

### Anti-hallucination strategy

> The verification system must be external to the generator.
> If the LLM generates code, fixtures AND tests, you're validating fiction with fiction.

Five layers: Rust compiler → differential tests vs Python → provenance tracking → crate verification → incident logging.

## Roadmap

| Phase | What | Milestone |
|-------|------|-----------|
| **0** | Scaffolding, CI, README | Public repo with green CI |
| **1** | Config, message bus, sessions, tools, agent loop, CLI channel | `rustyclaw agent` works in terminal |
| **2** | Multi-provider LLM router, SSE streaming, web tools, skills, subagents | Web search + file editing + persistent memory |
| **3** | Telegram (teloxide), Discord (raw WebSocket), channel manager | Working bots with tool use |
| **4** | MCP client, cron service, heartbeat, full CLI | Feature parity with nanobot |
| **5** | Parallel message processing, rate limiting, hot-reload skills, benchmarks | v0.1.0 release |

**Estimated:** 5-8 weeks, ~4,830 LOC Rust (from ~7,300 LOC Python, excluding Asian-market channels skipped in MVP).

## Blog series

Each phase gets a blog post at [frr.dev/keepcoding](https://frr.dev/keepcoding). Highlights:

- Real LLM consumption data (tokens, cost, duration)
- Unedited side-by-side Python vs Rust
- Hallucination counter
- Honest benchmarks with the running gag: *"Does it matter?"*

## Project structure

```
rustyclaw/
├── src/                    # Rust source (to be created)
├── reference/nanobot/      # Python source snapshot (porting truth)
├── CLAUDE.md               # AI assistant context
└── llm-usage.csv           # LLM consumption tracking
```

## License

MIT
