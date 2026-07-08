# Minutes IA and Agent UX Plan - 2026-06-02

> Canonical merged plan from the Codex IA pass and Claude's independent
> `PLAN.ia-claude.md` draft. Goal: make Minutes feel coherent for humans,
> terminal users, and agents without turning a local-first tool into redesign
> theater.

## Verdict

Minutes does not primarily have a navigation problem. It has a shared-model
problem.

The product already spans a Tauri desktop app, CLI, MCP server, command palette,
portable skills, local markdown files, live transcript streams, and an
append-only event log. Adding a larger sidebar would only make that sprawl more
visible. The right fix is to name the durable object model once, then project it
consistently into human desktop UX, CLI/palette commands, and MCP/agent tools.

The human-facing desktop projection should use four zones:

- **Capture** - get conversation into Minutes.
- **Review** - decide whether the latest capture worked and what needs action.
- **Recall** - search and reason across the conversation corpus.
- **Health** - keep the local system ready and understandable.

The agent-facing projection is the control plane: MCP tools/resources/prompts,
CLI commands, portable skills, command-palette actions, and the event bus. It
should not become a fifth top-level app tab unless a concrete UI requires it.

## Constraint Set

The plan must satisfy these constraints at the same time:

- **Local-first trust:** meeting markdown, event logs, graph data, and recovery
  artifacts remain inspectable and local.
- **Human and agent coherence:** desktop, CLI, MCP, and skills must describe the
  same objects and states.
- **No breaking CLI churn:** command regrouping should be conceptual and
  discoverable before any namespace migration is considered.
- **Failure visibility:** blank transcripts, audio risk, missing system audio,
  backend readiness, and low confidence cannot be silent.
- **Agent safety:** model-driven actions need authority tiers and provenance,
  especially when starting capture, mutating local state, or causing external
  effects.
- **Small first slice:** write the spec, redesign the default home, then build
  proof through scenarios. Visual polish comes after the IA spine.
- **Repo substrate reuse:** prefer existing `events.rs`, MCP resources,
  dashboard UI, command registry, and frontmatter data over parallel systems.

## Product Model

The durable IA is object-first. Zones are only projections.

| Object | Meaning | Existing substrate |
| --- | --- | --- |
| CaptureSession | Recording, live transcript, dictation, watch/import item in progress or recently finished | `capture.rs`, `live_transcript.rs`, `dictation.rs`, `watch.rs`, `pid.rs`, jobs |
| Transcript | Raw, cleaned, segmented, or live text | transcription coordinator, `transcribe.rs`, live JSONL, whisper/parakeet/apple-speech paths |
| MeetingArtifact | Markdown meeting/memo/dictation artifact with frontmatter | pipeline, templates, reader/search, `~/meetings/` |
| Extraction | Action item, decision, commitment, open question, insight | frontmatter, graph, insights, events |
| PersonOrSpeaker | Speaker identity, participant profile, voice confirmation | diarization, speaker map, people/graph surfaces |
| GraphMemory | Cross-meeting people, commitments, consistency, relationship intelligence | `graph.db`, graph/search/person commands |
| Readiness | Device, model, backend, permissions, disk, updater, platform support | health, setup, sources/devices, macOS permission flows |
| Template | Reusable extraction and summary shape for capture/review | bundled templates, project/user templates, RFC 0001 |
| Event | Append-only status/provenance record | `~/.minutes/events.jsonl`, `events.rs`, MCP event resources |
| Integration | Bridge to an external system (knowledge base, automation, vault, web research) | `qmd`, `vault`, `automate`, `autoresearch`, MCP/skill connectors |

Canonical verbs:

- `capture`
- `review`
- `recall`
- `annotate`
- `extract`
- `confirm`
- `export`
- `diagnose`

IA litmus test: every CLI command, MCP tool, palette action, and skill should
reduce to `<verb> <object>`. If it cannot, either the object model is missing a
real concept or the surface is product debt. Three lanes sit intentionally
outside the four human zones: `integration` (external systems - `qmd`, `vault`,
`automate`, `autoresearch`), `introspection` (agent-facing `schema`,
`capabilities`), and `internal` (plumbing - `parakeet-helper`,
`preflight-record`, `apple-speech`, `decode-hint-eval`). A surface is clean if it
is `<verb> <object>` in a zone, or explicitly tagged one of those three lanes.

## Surface Projections

| Object | Desktop zone | CLI/palette projection | MCP/agent projection |
| --- | --- | --- | --- |
| CaptureSession | Capture, Review | `record`, `live`, `dictate`, `stop`, `process`, `watch`, `note` | start/stop/status/process tools |
| Transcript | Review | `get`, `clean`, `process`, live reads | transcript resources, `read_live_transcript` |
| MeetingArtifact | Review, Recall | `list`, `search`, `export`, artifact commands | meeting resources, dashboard/detail UI |
| Extraction | Review, Recall | `actions`, `commitments`, `insights`, `consistency` | structured insight tools/resources |
| PersonOrSpeaker | Recall | `people`, `person`, `confirm`, `enroll`, `voices` | people/profile tools |
| GraphMemory | Recall | `graph`, `research`, `consistency` | graph/relationship memory tools |
| Readiness | Health | `health`, `setup`, `devices`, `sources`, `paths`, `storage` | health/status resources and safe storage tools |
| Template | Health/config, Capture | `template list`, `template show`, capture/process template flags | prompt/template guidance for agent workflows |
| Event | Review/control plane | `events`, `agent-annotate`, command history | live/recent events resources, agent annotation tools, provenance timeline |
| Integration | Health/integrations (not a core human zone) | `qmd`, `vault`, `automate`, `autoresearch` | external-system connectors; External-effect authority |

## Human Desktop IA

The desktop app should become a compact control room, not a feature catalog.

### Default Home Questions

The landing surface should be labeled **Now**, not **Review** or **Today**.
Review is the trust semantic and zone name; Now is the user-facing dashboard
that leads with live capture state plus anything needing attention. Today is too
date-scoped: needs-review items can span days.

The first screen should answer these in order:

1. Am I capturing, listening, processing, or idle?
2. Is anything broken, risky, or waiting for review?
3. What did I just capture?
4. What can I recall/search next?
5. Is the system ready for the next meeting?

### Zone Responsibilities

**Capture**

- Record meeting.
- Start live transcript.
- Dictate / Quick Thought.
- Process/import audio.
- Watch folder status.
- Show autonomy level: manual, live, background, or agent-controlled.

**Review**

- Recent captures.
- Per-capture trust state.
- Transcript, summary, action items, artifacts.
- Recovery paths for failed or incomplete captures.
- Event/provenance timeline.

First-class review states:

- `idle`
- `recording`
- `listening`
- `processing`
- `complete`
- `artifact-ready`
- `needs-review`
- `audio-risk`
- `system-audio-likely-missing`
- `blank-or-low-confidence`
- `backend-unavailable`
- `failed-with-recovery`

**Recall**

- Search.
- People.
- Commitments.
- Decisions.
- Consistency.
- Relationship graph.
- Assistant workspace.

Recall should feel like a real mode, not only an embedded terminal. The
terminal can remain a power-user surface inside it.

**Health**

- Mic/device readiness.
- System-audio/call-capture guidance.
- Model/backend readiness.
- Permissions.
- Storage/retention.
- Updates.
- Integrations.

Health should be boring on purpose. It prevents surprise.

## Event Bus as Trust and Provenance

The strongest unifying move is to render the event bus as the shared surface.

Minutes already has an append-only local event log at `~/.minutes/events.jsonl`.
`events.rs` includes `recording.started`, `recording.completed`,
`live.utterance.final`, and `agent.annotation`. The MCP server exposes recent,
live, and agent-annotation event resources, and the CLI has `minutes events`.

That means two UX needs collapse into one implementation path:

- Human trust center: "Did this capture work? What is risky? What should I do?"
- Agent provenance drawer: "What did the agent read, run, write, or request?"

Design one event-rendering component with filters:

- `trust`: capture health, processing, confidence, recovery, artifacts.
- `agent`: tool calls, resource reads, annotations, permission tier, approvals.
- `raw`: chronological local event stream for debugging.

This is better than building a separate "Review" surface and "Agent activity"
surface. They are the same timeline seen through different lenses.

Slice 2 should avoid event-type sprawl. The first Review timeline only needs a
small event contract:

- Derived states (`idle`/`recording`/`listening`/`processing`/`complete`) come
  from PID + job queue + artifact presence; no new events needed.
- Event-worthy facts extend the existing `recording.*`/`meeting.*` families
  (the `events.rs` convention is `noun.pastVerb`): `recording.flagged` with
  `kind: audio_risk | system_audio_missing | low_confidence | blank`,
  `recording.failed` with `{ recoverable, reason }`, and `meeting.ready`.
- `processing.failed` with `recoverable` and `reason`.
- `artifact.ready` when a user-visible artifact is available.

The 12 review states are state enum values, not 12 event variants. That keeps
the append-only log readable and preserves the small allowlist discipline.

## Agent IA and Authority Model

Agents should operate through structured, least-surprise authority tiers.

| Tier | Allowed behavior | Examples |
| --- | --- | --- |
| Read | Inspect local memory without mutation | search meetings, read transcript, read graph/person profile |
| Draft | Create new artifacts without changing canonical meeting state | debrief memo, weekly summary, proposed tags |
| Annotate | Append attributed event/provenance without editing human markdown | `agent.annotation`, confidence note, source note |
| Mutate local state | Change metadata or review state | mark reviewed, confirm speaker, apply tag |
| Capture control | Start/stop live or recording workflows | start recording, stop live transcript |
| External effect | Affect outside systems or delete/publish data | send/export/upload/delete raw audio |

Rules:

- Read and Draft can be low-friction.
- Annotate must be attributed and append-only.
- Mutate local state needs visible undo or review.
- Capture control needs obvious live state in the desktop app.
- External effects require explicit human confirmation and durable audit events.
- The local allowlist file remains the source of truth. Desktop UI may view or
  edit it, but CLI/MCP/headless agents need the same inspectable authority model
  without the app running.

## MCP Apps and Component Contracts

Research from current MCP Apps and OpenAI Apps SDK guidance points to the same
thing: interactive agent UI works best when tools return structured data plus a
portable component contract.

Relevant source anchors:

- MCP Apps are now an official MCP extension where tools can return interactive
  `ui://` resources rendered in sandboxed iframes, with auditable JSON-RPC and
  optional user consent for UI-initiated tool calls:
  <https://blog.modelcontextprotocol.io/posts/2026-01-26-mcp-apps/>
- OpenAI Apps SDK guidance says components are the human-visible half of the
  connector, and that a well-designed component/data contract can be portable
  across MCP Apps-compatible hosts:
  <https://developers.openai.com/apps-sdk/plan/components>
- MCP tools are model-controlled actions, which reinforces the need to separate
  tools from resources/prompts and pair them with explicit authority:
  <https://modelcontextprotocol.io/specification/2025-06-18/server/tools>
- NN/g's AI UI paradigm framing supports hybrid interfaces: intent-based AI,
  GUI, and command interfaces coexist rather than one replacing the others:
  <https://www.nngroup.com/articles/ai-paradigm/>

Minutes already has an MCP App dashboard under `crates/mcp/ui/` and the README
describes an interactive dashboard resource. The convergence path is contract
canonical, components per surface:

1. Define Review component JSON contracts first.
2. Reuse those contracts in MCP dashboard/tool results.
3. Render separate MCP and Tauri components against the same contract.
4. Keep a text-only fallback for clients without interactive UI.

This prevents building two divergent product models while avoiding brittle
component coupling across different runtimes. MCP Apps hosts render sandboxed
iframes over `postMessage`; Tauri renders a local webview over `cmd_*` IPC. The
payload contract should be portable even when the implementation is not.

## First Slice

Do not implement everything at once.

### Slice 0 - Spec and Inventory

- This document becomes the source-of-truth plan.
- Build a surface/action/state inventory for:
  - `tauri/src/index.html`
  - `crates/core/src/palette.rs`
  - `crates/cli/src/main.rs`
  - `crates/mcp/src/index.ts`
  - `.agents/skills/minutes/`
  - `.opencode/commands/`
- For each surface, map to object, verb, zone, review state, and authority tier.

### Slice 1 - Desktop Default Home

Scope: `tauri/src/index.html` only, unless a command already exists and needs
minor data plumbing.

Home layout:

- Now strip: capture state, timer/status, primary action.
- Review queue: risky/recent captures first, not a passive archive.
- Recent captures: complete artifacts with clear status.
- Recall entry: search/people/commitments shortcut.
- Health chip: next-meeting readiness and setup risks.

The crowded footer should become secondary command access. Capture remains
prominent, but Review becomes the default product spine.

Dictation history belongs in Review immediately as a surfaced capture artifact:
the user should see what was captured and find it later. The dictation-specific
interaction fixes, such as partial text and success preview, remain in
`PLAN.dictation-ux.md`.

**Constraint - preserve the v0.18.5 idempotent-render baseline (perf handoff).**
Slice 1 shares `tauri/src/index.html` with the v0.18.5 perf fix, which replaced
raw DOM writes with idempotent helpers (`setText`, `setDisabled`, `setClassName`,
`clearChildren`) plus change-guards, so the ~2s status poll stops mutating the
DOM when nothing changed. On the translucent ("glass" / Liquid Glass) window
every redundant repaint forces a full-window re-blur - the WindowServer jank
v0.18.5 fixes. A fresh "Now" render that reverts to `el.textContent = ...` or
unconditional `replaceChildren()` reintroduces that bug. Therefore:

1. Build Slice 1 on top of the committed v0.18.5 baseline, not concurrently in
   the shared worktree.
2. Every "Now"/Review status update routes through the idempotent helpers and
   touches the DOM only when the value actually changes (idle = zero repaints).

### Slice 2 - Review Timeline

- Render event-derived status for a selected capture.
- Add human/agent/raw filters.
- Show recovery actions beside failure states.
- Use the small event contract aligned to the existing `events.rs` convention:
  `recording.flagged{kind}`, `recording.failed{recoverable, reason}`, and
  `meeting.ready`; derive `idle`/`recording`/`processing`/`complete` from PID +
  jobs + artifact presence rather than emitting them.
- Keep fallback text for CLI/MCP.

### Slice 3 - MCP/Component Contract

- Define structured payloads for Review queue and capture detail.
- Ensure MCP dashboard and desktop consume the same canonical JSON shape.
- Use `ui://` resources where the current MCP Apps path supports it.

### Slice 4 - CLI and Palette Grouping

- Add conceptual grouping in help/palette sections.
- Avoid breaking command renames.
- Add docs mapping command -> object -> zone.

### Slice 5 - Visual Polish

- Apply `PLAN-tauri-design-pass.md` once the IA shell is stable.
- Verify in `~/Applications/Minutes Dev.app` for TCC-sensitive surfaces.

## Acceptance Scenarios

Goal complete means these scenarios are true, not merely documented:

1. First-time user can record a call and understand system-audio limits before
   losing the meeting.
2. User dictates a thought, sees what was captured, and finds it later.
3. Failed capture is recoverable, visible, and explainable.
4. Low-confidence or blank transcript is flagged instead of silently accepted.
5. Agent can answer "what did I promise Sarah?" from commitments/graph/history.
6. Agent can start, read, and stop live transcript without hidden state.
7. Human can inspect what an agent did: tool, resource, artifact, authority tier,
   and approval.
8. Review component works in the desktop app and degrades gracefully in MCP text
   clients.
9. CLI and command palette can be explained through the same object/verb model.
10. Visual polish supports the model instead of masking conceptual sprawl.
11. The "Now"/Review home performs no idle repaints: with capture state
    unchanged, the ~2s poll mutates no DOM, preserving the v0.18.5
    idempotent-render baseline on the translucent window.

## Path to 10/10

| Stage | Output | Excellence bar |
| --- | --- | --- |
| Spec | Canonical IA and agent UX plan | Shared object model is explicit and repo-grounded |
| Inventory | Surface/action/state map | Every command/tool/action maps to object, verb, zone, and authority |
| Home | Redesigned desktop default surface | User sees Now, Review, Recall, and Health without hunting |
| Review | Event-backed trust/provenance timeline | Failure, confidence, recovery, and agent actions are legible |
| MCP contracts | Shared component payloads | Human and agent UI stop diverging |
| QA | Scenario proof in dev app and MCP | Trust states and transitions are verified |
| Polish | Design-system pass | The app feels deliberate, not just more decorated |

## Resolved Decisions

- The landing label is **Now**. Review remains the semantic zone.
- Slice 2 uses a small event contract instead of one event variant per state.
- Dictation history enters Review as a capture artifact now; dictation UX polish
  remains a separate workstream.
- The local allowlist file is the authority source of truth; app UI is a
  viewer/editor over it.
- JSON contracts are canonical. MCP and Tauri components render separately.

## Non-Goals

- No immediate CLI namespace migration.
- No broad rewrite of Tauri app structure before the home slice proves the IA.
- No pure chat-first product shell.
- No visual-only redesign that leaves trust/recovery states hidden.
- No external cloud control plane for local Minutes state.
