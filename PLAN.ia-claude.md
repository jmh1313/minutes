# PLAN.ia-claude — Minutes Information Architecture (independent pass)

> Independent IA draft by Claude, written **without** reading the Codex IA doc, to be diffed against it.
> Date: 2026-06-02. Repo state: v0.18.4, `release/v0.18.4`.

## TL;DR

Minutes does not have a *navigation* problem. It has a **shared-model** problem: four input modes, ~45 CLI commands, 31 MCP tools, a command palette, and a single-screen Tauri app all expose the same concepts with no canonical object model underneath. The fix is not "add a sidebar." It is:

1. Name a **shared object model** (the nouns) + **verbs** (the actions). Every surface becomes a *projection* of it.
2. Recognize Minutes has **three user classes, not one**: human@desktop, human@CLI, **agent@MCP**. The IA must serve all three from one model.
3. Render the **event bus as the shared surface** — it is already the substrate that makes human *trust* and agent *legibility* the same thing, viewed two ways.
4. Put a four-zone **human projection** on the desktop: **Capture / Review / Recall / Health.** (This independently matches the Codex spine — convergence is the signal that this cut is discoverable from the substrate, i.e. low-risk to commit.)

The scope stays small: a one-page object+zone spec, then a targeted `tauri/src/index.html` home redesign. Not redesign theater. Justification is **marginal cost of the next feature**, not "the app is broken today."

---

## 1. The real problem: three audiences, one model

Most IA advice assumes one user class (a human looking at screens). Minutes is unusual — it is explicitly built to be *run and controlled by AI agents too* (CLI + MCP + palette + app). So the design target is a model that reads correctly for three very different consumers:

| User class | Surface | Wants | Failure mode if IA is weak |
|---|---|---|---|
| **Human @ desktop** | Tauri app | "Am I capturing? Did it work? What can I do next?" | Pile of overlays; no home; silent failures |
| **Human @ CLI / power user** | 45 commands + palette | Fast, scriptable verbs; discoverability via `--help` | Flat namespace; no mental grouping |
| **Agent @ MCP** | 31 tools + 7 resources + event bus | Clean nouns/verbs to call; legible state to read; provenance to write | Tools that don't map to objects = drift; hidden state = confused agent |

The screens are downstream. **The object model is the thing all three share.** Get it right once and every surface inherits coherence for free.

---

## 2. Object-first IA (the durable layer)

The durable IA is the **noun set**, because nouns survive redesigns, CLI refactors, and new MCP tools. Proposed canonical objects (all already exist in `crates/core`):

- **Session / Capture** — a recording in flight or just finished (`capture.rs`, `pid.rs`, `streaming.rs`)
- **Transcript** — raw + cleaned text (`transcribe.rs`, `whisper-guard`)
- **Meeting** — the markdown artifact w/ frontmatter (`markdown.rs`, `pipeline.rs`)
- **Memo** — voice memo via watch/dictation (`watch.rs`, `dictation.rs`, `daily_notes.rs`)
- **Extraction** — action item / decision / commitment (frontmatter YAML, `knowledge_extract.rs`)
- **Person / Speaker** — identity + voice profile (`diarize.rs`, `voice.rs`, `voices.db`)
- **Graph** — cross-meeting entities + relationships (`graph.rs`)
- **Event** — append-only JSONL on the bus (`events.rs`, RFC #194)
- **Readiness** — model / mic / disk / permissions / backend (`health.rs`)

Verbs (the action set every surface re-uses): **capture · review · recall · annotate · extract · confirm · export · diagnose.**

**IA litmus test** (this is the rigorous version of "map commands to zones"): *every CLI command, MCP tool, and palette entry must be `<verb> <object>`.* If a command doesn't reduce to a known verb+object, either the object model is missing a concept or the command is product debt. Examples of current commands re-read as verb+object: `Confirm` = confirm Person; `AgentAnnotate` = annotate Meeting (as Event); `Consistency` = diagnose Graph; `Insights` = recall Event.

---

## 3. Surfaces are projections of the model

| Object | Desktop zone | CLI verb(s) | MCP surface |
|---|---|---|---|
| Session/Capture | **Capture** | record, live, dictate, stop, note | start/stop/status tools |
| Transcript/Meeting | **Review** | list, get, clean, process, export | meeting resources + reader |
| Extraction | **Review** | actions, commitments | action/decision tools |
| Person/Speaker | **Recall** | person, people, enroll, confirm, voices | people tools |
| Graph | **Recall** | graph, consistency, insights | graph tools |
| Event | **Review** (as timeline) | events, agent-annotate | event-bus tools |
| Readiness | **Health** | health, setup, devices, sources, paths, storage | health resource |

Read top-to-bottom, this is also the answer to "do we need to regroup the CLI?" — **no breaking rename required.** The grouping is *conceptual* (which object does this touch), surfaced through `--help` sections and palette categories, not a forced `minutes meeting <x>` namespace migration. The object model gives you the grouping benefit without the breaking-change cost.

---

## 4. The key insight: the event bus **is** the shared surface

This is where I'd push hardest and where I think the strongest, most Minutes-specific move lives.

2026 agentic-UX consensus (sources in §6) converges on one idea: **context is a shared surface.** UI actions must flow to the agent (`updateModelContext()` in MCP Apps); agent actions must flow back to the human (provenance, data lineage, confidence). NN/g's agent guidance: *expose system status, show confidence and data lineage, keep humans in the loop with review/rollback.*

Minutes **already has this substrate**: the append-only event log (`events.rs`, RFC #194 — annotation discipline + allowlist enforced in code; agents write attributed `agent.annotation` events, never mutate human frontmatter).

So two things people usually build separately collapse into one:

- A **human trust center** ("did this capture work? what's the confidence? what needs review?")
- An **agent provenance drawer** ("what tool ran, what it read, what it wrote, did it need approval")

…are the **same timeline, filtered two ways.** Both are projections of the event stream. The Review zone should *render the event bus* as a per-capture status+provenance timeline. You don't design two features; you design one event-rendering view with a human/agent filter. That is the kind of unification that makes the app feel coherent instead of bolted-together, and it reuses architecture that already shipped.

---

## 5. The human projection: four zones + a default home

The desktop app's job is to make the object model *navigable for the human@desktop class.* Four zones:

1. **Capture** — record · live · dictate · quick-thought · watch/import. Make the **autonomy level** of each mode visible (live dictation = you're the operator, needs instant feedback; watch-folder = autonomous, needs after-the-fact review). This is the "variable autonomy" principle applied to capture.
2. **Review** — recent captures + the trust/provenance timeline (§4). First-class states: `complete · processing · needs-review · audio-risk · blank/low-confidence · system-audio-missing · backend-unavailable · artifact-ready`. **This is the trust center and it is the product's spine** — Minutes is excellent only when the user *knows whether a capture is trustworthy.*
3. **Recall** — the cross-meeting layer: search · people · commitments · consistency · graph · assistant workspace. A mode, not a bolted-on terminal.
4. **Health** — readiness/diagnostics/integrations/permissions/updates. Boring on purpose. Keep subsystem knobs out of the main experience.

**Default home state** (the thing the current single-screen app lacks): home answers, in priority order — *Am I capturing right now? · Is anything broken or waiting for review? · What did I just capture? · What can I recall? · Is the system ready for the next meeting?* Not every feature gets equal visual weight; "needs review" outranks "browse archive."

---

## 6. 2026 design grounding → concrete rules

From the current literature (June 2026), translated into rules for Minutes:

- **Variable autonomy / human-on-the-loop is a spectrum, not binary.** Pause / override / rollback / escalation are first-class. → Per-capture: retry, re-process, dismiss, "keep recording" already exist; make them consistent across all capture modes.
- **Show your work: goals, assumptions, confidence, provenance.** → The Review timeline (§4) renders confidence (whisper-guard `no_speech` prob, diarization confidence levels L0–L3) as visible signal, not buried metadata.
- **Context is a shared surface** (MCP Apps `updateModelContext()`, "same info flows to the agent and back to the user where work happens"). → The event bus is this surface; render it both directions.
- **MCP Apps standard (2026-01-26):** tools can return `ui://` UI resources rendered in sandboxed iframes; one server ships both the human UI and the agent tools and the protocol keeps them in sync. → **Converge the existing MCP dashboard (`crates/mcp/ui/`) onto this standard.** Define each component's data contract once (read-only vs writeback declared up front); then the *same* components can serve Claude Desktop / ChatGPT *and* seed the desktop Review views. This is the highest-leverage way to avoid building the UI twice.
- **Apps SDK component rules:** inline for simple viewers, fullscreen for complex workflows; max-width + graceful collapse; respect system dark mode (`color-scheme`); keyboard focus states; design the JSON payload before the server. → Direct checklist for both the MCP dashboard and any new desktop view.

Sources:
- [Designing Human-Agent Interaction: Principles for Trustworthy Collaboration (designative, Jan 2026)](https://www.designative.info/2026/01/15/designing-human-agent-interaction-principles-for-trustworthy-collaboration/)
- [Agentic UX: 7 principles for designing systems with agents (Bootcamp, Feb 2026)](https://medium.com/design-bootcamp/agentic-ux-7-principles-for-designing-systems-with-agents-019512c2caa9)
- [MCP Apps — Bringing UI Capabilities to MCP Clients (modelcontextprotocol.io, Jan 26 2026)](https://blog.modelcontextprotocol.io/posts/2026-01-26-mcp-apps/)
- [OpenAI Apps SDK — Design components](https://developers.openai.com/apps-sdk/plan/components)
- [AI as a UX Assistant (Nielsen Norman Group)](https://www.nngroup.com/articles/ai-roles-ux/)
- [MCP vs API: when to use each (Atlan, 2026)](https://atlan.com/know/when-to-use-mcp-vs-api/)

---

## 7. Where this pass agrees with / differs from the Codex pass

**Agrees (independently):** four-zone spine (Capture / Review / Recall / Health); Review-as-trust-center is the 10/10 insight; agent operation must be legible; align CLI/MCP/palette to the IA; visual polish last; one-page spec before redesign.

**Adds / differs:**
1. **Object-first, not zone-first.** Zones are the *human-desktop projection*; the durable layer is the noun/verb model that all three user classes share. This makes "map every command to a zone" rigorous: it becomes "every command is `<verb> <object>`."
2. **Trust center and provenance drawer are one surface,** both rendered from the **existing event bus** (RFC #194) — not two features to build.
3. **Concrete MCP Apps convergence path:** define component/data contracts once, project to both Claude/ChatGPT and desktop, so the UI isn't built twice. (Neither pass had this until the 2026 MCP Apps research.)
4. **CLI needs no breaking rename** — conceptual grouping via the object model + `--help`/palette categories beats a forced namespace migration.

---

## 8. Scope & sequencing

1. **One-page canonical spec** — the object/verb model (§2), the surface→object map (§3), the four zones + default home (§5). (Merge this doc with the Codex doc into one source of truth.)
2. **Desktop home redesign** against the spec — `tauri/src/index.html` only; default home state first.
3. **Review = event-bus timeline** — render `events.jsonl` as the human/agent trust+provenance view.
4. **MCP dashboard → MCP Apps contracts** — define data contracts; reuse components toward desktop.
5. **CLI/palette categorization** — group by object via `--help` sections; no renames.
6. **Acceptance scenarios** (§9) verified in `~/Applications/Minutes Dev.app`.
7. **Visual/design-system polish** (per `PLAN-tauri-design-pass.md`) — last.

**Why now, not later:** not because users are hurting today, but because the marginal cost of the *next* feature (graph, mirror, weekly, tagging) is "another modal on a single screen" until the spine exists. Do it before the next feature, not as emergency triage.

---

## 9. Acceptance scenarios (human + agent as users)

- First-time user records and **understands system-audio limits before** the meeting is lost.
- User dictates, **sees what was captured**, and finds it later.
- A **failed capture is recoverable**, not mysterious — Review shows the state and the next action.
- Confidence is **visible** (low-confidence transcript is flagged, not silently shipped).
- **Agent** can answer "what did I promise Sarah?" from the right substrate (Graph/commitments).
- **Agent** can start/read/stop live transcript **without creating hidden state** the human can't see.
- **Human can inspect what the agent did** — the provenance timeline shows tool, resource, artifact, permission tier, approval.
- Same Review component renders correctly **inline in Claude/ChatGPT and fullscreen in the desktop app** (MCP Apps contract holds).
