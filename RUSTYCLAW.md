# RustyClaw — Documento Maestro del Proyecto

> *"OpenClaw, mass-rewritten in Rust. Because the crab demanded it."*

Port de [nanobot](https://github.com/HKUDS/nanobot) (~8.300 LOC Python) a Rust.
Objetivo dual: agente de IA funcional + serie de contenido viral sobre aprender Rust desde cero.

**Estado:** Planificado. Inicio estimado: primera semana de marzo 2026.

---

## Índice

- [Parte I: Visión y logística](#parte-i-visión-y-logística)
- [Parte II: Análisis del código fuente](#parte-ii-análisis-del-código-fuente)
- [Parte III: Problemas potenciales y mitigaciones](#parte-iii-problemas-potenciales-y-mitigaciones)
- [Parte IV: Estrategia de porte](#parte-iv-estrategia-de-porte)
- [Parte V: Estrategia anti-alucinaciones](#parte-v-estrategia-anti-alucinaciones)
- [Parte VI: Relay entre LLMs](#parte-vi-relay-entre-llms)
- [Parte VII: Tracking de consumo](#parte-vii-tracking-de-consumo)
- [Parte VIII: Serie de blog](#parte-viii-serie-de-blog)
- [Parte IX: Backlog público](#parte-ix-backlog-público)
- [Parte X: Crates de Rust](#parte-x-crates-de-rust)

---

# Parte I: Visión y logística

## Qué es

Port de nanobot (agente de IA personal ultra-ligero derivado de OpenClaw) de Python a Rust.
nanobot conecta LLMs a canales de chat (Telegram, Discord, Slack...) con tool use,
memoria persistente y MCP.

## Objetivo dual

1. **Proyecto real:** Agente de IA funcional en Rust con mejoras concretas
2. **Serie de contenido:** Aprender Rust desde cero en público, con sátira RIIR

## Identidad

| Aspecto | Decisión |
|---------|----------|
| **Nombre** | RustyClaw |
| **Tagline** | "OpenClaw, mass-rewritten in Rust. Because the crab demanded it." |
| **Tono** | Autoironía + honestidad brutal + sátira RIIR |
| **Licencia** | MIT |
| **Org GitHub** | `rustyclaw` (org dedicada) |
| **Blog** | Serie en frr.dev/keepcoding (español → traducción automática) |
| **Cadencia posts** | Según avance (sin calendario fijo) |
| **LLMs** | Claude (Opus/Sonnet) principal + Codex CLI backup |

## Repositorios

| Repo | Contenido |
|------|-----------|
| `rustyclaw/rustyclaw` | Código Rust del agente + backlog + tracking |
| `rustyclaw/nanobot-analysis` | Fork/snapshot con análisis anotado |

---

# Parte II: Análisis del código fuente

## Estructura completa de nanobot

```
nanobot/ (~8.300 LOC Python, ~7.300 sin tests)
│
├── nanobot/
│   ├── agent/                    [NÚCLEO: ~1.280 LOC]
│   │   ├── loop.py          476  Agent loop: receive → context → LLM → tools → respond
│   │   ├── context.py       238  System prompt + message assembly
│   │   ├── skills.py        228  Skill loading, frontmatter parsing
│   │   ├── subagent.py      257  Background task execution
│   │   ├── memory.py         30  MEMORY.md / HISTORY.md file I/O
│   │   └── tools/                [HERRAMIENTAS: ~850 LOC]
│   │       ├── base.py      102  Tool ABC + JSON Schema validation
│   │       ├── registry.py   73  Tool dict + dispatch
│   │       ├── filesystem.py 211  read_file, write_file, edit_file, list_dir
│   │       ├── shell.py     144  exec: subprocess con safety guards
│   │       ├── web.py       163  web_search (Brave), web_fetch (readability)
│   │       ├── message.py    86  Send message to channel via bus
│   │       ├── spawn.py      65  Spawn subagent tool
│   │       ├── cron.py      127  Schedule cron jobs
│   │       └── mcp.py        80  MCP client tool wrapper
│   │
│   ├── bus/                      [MESSAGE BUS: ~124 LOC]
│   │   ├── events.py         37  InboundMessage, OutboundMessage dataclasses
│   │   └── queue.py          81  asyncio.Queue-based MessageBus
│   │
│   ├── channels/                 [CANALES: ~3.440 LOC]
│   │   ├── base.py          127  BaseChannel ABC + allowlist
│   │   ├── manager.py       227  Lazy init + outbound dispatcher
│   │   ├── telegram.py      390  python-telegram-bot, long polling
│   │   ├── discord.py       261  Raw WebSocket Gateway v10
│   │   ├── whatsapp.py      148  WebSocket to Node.js Baileys bridge
│   │   ├── slack.py         235  slack-sdk Socket Mode
│   │   ├── feishu.py        402  lark-oapi SDK, WebSocket in thread
│   │   ├── dingtalk.py      245  dingtalk-stream SDK
│   │   ├── email.py         403  stdlib imaplib/smtplib, IMAP polling
│   │   ├── mochat.py        895  Socket.IO + HTTP polling fallback (MÁS COMPLEJO)
│   │   └── qq.py            134  qq-botpy SDK
│   │
│   ├── config/                   [CONFIGURACIÓN: ~410 LOC]
│   │   ├── schema.py        304  Pydantic models para toda la config
│   │   └── loader.py        106  JSON load/save con camelCase conversion
│   │
│   ├── providers/                [LLM PROVIDERS: ~680 LOC]
│   │   ├── base.py           70  LLMProvider ABC, LLMResponse, ToolCallRequest
│   │   ├── registry.py      395  ProviderSpec dataclass, PROVIDERS tuple
│   │   ├── litellm_provider.py 208  LiteLLM wrapper
│   │   ├── openai_codex_provider.py 312  SSE stream para ChatGPT Codex
│   │   └── transcription.py  65  Groq Whisper API
│   │
│   ├── session/                  [SESIONES: ~184 LOC]
│   │   └── manager.py       179  JSONL file-based sessions
│   │
│   ├── cron/                     [TAREAS PROGRAMADAS: ~410 LOC]
│   │   ├── service.py       351  Scheduler con croniter + asyncio
│   │   └── types.py          59  CronJob, CronSchedule dataclasses
│   │
│   ├── heartbeat/                [HEARTBEAT: ~130 LOC]
│   │   └── service.py       130  Periodic HEARTBEAT.md check
│   │
│   ├── cli/                      [CLI: ~935 LOC]
│   │   └── commands.py      935  typer: agent, gateway, onboard, status, cron
│   │
│   └── utils/                    [UTILIDADES: ~85 LOC]
│       └── helpers.py        80  Path utils, safe_filename, timestamp
│
├── tests/                        [~1.030 LOC]
│   ├── test_cli_input.py     59
│   ├── test_commands.py      92
│   ├── test_consolidate_offset.py 477
│   ├── test_email_channel.py 311
│   └── test_tool_validation.py 88
│
└── workspace/                    [TEMPLATES]
    ├── AGENTS.md, SOUL.md, USER.md, TOOLS.md, IDENTITY.md
    └── skills/*.md (7 skills builtin)
```

## Arquitectura: flujo completo de un mensaje

```
Canal (Telegram/Discord/...)
  │ _on_message(update)
  │ check is_allowed(sender_id)
  ▼
BaseChannel._handle_message()
  │ bus.publish_inbound(InboundMessage)
  ▼
AgentLoop.run()                          ◄── asyncio.Queue consumer (1 msg at a time!)
  │
  ├── sessions.get_or_create(key)        ◄── JSONL file load + memory cache
  ├── check memory_window → consolidation task
  ├── _set_tool_context(channel, chat_id)
  │
  ├── ContextBuilder.build_messages()
  │   ├── build_system_prompt()
  │   │   ├── _get_identity()            ◄── static text + datetime + workspace path
  │   │   ├── _load_bootstrap_files()    ◄── AGENTS.md, SOUL.md, USER.md, TOOLS.md
  │   │   ├── memory.get_memory_context() ◄── reads MEMORY.md
  │   │   ├── skills.get_always_skills() ◄── always=true skills
  │   │   └── skills.build_skills_summary()
  │   ├── append session history
  │   └── _build_user_content()          ◄── text + optional base64 images
  │
  ├── _run_agent_loop(messages)
  │   └── LOOP (max 20 iterations):
  │       ├── provider.chat(messages, tools, model, temperature, max_tokens)
  │       │   └── LiteLLMProvider: litellm.acompletion() → _parse_response()
  │       │
  │       ├── IF tool_calls:
  │       │   ├── registry.execute(name, params) per tool
  │       │   │   ├── tool.validate_params(params)  ◄── JSON Schema
  │       │   │   └── tool.execute(**params)         ◄── returns string
  │       │   └── append "Reflect on results" prompt
  │       │
  │       └── ELSE: break with final_content
  │
  ├── session.add_message("user", msg.content)
  ├── session.add_message("assistant", final_content)
  └── sessions.save(session)             ◄── write JSONL
  │
  ▼
bus.publish_outbound(OutboundMessage)
  │
  ▼
ChannelManager._dispatch_outbound()
  └── channel.send(msg)                  ◄── markdown→HTML, chunking, API call
```

## Concurrencia

**Single-threaded async (asyncio event loop).** Una excepción: Feishu SDK
corre WebSocket en un thread separado con bridge `asyncio.run_coroutine_threadsafe()`.

**Limitación crítica:** El agent loop procesa UN mensaje a la vez.
Queue → process → respond → next. Sin paralelismo entre conversaciones.

## Estado

Todo es archivos. Sin base de datos.

| Estado | Formato | Ubicación |
|--------|---------|-----------|
| Config | JSON | `~/.nanobot/config.json` |
| Sessions | JSONL | `~/.nanobot/sessions/{key}.jsonl` |
| Memoria | Markdown | `{workspace}/memory/MEMORY.md` |
| Historial | Markdown (append) | `{workspace}/memory/HISTORY.md` |
| Cron jobs | JSON | `~/.nanobot/cron/jobs.json` |
| Skills | Markdown + YAML frontmatter | `{workspace}/skills/*/SKILL.md` |

## Dependencias clave

| Dependencia Python | Función | Equivalente Rust | Riesgo |
|-------------------|---------|------------------|--------|
| `litellm` | Multi-provider LLM routing | **No existe** — implementar propio | ALTO |
| `pydantic` | Config + validación | `serde` + validación custom | Bajo |
| `httpx` | HTTP async | `reqwest` | Bajo |
| `python-telegram-bot` | Canal Telegram | `teloxide` | Bajo |
| `websockets` | Discord/WhatsApp | `tokio-tungstenite` | Bajo |
| `json_repair` | JSON malformado de LLMs | `llm_json` (joven) | Medio |
| `mcp` | Model Context Protocol | `rmcp` (joven, v0.15) | Medio |
| `slack-sdk` | Canal Slack | `slack-morphism` | Medio |
| `readability-lxml` | HTML→texto | `scraper` + custom | Bajo |
| `croniter` | Cron expressions | `tokio-cron-scheduler` | Bajo |
| `python-socketio` | Mochat (Socket.IO) | `rust_socketio` | Medio |
| `lark-oapi` | Canal Feishu | **No existe** — raw HTTP/WS | ALTO (skip MVP) |
| `dingtalk-stream` | Canal DingTalk | **No existe** — raw HTTP/WS | ALTO (skip MVP) |
| `qq-botpy` | Canal QQ | **No existe** — raw HTTP/WS | ALTO (skip MVP) |

## Error handling

nanobot devuelve **todos** los errores como strings. No hay tipos de error.
El patrón es catch-and-return-string en todas las capas:

```python
# Tools
return f"Error reading file: {str(e)}"

# Registry
return f"Error executing {name}: {str(e)}"

# Agent loop
content=f"Sorry, I encountered an error: {str(e)}"

# LLM
return LLMResponse(content=f"Error calling LLM: {str(e)}", finish_reason="error")
```

## Seguridad

- **Allowlists por canal** — si vacía, acceso abierto
- **Shell command filtering** — regex blocklist (rm -rf, format, dd, fork bomb...)
- **Workspace restriction** — opt-in path containment
- **Sin rate limiting** — cualquiera puede bombardear el agente
- **Sin auth en gateway** — el puerto 18790 no tiene autenticación
- **Email requiere consent** — `consent_granted: true` explícito

---

# Parte III: Problemas potenciales y mitigaciones

## Problemas de porte (técnicos)

### P1: LiteLLM no existe en Rust (RIESGO ALTO)

**Problema:** nanobot depende de LiteLLM para routing a 100+ providers. La API
real usada es solo `litellm.acompletion()`, pero internamente LiteLLM:
- Rutea por prefijo de modelo (`anthropic/`, `openrouter/`, `deepseek/`...)
- Normaliza respuestas (tool_calls, streaming, usage)
- Gestiona API keys via env vars
- Droppea parámetros no soportados por provider

**Mitigación:** El `registry.py` de nanobot (395 LOC) contiene toda la lógica de
routing como datos declarativos (tuple de `ProviderSpec`). Es portable directamente.

**Estrategia:**
```rust
#[async_trait]
trait LlmProvider: Send + Sync {
    async fn chat(&self, req: ChatRequest) -> Result<ChatResponse>;
    fn supports_tools(&self) -> bool;
    fn supports_streaming(&self) -> bool;
}

// OpenAI-compatible: cubre OpenRouter, DeepSeek, Groq, vLLM, Moonshot, etc.
// Anthropic: formato nativo diferente (content blocks, no choices)
// Gemini: formato Google
```

El 80% de providers son OpenAI-compatible. Con `async-openai` + base_url custom
cubrimos la mayoría. Anthropic necesita implementación propia con `reqwest`.

**LOC estimadas:** ~300-400 Rust (vs 680 Python contando LiteLLM)

### P2: JSON malformado de LLMs (RIESGO MEDIO)

**Problema:** Los LLMs devuelven JSON roto con frecuencia:
- Markdown fences alrededor del JSON (` ```json ... ``` `)
- Trailing commas
- Comentarios
- Python True/False en vez de true/false
- JSON truncado por max_tokens

nanobot usa `json_repair` (maduro en Python). En Rust no hay equivalente igual de maduro.

**Mitigación:** Cascada de fallbacks:
1. `serde_json::from_str()` (estricto — cubre el 80% de respuestas)
2. Strip markdown fences + retry
3. `llm_json` crate (fuzzy parser para LLMs)
4. Regex extraction de primer `{...}` o `[...]` + retry
5. Devolver error al agente para que el LLM regenere

### P3: MCP Client inmaduro (RIESGO MEDIO)

**Problema:** El SDK oficial de Rust (`rmcp` v0.15) es joven con API inestable.
Breaking changes entre versiones menores son esperables.

**Mitigación:**
- El uso de MCP en nanobot es simple: spawn stdio → initialize → list_tools → call_tool
- Podemos implementar el protocolo directamente (JSON-RPC 2.0 sobre stdio)
  si `rmcp` resulta problemático — son ~200 LOC de JSON-RPC
- Aislar MCP detrás de un trait para poder cambiar la implementación

### P4: Ownership y estado compartido (RIESGO MEDIO)

**Problema:** En Python, el bus, las sesiones y la memoria son objetos mutables
accesibles desde cualquier sitio. La traducción directa a Rust resulta en
`Arc<Mutex<T>>` por todos lados, que es ineficiente e inidiomatic.

**Puntos de estado compartido en nanobot:**
- `MessageBus` — dos queues (inbound/outbound)
- `SessionManager` — cache de sesiones + file I/O
- `SubagentManager` — running tasks dict
- `CronService` — job store
- `ChannelManager` — channel instances + outbound dispatch

**Mitigación:** Rediseñar ownership desde el principio:
- Bus: `tokio::sync::mpsc` — sender es `Clone`, receiver es owned. Sin mutex.
- Sessions: `Arc<RwLock<SessionManager>>` — read-heavy, write-rare.
- Tools: registrados una vez en init, luego read-only → `Arc<ToolRegistry>` sin lock.
- Channels: cada uno owned por su task, comunica via bus channels.
- Config: loaded once, `Arc<Config>` inmutable compartido.

### P5: Canales sin SDK de Rust (RIESGO BAJO si scope correcto)

**Problema:** Feishu, DingTalk, QQ, Mochat no tienen SDKs de Rust.

**Mitigación:** No portar para MVP. Scope del MVP:
- **Telegram** → `teloxide` (maduro, excelente)
- **Discord** → raw WebSocket (nanobot ya lo hace así)
- **Slack** → `slack-morphism` (aceptable)
- **Email** → `lettre` + `imap` crates
- **CLI** → `rustyline` (nuevo, no existe en nanobot como canal separado)

Esto cubre el 90%+ de usuarios occidentales. Los canales asiáticos son P3/backlog.

### P6: SSE Streaming de providers (RIESGO MEDIO)

**Problema:** Cada provider tiene quirks en su streaming SSE:
- OpenAI: `data: {"choices":[{"delta":{"content":"..."}}]}`
- Anthropic: `event: content_block_delta` + `data: {"delta":{"text":"..."}}`
- DeepSeek: `reasoning_content` en formato diferente al de Anthropic

**Mitigación:**
- `reqwest-eventsource` para el transporte SSE (retry automático)
- Parser por provider con trait `StreamParser`
- Tests diferenciales con fixtures de respuestas reales capturadas

### P7: Feishu threading model (RIESGO BAJO — skip MVP)

**Problema:** El SDK de Feishu en Python es síncrono y corre en un thread separado,
con bridge a asyncio via `run_coroutine_threadsafe`. Es un antipatrón.

**Mitigación:** Si se porta algún día, implementar directamente sobre HTTP/WebSocket
con tokio. El SDK de Python es un wrapper que simplifica auth y event subscription,
pero la API subyacente de Feishu está documentada.

## Problemas de proceso (metodológicos)

### P8: Alucinaciones de crates inventados (RIESGO ALTO)

**Problema conocido de Tokamak/bfclaude:** El LLM inventa APIs, crates y campos
que no existen. Genera código, tests y fixtures que validan la ficción.

**Mitigación:** Ver [Parte V: Estrategia anti-alucinaciones](#parte-v-estrategia-anti-alucinaciones).

### P9: Context exhaustion en sesiones largas (RIESGO MEDIO)

**Problema de vjeux:** En sesiones largas de porting, Claude pierde instrucciones.
Compactar contexto hace que olvide convenciones establecidas al principio.

**Mitigación:**
- `CLAUDE.md` del proyecto con reglas de porting explícitas (siempre en contexto)
- `.llm/CONVENTIONS.md` con patrones de código adoptados
- Sesiones cortas y enfocadas (un módulo por sesión)
- Session summaries al final de cada sesión

### P10: Claude intenta "mejorar" el código (RIESGO MEDIO)

**Problema de vjeux:** En vez de traducir 1:1, Claude optimiza, refactoriza y
añade features. Introduce bugs que no existían en el original.

**Mitigación:**
- Instrucción explícita: "traducción literal, sin optimizaciones"
- Fase 1: port fiel. Fase 2: rustificar. Fase 3: mejorar.
- Nunca portar y mejorar en la misma sesión.

### P11: Circular validation (RIESGO ALTO)

**El incidente de Tokamak:** LLM genera código + fixture + test que valida el fixture.
90 tests pasan validando ficción.

**Regla de oro:** El sistema de verificación debe ser externo al generador.
- LLM genera código Rust → OK
- Tests diferenciales contra nanobot Python → fuente de verdad independiente
- Fixtures capturadas de ejecución real → no generadas por LLM

---

# Parte IV: Estrategia de porte

## Principios (de vjeux + Tokamak + experiencia propia)

1. **File-by-file.** Un módulo por sesión de trabajo.
2. **Nunca portar y mejorar a la vez.** Primero port fiel → luego rustificar → luego mejorar.
3. **El compilador de Rust es tu TODO list.** Port → compile → fix errors → repeat.
4. **Tests diferenciales antes de implementación.** Portar tests primero.
5. **Instrucción explícita al LLM:** "traducción literal, sin optimizaciones."
6. **Verificar crates manualmente.** Cada crate que el LLM sugiera → comprobar en crates.io.
7. **Sesiones cortas.** Un módulo, máximo 90 minutos, luego summary y commit.
8. **Ownership rediseñado.** No traducir `Arc<Mutex>` ciego — pensar ownership desde cero.

## Fases

### Fase 0: Scaffolding (1-2 días)

- Crear org GitHub `rustyclaw` y repos
- `cargo init` con estructura de carpetas
- CI: `cargo check` + `cargo test` + `cargo clippy`
- README satírico
- `.llm/` context files (CONTEXT.md, CONVENTIONS.md, DECISIONS.md)
- `llm-usage.csv` vacío + script de registro
- GitHub Project board
- Configurar taxonomía `series` en Hugo (blog)

**Entregable:** Repo público con CI verde.
**Post:** Post 0 (Manifiesto)

### Fase 1: Core loop funcional (1-2 semanas)

| Módulo | LOC Python | LOC Rust est. | Orden |
|--------|-----------|---------------|-------|
| `config/schema` | 410 | ~300 | 1 |
| `bus/events` | 124 | ~60 | 2 |
| `session/manager` | 184 | ~150 | 3 |
| `tools/base + registry` | 175 | ~120 | 4 |
| `tools/filesystem` | 211 | ~180 | 5 |
| `tools/shell` | 144 | ~130 | 6 |
| `agent/context` | 238 | ~180 | 7 |
| `agent/memory` | 30 | ~25 | 8 |
| `agent/loop` | 476 | ~380 | 9 |
| CLI channel (nuevo) | 0 | ~100 | 10 |

**Milestone:** `rustyclaw agent` funciona en CLI con OpenRouter.
**Tests:** Diferenciales para context building y session serialization.
**Posts:** Post 2 (Message Bus), Post 3 (Agent Loop)

### Fase 2: Multi-provider + tools completos (1-2 semanas)

| Módulo | LOC Python | LOC Rust est. |
|--------|-----------|---------------|
| Provider router (OpenAI-compat) | 395+208 | ~300 |
| Anthropic provider | (nuevo) | ~150 |
| SSE streaming | (dentro de litellm) | ~100 |
| `tools/web` | 163 | ~140 |
| Memory consolidation | (dentro de loop) | ~100 |
| Skills loader | 228 | ~180 |
| Subagent manager | 257 | ~200 |

**Milestone:** Conversación con web search, file editing, memoria persistente.
**Post:** Post 4 (Reemplazando LiteLLM)

### Fase 3: Canales (1-2 semanas)

| Canal | LOC Python | LOC Rust est. | SDK Rust |
|-------|-----------|---------------|----------|
| Telegram | 390 | ~300 | `teloxide` |
| Discord | 261 | ~220 | raw WebSocket |
| Channel manager | 227 | ~150 | — |

**Milestone:** Bot de Telegram + Discord funcional con tool use.
**Post:** Post 5 (Telegram en Rust)

### Fase 4: MCP + Cron + Polish (1 semana)

| Módulo | LOC Python | LOC Rust est. |
|--------|-----------|---------------|
| MCP client | 80 | ~100 |
| Cron service | 410 | ~250 |
| Heartbeat | 130 | ~80 |
| CLI completo | 935 | ~600 |

**Milestone:** Paridad funcional con nanobot (sin canales asiáticos).

### Fase 5: Diferenciación (1 semana)

Mejoras que nanobot no tiene:

| Mejora | Impacto | Crate |
|--------|---------|-------|
| Parallel message processing | ALTO | `tokio::spawn` |
| Rate limiting | MEDIO | `governor` |
| Hot-reload skills | MEDIO | `notify` |
| Type-safe errors | MEDIO | `thiserror` |
| Structured logging | BAJO | `tracing` |
| Binary distribution | ALTO | `cargo-dist` |
| Minimal Docker | BAJO | `FROM scratch` |

**Milestone:** v0.1.0 release con benchmarks.
**Posts:** Post 6 (Benchmarks honestos), Post 7 (Retrospectiva)

### Estimación total

| Fase | Duración | LOC Python | LOC Rust est. |
|------|----------|------------|---------------|
| 0 | 1-2 días | 0 | ~200 |
| 1 | 1-2 semanas | ~2.000 | ~1.500 |
| 2 | 1-2 semanas | ~1.050 | ~900 |
| 3 | 1-2 semanas | ~900 | ~700 |
| 4 | 1 semana | ~1.550 | ~1.030 |
| 5 | 1 semana | 0 | ~500 |
| **Total** | **5-8 semanas** | **~5.500** | **~4.830** |

No portamos: canales asiáticos (~1.670 LOC), bridge WhatsApp Node.js.

---

# Parte V: Estrategia anti-alucinaciones

## Lecciones de Tokamak (bfclaude)

**El incidente:** Un LLM inventó un campo de API (`active_flags`) que no existe.
Generó DTOs, fixtures y tests que validaban ese campo fantasma. 90 tests pasaron
validando pura ficción. El código nunca crasheó — hacía lo incorrecto en silencio.

**Principio fundamental:**

> El sistema de verificación debe ser externo al generador.
> Si el LLM genera código, fixtures Y tests, estás validando ficción con ficción.

## 5 capas de defensa

### Capa 1: El compilador de Rust

El propio `rustc` elimina categorías enteras de bugs: null, type mismatch,
data races, use-after-free. **Pero:** que compile no significa lógica correcta.

### Capa 2: Tests diferenciales (fuente de verdad: nanobot Python)

```
Mismo input → nanobot (Python) → output_python
Mismo input → rustyclaw (Rust) → output_rust
assert output_python == output_rust
```

Categorías:
- Context building: mismo historial → mismo prompt
- Tool execution: mismo comando → mismo resultado
- Session serialization: misma sesión → mismo JSONL
- Config parsing: mismo JSON → misma estructura
- Provider routing: mismo model name → misma URL

### Capa 3: Provenance tracking

Cada archivo portado incluye header:
```rust
// Ported from: nanobot/agent/loop.py (lines 1-476)
// Ported by: Claude Opus 4.6 / session 2026-03-XX
// Verified: differential tests pass ✓
// Manual review: [fecha]
```

### Capa 4: Verificación de crates y APIs

Antes de usar cualquier crate que el LLM sugiera:
1. ¿Existe en crates.io? → verificar manualmente
2. ¿La API es como dice el LLM? → comprobar en docs.rs
3. ¿La versión existe? → verificar en crates.io

Script `verify-deps.sh`: parsea Cargo.toml y verifica cada crate contra crates.io API.

### Capa 5: Logging de incidentes

Cada alucinación = issue con label `hallucination`:
- Qué inventó el LLM
- Cómo se detectó
- Qué capa lo pilló
- Material para los posts del blog

## Checklist pre-merge

```
[ ] Compila sin warnings (`cargo clippy`)
[ ] Tests unitarios pasan
[ ] Tests diferenciales contra nanobot Python pasan
[ ] Crates verificados en crates.io/docs.rs
[ ] APIs verificadas contra documentación real
[ ] Provenance header presente
[ ] Review manual (ejecución mental, no solo lectura)
[ ] Issue actualizado con métricas
```

---

# Parte VI: Relay entre LLMs

## Artefactos de estado persistente

```
rustyclaw/.llm/
├── CONTEXT.md       Estado actual del proyecto para onboarding rápido
├── CONVENTIONS.md   Patrones de código adoptados
├── DECISIONS.md     ADRs (Architecture Decision Records)
├── KNOWN-ISSUES.md  Problemas pendientes
└── sessions/
    ├── 2026-03-05-claude-session1.md
    └── 2026-03-06-codex-session1.md
```

## Protocolo de cambio de LLM

```
1. Sesión termina:
   └── LLM actualiza CONTEXT.md y DECISIONS.md
   └── Escribe resumen en sessions/YYYY-MM-DD-{llm}-sessionN.md
   └── Commit: "chore: update LLM context after session N"

2. Sesión nueva con otro LLM empieza:
   └── Lee CONTEXT.md, DECISIONS.md, CONVENTIONS.md
   └── Lee último session summary
   └── Continúa desde donde se dejó
```

## CONTEXT.md — Template

```markdown
# RustyClaw - Contexto actual

## Última sesión
- **Fecha:** YYYY-MM-DD
- **LLM:** Claude Opus / Codex
- **Módulo:** X
- **Estado:** [compila|tests pasan|en progreso]

## Qué funciona
- [x] Config loader
- [x] Message bus
- [ ] Agent loop (en progreso)

## Próximo paso
Completar X, luego empezar Y.

## Decisiones recientes
- ADR-NNN: ...

## Trampas conocidas
- crate X no soporta Y (issue #ZZZ)
```

## Diferencias Claude vs Codex

| Aspecto | Claude | Codex CLI |
|---------|--------|-----------|
| **Fortaleza** | Arquitectura, diseño, idiomático | Iteración rápida, debugging |
| **Debilidad** | Sobre-diseña, verbose | Menos contexto de proyecto |
| **Alucinaciones** | Inventa APIs de crates | Inventa flags de CLI |
| **Mejor para** | Porte de módulos completos | Fixes puntuales, tests, boilerplate |

**Regla:** Claude para diseño y módulos. Codex para debugging y muscle work.

---

# Parte VII: Tracking de consumo

## Métricas por sesión

| Métrica | Fuente |
|---------|--------|
| LLM + modelo | Manual |
| Tokens in/out | `/cost` en Claude Code, logs de Codex |
| Coste estimado ($) | Calculado según pricing |
| Duración (min) | Timestamp inicio/fin |
| Módulo trabajado | Manual |
| LOC Python portadas | Conteo del original |
| LOC Rust generadas | Conteo del resultado |
| Alucinaciones | Conteo + ref a issue |
| Tests passing | Después de sesión |

## Formato

`llm-usage.csv` en el repo:

```csv
date,llm,model,module,tokens_in,tokens_out,cost_usd,duration_min,loc_python,loc_rust,hallucinations,tests_pass
2026-03-05,claude,opus-4.6,agent/loop,45000,12000,1.20,90,476,380,1,12/15
```

## Dashboard

El CSV se renderiza como tabla en el README con totales acumulados.
Material para Post 6 (Benchmarks) y Post 7 (Retrospectiva).

---

# Parte VIII: Serie de blog

## Configuración técnica

Añadir taxonomía `series` a Hugo. Cada post lleva:
```yaml
series: ["RustyClaw: Rewrite It In Rust"]
```

Crear partial `series_nav.html` para navegación prev/next.
El blog genera automáticamente página índice de la serie.

## Posts planificados

### Post 0: "Manifiesto: Por qué voy a reescribir un agente de IA en Rust sin saber Rust"

Establece premisa, tono, hook. Qué es OpenClaw/nanobot, por qué Rust,
qué mejoras concretas esperamos, la tabla "Does it matter?", confesión de
ignorancia total en Rust, enlace al backlog público.

**Distribución:** HN, r/rust, r/programming, Twitter, LinkedIn

### Post 1: "El Zen de Rust para Pythonistas, Swifties y TypeScripters"

Tutorial conceptual. El axioma AXM (Aliasing XOR Mutation). Las 7 ausencias.
Marker traits como dunders. Tabla de analogías 4 lenguajes. La analogía de
la mudanza. Ejercicio side-by-side en 4 lenguajes.

**Distribución:** HN, r/rust, r/python, Dev.to

### Post 2: "El Message Bus (o cómo tokio::mpsc me hizo sentir tonto)"

Primera experiencia real. 124 LOC Python → ~60 Rust. asyncio.Queue → mpsc.
Primera pelea con borrow checker. Arc<Sender> y el ahá. Datos reales de consumo.

### Post 3: "El Agent Loop: 476 líneas de Python → ??? de Rust"

Corazón del agente. Dynamic dispatch. Error handling strings → Result.
"En Python eran 3 líneas" vs "ese bug llevaba 2 meses". Tests diferenciales.

### Post 4: "Reemplazando LiteLLM (o cómo aprendí a amar reqwest)"

Dependencia más crítica. Provider router en ~300 LOC. SSE streaming.
JSON fuzzy parsing. "Mi router tiene 300 LOC. LiteLLM tiene 50.000."

### Post 5: "Telegram en Rust (teloxide es una maravilla)"

Primer resultado visible. Screenshot del bot. Pattern matching.
"Menos RAM que el tab de Chrome donde leí cómo hacerlo."

### Post 6: "Benchmarks honestos (spoiler: no importan)"

**La pieza más viral.** Tabla completa con "Does it matter?".
Lo que SÍ mejoró. Datos de consumo de LLM. Bugs encontrados.

**Distribución:** HN (máxima tracción esperada)

### Post 7: "Retrospectiva: ¿Valió la pena?"

Cierre honesto. Coste total. Tasa de alucinaciones. Qué aprendí.
"RIIR es un meme. Pero a veces el meme tiene razón."

## Elementos recurrentes

1. **Cita de Don Cangrejo** — cada post abre con una cita ficticia sobre Rust
2. **Tabla "Does it matter?"** — running gag en cada benchmark
3. **Datos reales de consumo** — tokens, coste, duración al final de cada post
4. **Counter de alucinaciones** — "Alucinaciones hasta ahora: N"
5. **Side-by-side code** — Python vs Rust sin editar para favorecer
6. **Honestidad radical** — "esto tardé 3h en Rust, en Python serían 10min"

## Distribución

| Plataforma | Formato | Cuándo |
|-----------|---------|--------|
| frr.dev/keepcoding | Post largo (2000-3000 palabras) | Con cada avance |
| Twitter/X | Thread highlights + memes | Con cada post |
| HN | Submit principal | Posts 0, 1, 6, 7 |
| Reddit r/rust | Cross-post | Posts 1, 5, 6 |
| Dev.to | Cross-post | Todos |
| GitHub | README + releases | Continuo |

---

# Parte IX: Backlog público

## GitHub Project board

**Board:** "RustyClaw Porting Tracker"
**Columnas:** Backlog → Ready → In Progress → Review → Done

## Labels

| Label | Uso |
|-------|-----|
| `phase-0` a `phase-5` | Fase de porte |
| `blog-post` | Tiene post asociado |
| `benchmark` | Requiere medición |
| `hallucination` | Alucinación detectada |
| `improvement` | Mejora sobre nanobot |
| `good-first-issue` | Para futuros contribuidores |

## Issues iniciales por milestone

### Phase 0 - Scaffolding
- Crear org GitHub y repos
- Cargo init con estructura
- CI: check + test + clippy
- README satírico
- .llm/ context files
- llm-usage.csv + script
- GitHub Project board
- Configurar series en Hugo

### Phase 1 - Core Loop
- Port: config/schema
- Port: bus/events
- Port: session/manager
- Port: tools/base + registry
- Port: tools/filesystem
- Port: tools/shell
- Port: agent/context
- Port: agent/memory
- Port: agent/loop
- New: CLI channel
- Tests diferenciales: context
- Tests diferenciales: sessions

### Phase 2 - Providers + Tools
- Port: provider router
- Port: Anthropic provider
- Port: SSE streaming
- Port: tools/web
- Port: memory consolidation
- Port: skills loader
- Port: subagent manager

### Phase 3 - Channels
- Port: Telegram (teloxide)
- Port: Discord (WebSocket)
- Port: channel manager
- Typing indicators + media

### Phase 4 - MCP + Polish
- Port: MCP client
- Port: cron service
- Port: heartbeat
- Port: CLI completo

### Phase 5 - Differentiation
- Parallel message processing
- Rate limiting
- Hot-reload skills
- Benchmarks comparativos
- Release binaries
- Docker minimal

---

# Parte X: Crates de Rust

## Cargo.toml MVP

```toml
[package]
name = "rustyclaw"
version = "0.1.0"
edition = "2021"
description = "OpenClaw, mass-rewritten in Rust. Because the crab demanded it."
license = "MIT"
repository = "https://github.com/rustyclaw/rustyclaw"

[dependencies]
# Async runtime (obligatorio — todo el ecosistema depende de tokio)
tokio = { version = "1", features = ["full"] }

# HTTP + SSE streaming
reqwest = { version = "0.12", features = ["json", "stream", "rustls-tls"], default-features = false }
reqwest-eventsource = "0.6"

# LLM APIs
async-openai = "0.26"          # OpenAI/OpenRouter/DeepSeek/Groq (compatible)
# Anthropic: via reqwest directo (no hay SDK maduro oficial)

# MCP (Model Context Protocol)
rmcp = "0.15"                  # SDK oficial, joven pero funcional

# Telegram
teloxide = { version = "0.17", features = ["macros"] }

# Discord (raw WebSocket)
tokio-tungstenite = { version = "0.26", features = ["rustls-tls-webpki-roots"] }

# JSON
serde = { version = "1", features = ["derive"] }
serde_json = "1"
# llm_json — activar cuando necesitemos fuzzy JSON

# CLI
clap = { version = "4", features = ["derive"] }
rustyline = "15"

# Config
figment = { version = "0.10", features = ["toml", "json", "env"] }

# Logging
tracing = "0.1"
tracing-subscriber = { version = "0.3", features = ["env-filter"] }

# Scheduling
tokio-cron-scheduler = "0.15"

# Web scraping
scraper = "0.22"

# Markdown
pulldown-cmark = "0.13"

# Templates (prompts)
minijinja = "2"

# File watching
notify = "8"

# Error handling
anyhow = "2"
thiserror = "2"

# Misc
regex = "1"
chrono = { version = "0.4", features = ["serde"] }
uuid = { version = "1", features = ["v4"] }
dirs = "6"
```

## Mapa de equivalencias detallado

| Categoría | Python | Rust crate | Madurez | Gotchas |
|-----------|--------|------------|---------|---------|
| HTTP async | `httpx` | `reqwest` | Producción | Usar `rustls-tls` no `native-tls` |
| SSE | (dentro de litellm) | `reqwest-eventsource` | Estable | Reconexión automática incluida |
| Telegram | `python-telegram-bot` | `teloxide` | Alta | Curva de aprendizaje pronunciada |
| Discord | `websockets` (raw) | `tokio-tungstenite` | Alta | Sin reconexión auto — implementar |
| Slack | `slack-sdk` | `slack-morphism` | Media | Único SDK serio en Rust |
| JSON estricto | `json` | `serde_json` | Producción | 614M+ descargas |
| JSON fuzzy | `json_repair` | `llm_json` | Joven | Fallback: strip fences + regex |
| CLI | `typer` | `clap` | Producción | Derive macros para structs |
| Config | `pydantic-settings` | `figment` | Alta | Provenance tracking en errores |
| Logging | `loguru` | `tracing` | Producción | Spans async — esencial para tokio |
| Cron | `croniter` | `tokio-cron-scheduler` | Media | API cambia entre versiones |
| Markdown | (no usa) | `pulldown-cmark` | Producción | Usado en cargo doc |
| Templates | (f-strings) | `minijinja` | Alta | Autor de Jinja2 original |
| Errors | strings | `anyhow` + `thiserror` | Producción | anyhow en app, thiserror en libs |
| WebSocket | `websockets` | `tokio-tungstenite` | Alta | Sin reconexión auto |
| MCP | `mcp` | `rmcp` | Joven | Breaking changes esperables |
| Process exec | `subprocess` | `tokio::process` | Producción | kill_on_drop es best-effort |
| File watch | (no usa) | `notify` | Producción | Usado en rust-analyzer |
| Rate limit | (no tiene) | `governor` | Alta | Nueva mejora |
| Multi-provider | `litellm` | **No existe** | — | Implementar propio (~300 LOC) |

---

# Próximos pasos (cuando arranquemos)

1. Crear org `rustyclaw` en GitHub
2. Crear repo con scaffold
3. Escribir Post 0 (Manifiesto)
4. Configurar series en Hugo
5. Empezar Fase 1

---

*Documento consolidado: 2026-02-16*
*Inicio estimado: primera semana de marzo 2026*
