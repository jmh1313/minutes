# Minutes IA — Slice 0 Inventory (2026-06-02)

> Read-only surface/action/state map. Companion to
> `minutes-ia-agent-ux-2026-06-02.md`. Maps every user-reachable action across
> the five surfaces to **object → verb → zone → authority**, then flags what
> fails the `<verb> <object>` litmus test. No code changed to produce this.

**Surfaces inventoried:** `tauri/src/index.html` (desktop) · `crates/core/src/palette.rs` (command palette / shared action registry) · `crates/cli/src/main.rs` (~45 CLI commands) · `crates/mcp/src/index.ts` (MCP tools) · `.agents/skills/minutes/` + `.opencode/commands/` (skills).

**Model recap.** Objects: CaptureSession · Transcript · MeetingArtifact · Extraction · PersonOrSpeaker · GraphMemory · Readiness · Event · Template. Verbs: capture · review · recall · annotate · extract · confirm · export · diagnose. Zones: Capture · Review · Recall · Health (+ Control-plane for agent-only surfaces). Authority tiers: Read · Draft · Annotate · Mutate-local · Capture-control · External-effect.

---

## 1. Zone coverage by surface (the balance check)

| Surface | Capture | Review | Recall | Health | Control-plane / internal |
|---|---|---|---|---|---|
| Desktop (`index.html`) | record/live/dictate/quick-thought, call-detect | detail, md-viewer, jobs, recovery, artifact-template | recall-panel | settings, readiness, about, cli-setup, whats-new, update | discuss-ai / assistant |
| Palette (`palette.rs`) | 5 actions | 5 actions | 7 actions | — | (registry is the shared key) |
| CLI (`main.rs`) | ~12 | ~10 | ~9 | ~9 | ~8 internal + 2 introspection |
| MCP (`index.ts`) | record/stop/live/dictate/note | get/list/transcript/insights/jobs | search/people/person/commitments | health/ready/status | events/annotations/schema |
| Skills (20) | record, note | debrief, recap, list, tag, cleanup | search, graph, prep, brief, mirror, ideas, lint, ingest | setup, verify | (video-review) |

**Read:** the model covers every surface. Recall is the densest in palette/CLI; Health is the densest in CLI (operational commands) and in desktop *overlays*. That overlay concentration is the Slice 1 signal (see §5).

---

## 2. Master shared-action table (palette registry = canonical key)

`palette.rs` calls itself "the one source of truth for every consumer," so it is the natural canonical action key. Each row is one `ActionId`, with its projection and classification.

| ActionId | Object | Verb | Zone | Authority |
|---|---|---|---|---|
| `start-recording` / `stop-recording` | CaptureSession | capture | Capture | Capture-control |
| `start-live-transcript` / `stop-live-transcript` | CaptureSession | capture | Capture | Capture-control |
| `start-dictation` / `stop-dictation` | CaptureSession | capture | Capture | Capture-control |
| `add-note` | Event (on CaptureSession) | annotate | Capture | Annotate |
| `read-live-transcript` | Transcript | review | Review | Read |
| `open-latest-meeting` / `…-from-today` | MeetingArtifact | review | Review | Read |
| `open-meetings-folder` / `open-memos-folder` | MeetingArtifact | review | Review | Read |
| `show-upcoming-meetings` | Readiness (calendar) | recall | Capture (prep) | Read |
| `search-transcripts` | Transcript/MeetingArtifact | recall | Recall | Read |
| `research-topic` | GraphMemory | recall | Recall | Read→Draft |
| `find-open-action-items` | Extraction | recall | Recall | Read |
| `find-recent-decisions` | Extraction | recall | Recall | Read |
| `copy-meeting-markdown` | MeetingArtifact | export | Review | Read |
| `create-debrief-draft-from-current-meeting` | MeetingArtifact | extract/export | Review | **Draft** |
| `confirm-current-speaker` | PersonOrSpeaker | confirm | Recall | **Mutate-local** |
| `rename-current-meeting` | MeetingArtifact | annotate | Review | **Mutate-local** |
| `open-assistant-workspace` | (assistant) | recall | Recall | Read |

Every palette action reduces to `<verb> <object>`. The authority column already shows the natural gate points: only `confirm-speaker` and `rename-meeting` are Mutate-local; only `create-debrief-draft` is Draft; all capture toggles are Capture-control. Nothing here is External-effect — consistent with local-first.

---

## 3. CLI commands grouped by zone

(Conceptual grouping for `--help` sections — **no renames**, per Non-Goals.)

- **Capture:** `record` `stop` `extend` `note` `mic-toggle` `live` `dictate` `watch` `process` `process-queue` `import` `preflight-record`
- **Review:** `list` `get` `clean` `export` `transcript` `actions` `jobs` `template` `cleanup` `storage`
- **Recall:** `search` `person` `people` `commitments` `consistency` (diagnose GraphMemory) `research` `insights` `context`
- **Health:** `setup` `status` `health` `devices` `sources` `paths` `logs` `service` `vault` `enroll` `voices` `vocabulary`
- **Control-plane (agent):** `events` `agent-annotate` `dashboard`

---

## 4. MCP tools grouped by object

From `index.ts` registrations:

- **CaptureSession:** `record` `stop_recording` `stop` `live` `dictate` `stop_dictation` `note` `add_note` `recording_status` `get_status`
- **Transcript:** `read_live_transcript` `transcript` `live_events` `live_events_since_seq`
- **MeetingArtifact:** `get_meeting` `list_meetings` `list` `get_moment` `get`
- **Extraction:** `get_meeting_insights` `insights` `decision` `commitment` `commitments`
- **PersonOrSpeaker:** `get_person_profile` `person` `people` `confirm_speaker` `confirm` `list_voices`
- **Event / AgentAction:** `add_agent_annotation` `get_agent_annotations`
- **Readiness:** `health` `ready` `status` `list_processing_jobs`
- **Recall:** `search_meetings` `search_context` `search`

Clean object alignment. The `add_agent_annotation` / `get_agent_annotations` pair is exactly the Event-as-shared-surface substrate the plan leans on.

---

## 5. Desktop overlay inventory → Slice 1 implication

The current app is one screen + ~13 overlays/panels. Mapped to zones, the redesign decision falls out almost mechanically:

| Surface (id) | Zone | Slice 1 disposition |
|---|---|---|
| `detail-overlay`, `md-viewer` | Review | **Promote** → persistent Review detail pane |
| `jobs-overlay` | Review | **Promote** → Review "processing" state |
| `recovery-overlay` | Review | **Promote** → Review "needs-review / failed" state (this is the trust center) |
| `artifact-template-overlay` | Review/Template | Keep modal, launched from Review |
| `recall-panel` | Recall | **Promote** → the Recall zone |
| `settings-overlay` | Health | Stay modal (Health is "boring on purpose") |
| `readiness-overlay` | Health | Fold into Health chip + modal |
| `about-overlay`, `cli-setup`, `whats-new`, update-banner | Health | Stay modal/system |
| `call-detected`, `call-ended` | Capture | Stay as inline prompts |

**Finding:** the four overlays that should *stop being overlays* (`detail`, `md-viewer`, `jobs`, `recovery`) are all Review. That is the concrete evidence for "Review becomes the default product spine" — the app already has the Review surfaces, they're just stacked as modals instead of living in a zone.

---

## 6. Litmus-test findings (the debt)

Everything user-facing maps cleanly. The residue is **not** product debt to delete — it's surface that should be explicitly classified into non-zone lanes so the human IA stays clean:

- **Internal / infra (hide from human zones + palette):** `parakeet-helper` `parakeet-benchmark` `preflight-record` `apple-speech` `decode-hint-eval` `process-queue`. These are subprocess/diagnostic plumbing, not user verbs.
- **Agent-introspection (Control-plane, agent-facing only):** `schema` `capabilities`. Machine-readable; correctly absent from the human app.
- **Integration (edge objects, not core model):** `qmd` `automate` `vault` `autoresearch`. They touch external systems (knowledge base, automation, Obsidian, web). Recommend a 10th object — **Integration** — or tag them `zone: Health/integrations` so they don't pollute the four core zones.
- **Genuinely ambiguous (decide before Slice 4):**
  - `mic-toggle` → Capture vs Health (it's a device action used mid-capture). Lean Capture.
  - `vocabulary` → Health (transcription tuning) vs Recall (it's terms). Lean Health.
  - `enroll`/`voices` → Recall (PersonOrSpeaker) vs Health (setup). Object says Recall; action feels like setup. Surface in both, canonical object = PersonOrSpeaker.

**Recommendation:** add an `Integration` object (covers `qmd`/`vault`/`automate`/`autoresearch`) and an explicit `internal` / `introspection` lane tag. Then the litmus test holds with zero exceptions: every surface is either `<verb> <object>` in a zone, or tagged internal/introspection/integration on purpose.

---

## 7. What Slice 0 tells Slice 1

1. **Build order is forced:** the four Review overlays already exist — Slice 1 is mostly *relocating* `detail`/`md-viewer`/`jobs`/`recovery` into a persistent Review zone, not inventing UI. Lower risk than it looked.
2. **`recovery-overlay` is the trust center seed** — it already renders failed/needs-review captures; the Slice 2 event timeline grows out of it, not from scratch.
3. **Health stays modal** — settings/readiness/about/cli-setup don't need zone real estate; a Health *chip* + existing modals is enough.
4. **One model gap to close:** add the `Integration` object so `qmd`/`vault`/`automate` stop being unclassified. Everything else maps.
5. **No CLI renames needed** — §3's grouping is purely `--help`/palette section labels.
