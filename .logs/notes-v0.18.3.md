## What changed

v0.18.3 is a reliability batch. The headline is a Windows fix for live transcript (#258), reported by Quinn (@mquinn614) with a precise root-cause diagnosis.

On Windows, `fs2` file locks are mandatory rather than advisory, so reading a live-transcript PID file held under the session's own lock failed with a lock violation. Minutes was treating that failure as "no session," which froze the in-app Live timer and line count at 0:00 during a perfectly healthy session, and could let a concurrent `minutes record` write the same `live-transcript.jsonl` and clobber it. Minutes now treats a held lock as proof the session is alive (Windows releases locks when the owning process exits), so status, the recording-sidecar guard, start/stop, the update guard, and the background worker all read live state correctly on Windows. macOS and Linux were never affected.

This release also folds in the fixes that landed since v0.18.2:

- Richer MCP `get_meeting` structured content (#255, #256).
- Desktop now surfaces recording-start failures instead of failing silently, and rejects conflicting recording launches more clearly.
- Clearer desktop processing stages and a more compact processing status pill.

## Who should care

Windows users running live transcript should update. The frozen 0:00 timer is fixed, any tooling built on `session_status()` line count or duration now reports correctly, and the `live-transcript.jsonl` clobbering hazard is closed.

Everyone else gets the MCP `get_meeting` enrichment plus the desktop recording-failure and processing-stage clarity improvements.

## CLI / MCP / desktop impact

- CLI: `minutes stop` now stops a standalone live session on Windows even when its PID file is locked; `minutes record` correctly refuses to start alongside a live session.
- Desktop: Live mode timer and line count update correctly on Windows; recording-start failures are surfaced; processing stages read more clearly.
- MCP: `get_meeting` returns richer structured content.
- Shared engine: new `pid::inspect_pid_file` distinguishes a live-but-locked PID owner from an absent one, and every reader of a persistently-locked PID file uses it.

## Breaking changes or migration notes

No breaking changes. Pure reliability and correctness.

## Install

- `npx minutes-mcp` for search, browsing, and the dashboard with zero install.
- `brew install silverstein/tap/minutes` or `cargo install minutes-cli` for recording and transcription, then `minutes setup --model tiny`.
- Desktop DMG attached to this release.

## Known issues

The npm packages (`minutes-sdk`, `minutes-mcp`) may lag this release by a short window: npm auth was not available in the build environment, so they publish from the maintainer machine. The GitHub binaries, desktop DMG, `.mcpb` bundle, and Homebrew formula are the immediate delivery path; `npx minutes-mcp` picks up the new version once npm publish completes.

call-end auto-stop reliability (#129) and multi-party remote-speaker diarization quality (#169) remain open field reports, unaffected by this release.
