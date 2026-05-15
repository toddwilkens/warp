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

## Phase B recon notes (pre-build, line numbers from warp@0d5da4d2)

Captured during the cold-build wait so the next iteration goes straight to editing.

### Sites 1–3 (LANDED on this branch, commit `4845677`)

Mechanical scaffolding committed; compiles standalone as observable no-op.

### Site 4 — `app/src/terminal/model/terminal_model.rs`

**Model: `sourced_rc_file()` at `:2988–3013`** — closest analog. Pattern to mirror:

```rust
fn sourced_rc_file(&mut self, data: SourcedRcFileForWarpValue) {
    if self.block_list.is_bootstrapped() {
        self.did_receive_rc_file_dcs = Some(true);     // <-- WE SKIP THIS
        let shell_type = ShellType::from_name(data.shell.as_str());
        match shell_type {
            Some(shell_type) => self.event_proxy.send_terminal_event(
                Event::SourcedRcFileInSubshell(SourcedRcFileInSubshellEvent { ... })
            ),
            None => log::error!(...),
        }
    }
}
```

Our `resume_warpify_session()` should:

- Add `if !self.ignore_bootstrapping_messages { ... }` outer guard (mirrors `init_shell` at `:2906–2907`; `sourced_rc_file` actually omits this guard — TBD whether to follow `init_shell` strict pattern or `sourced_rc_file` permissive pattern).
- Gate on `self.block_list.is_bootstrapped()` (line `:2991` pattern).
- Do **NOT** touch `self.did_receive_rc_file_dcs` (design intent: this is a fast path, not an RC-file path).
- Map `data.shell` → `ShellType` via `ShellType::from_name`.
- On success, emit `Event::ResumeWarpifySession(ResumeWarpifySessionEvent { shell_type, uname, session_id })` via `self.event_proxy.send_terminal_event(...)`.
- On unknown shell, `log::error!` (same shape as `:3005–3010`).
- On `!is_bootstrapped()`, `log::warn!` and ignore (design doc §4).

Other relevant landmarks confirmed:

- `terminal_model.rs:519` — `ignore_bootstrapping_messages: bool` field decl.
- `:554` — `did_receive_rc_file_dcs: Option<bool>` field decl.
- `:1505–1507` — `pub fn ignore_bootstrapping_messages(&mut self)` setter.
- `:1605, :1621` — existing `block_list().is_bootstrapped()` / `active_block().is_bootstrapped()` callers.
- `:2821` — `ignore_bootstrapping_messages = false` reset.

### Site 4b — `app/src/terminal/event.rs` (and `model_events.rs`)

Two parallel `Event` enums need the new variant. Both follow the same pattern as `SourcedRcFileInSubshell`.

**`app/src/terminal/event.rs:30` (outer Event enum):**

- Add `ResumeWarpifySession(ResumeWarpifySessionEvent)` near `:94` (next to `SourcedRcFileInSubshell`).
- Add new struct `ResumeWarpifySessionEvent` near `:161` (next to `SourcedRcFileInSubshellEvent`):

```rust
#[derive(Debug, Clone)]
pub struct ResumeWarpifySessionEvent {
    pub shell_type: ShellType,
    pub uname: Option<String>,
    pub session_id: Option<String>,
}
```

**`app/src/terminal/model_events.rs:6,444` (ModelEvent enum):**

- Add `SourcedRcFileInSubshellEvent`-style import on line `:6` (already imports it — add `ResumeWarpifySessionEvent` to the same import line, OR add a new `pub use` if the existing import is glob).
- Add `ResumeWarpifySession(ResumeWarpifySessionEvent)` variant near `:444`.

Note the existing pattern: `event.rs::Event` is what the model emits via `event_proxy.send_terminal_event(...)`. The bridge between `Event` and `ModelEvent` is somewhere in the model→view event pump (not yet located — Phase B will surface it if the variant is missing on either side; the compiler will tell us).

### Site 5 — `app/src/terminal/view.rs`

**Model: `ModelEvent::SourcedRcFileInSubshell` arm at `:11742–11783`** — adjacent to our patch site.

What that arm does (the path we are NOT taking):
1. Fires `TelemetryEvent::ReceivedSubshellRcFileDcs`.
2. Spawns a delayed task that waits `TRIGGER_RC_FILE_SUBSHELL_BOOTSTRAP_DELAY`.
3. After the delay, checks `is_ssh`, `tmux_control_mode_active`, `has_ai_metadata`, returns early on agent / tmux.
4. Calls either `continue_warpify_ssh_session(&uname, shell_type, ctx)` (the SSH path) OR `trigger_subshell_bootstrap(Some(shell_type), true, ctx)` (the local subshell path).

`continue_warpify_ssh_session()` is at `:24317` — it's the function that actually lights up warpify state for an SSH session WITHOUT injecting a subshell bootstrap. **This is probably the function we want to call from the new arm**, NOT `trigger_subshell_bootstrap` (which calls `start_bootstrap_timer` + writes init bytes to the pty — the failure mode).

Proposed new arm shape (subject to empirical refinement):

```rust
ModelEvent::ResumeWarpifySession(event) => {
    send_telemetry_from_ctx!(TelemetryEvent::ResumeWarpifySession, ctx);
    let shell_type = event.shell_type;
    let uname = event.uname.clone().unwrap_or_default();
    // Same agent / tmux guards as the SourcedRcFileInSubshell arm.
    let (is_ssh, is_tmux_control_mode_active, has_ai_metadata) = { ... };
    if has_ai_metadata { return; }
    if is_tmux_control_mode_active { return; }
    if is_ssh {
        self.continue_warpify_ssh_session(&uname, shell_type, ctx);
    } else {
        // Local-only resumed sessions are out of scope for #560 — log + ignore.
        log::warn!("ResumeWarpifySession on non-SSH block; ignoring");
    }
}
```

Key differences from `SourcedRcFileInSubshell` arm:
- **No spawn delay.** The SourcedRcFileInSubshell delay exists to coalesce with the RC-file flow; we have nothing to coalesce with.
- **No `trigger_subshell_bootstrap` branch.** That's the byte-interleave + watchdog culprit for our case.
- **`is_ssh` is the expected case** (claude-session always runs over ssh). Non-SSH is logged + ignored.

Empirical questions for Phase B:
- Does `continue_warpify_ssh_session` alone fully light up the warpify UI (command corrections, file tree, AI hints, OSC52 clipboard), or are there state fields it doesn't touch that the SourcedRcFileInSubshell→trigger_subshell_bootstrap path does?
- If yes, follow up by reading `continue_warpify_ssh_session` body at `:24317` and `trigger_subshell_bootstrap` body at `:8549` and diff'ing the state mutations.

### Site 6 — Telemetry parity (`app/src/server/telemetry/events.rs`)

`TelemetryEvent::ReceivedSubshellRcFileDcs` is a unit variant — 5 mechanical sites:

| Line  | What                                                                                  |
| ----- | ------------------------------------------------------------------------------------- |
| `:1683` | variant decl (`ReceivedSubshellRcFileDcs,`)                                       |
| `:4133` | big match arm (likely category bucketing)                                          |
| `:4863` | big match arm (likely a second bucketing pass)                                     |
| `:5436` | `EnablementState::Always` arm                                                       |
| `:5920` | display-name arm (`"Received Subshell RC File DCS"`)                              |
| `:6623` | description arm (`"Spawned a subshell to be automatically Warpified"`)             |

Add `TelemetryEvent::ResumeWarpifySession` as a sibling unit variant + 5 parallel arms with display name `"Resumed Warpify Session"` and description `"Marked a tab warpified via fast-path on attach to an already-bootstrapped backing shell"`.

### Test sites (for the eventual upstream PR)

- `app/src/terminal/model/ansi/mod_tests.rs` — JSON round-trip for `ResumeWarpifySession` (mirror existing hook round-trip tests).
- Mock handler — already covered by the default no-op landed in `:296`.
- Integration test in `crates/integration/` — exercise the fast path on a tab whose local `block_list` is already bootstrapped.

---

## Phase B validation results (2026-05-15)

**Setup:** `./script/run --release` on Mac (M-series). HEAD `28dd333` (sites 1–6 + diagnostic logs). Two tabs in WarpOss, both `ssh isidore claude-session warpify-attach-test`. Tab A first attach = warpified. Tab B reattach = DCS dispatched, **UI did not warpify**.

### Diagnostic log output (the `log::info!` lines added at every guard + branch in site 5)

```
[INFO] Received ResumeWarpifySession hook
[INFO] ResumeWarpifySession arm entered: shell_type=Zsh uname="Linux"
[INFO] ResumeWarpifySession guards: is_ssh=false tmux_control_mode_active=false has_ai_metadata=false
[INFO] ResumeWarpifySession: is_ssh=false, no-op
```

### Finding 1 — `is_ssh` guard is timing-sensitive and false on reattach

`is_ssh_block()` (terminal_model.rs:2329) returns true only while `notify_on_end_of_ssh_login.is_some()`. That flag is set when Warp starts monitoring for the SSH `Last login:` marker and cleared on login complete. On Tab B our DCS lands **before** dtach attaches, before any zsh output, before Warp's SSH-prompt detection fires. Hence `is_ssh=false`.

Tab A works because its DCS (`SourcedRcFileForWarp` from the rcfile bootstrap) fires **after** zsh has rendered the prompt and Warp has detected the SSH login window.

### Finding 2 — Relaxing the `is_ssh` guard is necessary but insufficient

Even if we drop the guard, `continue_warpify_ssh_session` (view.rs:24355) writes the warpify-init script to Tab B's PTY via `clear_line_editor_and_write_to_pty_with_mac_workaround_hack`. Those bytes route through:

```
Tab B Warp → local zsh → ssh → remote sshd → dtach stdin → shared zsh (broadcast to all attached clients)
```

So the warpify-init bytes (and the resulting rcfile-source bytes + a fresh `SourcedRcFileForWarp` echo) would appear in **both** Tab A and Tab B — the exact byte-interleave failure mode the NIGHT handoff flagged for `trigger_subshell_bootstrap`. Same disease, different function.

### Finding 3 — Proper fix path is "synthesize InitShell view side effects, no PTY write"

The Tab A normal flow is:

1. Shell sources rcfile → emits `Auto-Warpify` OSC.
2. Warp writes init script to PTY (**PTY write — broadcasts through dtach**).
3. Shell sources init script → emits `InitShell` DCS.
4. `terminal_model::init_shell` builds `pending_session_info` → fires `HandlerEvent::InitShell`.
5. View receives `HandlerEvent::InitShell` → activates warpify UI (file tree, AI hints, OSC52, command blocks).

The resume flow needs to skip steps 2–3 and synthesize step 5 directly using values shipped in `ResumeWarpifySessionValue` (currently `shell` + `uname`; may need `pwd`, env_var_collection_name, or other fields that `SubshellInitializationInfo` normally collects from the bootstrap).

**Open implementation questions:**

1. What does `HandlerEvent::InitShell` actually do view-side beyond creating `SessionInfo`? Which calls light up which UI features?
2. Is there a single `set_warpified_for_active_block()`-style entrypoint, or is the warpify-UI state scattered across `warpify_state`, `block_list`, and `sessions`?
3. Does `SubshellInitializationInfo` require fields we cannot reasonably ship in the DCS (e.g. dynamic shell state computed during bootstrap)?

### Status

- Caller (`isidore-infra/installers/dtach.sh`) DCS emission verified working end-to-end (DCS reaches Warp, is parsed, dispatched as `ModelEvent::ResumeWarpifySession`).
- Sites 1, 2, 3, 4, 4b, 6 (mechanical scaffolding + telemetry) all correct.
- Site 5 (view arm) **needs rework** along the lines of Finding 3 above. Current implementation is a no-op on reattach due to Finding 1, and would be byte-interleave-broken if the guard were relaxed (Finding 2).

### Approach C (planned next — quick empirical probe)

Try replacing the `ResumeWarpifySession` DCS in `claude-session` with a synthetic `SourcedRcFileForWarp` DCS on reattach. Routes through the existing handler. Almost certainly hits the same PTY-write byte-interleave problem (the existing `SourcedRcFileInSubshell` arm calls either `continue_warpify_ssh_session` or `trigger_subshell_bootstrap` after a spawn-delay), but the empirical confirm is cheap and rules out "did we just pick the wrong DCS." No Warp patch needed for this probe.
