REVIEW ONLY — do NOT modify any files, do NOT run cargo, just critique my diagnosis and plan adversarially. You are reviewing a fix for github issue silverstein/minutes#258 in this repo (~/Sites/minutes).

## The bug (Windows only)
`live_transcript::session_status()` reports `line_count: 0` / `duration_secs: 0.0` for a healthy in-app (Tauri) live transcript session on Windows, even while utterances are being captured and written to live-transcript.jsonl. macOS is fine.

## My confirmed root cause
- `session_status()` (crates/core/src/live_transcript.rs:2344) does `lt_process_pid = pid::check_pid_file(&lt_pid).ok().flatten()`.
- `check_pid_file` (crates/core/src/pid.rs:234) returns Ok(None) only if `!path.exists()`, otherwise does `fs::read_to_string(path)?`.
- The live session holds that same PID file under an exclusive fs2 lock (`create_pid_guard` -> `try_lock_exclusive`) for its whole lifetime.
- On Unix fs2 uses flock (advisory) so a separate read_to_string succeeds -> check_pid_file returns Some(pid).
- On Windows fs2 uses LockFileEx(LOCKFILE_EXCLUSIVE_LOCK) (mandatory) -> ReadFile on the locked range fails with ERROR_LOCK_VIOLATION -> read_to_string returns Err -> check_pid_file returns Err -> `.ok().flatten()` = None.
- So `standalone_active = lt_process_pid.is_some()` = false. In the in-app case `check_recording()` is also None (UI tracks recording flag separately), so `should_report_stats = standalone_active || recording_pid.is_some()` = false -> stats forced to (0, 0.0). The whole standalone session actually reports active:false, source:None too.

## My planned fix (issue's recommended Option 2, hardened + testable)
Reuse the existing fresh-heartbeat machinery the recording-sidecar path already uses. The LiveStatus sidecar (live-transcript-status.json) is written ~1x/sec via atomic temp+rename (NOT locked), so read_live_status succeeds on Windows.

1. In `session_status()`: after computing `lt_process_pid`, also compute `standalone_pid_present = lt_pid.exists()` (computed AFTER check_pid_file, so a stale-dead file that check_pid_file already removed reports present=false).
2. Add param `standalone_pid_present: bool` to `derive_session_status`.
3. Change standalone detection to:
   - if lt_process_pid.is_some() -> standalone_active = true (primary, Unix + readable Windows)
   - else if standalone_pid_present -> standalone_active = (evaluate_recording_sidecar_status(live_status, now).0 == true)  // fresh+Healthy heartbeat proves liveness
   - else false
4. Everything downstream (active, source=Standalone, should_report_stats, status_metrics) then works. `pid` stays None in the fallback (we genuinely can't read it; do NOT fake std::process::id()).

## Why I think the fallback can't false-positive
- Unix alive: check_pid_file returns Some -> primary path, fallback never consulted.
- Unix/Windows dead+leftover file: read succeeds (lock released on death) -> is_process_alive false -> check_pid_file removes file -> exists()=false -> fallback not engaged.
- Windows alive standalone: file present + unreadable(locked) + fresh Healthy LiveStatus -> fallback engages -> active. This is exactly the bug case.
- Windows alive but hung (stale heartbeat): present but evaluate_recording_sidecar_status returns (false,...) -> inactive. Correct.

## Questions for you to attack
1. Any way the fallback wrongly reports active (e.g., a crashed Windows session whose PID file stayed locked, or a stale status.json left from a prior session)? Walk the staleness gates.
2. Is the AFTER-check_pid_file ordering for `standalone_pid_present` actually race-safe enough, or is there a TOCTOU that matters?
3. Should I instead fix check_pid_file generally (Option 1) so other callers benefit? Tradeoff: it can't return the real PID when locked.
4. Am I wrong that recording_pid is None in the in-app case? If the in-app live session DID create a recording PID, the bug wouldn't manifest — so does my model hold?
5. Any concern reusing evaluate_recording_sidecar_status (its diagnostics say "sidecar ...") for the standalone fallback? I only consume the bool, diagnostic stays None for active standalone.
6. Anything in the SIDECAR_HEALTH_STALE_AFTER_SECS=3s window that makes this flaky (heartbeat is 1s)?

Give me your sharpest objections and anything I'm missing. No code edits.
