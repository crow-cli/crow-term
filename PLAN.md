## **DO NOT ASK USER FOR FEEDBACK — THIS IS THE USER FEEDBACK.**
## **DO NOT ASK USER FOR NEXT STEPS — THESE ARE THE NEXT STEPS.**

# PLAN — ACP v2 support in crow-term (dual-stack: v1 AND v2, negotiated per connection)

Build/test gate (ALWAYS through the guard, never bare cargo):

    cd /home/thomas/src/crow-term/Martty
    DSH_TUI_CARGO_TARGET_DIR=$PWD/target ./scripts/cargo-guard.sh test --locked
    DSH_TUI_CARGO_TARGET_DIR=$PWD/target ./scripts/cargo-guard.sh check --locked
    DSH_TUI_CARGO_TARGET_DIR=$PWD/target ./scripts/cargo-guard.sh build --locked --release

Baseline: 851 unit + 10 integration, 0 failed. Two pre-existing warnings
(ui__tests.rs unused `ctl`, input__vim__tests.rs snake_case). `cargo fmt --check`
is NOT clean repo-wide (591 diff markers, hand-formatted files) — never run
repo-wide `cargo fmt`; re-check the 591 total after every edit.

## WHERE THIS STANDS

Current: **918 unit + 2 cli_help + 1 sigterm + 12 startup_e2e + 1 tcp_attach**,
all green, plus **11 + 434 JS** (432 pass, 2 skipped). Two pre-existing warnings
only. `npm/node_modules` had never been installed here — run `npm ci` in `npm/`
before trusting any JS result.

**v2 works end to end through the negotiator, both protocols, and it is now
pinned by tests.** Phases 0, 2, 3 and 8 are done; 4 is half done.

**The secondary mandate — picking a harness inside the TUI — is also done**
(`b3e129d` + `761f7a5`), live-verified in both switch directions on a real PTY
and pinned by 14 mutation-checked tests. See the section below. It surfaced two
real bugs, both fixed. The `/harness` name collision with the Node plugin is
**decided, not open**: `npm/lib` is dead code that never executes, so the
builtin owns the name and there is nothing to preserve. See that section, and
`AGENTS.md`'s "⛔ READ THIS FIRST".

⚠️ **`scripts/cargo-guard.sh` no longer auto-cleans.** It used to `cargo clean`
the shared `target/` at 20 GiB, which deleted a release build mid-test-run.
Over the limit now warns; pruning is explicit (`cargo-guard.sh prune`, or
`DSH_TUI_RUST_CACHE_AUTOCLEAN=1`). Do not reintroduce auto-clean. The cache is
currently ~22 GiB and the warning is expected.

**The operating order for this work is: make it work live against the real
agent over a teed PTY, THEN write the test that pins the shape that broke.**
TDD against a protocol nobody has spoken to yet produced guesses; the wire
capture produced facts. Every payload in the stub fixture and in
`tests/unit/events__tests.rs` is copied verbatim from a capture in `/tmp/wire-v2*.jsonl`.

**Mutation-check every assertion that guards a bug fix.** Three were verified by
breaking the source on purpose and watching the test fail with the original
symptom (see Phase 8). An assertion that has never been seen to fail is a
comment, not a test.

## SECONDARY MANDATE — the in-TUI `/harness` picker                    [x] DONE

> "okay that's what I was thinking. we don't have /harness. we need to implement
> a way to pick inside TUI."

Not a numbered phase: it is the standing secondary mandate, and it is the reason
the v2 work is reachable by a human at all. Running the bare binary
(`./target/release/crow-term --agent crow-cli --agent-arg acp2`) had no way to
change agent — `harness::selected()` is read once at boot.

**Shipped (`b3e129d`, 9 files):** `/harness [id]` as a Rust builtin. With an id
it switches directly; with no argument it opens a picker snapped to the active
recipe showing each one's command. A switch persists `defaultHarness` into
`settings.json` (patch, not rewrite; quarantine an unparseable file rather than
replace it), drops the sessions belonging to the agent being replaced, and sends
`Cmd::SwitchHarness { argv }` — the RESOLVED argv, because the painter owns
settings and the controller never reads it.

**The seam is a relay above `run()`, not inside a stack.** `cmd_rx` cannot be
recovered from inside either stack (both move it into a `dsh-acp-cmds` forwarder
thread that dies when its send fails), so `run_blocking` became a supervisor
loop: ONE tokio runtime for every generation, and per generation a
`dsh-acp-relay` thread running `relay_commands`, which forwards every command
except `SwitchHarness`. On a switch it emits `Starting { runtime: "harness" }`
(the painter's cue that a switch, not a cold start, owns the terminal — the UI
scaffolding for that event already existed and was dead), hands the command
receiver and the new endpoint back over `switch_tx`, and returns, dropping
`relay_tx` so the generation unwinds. A stack that cannot switch (v2, the legacy
controller loop) refuses in the agent's own words via `refuse_harness_switch`.

**Picker key sequence, for anyone driving it on a PTY:** `/harness ` → **Esc**
(dismisses the completion menu, keeps the input) → **Enter** (submits
`/harness ""`, which opens the picker) → **Down** → **Enter**. A bare
`/harness` + Enter submits `/harness <current>` because `builtin_option_current`
snaps an argument menu onto the running value, and hits the already-active guard.

**Two real bugs, both found by testing, both mine:**
1. ⭐ **The replaced agent process leaked.** `negotiate_and_run` did
   `tokio::spawn(driver)` and dropped the handle — safe only because
   `run_blocking` used to build a runtime per call and drop it on return, which
   shuts every task down → transport dropped → child guard dropped → process
   group killed. A long-lived supervisor runtime keeps the detached driver alive
   holding the transport: one leaked agent per switch. Fixed by keeping the
   `JoinHandle` and calling `driver.abort()` on every path out of the connect
   block. **Lesson: making a runtime long-lived turns every detached task in it
   into a leak.** Only the e2e's `assert_agent_gone` saw it — a `ps` after the
   run looks clean, because crow-term's own exit kills the group.
2. **The switch confirmation was invisible on a fresh pane.** The chat pane draws
   the welcome banner *instead of* the transcript while `show_banner` is set, so
   a user who switched before sending a prompt saw nothing. Fixed by pushing the
   transcript notice always and additionally tipping when the banner is up —
   the existing `CtlEvent::TuiOpDone` precedent.

**✅ COLLISION DECIDED — the builtin owns `/harness`, and there was never a
collision.** `/harness` also exists as a Node client plugin
(`npm/lib/harness-view.js:540`, input
`[id] [--new] | add | remove [id] | find [query]`) whose overlay the builtin now
shadows. That was filed as an open decision on the assumption the Node host was
a live surface. **It is not: `npm/lib` is dead code and never executes.** The
product is the Rust binary, run directly; there is no Cordis host, no plugin
tree, no agent pool. So there is no management overlay to preserve, no
"host-overridable builtin" category to invent, and nothing to rename. The
registry catalog / downloads / install / removal that plugin offered are not
"moved elsewhere" — they were never reachable in this product, and if harness
installation is ever wanted it gets built in Rust.

**Recorded in `AGENTS.md` under "⛔ READ THIS FIRST — this is crow-term, not
Martty"** so no future agent re-derives the Node layer as live. Corollaries:
the 434-test JS suite is not a gate and its totals must not be reported as
product health; Phase 7.3 is answered (the JS layer never needs v2); Phase 8.1
is obsolete rather than blocked; `npm/lib/tui-plugin-store.js marttyHome()`'s
missing `CROW_HOME` is moot; and the harness badge cannot come from a Cordis
slot snapshot, because nothing sends one.

**Scope, deliberately:** pick among *configured* harnesses and respawn. The
registry catalog, downloads, install and removal stay in Node.

## THE MANDATE

crow-term must speak ACP v2 to `crow-cli acp2` **without losing v1**. v1-only
agents stay common; the spec's own ruling is *"Treat v2 support as additive…
a Client that drops v1 loses access to existing Agents."* One connection speaks
exactly one negotiated version after `initialize`. v2 is gated behind
`unstable_protocol_v2` until it stabilizes.

## GROUND TRUTH — three sources, read them before coding

1. **The working Rust v2 client to port**: `~/.agents/crow/src/worktrees/crow-cli-rs/crow-cli/src/main.rs:1077-1233`
   (`run_session`) — `Client.v2()`, `UpdateSessionNotification` handler, the
   `StateUpdate::{Running,Idle}` → `watch::channel(TurnState)` machine,
   `wait_turn` (:1035) = cancel-then-wait-for-idle, `build_session_from` /
   `build_session_cwd` / `resume_session_from(..replay_from(ReplayFrom::Start))`,
   `send_prompt().block_task()`, `cancel_active_work()`, `close()`.
   `main.rs:1266-1308` (`auth_session`) = v2 `auth/login` + `auth/logout`.
   `render.rs:226-320` (`handle_update_to`) = the v2 `SessionUpdate` → display match.
   `agent.rs` (986 lines) = the v2 **agent** side; use it as the counterparty
   contract and as the model for a scripted Rust test peer.
   That worktree is ABANDONED as a product — mine it, do not revive it.
2. **The dual-stack ruling, already litigated in Python**: `~/.agents/crow/src/crow-cli/src/crow_cli/discover.py`
   (317 lines) + `~/.agents/crow/src/crow-cli/ACP_V2.md` §2 and §6.
   **The protocol is NEGOTIATED, not declared.** A config `protocol:` field is a
   second source of truth whose only talent is disagreeing with the first — it
   produced a hang with no error on either side. A declared value is at most an
   override that skips the round trip.
3. **The spec**: <https://agentclientprotocol.com/protocol/v2/migration> (fetched
   to `/tmp/acp-v2-migration.md`), and the machine-readable schema at
   `~/.agents/crow/src/python-sdk/schema/v2/schema.json` (265 `$defs`, alpha.3).

## ⚠️ RULING 1 — the hosted docs are AHEAD of both installed SDKs. Do not "fix" this.

The migration page says the v2 `session/prompt` response carries a **required
`messageId`**. It does not, in anything installed on this box:

| source | `PromptResponse` fields |
|---|---|
| website `protocol/v2/migration` | `messageId` (required), `_meta` |
| Rust `agent-client-protocol-schema` 1.5.0 (what crow-term pins) | `_meta` only |
| Rust `agent-client-protocol-schema` 1.7.0 | `_meta` only |
| Python `acp.experimental.v2` alpha.3 (`schema/v2/schema.json`) | `_meta` only |

`crow-cli acp2` is built on the Python alpha.3, so **the wire we must interop
with has no `messageId` in the prompt response.** Target the installed shapes.
Tolerate a `messageId` if a future agent sends one; never require it. The user
message's canonical id arrives as the `user_message` session update.

## ⚠️ RULING 2 — no dependency bump is needed, and 2.1.0 is a trap for now

`Cargo.toml` already pins `agent-client-protocol = "2.0.0"`, and **2.0.0 already
ships everything v2 needs**: the `unstable_protocol_v2` feature, `schema::v2`
(via `agent-client-protocol-schema` 1.5.0, whose `src/v2/` has the complete
stable `SessionUpdate` set), `Client::v2()`, and `Client::protocol_connector()`.
Verified: 1.5.0's v2 structs match Python alpha.3 field-for-field on
`ToolCallUpdate`, `PlanUpdate`/`PlanItems`, `TerminalUpdate`, `IdleStateUpdate`,
`SessionConfigOption` (`configId`), `RequestPermissionRequest` (`title`+`subject`),
`UpdateSessionNotification`.

2.1.0 adds `unstable_tool_call_name` + `unstable_session_compaction` but
**removes the `unstable_auth_methods` and `unstable_elicitation` features this
crate enables** — bumping is a separate, later decision (Phase 9), not a
prerequisite.

**The one real gap:** agent2 emits `ToolCallUpdate.name`; schema 1.5.0's
`ToolCallUpdate` has no `name` field. serde ignores unknown fields, so nothing
fails to parse — the tool name is simply absent from the typed struct. Because
Phase 4 forwards updates as **raw JSON**, `events.rs` can read `name` directly
and the gap costs nothing. Do not bump the dep to close it.

## ⚠️ RULING 3 — the connector was REJECTED; the union probe is hand-rolled

**This ruling was written backwards and has been corrected. Do not "fix" the
code back to `ClientProtocolConnector`.** The original text said the Rust SDK
does the union-probe for us (`ClientProtocolConnector::connect_to`,
`role/acp.rs:153-234`) and that `discover.py` need not be ported. That is true
of the SDK and false for crow-term. What was actually built is the hand-rolled
union probe in **`src/acp/negotiate.rs`** — i.e. `discover.py`'s mechanism,
ported. Two reasons, both fatal to the connector:

1. **It re-spawns every v1 agent.** The connector sends a *v2* initialize, and
   reuses the live connection only when both sides' normalized params match
   **exactly**. v2's `ClientCapabilities` has no `fs` and no `terminal`, while
   crow-term's v1 caps advertise both — so equality is unreachable by
   construction and every v1 agent falls through to the factory a second time.
   A v2-first probe with a re-spawn is not a union probe; it is two spawns.
2. **An attach endpoint cannot be re-created at all.** `AttachStdio{incoming,
   outgoing}` and `AttachTcp(TcpStream)` are one-shot inherited fds/sockets.
   The connector's contract is `FnMut() -> impl ConnectTo<Client>`, so the
   fallback path would have to hand back a peer that has already eaten the v2
   initialize. There is no dup that makes that legal.

The ruling that survives is the *ruling*, not the mechanism: **negotiate, don't
declare.** `negotiate.rs` builds ONE union initialize (`union_initialize_params`,
`v2_initialize_request`) and sends it over an already-live `Channel`
(`Channel::duplex()` at `:248`), then `classify`s the answer and `adopt`s it
into `Negotiated{protocol, init}`. The probe IS the connection — exactly
`crow-cli`'s `cli/main.py:_dispatch`, which is 30 lines and hands the already
handshaked child to whichever stack the answer selects. No factory, no second
spawn, and attach endpoints work unchanged because nothing is re-created.

**Corollary:** because there is no re-spawn, there is no v1-retry fallback
either. If the union initialize errors, fail fast with the agent's own message
(a settled decision — see Phase 2).

## ARCHITECTURE — where v2 lands

Today `src/acp.rs` (2979 lines) is ONE `connect<T: ConnectTo<Client>>()` that
builds `Client.builder()` with v1 handlers and runs a single giant
`connect_with(transport, |cx: ConnectionTo<Agent>| async { … })` command loop.
~~The connector needs two separate `ConnectTo<Agent>` implementations~~ — it
does not, and neither does the negotiator that shipped (see Phase 1 OBSOLETE
and RULING 3). The layout below is the ORIGINAL sketch, kept for provenance;
`src/acp/v1.rs` was never created and there is no `V1Client`/`V2Client` pair:

    src/acp.rs        → shared plumbing only: Surface, SessionSurface, bus
                        emission helpers, endpoint/transport construction,
                        the connector entry point
    src/acp/v1.rs     → today's client body, MOVED verbatim: `struct V1Client`
                        impl `ConnectTo<Agent>` (fs, terminal, set_mode,
                        session/load, prompt-response-is-the-turn)
    src/acp/v2.rs     → NEW: `struct V2Client` impl `ConnectTo<Agent>`
    src/acp/control.rs→ unchanged (already a submodule)

**The bus stays the seam.** v1 already forwards `session/update` as *raw JSON*
(`AppEvent::Rpc{method:"session/update", params}`) and `src/events.rs::parse_session_update`
is the single interpreter producing `Vec<UiEvent>`. v2 does the same, so the
entire UI/transcript layer is protocol-agnostic and the two protocols differ
only in (a) which JSON shapes arrive and (b) where turn-end comes from.

**The one thing that cannot stay shared:** turn lifecycle. v1 ends the turn on
the `PromptRequest` RESPONSE (`spawn_session_prompt` → `PromptFinish` →
`apply_prompt_finish` maps `response.stop_reason` → `TurnEnd{kind}` and
`response.usage` → `UiEvent::Usage`). v2's response is an ack; end-of-turn is
`state_update: idle{stopReason, usage}`. That is Phase 3 and it is the semantic
core of the whole sprint.

---

## Phase 0 — feature gate + proof of life                              [x]
0.1 Add `"unstable_protocol_v2"` to the existing `agent-client-protocol`
    features in `Cargo.toml`. Do NOT change the version. `Cargo.lock` must not
    move (schema stays 1.5.0).
0.2 A unit test that builds `schema::v2::InitializeRequest::new(ProtocolVersion::V2,
    Implementation::new("crow-term", env!("CARGO_PKG_VERSION")))`, serializes it,
    and asserts the wire keys are exactly `protocolVersion`/`info`/`capabilities`
    with `protocolVersion: 2` — and that a v2 `InitializeResponse` JSON with
    `capabilities.session.prompt.image = {}` deserializes.
0.3 A unit test that deserializes one JSON fixture per v2 `SessionUpdate`
    variant crow-term will handle (state_update idle/running/requires_action,
    agent_message, tool_call_update with `name`, tool_call_content_chunk,
    terminal_update, terminal_output_chunk, plan_update, config_option_update
    with `configId`) into `schema::v2::SessionUpdate`, asserting the variant
    landed and that `name` survives in the raw JSON even though the struct
    drops it (RULING 2's gap, pinned so a dep bump is visible).
    Criteria: `check --locked` + `test --locked` green, Cargo.lock unchanged,
    851+10 baseline still passes.

## Phase 1 — split the monolith, behavior-identical              [~] OBSOLETE

**Not done, and it should not be.** `ConnectionTo<Agent>` is ONE type for both
protocols in SDK 2.0.0 (`V2ConnectionTo` does not exist; the protocol lives in
the *Builder*, not the connection), so there is no seam along which to split a
`V1Client`/`V2Client` pair — both stacks take the same `cx` and the same shared
helpers. What was built instead: `src/acp/v2.rs` is a NEW file, nothing moved,
`connect()` stayed a free function in `src/acp.rs`, and the genuinely shared
surface (`TurnOutcome`, `apply_prompt_finish`, `bind_session`, `call_tui_extension`,
the auth helpers) is used by both. `src/acp/negotiate.rs` and `src/acp/control.rs`
are the two new seams that mattered. Splitting the monolith anyway would have
been churn with a compile error at every shared helper.

1.1 Create `src/acp/v1.rs`; move today's `connect()` body into
    `struct V1Client { cfg, bus, cmd_rx }` + `impl ConnectTo<Agent> for V1Client`.
    Move the v1-only helpers with it (`acp_fs`/`acp_term` wiring,
    `apply_session_modes`, `load_session_supported`, `initialize_request`).
1.2 `src/acp.rs` keeps: `AcpEndpoint`, `Surface`/`SessionSurface`, the shared
    helpers both stacks need (`bind_session`, `retarget_session`, `emit_auth`,
    `parked_prompt`, `create_prompt_session`/`resume_prompt_session` become
    per-stack or generic — decide by what compiles, not by taste), and transport
    construction.
1.3 Entry point stays `Client.builder()…connect_with(transport, …)` for now —
    NO connector yet. This phase is a pure move.
    Criteria: full suite green with ZERO test edits other than `mod` paths;
    `tests/tcp_attach.rs`, `tests/startup_session_e2e.rs`,
    `scripts/tui-multi-session.e2e.mjs` all still pass; a live PTY against a v1
    harness is pixel-identical to pre-split.

## Phase 2 — the negotiator: negotiate, don't declare                  [x]

**Built differently from 2.1–2.3 as originally written.** Per the corrected
RULING 3, `Client::protocol_connector()` was rejected. What shipped is
`src/acp/negotiate.rs`: one union `initialize` sent over an already-live
`Channel`, classified and adopted into `Negotiated{protocol, init}`, which the
v1 stack (`acp.rs`) or the v2 stack (`acp/v2.rs::connect`) then takes over.

2.1 ~~`Client::protocol_connector().with_v1(…).with_v2(…).connect_to(factory)`~~
    → `negotiate(channel) -> Negotiated` over `Channel::duplex()`. No factory,
    no second spawn. `Negotiated` is directly constructible from tests
    (`v1()`, `v2()`, private `expect(protocol)` guard), with `agent_name()` and
    `describe()` for the banner.
2.2 `AcpEndpoint::Spawn(argv)` → `AcpAgent::from_args(argv)` (already what
    `run()` uses at `acp.rs:1290`). Unchanged — but it is spawned ONCE, before
    the probe, not per factory call.
2.3 `AttachStdio{incoming: File, outgoing: File}` / `AttachTcp(TcpStream)` →
    **no special case needed any more.** The endpoint is still consumed by
    value, but nothing re-spawns, so the one-shot fds/socket are fine: the
    probe rides the connection the endpoint already produced. `tests/tcp_attach.rs`
    is green. (The old single-impl-only restriction existed solely to avoid the
    connector's re-spawn.)
2.3b **No v1-retry fallback.** If the union initialize errors, fail fast with
    the agent's own message rather than retrying as v1 — settled decision.
2.6 **DONE — `check_blocking` (`acp.rs:238`) was a SECOND v1-initialize site.**
    The `--check` path spawned an agent, sent `initialize_request()`, printed
    `init.agent_info.name`, exited. Against a v2 agent that is a `-32602`
    (`info` required) or a silent hang. It now dual-stacks and reports the
    negotiated version alongside the name — verified live:
    `target/debug/crow-term --check-runtime` against `crow-cli acp2` prints
    `agent /home/thomas/.local/bin/crow-cli acp2 / initialize ok in 1.8s →
    crow-cli acp2`.
2.4 **Not needed, and deliberately not built.** There is no `protocol:` field.
    Because the protocol is NEGOTIATED, a v2 harness entry is the same shape as
    a v1 one with a different argv (`["acp"]` vs `["acp2"]`) — see the two
    entries in `~/.agents/crow/settings.json`. A pin would only add a way to be
    wrong. Never guess from argv either.
2.5 **OPEN — surface the negotiated version in the TUI.** `check_blocking`
    reports it, the running pane does not. `CtlEvent::Initialized{server}`
    gains the protocol (only 3 sites: `bus.rs:98`, `acp.rs`, `app.rs:3861`), and
    the banner/cap row shows `acp` vs `acp2` — reuse the harness badge
    (`npm/lib/harness-badge.js` / `src/harness_badge.rs`) with `Protocol::tag()`.
    Criteria for 2.1–2.3: **MET** — `tests/unit/acp__negotiate_tests.rs` (11
    tests) drives two scripted peers over one `Channel::duplex()`: a v1-only
    peer that answers `protocolVersion: 1` (it MUST answer 1, not echo 2 —
    crow's settled ruling #6) lands on the v1 stack; a v2 peer lands on the v2
    stack. Plus: a banner printed before the answer reaches the adopter; a
    request the peer opened with is not eaten by the probe; an answer batched
    with other traffic is split not dropped; a refusal carries the agent's own
    words; an unspoken version is refused by number; a missing version is
    refused; a peer that hangs up is an error not a hang; a silent peer times
    out; the probe reuses id zero and the stack may reuse it too.
    `tests/tcp_attach.rs` still green. `check_blocking` against a v2 peer prints
    the agent name AND `acp2` instead of erroring or hanging (2.6, verified
    live). 2.5 remains open.

## Phase 3 — v2 turn lifecycle off `state_update` (THE CORE)           [x]
3.1 `src/acp/v2.rs`: port `run_session`'s notification handler. On
    `SessionUpdate::StateUpdate`: `Running` → mark the session busy;
    `Idle{stop_reason, usage}` → emit `TurnEnd{kind}` + `UiEvent::Usage` +
    `session.status idle`; `RequiresAction` → a waiting state (permission
    pending) that does NOT end the turn.
3.2 Stop-reason mapping, identical strings to v1's `apply_prompt_finish`:
    end_turn→completed, max_tokens→max-tokens, max_turn_requests→max-turn-requests,
    refusal→blocked, cancelled→interrupted, unknown→unknown.
3.3 `spawn_session_prompt` v2 variant: emit `TurnStart` + `session.status running`
    + `PromptQueued`, send the prompt, and treat the RESPONSE as *accepted*
    (RULING 1: it carries only `_meta`). Do NOT emit `TurnEnd` from it. Keep the
    `prompt_gen` tag so a stale idle can't clear a newer turn.
3.4 Cancel: port `wait_turn`. `session/cancel` (`CancelSessionNotification`) then
    WAIT (bounded, ~30 s) for `idle{cancelled}`; only then `TurnEnd{interrupted}`.
    Keep accepting updates that arrive after the cancel. Resolve any pending
    permission as cancelled immediately.
3.5 One foreground prompt per session: a v2 agent rejects a second
    `session/prompt` while foreground work is active. `Cmd::Steer` and the
    `parked`/`requeue_parked_prompts` machinery must key off the state machine,
    not off "is the request future still pending". A steer during `running`
    queues; during `idle` it sends.
3.6 Background updates after `idle` must NOT reopen a turn (spec: *"`idle` is
    not a wire boundary"*).
    Criteria: unit tests on a scripted v2 peer — (a) prompt → running → chunks →
    idle(end_turn) yields exactly one TurnStart and one TurnEnd{completed};
    (b) the ack response alone yields NO TurnEnd; (c) cancel → idle(cancelled)
    yields TurnEnd{interrupted} and nothing before the idle; (d) a second prompt
    while running is queued and fires after idle; (e) a post-idle
    `agent_message_chunk` paints but does not flip status to running.

    **Coverage as shipped — read this before trusting the `[x]`.** The
    implementation is complete; the criteria are pinned at a different altitude
    than worded. `tests/unit/acp__v2_tests.rs` pins the *board* (one outcome per
    armed turn, an unarmed idle is a heartbeat, rearming replaces the waiter,
    disarm leaves nothing to settle, a poisoned board still answers) and the
    stop-reason/usage mapping. (b) and (c) are pinned end-to-end on a real PTY by
    `a_v2_turn_ends_on_the_idle_state_not_on_the_prompt_acknowledgement`, which is
    airtight by construction: `app.rs:8071-8077` only sends `Cmd::Interrupt` from
    Running|Starting and `v2.rs:1373-1379` only emits `Interrupted` once the board
    settles on the cancelled idle, so drawing `interrupted` proves the `{}` ack did
    not end the turn. **(d) and (e) are implemented but NOT separately pinned** —
    `Cmd::Steer` is at `v2.rs:1073` with the `parked`/`requeue_connection_prompts`/
    `drain_ready_sessions` machinery, and (e) follows from the arm/disarm rule.
    If either regresses, nothing goes red. Adding a scripted-peer test for
    steer-while-running is the cheapest way to close that gap.

## Phase 4 — `events.rs`: the v2 discriminators                   [~] HALF

**Done and pinned** (`tests/unit/events__tests.rs`, all payloads verbatim from
`/tmp/wire-v2g.jsonl` and `-v2resume.jsonl`): the whole-message upserts
(`user_message`/`agent_message`/`agent_thought`), `state_update` belonging to
the stack rather than the parser, terminal updates claimed but not painted, a
first `tool_call_update` creating a cell nobody announced, `usage_update`
feeding the context meter, single-block live chunks, and `configId` vs `id`
through the one `config_id_of` reader.

**Still open:** per-`tool_call_id` patch semantics (omitted = unchanged,
null = cleared, value = replaced, chunks append — crow-cli's `_tools` dict);
reading `ToolCallUpdate.name` from raw JSON; `available_commands_update`'s
tagged-union input; `coalesce_session_updates`/`merge_update` must not merge two
v2 upserts; v2's `category`. And the standing ruling below is still unimplemented.
4.1 `parse_session_update` gains: `state_update`, `agent_message`,
    `user_message`, `agent_thought` (whole-message upserts — `content` is a
    LIST; chunks carry ONE block), `tool_call_content_chunk`, `terminal_update`,
    `terminal_output_chunk`. Keep the existing `plan`|`plan_update` arm.
4.2 Upsert semantics, per spec: omitted = unchanged, `null` = cleared, value =
    replaced, chunks append. A whole-message update REPLACES content accumulated
    from earlier chunks. crow-term's transcript is append-only today — decide
    per event: `agent_message` after chunks is a *replace* of the current
    message, so it needs a messageId-keyed patch path in `transcript.rs` or a
    documented suppression when the chunks already painted the same text.
    **Write the decision into the code comment.** Do not silently double-paint.
4.3 `tool_call` (v1 create) does not exist in v2 — the first `tool_call_update`
    for an unseen `toolCallId` CREATES it. The existing `tool_call_update` arm
    already half-does this; make it explicit and read `name` from the raw JSON
    (RULING 2).
4.4 `terminal_update`/`terminal_output_chunk`: base64, **decode each chunk
    independently then append bytes** (never concatenate encoded strings);
    chunk boundaries may split UTF-8 and ANSI, so the parser retains state;
    a later `output` snapshot REPLACES all accumulated bytes. Display-only:
    no input, resize, interrupt, kill, wait, or release. This is NOT
    `acp_term.rs`'s client-executed terminal — do not reuse its broker.
4.5 Config options: read `configId` (v2) alongside `id` (v1) in
    `config_option_events`, `catalog_from_config_options`,
    `reasoning_effort_option`, `composition_option`, and `Surface::apply_config_options`
    (which currently sniffs `o.get("id") == Some("mode")`). Use v2's `category`
    (`mode`|`model`|`model_config`|`thought_level`) when present, tolerate
    unknown categories, and keep the `id == "mode"` sniff for v1.
4.6 `available_commands_update`: a command's `input` is now a tagged union
    (`{"type":"text","hint":…}`). `skills_from_available_commands` must read both.
4.7 `coalesce_session_updates`/`merge_update` must not merge two v2 upserts into
    one — merging a replace into an append loses the replace.
    Criteria: `tests/unit/events__tests.rs` gains a fixture per new discriminator
    (reuse Phase 0.3's JSON); a v2 `config_option_update` with `configId`
    populates the model catalog; a terminal chunk pair split mid-UTF-8 decodes
    correctly; a whole-message update after chunks paints once.

## Phase 5 — v2 auth                                                   [ ]
5.1 `acp_auth.rs`: `authenticate` → `auth/login`
    (`LoginAuthRequest::new(AuthMethodId::new(..))` + `_meta.value` for a
    supplied secret — see `crow-cli-rs/main.rs:1286-1294`); add `auth/logout`
    (`LogoutAuthRequest::new()`).
5.2 `authMethods[]`: `id` → `methodId`, plus a REQUIRED `type` discriminator
    (`"agent"` = protocol-driven login; custom types start with `_`).
    `parse_auth_methods`/`declared_auth_methods`/`snapshot_from_methods` read
    both spellings.
5.3 Non-empty `authMethods` ⇒ the agent MUST implement both login and logout,
    and there is no logout capability marker. Omitted/empty ⇒ the client MUST
    NOT call either. Enforce both directions.
5.4 Capability reads become PRESENCE checks: v1's
    `promptCapabilities.image == true` → v2's `capabilities.session.prompt.image != null`.
    `prompt_image_supported`, `load_session_supported`, `resume_session_supported`,
    `list_session_supported` all need a v2 branch — and in v2, advertising
    `capabilities.session` at all implies list/resume/close/prompt/cancel/update
    are REQUIRED, so no probing. `session.delete` and `session.additionalDirectories`
    keep their markers.
5.5 The `marttyConnection` error-data stall path, `failed_auth_setups`, and
    prompt parking must work unchanged on the v2 stack.
    Criteria: `tests/unit/acp_auth__tests.rs` v2 fixtures — a v2 agent
    advertising one `type:"agent"` method drives the existing overlay to a
    successful `auth/login` and then resumes the parked prompt; an empty
    `authMethods` never emits an auth CTA; a v2 `initialize` response sets
    `prompt_image` from `capabilities.session.prompt.image`.

## Phase 6 — retire the removed client surface on v2 connections   [~] PART

**Done and pinned:** 6.1, 6.2, 6.4, and the `title` half of 6.6.
**Still open:** 6.3 (modes as config options), 6.5 (v2 diffs), and the
`command`-subject *rendering* half of 6.6.

6.1 **DONE** — v2 advertises auth only: `negotiate.rs:157` builds
    `v2::ClientCapabilities::new().auth(v2::AuthCapabilities::new())`. No `fs`,
    no `terminal`, no elicitation, no nes; `acp_fs.rs`/`acp_term.rs` handlers are
    not registered on the v2 builder (`v2.rs:734` is a bare `.v2()`). (v2's
    `ClientCapabilities` has only `auth`/`elicitation`/`nes`/`position_encodings`;
    there is nothing to decline.)
6.2 **DONE** — `session/load` → `session/resume` with `replayFrom: {"type":"start"}`
    (`ResumeSessionRequest::new(sid, cwd).replay_from(ReplayFrom::Start(..))` →
    `connection.resume_session_from(req)`). Install the update handler BEFORE
    resuming — replay updates precede the resume response on the wire.
    Plain reconnect = `session/resume` with no `replayFrom`.
    `v2.rs:531` does exactly this and the handler is installed before the send;
    pinned by `a_v2_resume_repaints_the_transcript_the_agent_replays`, which
    asserts `request_params("session/resume")["replayFrom"]["type"] == "start"`
    and that the replayed rows are drawn once each.
6.3 **OPEN.** `session/set_mode` + `current_mode_update` + `apply_session_modes` become
    v1-only. On v2 the same state is a config option with `category:"mode"`,
    changed via `session/set_config_option` and reported via `config_option_update`.
    The mode picker UI must work against both.
6.4 **DONE** — `session/set_config_option` v2 shape: `(sessionId, configId, type, value)`
    where `type` is `"id"` (string option id) or `"boolean"`. The response
    returns the FULL updated `configOptions` array. v1's `SetSessionConfigOptionRequest`
    path in `apply_config_response` needs a v2 sibling. Shipped as the
    `set_config_option` helper at `v2.rs:587`, called from all five sites
    (`:1216`, `:1228`, `:1244`, `:1297`, `:1324`) — model select, efforts,
    presets and `Cmd::SetConfigOption` all ride it. Pinned by the resume test's
    `config["configId"] == "model"` / `config.get("id").is_none()` assertions.
6.5 **OPEN.** Diffs: v2 replaces `oldText`/`newText` with `changes[]` (add/delete/modify/
    move/copy, `fileType`, `mimeType`) + optional `patch{format:"git_patch", text}`.
    Render `patch.text` when present, drive file trees/summaries from `changes`,
    and handle a patch-less diff (binary, symlink, directory). There is NO
    mechanical mapping back to oldText/newText — do not attempt one.
6.6 **HALF DONE** — the ask renders and the answer reaches the agent; the
    `command` subject is parsed but its `command`/`cwd` are not yet drawn.
    Pinned by `a_v2_permission_ask_reaches_the_user_and_the_answer_reaches_the_agent`
    (title, both option names and kinds on screen; Enter selects the first
    `allow_once`; the wire reply is `{"outcome":{"outcome":"selected","optionId":"allow"}}`).
    Permissions: v2 `session/request_permission` has a REQUIRED `title` (prompt
    copy) and an optional `subject` tagged union (`tool_call` | `command`). Stop
    reading the tool call's `title` as the prompt text (`acp.rs:1653` does
    exactly that today). Apply a `tool_call` subject like an ordinary tool-call
    upsert; render a `command` subject from `command` + required absolute `cwd`;
    show a generic prompt when `subject` is absent or unknown. While pending,
    expect `requires_action`.
    Criteria: a v2 connection registers no fs/terminal handler and a scripted
    peer that sends `fs/read_text_file` gets method-not-found, not a hang;
    resume-with-replay paints history then goes idle; the mode picker switches a
    v2 agent's mode via `set_config_option`; a v2 permission request renders its
    `title` and a `command` subject.

## Phase 7 — launching a v2 agent: harness + config + the JS layer     [ ]
7.1 Add the real target as a harness entry so it is launchable from the TUI:
    `uv --project /home/thomas/.agents/crow/src/crow-cli run crow-cli acp2`
    with `mcpServers: {crow-mcp2: {transport: stdio, command: …/.venv/bin/crow-cli, args: [mcp2]}}`
    — this is the working `v2-agent` entry in `~/.agents/crow/config.yaml`.
    Note: `mcp2` takes NO `--include-tools`, and agent2 reads tool supply from
    `request.mcp_servers`, never from config.
7.2 Per RULING 1 of the mandate: NO `protocol:` field in the harness schema —
    **settled, and stronger than "by default".** The negotiator asks; there is
    no pin override at all (Phase 2.4 was rejected, not deferred). A v2 harness
    entry is the same shape as a v1 one with a different argv, which is exactly
    how `crow-cli` / `crow-cli-v2` are configured in `~/.agents/crow/settings.json`
    today. Nothing to build here.
7.3 [x] ANSWERED, and the answer is stronger than "stays v1": **the JS layer is
    dead code and never needs v2 at all.** `npm/lib/acp-client.js`,
    `acp-host.js`, `acp-agent-pool.js`, `acp-session-*.js` and
    `acp-registry.snapshot.json` do not execute — the product is the Rust binary
    run directly, with no Cordis host and no plugin tree (see `AGENTS.md`,
    "⛔ READ THIS FIRST"). Nothing there has to list a v2 agent or hand over an
    argv; `src/harness.rs` reads `~/.agents/crow/settings.json` itself. The only
    live question left about `npm/` is whether the delivery wrapper
    (`bin/martty.js` + the vendored static ELF) stays as the install path or the
    package is dropped — that is packaging, not protocol, and it is the user's
    call, not this plan's.
7.4 `src/harness.rs` / `harness_badge.rs` / `harness-discovery.js`: a v2 agent
    is discovered, badged, and switchable like any other.
    Criteria: `crow-term` launched from a clean HOME with the v2 harness
    selected initializes, shows `acp2` in the badge, prompts, and paints; a v1
    harness in the same install still works; `scripts/harness-*.test.mjs` green.
    **Switchable-in-Rust is DONE** — see the SECONDARY MANDATE section; the
    picker switches between a v1 and a v2 recipe live and the e2e proves the
    protocol changed (the `crow-mcp` stdio entry gains `"type":"stdio"`).
    **Badged is the remaining half**, and it is Phase 2.5's job: the badge comes
    from a Cordis slot snapshot (`conversation.harness`), i.e. Node-driven, so
    the bare binary shows nothing. Reuse it with `Protocol::tag()` = `acp`/`acp2`
    once `CtlEvent::Initialized { server }` carries the negotiated version.
    The badge must be driven from the Rust side — the Cordis slot snapshot that
    feeds `app.harness_badge` today never arrives, because nothing sends one.

## Phase 8 — e2e against a v2 agent                             [x] (8.1 obsolete)

Done against the STUB, not the live agent, and that was the right call: a live
`crow-cli acp2` run burns model tokens, needs `alibaba` reachable, and cannot
run in CI. The stub now answers the union `initialize` as a v2 agent
(`STUB_PROTOCOL=2`), parks a turn mid-flight (`STUB_HOLD=1`), and asks for
permission (`STUB_ASK_PERMISSION=1`). Every shape it sends was copied from a
real capture (`/tmp/wire-v2g.jsonl`, `-v2resume.jsonl`), so the stub is a
recording of the live agent, not an invention. Live verification against
`crow-cli acp2` was done separately over a teed PTY and is recorded in the
session notes.

8.1 ⛔ OBSOLETE, not blocked. `scripts/real-agent-e2e.py` drives
    `dsh --profile tui-test` — the NODE host — which does not exist in this
    product (`npm/lib` never executes; Phase 7.3). It cannot be "unblocked" by
    giving the JS layer v2, because there is no JS layer to give it to. The
    equivalent coverage lives in `tests/startup_session_e2e.rs`, which drives the
    shipped Rust binary on a real PTY. Delete the script and its
    `real-agent-e2e.test.mjs` / `make real-agent-e2e` when the npm packaging
    question is settled.
8.2 [x] `a_v2_resume_repaints_the_transcript_the_agent_replays` — `session/resume`
    with `replayFrom:{type:start}`, no `session/load` on the wire, whole-message
    upserts repaint, and the replayed prompt is drawn exactly once.
8.3 [x] `a_v2_turn_ends_on_the_idle_state_not_on_the_prompt_acknowledgement` —
    Esc only sends `Cmd::Interrupt` from Running, and `CtlEvent::Interrupted`
    only fires once the board settles on the cancelled idle, so drawing
    `interrupted` is proof the `{}` ack did not end the turn.
8.4 [x] `a_v2_permission_ask_reaches_the_user_and_the_answer_reaches_the_agent` —
    the overlay draws v2's required top-level `title` plus the option list; the
    agent gets `{"outcome":{"outcome":"selected","optionId":"allow"}}`.
8.5 [x] folded into 8.2: `--model` writes `configId`, and `id` is absent.
8.6 [x] the six pre-existing v1 e2e tests are untouched and green, and the stub's
    v1 path is byte-for-byte the behaviour it had before (`STUB_PROTOCOL` defaults
    to 1). Also `a_v2_agent_is_negotiated_and_its_tool_supply_carries_its_transport`
    fails on a v1 connection by construction — an untagged stdio server — so it
    proves the v2 stack actually ran rather than passing vacuously.

Mutation-verified (each assertion was made to fail on purpose, then restored):
  * forwarding every update → the prompt prints twice, verbatim the reported bug;
  * never opening the replay window → the resumed transcript loses its prompt;
  * dropping the version-neutral lane → the `acp2: AgentsSnapshot { … } is not
    supported on a v2 connection` flood, one row per status change.

Suite: 902 unit + 2 cli_help + 1 sigterm + 11 startup_e2e + 1 tcp_attach, and
11 + 434 JS (432 pass, 2 skipped). `npm/node_modules` had never been installed
in this checkout — `npm ci` before trusting any JS result.

## Phase 9 — decisions to revisit once v2 works                        [ ]
9.1 Dep bump to `agent-client-protocol` 2.1.0: gains `unstable_tool_call_name`
    (typed `name`) + `unstable_session_compaction`, but drops the
    `unstable_auth_methods`/`unstable_elicitation` features we enable. Only worth
    it if the raw-JSON `name` read (4.3) proves insufficient.
9.2 The git-rev dep `crow-cli-rs` uses (`rust-sdk` rev `7d21931`, carrying
    conductor v2 proxy init #302, http slow-stream #292, unstable v2 APIs
    #295-#298) — only if a crates.io release lacks something we need. crow-term
    ships via npm + a static ELF; a git dep breaks `--locked` reproducibility.
9.3 Multi-client observe / a verifier that attaches to the same `sessionId` —
    the thing v2's notification-driven lifecycle actually unlocks. Out of scope
    here; note it in `docs/acp-v2.md` as the reason v2 matters beyond parity.
9.4 JSON-RPC batch arrays on stdio (v2 explicitly allows them). Check whether
    the SDK's transport already handles a batch line; if not, that is a bug to
    file upstream, not to patch locally.

## Phase 10 — docs + CHANGELOG                                         [ ]
10.1 `docs/acp-v2.md` (new): the two rulings, the architecture split, the
     turn-lifecycle difference, what was removed, and how to add a v2 harness.
10.2 `docs/architecture.md` + `docs/architecture.en.md`: the `acp/negotiate.rs`
     + `acp/v2.rs` seams (NOT an `acp/v1.rs` split — it does not exist) and the
     union probe.
10.3 `docs/harness-management{,.en}.md` / `docs/harness-cli.md` / `docs/harness-tui.md`:
     negotiated-not-declared, i.e. a v2 harness is an argv not a flag. No pin
     override exists to document.
     **These are Chinese-first docs** — write them in Chinese directly, do not
     treat the `.en.md` as the source.
     **Unblocked** — the `/harness` collision is decided (SECONDARY MANDATE):
     the Rust builtin owns the name and the Node plugin is dead code. Document
     the builtin as the only `/harness` there is, and do not document the
     `npm/lib/harness-view.js` overlay as a live surface.
     ⚠️ `docs/plugins.md` and `docs/migration.md` describe the dead Cordis/plugin
     architecture. Either mark them historical the way `AGENTS.md` now does, or
     leave them out of the "read this before changing UI" path — an agent that
     reads them as current will design against a host that does not exist.
10.4 `CHANGELOG.md` `[Unreleased] ### Added`. The `/harness` picker entry is
     written; the v2 negotiation entries are not.
     Criteria: no doc claims a behavior the tests do not pin.
