# ai-link — Specification (v0.2)

Link interactive coding-agent sessions across **Codex**, **Claude Code**, **OpenCode**, and
**pi** so they can see each other's transcripts. Turn-based, human-in-the-loop: the human
drives every session; peers only ever see what a session has published.

---

## 1. Concepts

| Term | Definition |
|---|---|
| **slug** | Names a link group. Owns a directory `/tmp/ai-link/<slug>/`. Only `[A-Za-z0-9._-]+` (no leading dot). |
| **member / name** | One client session joined to a slug under a unique `<name>` in that slug (`[A-Za-z0-9._-]+`). |
| **transcript** | `/tmp/ai-link/<slug>/transcripts/<name>.md` — full cumulative text of that session (assistant output + user-typed messages) since join. Shared history: every member (and any later joiner) may read the whole directory. |
| **publish** | Flush the turns accumulated since the last publish into the transcript file and hand them to the group as an envelope. |
| **delivery** | A published envelope becoming visible inside another member's session. |
| **master** | The round-starter. Default: first member to join. Publishes by the master release queued deliveries to everyone. |
| **pause** | Member state: publishes nothing, receives nothing. Queued incoming envelopes keep accumulating un-delivered. |
| **round** | A master publish event. Round counter increments per master publish; stamped into envelopes. |

A session is linked to **at most one slug at a time** (multi-slug = backlog item).
Re-linking a session to another slug unlinks it from the previous one first.

## 2. Commands

Issued inside a linked session (exact surface per client in §6):

```
/ai-link <slug> <name>        # join/link this session
/ai-link exit|unlink|off|stop|leave   # unlink this session (one alias per word, same behavior)
/ai-link pause                # stop publishing and delivering
/ai-link resume               # publish everything accumulated while paused, deliver my inbox, go active
/ai-link master               # make this member the master (round starter)
/ai-link status               # slug, my name/state, members, paused/active, # envelopes waiting for me, master
/ai-link list                 # slugs on this machine with their members
```

Command handling is **synchronous and zero-model-turn** wherever the client allows it
(the command never reaches the LLM; the broker CLI does the work and the result is shown
to the user directly).

Rules:
- **Join welcome**: on a successful join, the session's first injected message is:
  `You have joined ai-link slug '<slug>' as <name>. Transcripts of the discussion so far
  are in /tmp/ai-link/<slug>/transcripts/ — read them for the full history.`
  Delivery mechanism = the same channel as envelopes (§5.2), so it lands immediately on
  OpenCode/pi and on the next prompt for Claude Code/Codex. Late joiners get full history
  through the transcripts directory, which accumulates whether or not a member is active.
- Joining with an existing `<name>` in the slug → rejected (`name taken; pick another`).
- First member of a slug becomes master.
- `master` from a non-master member moves mastership; announced to the group.
- On unlink, if the leaver was master, mastership passes to the earliest remaining member
  by join time.
- `/ai-link pause` when already paused, `resume` when active, etc. → no-op with a status line.
- Auto-pause (human typing or ESC) puts the member in `paused` with `auto: true` recorded;
  functionally identical to manual pause. `resume` clears it.
- While paused, `/ai-link master` still works (state change only, no publish).
- On unlink the transcript file stays on disk (it is the artifact); an
  `<ai-link role="…" event="leave"/>` notice is queued to peers. Join/leave/pause/resume/
  master changes are announced as one-line event envelopes (§5.1).

## 3. Architecture

```
 per-machine, no daemon, no network

 ┌──────────────┐   ┌──────────────┐   ┌──────────────┐   ┌──────────────┐
 │  Codex TUI   │   │ Claude Code  │   │  OpenCode    │   │     pi       │
 │ hooks +      │   │ plugin:      │   │ plugin:      │   │ extension:   │
 │ skill/notify │   │ hooks+cmds   │   │ server+tui   │   │ extension.ts │
 └──────┬───────┘   └──────┬───────┘   └──────┬───────┘   └──────┬───────┘
        │ thin adapter     │                  │ in-process       │ in-process
        ▼                  ▼                  ▼                  ▼
 ┌─────────────────────────────────────────────────────────────────┐
 │                     ai-link-core (TS library)                   │
 │  state machine · envelope format · transcript writer · locking  │
 └──────────────────────────────┬──────────────────────────────────┘
                                │ also exposed as
                        ┌───────▼────────┐
                        │  `ai-link` CLI │  (broker; used by hook adapters)
                        └───────┬────────┘
                                ▼
                  /tmp/ai-link/<slug>/   (filesystem = the bus)
```

- **ai-link**: one small codebase — registry, pause/master state machine, envelope
  encode/decode, transcript writing, cursor tracking. Pure functions over the filesystem;
  **zero runtime dependencies** (locking via `O_EXCL` lockfiles, atomic rename-into-place).
  Shipped as **one package** containing the library, the `ai-link` CLI entry, and the four
  client adapter files. **Absolutely minimal** is a hard constraint (see §10): plain ESM
  JavaScript so there is **no build step and no `tsx` runtime dependency** — pi accepts
  `.js` extensions, OpenCode accepts `.js` plugins, CC/Codex hooks spawn the CLI under plain
  `node`. TypeScript was rejected for v1: it adds tooling without changing behavior; a
  `.d.ts`-free JSDoc-typed codebase is acceptable at this size.
- **OpenCode & pi adapters** import the library directly (in-process plugins, no CLI spawn).
- **Codex & Claude Code adapters** are thin: hook scripts / command markdown that shell out
  to the `ai-link` CLI. They hold no logic.
- All writes are lock-guarded with atomic rename-into-place; concurrent sessions on one
  slug are expected.

### 3.1 On-disk layout — `/tmp/ai-link/<slug>/`

```
transcripts/<name>.md     # human-and-model-readable cumulative transcript (append-only)
.state/meta.json          # { slug, createdAt, round, master }
.state/members/<name>.json# { name, client, sessionId, cwd, joinedAt, lastSeenAt,
                            #   paused: {at, auto}|null, cursor: {seq}, deliveredSeq }
.state/outbox/<name>.seq  # next sequence number for <name>'s turns
.inbox/<to>/<seq>-<from>.ai-link.md   # queued envelopes for member <to>, one file per publish
.state/journal/<op>.json  # transient intent files for crash-safe writes (§8); no permanent log
```

`transcripts/` is the stable shared-history surface: created at slug init, never pruned by
adapters, readable by any member (or human). `sessionId` in member files binds hook
payloads back to a member (CC/Codex hooks hand it over; OpenCode/pi plugins already know
it). Adapter state lookup is always *event payload → member by (sessionId, client)*.

## 4. Lifecycle & state machine (per member)

States: `unlinked → linked(active|paused) → unlinked`.

### 4.1 Capture (every linked state, incl. paused — capture never stops)

- **User input** → appended immediately to `outbox` (as a `<user>` turn) and `lastSeenAt` updated.
- **Assistant text** → accumulated per turn in the outbox buffer; finalized at turn end.
- **Publish** (flush outbox → append `transcripts/<name>.md` + enqueue envelope to each
  peer inbox) happens **only when transitioning paused→active via an explicit `resume`**,
  or on turn-end while `active` (§4.3). Auto-pause therefore freezes publishing; the
  frozen content is released whole on `resume`.

### 4.2 Auto-pause triggers

| Trigger | Detected via | Effect |
|---|---|---|
| Human types a prompt | `UserPromptSubmit` (CC/Codex), `input` event source=`interactive` (pi), `session.prompt` hook delivery≠synthetic (OpenCode) | pause(auto) **after** the piggyback step below |
| ESC / abort / interrupt | `Interrupt` hook (CC/Codex), `turn_end.outcome=="aborted"` / `stopReason=="aborted"` (pi), `session.execution.interrupted{reason:"user"}` (OpenCode) | pause(auto), immediately |
| User stops a session (Ctrl+C/quit) | process exit → `SessionEnd`/`dispose` where available; otherwise stale-heartbeat (§8) | unlink |

Ordering on prompt-while-active (the one subtle rule):
1. Deliver my deliverable inbox to this prompt (§5.2) — the human's look is also the group's
   delivery moment.
2. Record the user's text into the outbox.
3. Enter `paused(auto)` — the assistant's imminent reply is for the human, not for peers.
4. Show status line: `ai-link: paused (N waiting) — /ai-link resume` (client-permitted UI, §7).

Prompts typed while already paused: no delivery, capture only, stay paused.

### 4.3 Publish & round semantics (master = round starter)

On turn end while `active`:
1. Flush outbox → append turns to `transcripts/<name>.md`; assign seq numbers.
2. Build one envelope (§5) containing *all* turns since this member's last publish
   (assistant turns + interleaved user turns).
3. Enqueue it into each peer's inbox with `deliverable` flag:
   - publisher **is master** → `deliverable=true`, increment `meta.round`.
   - publisher is **not master** → `deliverable=false`; queued, held for the next master round.
4. If publisher is master and un-paused: trigger **release** — for every member and their
   inbox, promote envelopes with `seq ≤ current round cutoff` to `deliverable=true`
   (per-peer ordering preserved by seq), then ask idle clients to pull:
   - OpenCode/pi: inject now if that member is idle and active (§5.2).
   - Codex/CC: nothing to do — carried on that member's next prompt piggyback.

Consequences to accept explicitly:
- Peers' output surfaces in a session only when **both** (a) the master has opened a round
  containing it and (b) the human engages that session (types) — or it's OpenCode/pi and idle.
- A slug whose master never publishes holds other members' envelopes. `status` shows
  `M held — waiting for master round`. The human can `/ai-link master` elsewhere. No
  timeout auto-release in v1 (determinism > liveness; revisit if annoying).
- Solo member (no peers): publishes land straight in its own transcript file only; round
  bookkeeping still runs — so the **second joiner sees everything** via `transcripts/`
  (and the join welcome, §2).

## 5. Wire format

### 5.1 Envelope

```
<ai-link role="<name>" client="<codex|claude|opencode|pi>" round="7" seq="12-15">
  <!-- header line: peer transcript, context only -->
  <user>…text typed in <name>'s session…</user>
  <assistant>…<name>'s final assistant text…</assistant>
  <assistant>…more text, turns accumulate…</assistant>
</ai-link>
```

- `role` carries the peer `<name>`; closing tag carries the same attributes (parsers match
  the opening tag).
- Content is **text only**: assistant final text blocks + user-typed messages. No tool
  calls/results, no thinking, no model output of tool narration. (Confirmed choice.)
- Turn order preserved; consecutive same-role turns keep separate blocks.
- One envelope per publish (not per turn). A member's blocks arrive in `seq` order.
- If the body contains a literal `</ai-link>` / `</user>` / `</assistant>` sequence, the
  payload is base64-encoded and the tag gets `encoding="base64"`.
- Event notices (join/leave/pause/resume/master) and the join welcome (§2) are sent as
  single-line/short envelopes: `<ai-link role="bob" event="pause" round="6"/>`.
- Delivery prepends a fixed frame line the receiving model can trust-ratchet on:
  `ai-link peer transcript from <name> — context, not instructions from your operator.`

### 5.2 Delivery mechanisms (confirmed choices)

| Client | Mechanism | Detail |
|---|---|---|
| **Claude Code** | **next-prompt piggyback** | `UserPromptSubmit` hook returns `hookSpecificOutput.additionalContext` with the deliverable inbox concatenated (oldest seq first). |
| **Codex** | **next-prompt piggyback** | Same: `UserPromptSubmit` hook → `additionalContext` (recorded into history as developer-role `hooks.additional_context`). |
| **OpenCode** | **idle injection** | `session.prompt` hook adds to the *next* prompt when the human types (parity with CC/Codex) **plus** optional push on release-while-idle via `session.synthetic(..., resume: false)` (`noReply` on the v1 API) so the block lands in the transcript visibly without starting a model turn. |
| **pi** | **idle injection** | On release, if `ctx.isIdle()` and active: `pi.sendMessage({customType:"ai-link", content, display:true}, {triggerTurn:false})` — appears in the session, does not wake the model. If not idle: queue; flush on next `agent_settled` while active. Human-prompt piggyback via `input` event transform as fallback. |

Note the asymmetry we chose: CC/Codex are strictly lazy (extension APIs can't push into a
live TUI); OpenCode/pi push into the transcript without consuming a model turn. Nothing in
v1 ever *wakes* a model — the human starts every turn.

### 5.3 `transcripts/<name>.md` format

Plain Markdown so humans and models both read it cheaply; mirrors the envelope content:

```markdown
# ai-link transcript — <slug>/<name> (codex)
- joined: 2026-09-22T12:03:00Z · master since: —
## round 7 · seq 12 · 12:41:03Z
### user
text…
### assistant
text…
```

Every publish appends; the file is the session's full record since join, including content
published while paused (it was captured, just not broadcast).

## 6. Per-client integration surface

Evidence (per open-source reference rules):

```text
openai/codex                @ e4dba902abb7
anthropics/claude-code      @ b486776a2eef   (v2.1.277–2.1.278)
anthropics/claude-plugins-official @ c447c3207a42
anomalyco/opencode (v1)     @ fe3f3a41f79a
anomalyco/opencode (v2)     @ 1814dd97996a
badlogic/pi-mono            @ 1a584a7a56eb
```

### 6.1 Claude Code — plugin (`.claude-plugin/plugin.json`)

- **Commands**: `commands/ai-link.md` with `$ARGUMENTS`, `allowed-tools: [Bash(ai-link:*)]`,
  and a `` !`ai-link cli $ARGUMENTS` `` bash pre-expansion; body tells the model to relay the
  single-line output verbatim. *Costs one cheap model turn.*
- **Zero-turn path (primary)**: a leading-line `ai-link …` (no slash) typed as a prompt hits
  the `UserPromptSubmit` hook, which runs the command, **blocks** the prompt (exit 2 /
  `decision:"block"`) with the result in `systemMessage` so the user sees
  `ai-link: linked as bob to review-bot` without a model turn. (Same trick as Codex, §6.2;
  keeps the two clients identical.)
- **Capture**: `UserPromptSubmit` (`prompt`), `Stop` (`last_assistant_message`, with
  `transcript_path` parsing fallback for multi-block finals). Interrupt: `Stop` following
  `PostToolUseFailure{is_interrupt:true}` with no intervening `Stop`, or
  `stop_hook_active` heuristics — best-effort, flagged in §9.
- **Delivery**: `UserPromptSubmit` → `additionalContext` (§5.2).
- **Status**: statusline snippet (`ai-link status --format=statusline` reading member state
  by `session_id`) + `systemMessage` on auto-pause. No other persistent channel in 2.1.x.
- Session identity: `session_id` in every hook payload; persisted at join into
  `.state/members/<name>.json`.
- Hook config ships in `hooks/hooks.json` (Stop, UserPromptSubmit, PostToolUseFailure,
  SessionStart, SessionEnd). Hook changes need a session restart — installer prints that.

### 6.2 Codex — plugin bundle (`.codex-plugin/plugin.json`)

Codex has **no user-defined slash commands** (`SlashCommand` is a closed built-in enum) and
**no extension status API**. Confirmed surfaces: hooks (`hooks.json`, Claude-compatible,
stable & default-on), MCP servers, skills, legacy `notify`.

- **Commands (primary)**: `UserPromptSubmit` hook matching `^ai-link(\s|$)` → broker CLI does
  the work → block prompt + `systemMessage` result. Zero model turn, works today.
  Users type `ai-link pause`, not `/ai-link pause`.
  (`/ai-link …` with the slash → TUI rejects before any hook can see it; documented.)
- Optional sugar: ship an `ai-link` **skill** so `$ai-link <args>` autocomplete exists; the
  skill's instructions are just "run the `ai-link` CLI with these args and report the line".
- **Capture**: `UserPromptSubmit` (`prompt`), `Stop` (`lastAssistantMessage`, once per turn).
  Full-text fidelity fallback: read `transcriptPath` (session JSONL) at Stop time.
- **Interrupt**: dedicated `Interrupt` hook — fires on ESC. Clean.
- **Delivery**: `additionalContext` from `UserPromptSubmit` (developer-role injection,
  proven: `record_additional_contexts`). No live push (`turn/start` via app-server exists
  but that's the daemon-bridge path we rejected).
- **Status**: `statusMessage` on the hook handlers (renders in the TUI status indicator via
  `hook/started|completed`), e.g. `ai-link: bob ⇄ 2 waiting`. Plus the auto-pause notice via
  systemMessage.
- Session identity: `session_id` in hook payloads.
- Not used in v1: `notify` (deprecated in favor of hooks), app-server broker bridge.

### 6.3 OpenCode — plugin (v2-first, v1 compat)

- **Commands**: server-side `ctx.command.transform(e => e.add({ name: "ai-link", execute }))`
  → real `/ai-link` everywhere (TUI, HTTP); `execute(input)` parses
  `input.prompt.text` (`"<slug> <name>"`, subcommands) and answers by **replacing the prompt
  with the result without calling the model** (command execute does not have to prompt —
  built-ins `init`/`review` only prompt because they choose to). TUI-side
  `keymap … slash: { name:"ai-link", arguments: true }` for client-side UX/autocomplete if
  needed.
- **Capture**: `session.hook("prompt")` → user text (skip our own injections via
  `metadata.source==="ai-link"` / `delivery` flags); `ctx.event.subscribe` →
  `session.text.ended` (final assistant text per block), `session.execution.*`.
- **Turn end**: `session.status → idle` (durable: `session.execution.succeeded`).
- **Interrupt**: `session.execution.interrupted` with `reason:"user"` — exact.
- **Delivery**: `session.synthetic(text, { resume: false, metadata })` when idle — visible
  transcript block, no model turn; piggyback in `session.hook("prompt")` otherwise.
- **Status**: TUI plugin claiming `prompt.footer.status` slot
  (`ai-link · myslug/bob · active · 2 waiting`) + `ui.toast` on join/pause/resume.
- v1 fallback (`@opencode-ai/plugin` 1.x): `chat.message` + `event` (message.part.updated,
  `session.idle`), SDK `session.prompt({ noReply: true })`, slots
  (`prompt_footer`-equivalent `session_prompt_right`), `command.execute.before`. Shipped as
  a compat adapter, thinner UI.

### 6.4 pi — extension (`~/.pi/agent/extensions/ai-link/index.ts` or pi package)

- **Commands**: `pi.registerCommand("ai-link", { handler(args, ctx) })` — real command,
  runs before the input pipeline, never reaches the LLM. `getArgumentCompletions` for
  slug/name autocomplete.
- **Capture**: `input` event (`source: "interactive"` = human-typed; `"extension"` = ours) →
  user turns; `message_end` (`role==="assistant"`, stopReason≠error) → assistant text;
  accumulate per `turn_start`/`turn_end`.
- **Turn end**: **`agent_settled`** (the true "won't auto-continue" boundary), not `agent_end`.
- **Interrupt**: `turn_end.outcome === "aborted"` / assistant `stopReason === "aborted"`.
  (No dedicated abort event; `ctx.abort()` exists for us, not as a signal.)
- **Delivery**: §5.2 (`sendMessage` triggerTurn:false when idle; `input`-event transform
  piggyback otherwise).
- **Status**: `ctx.ui.setStatus("ai-link", "myslug/bob · paused · 2 waiting")` in the footer;
  `notify()` for command results (zero-turn, no dialog spam).
- Guards: `ctx.hasUI` / `ctx.mode` — headless (`-p`, rpc) still links & captures, skips UI.

### 6.5 Adapter contract (what every adapter must implement)

```
link(sessionRef, slug, name)      → registers member, announces join, queues welcome
unlink(sessionRef, alias)         → leave notice, cleanup of live state
pause(sessionRef) / resume(sessionRef)
setMaster(sessionRef)
status(sessionRef)                → printable summary
onUserInput(sessionRef, text)     → capture + auto-pause + piggyback return (may return
                                    context string and/or suppression signal)
onAssistantTurnEnd(sessionRef, text, {aborted}) → flush; publish when active
onSessionEnd(sessionRef, reason)  → unlink (graceful) / stale-mark
onIdle(sessionRef)                → try deliver (OpenCode/pi push; CC/Codex no-op)
```

`sessionRef = { client, sessionId, cwd }`. `core` never knows what a client is beyond this.

## 7. User-visible feedback matrix

| Event | CC | Codex | OpenCode | pi |
|---|---|---|---|---|
| command result | systemMessage / model-relayed line | systemMessage | command output in TUI | `ui.notify` |
| persistent state | statusline (opt-in snippet) | hook `statusMessage` | footer slot | `setStatus` |
| auto-pause notice | systemMessage on next prompt | systemMessage on next prompt | toast + footer immediately | notify + status immediately |
| envelope / welcome arrival | (inside next prompt context) | (same) | transcript block + toast | transcript block + notify |

Auto-pause on ESC is *immediately* visible only on OpenCode/pi; CC/Codex learn it on the
next prompt (their hooks can't fire between prompts). Accepted limitation of lazy clients.

## 8. Robustness & lifecycle

- **Stale detection**: member files carry `lastSeenAt` (bumped on every hook/event touch +
  30s heartbeat where an in-process plugin exists — OpenCode/pi/CC-Stop-ping). A member not
  heard from for 24h is marked `stale` in `status` and excluded from fresh delivery targets;
  never deleted. `/tmp` reboot-cleanup is the eventual janitor.
- **Crash mid-publish**: writes are journal-first (intent → write → commit); a half-written
  envelope never delivers (atomic rename). `seq` numbers dedupe on the receive side
  (`deliveredSeq` cursor).- **Two sessions, same name**: rejected at join (advisory lock on member file creation).
- **Slug dir gone** (someone rm'd /tmp/ai-link/x, including transcripts): adapters degrade
  to a loud error on next command, recreate empty dir on next join, sessions must re-link.
  Transcript durability beyond `/tmp` lifetime = backlog (§11), not v1.
- **Ordering**: per-sender ordering guaranteed (seq). Cross-sender interleaving = by release
  round; document as such.
- **Trust**: envelope header marks content as peer context (§5.1). Inbound is treated as
  data by convention; no auto-execution of anything in an envelope, ever.
- **Privacy**: transcripts are plaintext under `/tmp` (mode `0700` dir, `0600` files,
  single-user workstation assumption; multi-user hosts = backlog: use
  `${XDG_RUNTIME_DIR}/ai-link`).

## 9. Known client gaps (explicit, accepted)

1. **Codex has no real `/ai-link`** — slashless `ai-link …` interception is the command
   surface; `$ai-link` skill is optional sugar. Requires no Codex patches.
2. **CC/Codex cannot be pushed into** while idle — lazy piggyback only (chosen).
3. **CC ESC detection is heuristic** (no interrupt hook in 2.1.x): `PostToolUseFailure{
   is_interrupt}` + transcript interrupt-marker parsing. Codex/pi/OpenCode have exact events.
4. **CC hook config changes need session restart** — installer/upgrades say so.
5. **No wake-the-model delivery anywhere** in v1; every model turn remains human-initiated.
6. **Master stall holds rounds** (§4.3) — visible in `status`, human resolves via
   `/ai-link master`.
7. Codex `notify` and legacy paths deliberately unused; hooks are stable/default-on.

## 10. Decisions log

- Delivery to idle CC/Codex: **next-prompt piggyback** (not auto-wake bridge).
- Codex commands: **slashless prefix intercepted by UserPromptSubmit hook**.
- Master: **round starter** — only master publishes release queued deliveries.
- Pause: **both directions** (no publish, no deliver; inbound still queues).
- Envelope: **text only** (no tool calls/results, no thinking).
- Envelope attribute: **`role="<name>"`** (was `from` in v0.1 draft).
- **`/ai-link status` and `list` are in scope**; event notices (join/leave/pause/resume/
  master) are delivered to inboxes.
- **History for newcomers**: cumulative transcripts in `transcripts/`; join welcome names
  the directory; late joiner receives everything prior members published there.
- Solo-member slug: publish still writes the transcript and round bookkeeping runs —
  the second joiner thus receives full history (§4.3).
- OpenCode/pi additionally support zero-turn transcript injection when idle.
- Envelope feedback on CC/Codex via hook `systemMessage`: approved (zero model turn).

- Node-only runtime (v1): CLI and adapters run under `node` ≥ 20; no bun/deno requirement.
- **Absolutely minimal implementation** (user constraint 2026-09-22): the complete specified
  functionality with the smallest possible footprint — zero runtime deps, no build step,
  plain ESM JavaScript in one package, minimal file tree, no extra abstraction beyond what
  the spec requires. Where the spec offers choices, the smaller one wins.

## 11. Out of scope (backlog)

- Multi-slug per session; cross-machine links (broker over ssh/socket); auto-wake mode
  (`wake: true` per slug using codex app-server `turn/start` / CC resume — the rejected
  option, kept as a future toggle); team shared state dir; TUI status parity for CC/Codex
  waiting on upstream (hook statusMessage is the ceiling today); `oh-my-pi` fork-specific
  `deliverAs:"aside"` handling; transcript encryption; Windows; transcript history beyond
  `/tmp` lifetime (durable store location/config).

## 12. Repo layout

Minimal on purpose (§10): one package, no workspaces, no build step.

```
ai-link/
  lib/                      # core: state machine, envelope codec, transcript writer, locks
  cli/ai-link.js            # CLI entry (thin over lib) — `bin` in package.json
  clients/pi/index.js       # pi extension (imports ../lib directly)
  clients/opencode/index.js # OpenCode plugin (v2; v1 compat only if §6.3 shim fits in same file)
  clients/claude/           # .claude-plugin/plugin.json, commands/, hooks/*.js
  clients/codex/            # .codex-plugin/plugin.json, hooks.json
  docs/clients/*.md         # per-client install guides
  test/                     # node:test suites incl. fake-client adapter harness
  package.json              # name, bin, files — no build, no deps
```

Milestones: **M1** lib+CLI+tests · **M2** pi + OpenCode (rich APIs, fastest
e2e) · **M3** Claude Code · **M4** Codex · **M5** install story + docs.
