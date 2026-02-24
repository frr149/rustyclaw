# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Qué es RustyClaw

Port de [nanobot](reference/nanobot/) (~8.300 LOC Python) a Rust. Agente de IA personal que conecta LLMs a canales de chat (Telegram, Discord, Slack...) con tool use, memoria persistente y MCP.

Objetivo dual: agente funcional en Rust + serie de contenido viral sobre aprender Rust desde cero (blog en frr.dev/keepcoding).

El documento maestro completo está en `RUSTYCLAW.md`.

## Comandos de desarrollo

```bash
cargo check                    # Verificación de tipos (rápido)
cargo build                    # Compilar
cargo test                     # Todos los tests
cargo test test_name           # Un test específico
cargo test module::            # Tests de un módulo
cargo clippy                   # Linting (DEBE pasar sin warnings)
cargo fmt                      # Formatear
cargo run -- agent             # Ejecutar en modo CLI
cargo run -- agent -m "msg"    # One-shot message
cargo run -- gateway           # Arrancar canales (Telegram, Discord...)
```

## Arquitectura

### Flujo de un mensaje

```
Canal (Telegram/Discord/CLI) → MessageBus (tokio::mpsc) → AgentLoop → LLM Provider → Tool execution → respuesta por el bus → Canal
```

### Módulos principales

| Módulo | Responsabilidad | Referencia Python |
|--------|----------------|-------------------|
| `config/` | Carga y validación (serde + figment) | `reference/nanobot/nanobot/config/` |
| `bus/` | Message bus (tokio::mpsc) | `reference/nanobot/nanobot/bus/` |
| `session/` | Sesiones JSONL file-based | `reference/nanobot/nanobot/session/` |
| `agent/` | Loop, context builder, memoria | `reference/nanobot/nanobot/agent/` |
| `tools/` | Registry + herramientas (filesystem, shell, web, mcp) | `reference/nanobot/nanobot/agent/tools/` |
| `providers/` | Router multi-provider LLM | `reference/nanobot/nanobot/providers/` |
| `channels/` | Telegram (teloxide), Discord (WebSocket), CLI | `reference/nanobot/nanobot/channels/` |
| `cron/` | Tareas programadas | `reference/nanobot/nanobot/cron/` |
| `cli/` | Comandos clap | `reference/nanobot/nanobot/cli/` |

### Estado — todo ficheros, sin base de datos

| Estado | Formato | Ubicación |
|--------|---------|-----------|
| Config | JSON | `~/.rustyclaw/config.json` |
| Sessions | JSONL | `~/.rustyclaw/sessions/{key}.jsonl` |
| Memoria | Markdown | `{workspace}/memory/MEMORY.md` |
| Cron jobs | JSON | `~/.rustyclaw/cron/jobs.json` |
| Skills | Markdown + YAML frontmatter | `{workspace}/skills/*/SKILL.md` |

### Ownership (diseño Rust, NO traducción ciega de Python)

- **Bus**: `tokio::sync::mpsc` — sender es Clone, receiver es owned. Sin mutex.
- **Sessions**: `Arc<RwLock<SessionManager>>` — read-heavy, write-rare.
- **Tools**: registrados una vez, luego read-only → `Arc<ToolRegistry>` sin lock.
- **Channels**: cada uno owned por su tokio task, comunica via bus.
- **Config**: loaded once, `Arc<Config>` inmutable.

## Reglas de porte

### Metodología (CRÍTICO)

1. **File-by-file.** Un módulo Python por sesión de trabajo.
2. **NUNCA portar y mejorar a la vez.** Primero port fiel → luego rustificar → luego mejorar.
3. **El compilador es tu TODO list.** Port → compile → fix errors → repeat.
4. **Tests diferenciales primero.** Portar tests antes que implementación.
5. **Traducción literal, sin optimizaciones** en la primera pasada.
6. **Verificar crates manualmente.** Cada crate → comprobar en crates.io/docs.rs. No inventar APIs.
7. **Sesiones cortas.** Un módulo, máximo 90 minutos, luego summary y commit.

### Anti-alucinaciones

> **El sistema de verificación debe ser externo al generador.**
> Si generas código, fixtures Y tests, estás validando ficción con ficción.

- Tests diferenciales: mismo input → nanobot Python vs rustyclaw → mismo output.
- Fixtures capturadas de ejecución real, NUNCA generadas.
- Verificar cada crate en crates.io antes de usarlo.
- Si no estás seguro de una API, di "no sé" en vez de inventar.

### Provenance header (obligatorio en cada fichero portado)

```rust
// Ported from: reference/nanobot/nanobot/agent/loop.py (lines 1-476)
// Ported by: Claude Opus 4.6 / session 2026-XX-XX
// Verified: differential tests pass ✓
```

### Checklist pre-merge

- [ ] `cargo clippy` sin warnings
- [ ] Tests unitarios pasan
- [ ] Tests diferenciales contra nanobot pasan
- [ ] Crates verificados en crates.io/docs.rs
- [ ] Provenance header presente

## Fases del porte

| Fase | Contenido | Milestone |
|------|-----------|-----------|
| 0 | Scaffolding, CI, README | Repo público con CI verde |
| 1 | Config, bus, sessions, tools, agent loop, CLI | `rustyclaw agent` funciona en CLI |
| 2 | Provider router, SSE, web tools, skills, subagents | Web search + file editing + memoria |
| 3 | Telegram (teloxide), Discord (WebSocket) | Bot funcional con tool use |
| 4 | MCP client, cron, heartbeat, CLI completo | Paridad con nanobot |
| 5 | Parallel msgs, rate limiting, hot-reload, benchmarks | v0.1.0 release |

## Problemas conocidos de porte

- **LiteLLM no existe en Rust.** Implementar provider router propio (~300 LOC). 80% de providers son OpenAI-compatible (`async-openai` + base_url). Anthropic necesita reqwest directo.
- **JSON malformado de LLMs.** Cascada: serde_json → strip markdown fences → llm_json → regex → error al agente.
- **MCP (`rmcp`) es joven.** Breaking changes esperables. Aislar detrás de trait. Fallback: JSON-RPC directo (~200 LOC).
- **Canales asiáticos (Feishu, DingTalk, QQ)** no se portan en MVP. Sin SDKs Rust.

## Relay entre LLMs

Los artefactos de estado para cambio de contexto entre sesiones/LLMs están en `.llm/`:

```
.llm/
├── CONTEXT.md       # Estado actual del proyecto
├── CONVENTIONS.md   # Patrones de código adoptados
├── DECISIONS.md     # Architecture Decision Records
├── KNOWN-ISSUES.md  # Problemas pendientes
└── sessions/        # Resúmenes de sesiones anteriores
```

Al terminar una sesión: actualizar CONTEXT.md y escribir resumen en `sessions/`.

## Tracking de consumo

Registrar métricas de cada sesión en `llm-usage.csv`:

```csv
date,llm,model,module,tokens_in,tokens_out,cost_usd,duration_min,loc_python,loc_rust,hallucinations,tests_pass
```

## Referencia Python

El código fuente de nanobot está en `reference/nanobot/` como snapshot estático (sin `.git/`). Es la fuente de verdad para el porte. Consultar siempre el código Python original antes de portar un módulo.
