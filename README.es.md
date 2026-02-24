# RustyClaw 🦀

> *"OpenClaw, AI-rewritten in Rust. Because our AI overlords demand rust!"*

Port de [nanobot](https://github.com/HKUDS/nanobot) (~8.300 LOC Python) a Rust — un agente de IA personal ligero que conecta LLMs a canales de chat (Telegram, Discord, Slack...) con tool use, memoria persistente y MCP.

**Objetivo dual:**

1. **Proyecto real** — Construir un agente de IA funcional en Rust con mejoras concretas sobre la versión Python
2. **Serie de contenido** — Aprender Rust desde cero en público, con benchmarks honestos y sátira RIIR

## Qué es nanobot

Un asistente personal de IA derivado de OpenClaw. Se sitúa entre LLMs y plataformas de mensajería, dándole manos al modelo: acceso al filesystem, comandos shell, búsqueda web, tareas programadas y skills extensibles via Markdown + YAML.

**El problema:** Es Python. Single-threaded. Procesa un mensaje a la vez. Usa ~50MB de RAM para rutear JSON entre APIs.

**El plan:** Portarlo a Rust. Mantener la simplicidad. Arreglar la concurrencia. Ver si importa.

## Stack

| Capa | Crate | Reemplaza (Python) |
|------|-------|--------------------|
| Runtime async | `tokio` | `asyncio` |
| HTTP + SSE | `reqwest` + `reqwest-eventsource` | `httpx` + internals de `litellm` |
| LLM (OpenAI-compat) | `async-openai` | `litellm` |
| LLM (Anthropic) | `reqwest` (directo) | `litellm` |
| Telegram | `teloxide` | `python-telegram-bot` |
| Discord | `tokio-tungstenite` (WS raw) | `websockets` (WS raw) |
| Config | `figment` | `pydantic` |
| Serialización | `serde` + `serde_json` | `json` + `pydantic` |
| CLI | `clap` | `typer` |
| Logging | `tracing` | `loguru` |
| Errores | `anyhow` + `thiserror` | `str(e)` en todas partes |
| MCP | `rmcp` | `mcp` |
| Cron | `tokio-cron-scheduler` | `croniter` |
| Templates | `minijinja` | f-strings |

**Ausencia notable:** No existe equivalente en Rust a LiteLLM (el router multi-provider de Python). Construiremos un provider router propio en ~300 LOC. El 80% de providers son OpenAI-compatible; Anthropic necesita implementación propia.

## Metodología de porte

Lecciones del [port a C++ de vjeux](https://blog.vjeux.com/) y el [incidente Tokamak](https://blog.val.town/blog/codegen-hallucinations/):

1. **Fichero a fichero.** Un módulo Python por sesión de trabajo.
2. **Nunca portar y mejorar a la vez.** Primero port fiel → luego rustificar → luego mejorar.
3. **El compilador es tu lista de TODO.** Port → compile → fix errors → repeat.
4. **Tests diferenciales primero.** Mismo input → nanobot Python vs rustyclaw Rust → mismo output.
5. **Verificar cada crate manualmente.** Cada crate que sugiera el LLM → comprobar en crates.io/docs.rs.
6. **Sesiones cortas.** Un módulo, máximo 90 minutos, luego resumen y commit.

### Estrategia anti-alucinaciones

> El sistema de verificación debe ser externo al generador.
> Si el LLM genera código, fixtures Y tests, estás validando ficción con ficción.

Cinco capas: compilador Rust → tests diferenciales contra Python → tracking de proveniencia → verificación de crates → logging de incidentes.

## Hoja de ruta

| Fase | Qué | Milestone |
|------|-----|-----------|
| **0** | Scaffolding, CI, README | Repo público con CI verde |
| **1** | Config, message bus, sesiones, tools, agent loop, canal CLI | `rustyclaw agent` funciona en terminal |
| **2** | Router multi-provider, SSE streaming, web tools, skills, subagents | Web search + edición de ficheros + memoria persistente |
| **3** | Telegram (teloxide), Discord (WebSocket raw), channel manager | Bots funcionales con tool use |
| **4** | Cliente MCP, servicio cron, heartbeat, CLI completo | Paridad funcional con nanobot |
| **5** | Procesamiento paralelo de mensajes, rate limiting, hot-reload de skills, benchmarks | Release v0.1.0 |

**Estimación:** 5-8 semanas, ~4.830 LOC Rust (desde ~7.300 LOC Python, excluyendo canales asiáticos fuera del MVP).

## Serie de blog

Cada fase tiene su post en [frr.dev/keepcoding](https://frr.dev/keepcoding). Lo interesante:

- Datos reales de consumo de LLM (tokens, coste, duración)
- Side-by-side Python vs Rust sin editar
- Contador de alucinaciones
- Benchmarks honestos con el running gag: *"Does it matter?"*

## Estructura del proyecto

```
rustyclaw/
├── src/                    # Código Rust (por crear)
├── reference/nanobot/      # Snapshot del código Python (fuente de verdad)
├── CLAUDE.md               # Contexto para asistente IA
└── llm-usage.csv           # Tracking de consumo de LLMs
```

## Licencia

MIT
