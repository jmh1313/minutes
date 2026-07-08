# Screen Recording TCC prompt re-appears on every recording despite Settings showing Minutes enabled

## Symptom

Every app-initiated recording pops the macOS "Minutes would like to record this computer's screen and audio" TCC prompt, even though System Settings → Privacy & Security → Screen & System Audio Recording shows every Minutes entry (production and dev) enabled. The prompt offers only "Open System Settings" / "Deny" — and since the Settings toggle already reads as on, there is nothing to flip, so the loop never resolves.

Likely downstream casualty: a denied/unattributed capture grant makes the ScreenCaptureKit helper produce an all-zero system stem — see `ISSUE_silent_system_stem_diarization.md` (2026-05-04 meeting collapsed to one speaker because `.system.wav` was pure zeros).

## Root cause (diagnosed 2026-06-10)

TCC stores a Screen Recording grant per bundle ID with a **code requirement (csreq) snapshot of whichever binary was running when the grant was recorded**. Three different bundles on this machine share `com.useminutes.desktop`:

1. `/Applications/Minutes.app` — production, Developer ID (63TMLKT8HN), stable.
2. `~/Sites/minutes/Minutes.app` → symlink to `target/release/bundle/macos/Minutes.app` — **rebuilt and re-signed on every `cargo tauri build`**.
3. Historical local rebuilds that used to replace `/Applications/Minutes.app` directly (the practice CLAUDE.md's hard rule now forbids).

If the TCC row was recorded against an ad-hoc/local build (ad-hoc csreqs pin to one exact cdhash), every other binary with the same bundle ID — including the real production app — fails the csreq check at `SCShareableContent` time and re-prompts. Settings keeps showing the stale row as "enabled," so the user can't repair it from the UI.

Both the user and system TCC databases held **two rows each** for `com.useminutes.desktop` at reset time — duplicate/stale entries confirmed.

## Remedy (applied 2026-06-10)

```bash
tccutil reset ScreenCapture com.useminutes.desktop
tccutil reset ScreenCapture com.useminutes.desktop.dev
```

Then launch the production app, start a recording, and grant once via System Settings (the toggle now correctly shows off until granted). The fresh row records against the Developer ID csreq, which is identity-based and stable across updates — the prompt should not return (apart from macOS's occasional OS-level reauthorization cadence).

## Prevention / follow-ups

- [ ] Remove the repo-root `Minutes.app` symlink, or re-point dev flows exclusively at `Minutes Dev.app` (`com.useminutes.desktop.dev` — already correctly separated by `install-dev-app.sh`).
- [ ] Ensure no script ever launches `target/release/bundle/macos/Minutes.app` directly; it shares the production bundle ID with a per-build signature.
- [ ] `minutes verify` / app onboarding: detect the wedged state — `CGPreflightScreenCaptureAccess() == false` immediately after a recording attempt while a TCC row exists for the bundle ID is exactly this condition; surface the `tccutil reset` remedy instead of letting the user stare at an already-enabled toggle.
- [ ] Re-verify diarization stems on the next real call after re-grant (`.system.wav` should be non-zero) — likely closes the silent-system-stem issue's trigger.
