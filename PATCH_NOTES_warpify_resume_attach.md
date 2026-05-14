# Warp patch: warpify-on-attach for shared backing shells

Tracking issue: <https://github.com/toddwilkens/isidore-infra/issues/560>
Underlying user-visible bug: <https://github.com/toddwilkens/isidore-infra/issues/556>

## Problem

Warp's bootstrap state machine assumes one Warp tab per backing shell. When N Mac tabs attach to a shared `dtach` socket (`/tmp/dtach-<name>.sock` on the remote host), only the first tab gets warpified — subsequent tabs cannot trigger Warp's subshell-bootstrap path without destructive byte-interleave in the shared inner zsh stdin (already-running inner zsh idempotency-skips, no `InitShell`/`Bootstrap` reply, Warp hangs at "starting shell..." until `BOOTSTRAP_FAILED_DURATION` watchdog fires).

End-to-end mechanism walk: see `~/.claude/projects/-home-todd-code-isidore-infra/memory/handoff.md` (2026-05-14 EVE) and the comment chain on isidore-infra#556.

## Proposed shape

Add a new DCS hook that says "this tab is attaching to a shell that is already warpified; mark me bootstrapped using inferred/minimal session info, do NOT inject a re-bootstrap script, do NOT start the failure watchdog."

### Working name

`ResumeWarpifySession` (DProtoHook variant) / `ResumeWarpifySessionValue` (payload).

### Payload (minimal)

```rust
pub struct ResumeWarpifySessionValue {
    pub shell: String,           // "zsh" | "bash" | "fish"
    pub uname: Option<String>,   // remote OS for warpify command-corrections
    pub session_id: Option<String>,  // for telemetry / log correlation, optional
}
```

Deliberately omits the 20+ `BootstrappedValue` fields. The fast-path handler should treat them as "unknown — use safe defaults" rather than requiring the caller to fake them.

## Patch sites

### 1. `app/src/terminal/model/ansi/dcs_hooks.rs`

- Add `ResumeWarpifySession { value: ResumeWarpifySessionValue }` variant to `DProtoHook` enum (`:30`).
- Add `ResumeWarpifySession` arm to `name()` (`:88`).
- Add `ResumeWarpifySession` arm to `default_from_name()` (`:114`) for test fixtures.
- Add new `pub struct ResumeWarpifySessionValue { ... }` with `#[derive(Debug, Default, Deserialize, Serialize, PartialEq, Eq)]`.

### 2. `app/src/terminal/model/ansi/mod.rs`

- Add `Ok(DProtoHook::ResumeWarpifySession { value }) => self.handler.resume_warpify_session(value)` to `handle_decoded_hook` (`:604`).
- Decide: should this hook be hex-encoded (the default) or unencoded (like `SourcedRcFileForWarp`)? Likely hex-encoded — it's emitted from `claude-session` shell script, hex encoding is fine.

### 3. `app/src/terminal/model/ansi/handler.rs`

- Add default no-op `fn resume_warpify_session(&mut self, _data: ResumeWarpifySessionValue) {}` to `Handler` trait (near `:289`, the existing `sourced_rc_file` slot).

### 4. `app/src/terminal/model/terminal_model.rs`

- Implement `fn resume_warpify_session(&mut self, data: ResumeWarpifySessionValue)`:
  - **Pre-check**: if `self.ignore_bootstrapping_messages` is set, return early (mirrors `init_shell` at `:2907`).
  - **Branch on `block_list.is_bootstrapped()`**:
    - If already bootstrapped (the expected case — local Mac zsh has already done its own bootstrap): emit a NEW model event `Event::ResumeWarpifySession(ResumeWarpifySessionEvent { shell_type, uname, ... })`. Do NOT touch `did_receive_rc_file_dcs`. Do NOT go through the `InitShell` → `Bootstrapped` cycle.
    - If not bootstrapped: log a warning and ignore. This hook is only meaningful when the tab's own local block_list has already bootstrapped (which it will have via the standard Warp launch-config path).

- Add `ResumeWarpifySession(ResumeWarpifySessionEvent)` variant to terminal `Event` enum (cross-ref `app/src/terminal/event.rs` and `app/src/terminal/model_events.rs`).

### 5. `app/src/terminal/view.rs`

- Add a `ModelEvent::ResumeWarpifySession(event)` arm near `:11742` (the existing `SourcedRcFileInSubshell` site).
- The handler should:
  - **NOT** call `trigger_subshell_bootstrap` (that's the byte-interleave + watchdog culprit).
  - Mark `warpify_state` as active for this tab using `event.shell_type`. **Open question — what's the minimum state mutation?** Need to compare against the state set inside `trigger_subshell_bootstrap` → `continue_warpify_ssh_session` → eventual successful warpification path. The active end-state we care about is whatever flag the warpify UI checks before showing decorations (command corrections, file-tree generator, AI hints, prompt rendering). **Phase B validates this empirically: run the patched build, see which UI bits fail to engage with just the fast-path handler, add minimum required state mutations to satisfy them.**
  - Send `TelemetryEvent::ResumeWarpifySession` for telemetry parity.
  - Skip `start_bootstrap_timer` entirely.

### 6. Tests

- `app/src/terminal/model/ansi/mod_tests.rs`: add round-trip JSON serialize/deserialize test for `ResumeWarpifySession`.
- `app/src/terminal/model/ansi/handler.rs` test mock: add no-op `resume_warpify_session`.
- Integration test (in `crates/integration/` per CONTRIBUTING): exercise the fast-path on a tab whose local block_list is already bootstrapped, assert warpify UI activates without `write_init_subshell_bytes_to_pty` being called.

## Caller change (`isidore-infra/installers/dtach.sh` → `~/.local/bin/claude-session`)

On subsequent attach (socket pre-exists), emit the new DCS to the new tab's own stdout, then `exec dtach -A`:

```bash
sock="/tmp/dtach-${name}.sock"
if [ -S "$sock" ]; then
    # Subsequent attach — emit ResumeWarpifySession DCS so Warp marks this tab
    # warpified without trying to re-bootstrap the already-running inner zsh.
    # Hex-encoded payload:
    payload=$(printf '{"hook":"ResumeWarpifySession","value":{"shell":"zsh","uname":"Linux"}}' | xxd -p -c0)
    printf '\eP$d%s\x9c' "$payload"
fi
exec dtach -A "$sock" -E -z zsh -l
```

DCS prefix `\eP$d` is the hex-encoded variant (per existing `SourcedRcFileForWarp` is `\eP$f` unencoded; verify Warp's encoded-DCS introducer in `app/src/terminal/model/ansi/mod.rs`).

The bytes go up the new tab's own ssh stream — NOT through dtach's broadcast — so the existing already-warpified tabs are unaffected.

## Phase B validation checklist (Mac-side, after build)

- [ ] `cargo build --release` succeeds.
- [ ] Patched Warp launches; existing single-tab warpify still works (regression).
- [ ] First Mac tab attaching to a fresh dtach socket: warpifies (existing behavior).
- [ ] Second Mac tab attaching to same socket: emits `ResumeWarpifySession` DCS, warpify UI lights up, NO "starting shell..." hang.
- [ ] Tab A's warpify state is undisturbed when Tab B attaches.
- [ ] No byte-interleave in inner zsh (check `print -l $(jobs)` / `echo $WARP_BOOTSTRAPPED` after both tabs attach).
- [ ] Disconnect/reconnect cycle: close Tab B, reopen → re-emits DCS → re-warpifies.
- [ ] iTerm tabs attaching to the same socket: DCS silently discarded (iTerm doesn't parse Warp hooks) → no regression.
- [ ] Warpify UI features that need empirical validation (each is potentially a follow-up patch site):
    - [ ] Command corrections
    - [ ] File-tree generator (`warp_run_generator_command`)
    - [ ] Prompt rendering / Block boundaries
    - [ ] AI hints
    - [ ] OSC 52 clipboard out of Claude TUI

If any UI feature is broken on the fast path, the fix is either (a) add that feature's state mutation into the fast-path handler in `view.rs`, or (b) accept it as a documented limitation of resumed sessions.

## Upstream contribution path

If Phase B validates cleanly: open PR to `warpdotdev/warp@master` from a public branch (or via diff if the private mirror stays private). Frame as: "Support `tmux`/`dtach`/`screen` multi-attach to a shared backing shell — new opt-in DCS for clients that have an out-of-band guarantee the shell is already warpified."

Warp's `Oz for OSS` framing suggests they're receptive to ecosystem contributions. The patch is small, additive, opt-in, and doesn't alter any existing code path.

## Known unknowns / risks

- **Warpify UI state mutation set is empirically determined.** This design doc lists the patch sites; the exact `warpify_state` field touches in `view.rs::ResumeWarpifySession` handler will be discovered by running the patched build. Likely 1–3 small fields, but worth budgeting a second iteration.
- **Telemetry**: Warp has `TelemetryEvent::ReceivedSubshellRcFileDcs` at `view.rs:11743`. The new path needs its own telemetry event for upstream observability — Warp will want this in the PR.
- **`block_list.is_bootstrapped()` precondition**: this assumes the local Mac tab has already finished its own zsh bootstrap before `claude-session` runs. The launch-config flow guarantees this (`commands.exec` runs in interactive shell). If a future launch path bypasses the local shell, the fast path would silently no-op — log a clear warning so it's diagnosable.
- **Encoded vs unencoded DCS**: choose hex-encoded (default). `SourcedRcFileForWarp` is special-cased unencoded because the original RC file snippet is human-readable; we don't need that property here.
