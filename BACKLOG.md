# Minutes — Backlog

| # | Item | LOE | Theme | Status |
|---|------|-----|-------|--------|
| 1 | **Rust workspace scaffold + core transcription pipeline** | L | Core | SHIPPED 2026-03-17 |
| 2 | **MCP server + Claude Code plugin (initial)** | L | MCP | SHIPPED 2026-03-17 |
| 3 | **Tauri v2 menu bar app (scaffold + compiles)** | L | Desktop | SHIPPED 2026-03-18 |
| 4 | **First-run onboarding wizard + calendar integration in Tauri** | M | Desktop | SHIPPED 2026-03-18 |
| 5 | **Meeting lifecycle skills (prep, debrief, weekly) + MCP dashboard** | M | MCP | SHIPPED 2026-03-19 |
| 6 | **Windows / Linux cross-platform support + signed macOS release CI** | M | Infra | SHIPPED 2026-03-19 |
| 7 | **QMD semantic search for meeting lookup (v0.2.0)** | M | Core | SHIPPED 2026-03-20 |
| 8 | **Vault sync for Obsidian/Logseq + meeting-prompt overlay (v0.3.0)** | M | Core | SHIPPED 2026-03-21 |
| 9 | **`npx minutes-mcp` + pure-TS reader + minutes-sdk npm package (v0.5.0)** | M | SDK | SHIPPED 2026-03-22 |
| 10 | **Claude Code plugin marketplace distribution (v0.6.0)** | S | MCP | SHIPPED 2026-03-24 |
| 11 | **Dictation mode + streaming whisper (text appears as you speak) (v0.6.0)** | L | Voice | SHIPPED 2026-03-24 |
| 12 | **Cross-device ghost context — phone voice memos become AI memory (v0.7.0)** | L | Voice | SHIPPED 2026-03-24 |
| 13 | **Agent memory SDK (`minutes-sdk@0.7.1`)** | M | SDK | SHIPPED 2026-03-24 |
| 14 | **Live transcript + agent-agnostic coaching context (v0.8.1)** | M | Desktop | SHIPPED 2026-03-28 |
| 15 | **whisper-guard crate + nnnoiseless noise reduction (published to crates.io)** | M | Core | SHIPPED 2026-03-27 |
| 16 | **Meeting Intelligence Dashboard (`minutes dashboard`) + silence detection** | M | Core | SHIPPED 2026-03-30 |
| 17 | **Subscribable meeting insights (typed events, confidence model, MCP tool)** | M | MCP | SHIPPED 2026-03-30 |
| 18 | **Parakeet ASR opt-in transcription engine** | M | Core | SHIPPED 2026-03-27 |
| 19 | **Call capture + detection flow + `minutes service` install/uninstall (v0.9.x)** | M | Desktop | SHIPPED 2026-04-02 |
| 20 | **`minutes-cli` on crates.io — `cargo install minutes-cli` (v0.10.0)** | S | Infra | SHIPPED 2026-04-05 |
| 21 | **Knowledge base integration — Karpathy-style wiki from meeting data (v0.10.x)** | L | Core | SHIPPED 2026-04-06 |
| 22 | **Plugin v0.8.0 — brief, mirror, tag, graph skills + MCPB marketplace (v0.11.0)** | M | MCP | SHIPPED 2026-04-08 |
| 23 | **Recall panel + artifact workspace bridge (v0.11.1)** | M | Desktop | SHIPPED 2026-04-09 |
| 24 | **Lifecycle nudges + multi-agent sync + service management UI (v0.11.2)** | M | Desktop | SHIPPED 2026-04-10 |
| 25 | **Identity card (name, emails, aliases) + self-name correction + settings overhaul (v0.13.0)** | M | Core | SHIPPED 2026-04-16 |
| 26 | **Speaker overlay propagation across desktop, CLI, and MCP (v0.13.x)** | M | Core | SHIPPED 2026-04-24 |
| 27 | **OpenAI-compatible summarization backend + keychain gateway setup (v0.14.x)** | M | Core | SHIPPED 2026-04-25 |
| 28 | **MCP auto-install CLI + same-major compatibility check (v0.14.0)** | S | MCP | SHIPPED 2026-04-23 |
| 29 | **SQLite FTS5-backed search index (v0.15.0)** | L | Core | SHIPPED 2026-04-27 |
| 30 | **Agent event bus — recording lifecycle + meeting insight events (v0.16.0)** | M | MCP | SHIPPED 2026-04-29 |
| 31 | **First-class templates system RFC 0001 Phase 1 (v0.16.0)** | M | Core | SHIPPED 2026-04-29 |
| 32 | **macOS permission center + readiness probes (v0.16.1)** | M | Desktop | SHIPPED 2026-04-30 |
| 33 | **Bundle CLI inside Minutes.app for zero-config auto-update (v0.16.1)** | M | Desktop | SHIPPED 2026-05-04 |
| 34 | **Dictation multi-backend: Apple Speech, Parakeet final, local recents memory (v0.16.x)** | M | Voice | SHIPPED 2026-05-01 |
| 35 | **Core Audio process tap backend for system audio capture (v0.16.2)** | L | Core | SHIPPED 2026-05-06 |
| 36 | **Streaming Silero VAD via ORT (ort-silero becomes default engine) (v0.17.0)** | L | Core | SHIPPED 2026-05-10 |
| 37 | **Post-stop crash recovery + bounded retry + truncated model detection (v0.17.x)** | M | Core | SHIPPED 2026-05-11 |
| 38 | **Voice Memos audio probe + WAV stem-mix pipeline fix (v0.17.3 / v0.18.0)** | M | Core | SHIPPED 2026-05-13 |
| 39 | **Audio retention planner + cleanup CLI** | S | Core | SHIPPED 2026-05-12 |
| 40 | **whisper-guard hallucination phrase stripping + noise marker detection (v0.18.0)** | S | Core | SHIPPED 2026-05-13 |
| 41 | **Dictation shortcut settings simplification (merge two shortcut systems into one unified dropdown)** | M | Voice | QUEUED — top follow-up |
| 42 | **Full ambient memory — LLM auto-classification + intent extraction on voice memos** | L | Voice | parked |
| 43 | **Weekly synthesis as first-class Recall panel view** | M | Desktop | parked |
| 44 | **WASM compilation of minutes-reader for SDK (eliminate TS/Rust parser divergence)** | M | SDK | parked |
| 45 | **Multi-thread per-meeting conversation history (each meeting gets its own thread)** | L | Desktop | parked |
| 46 | **Post-process [[name]] links for person slugs in knowledge.rs** | S | Core | parked |
| 47 | **Voice Memos watch.rs real-device fixture (replace placeholder TODO)** | S | Core | parked |

## Components

- **core** (Core): Rust crate — transcription pipeline, diarization, summarization, VAD, knowledge base
- **cli** (Core): `minutes` binary — record, watch, search, dashboard, service management
- **reader** (SDK): Rust crate — frontmatter + artifact parsing (no audio deps; crates.io published)
- **whisper-guard** (Core): standalone Rust crate — noise reduction, hallucination filtering (crates.io published)
- **mcp** (MCP): TypeScript MCP server — `npx minutes-mcp`, MCPB marketplace, event streaming
- **sdk** (SDK): `minutes-sdk` npm package — agent memory API, humanizeTranscript, overlay-aware parsing
- **tauri/src-tauri** (Desktop): Tauri v2 menu bar app — recording, recall panel, settings, onboarding
- **site** (Infra): Next.js landing page at useminutes.app
- **.agents / .claude-plugin** (MCP): Claude Code plugin skills — brief, mirror, tag, graph, prep, debrief, weekly

## Minor follow-ups

- knowledge.rs: post-process to add `[[name]]` links for any person slugs mentioned in fact text (TODO at line 520)
- watch.rs: add a real Voice Memos audio fixture for the watch probe test (TODO at line 738)
