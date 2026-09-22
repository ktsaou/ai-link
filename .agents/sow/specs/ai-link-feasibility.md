# ai-link — Client Capability Feasibility (v1, 2026-09-22)

**Draft state** (SOW-0001). The exhaustive capability matrix behind the product spec.
Method: four very-thorough source studies (Claude Code, Codex, OpenCode v1+v2, pi) +
one retry. Every cell cites `owner/repo @ commit` + repo-relative path:line.

Evidence baseline (re-verified this round):

```text
openai/codex            @ e4dba902abb7   (codex-rs)
anthropics/claude-code  @ b486776a2eef   (d.ts 2.1.277 — authority over docs)
anomalyco/opencode (v1) @ fe3f3a41f79a   (@opencode-ai/plugin 1.x — USER TEAM'S BUILD)
anomalyco/opencode (v2) @ 1814dd97996a
badlogic/pi-mono        @ 1a584a7a56eb   (mirror may drift from installed — spike)
```

**Columns** (Codex and Claude Code each have two planes — the study proved they differ
enormously):

- **CDX-H** Codex hooks plane (spawned processes) · **CDX-B** Codex app-server client
  plane (**requires daemon mode**; default `codex` runs embedded ⇒ plane B unavailable)
- **CC** Claude Code classic hooks · **CC-Mod** Claude Code early-access mods
  (function-hooks; feature-gated, breaking changes without notice — design must not
  depend on it)
- **OC1** OpenCode v1 plugin (in-process) · **OC2** v2 plugin · **PI** pi extension
  (in-process; mirror-drift caveat)

Legend: ✅ supported · 🟡 partial/heuristic · ⚠️ conditional (noted) · ❌ unsupported ·
❓unverified-needs-spike

## 1. Input: observe / enrich / block / attribute

| Cell | CDX-H | CDX-B | CC | CC-Mod | OC1 | OC2 | PI |
|---|---|---|---|---|---|---|---|
| 1a observe pre-model | ✅ `UserPromptSubmit{prompt,…}` | ✅ item/turn notifications (subscribed only) | ✅ `UserPromptSubmit{prompt, source?, prompt_id}` | ✅ same, +in-process | ✅ `chat.message` (parts mutable, pre-persist) | ✅ `session.hook("prompt")` (deep-mutable) | ✅ `input{text,source,streamingBehavior}` |
| 1b enrich model input | ✅ `additionalContext` (developer role, 2500-tok budget, spill file) | ✅ `turn/start.additionalContext` → user-role `<external_KEY>` fragments | ✅ `additionalContext` — **never shown to user** | ✅ `prompt.submit{context[]}` | ✅ mutate parts; `synthetic` parts reach model, hidden from TUI | ✅ mutate `prompt.text`; `context`/`generate` hooks | ✅ `input:transform`, `before_agent_start.message` |
| 1c block input, zero turn | ✅ exit2/block; **prompt dropped from history**; reason→Feedback row | n/a (client submits) | ✅ block/exit2; text fate ❓spike; `suppressOriginalPrompt` exists | ✅ `{drop:reason}` | 🟡 `command.execute.before` w/ empty parts **does NOT stop the loop** (model runs vs stale history); zero-turn = throw-from-hook (renders `session.error`) or noReply trick | ✅ `CommandDefinition.execute` without prompting = **true zero-turn**, human sees nothing unless we surface it | ✅ `input:{action:"handled"}` — fully consumed, zero model, human sees only what we render |
| 1d human vs machine | ❌ no source field | 🟡 own submissions tagged (`turnTrigger`,`clientUserMessageId`); human's = absence of markers | ✅ `source ∈ user,sdk,system,loop_wakeup,schedule_wakeup,poll_event` — **optional while rolling out** ⇒ missing = assume human | ✅ `PromptOrigin{kind}` richer, mod-only | 🟡 no flag on stored msg; tag via `synthetic` part marker (ours fires through `chat.message` too) | ✅ `delivery`+`metadata` on inbox items (user/synthetic/compaction) | 🟡 `input.source:"extension"` ✅ BUT persisted user-role message has **no provenance field** ⇒ marker-in-text required |

Evidence: codex `hooks/src/schema.rs:566-583`, `context-fragments/src/additional_context.rs`, `tui/src/history_cell/hook_cell.rs:628`; cc `claude-code.d.ts:11088-11096, 7096-7107, 1061`; oc1 `plugin/src/index.ts:234-243`, `session/prompt.ts:999-1073,1460-1473,1111-1130`, `message-v2.ts:200-210`; oc2 `plugin/src/promise/session.ts:14-20`, `core/src/command.ts:81-89`, `schema/src/session-inbox.ts:37-44`; pi `types.ts:947-961`, `agent-session.ts:1634-1643,2030-2035`.

## 2. Output: reasoning / tools / text / background / the-four-states

| Cell | CDX-H | CDX-B | CC | CC-Mod | OC1 | OC2 | PI |
|---|---|---|---|---|---|---|---|
| 2a reasoning | ❌ | ✅ `item/reasoning/*Delta` | 🟡 transcript JSONL only (no event) | ✅ `turn.step` thinking chunks live | ✅ `ReasoningPart` streamed | ✅ durable reasoning events | 🟡 provider-dependent (`thinking_delta`, `ThinkingContent`) |
| 2b tool req pre-exec | ✅ PreToolUse (matcher, allow/deny, updated_input) | ✅ items | ✅ PreToolUse full args | ✅ ToolCallEnvelope | ✅ `tool.execute.before` (args mutable, no veto return) | ✅ + **true veto** (Tool.Error rejects pre-run) | ✅ `tool_execution_start` + `tool_call{block}` |
| 2c tool responses | ✅ PostToolUse | ✅ | ✅ full + failure/interrupt variants | ✅ | ✅ mutable output | ✅ discriminated completed/error | ✅ end + `tool_result` modify chain |
| 2d assistant text | 🟡 whole-message at `Stop{lastAssistantMessage}` only | ✅ `item/agentMessage/delta` + completed | ✅ `MessageDisplay{delta,final,…}` **display-only** + Stop; ❓JSONL flush timing vs Stop | ✅ turn.complete text | ✅ part deltas (`message.part.delta`) + final via `text-end`/`experimental.text.complete` | ✅ durable `session.text.*` | ✅ `message_update` token deltas + `message_end` final |
| 2e background activity | ✅ Subagent hooks | 🟡 backgroundTerminals + subagent-threads | ✅ Subagent*, `Stop.background_tasks[]`, `session_crons[]`, Task*/TeammateIdle | ✅ `$.agent.list()` | 🟡 child sessions via task tool; poll `/session` | 🟡 parentID + per-session execution events | ❌ core none; extensions ARE the background |
| 2f done/ESC/perm-wait/user-wait | ✅ Stop/Interrupt hooks; PermissionRequest before dialog | ✅ `turn/completed{status:interrupted}`; approval requests fan out to subscribers | 🟡 Stop/StopFailure/`PostToolUseFailure{is_interrupt}` heuristic; Notification at dialog show; ❓pending-dialog heartbeat | ✅ turn.abort / turn.complete | ✅ idle never under permission (busy+`permission.asked`); ESC = abort error on msg | ✅ `execution.interrupted{reason:user}`; ❓permission-blocks-execution (spike) | 🟡 `agent_settled` done / `turn_end.outcome=="aborted"` **mirror-only-version risk** / no perms / `ui_prompt_start/end` |

Evidence: codex `protocol/common.rs:1901+`, `events/interrupt.rs`, `outgoing_message.rs:330-445`; cc `d.ts:5002-5027, 9245-9267, 6036-6042, 5330-5335`; oc1 `permission/index.ts:98-106`, `processor.ts:500-546`; oc2 `session-event.ts:232-252,386-448`; pi `types.ts:754-860`, `agent-loop.ts:285-307`, `security.md:33`.

## 3. Display

| Cell | CDX-H | CDX-B | CC | CC-Mod | OC1 | OC2 | PI |
|---|---|---|---|---|---|---|---|
| 3a visible injected block | ❌ additionalContext = developer-role, human-invisible | 🟡 turn/start input items render as user messages in the human's TUI | ❌ "never shown the user" | 🟡 `$.ui.log` transcript rows | ✅ noReply user msg w/ non-synthetic part = renders like a message | ✅ `session.synthetic{description}` typed item, renders distinctly | ✅ `sendMessage display:true`+renderer; `appendEntry` = human-visible, never model |
| 3b UI w/o model activity | 🟡 `systemMessage` (Warning row) at hook moments only; `statusMessage` **static config**, only while hook runs | ❌ | ✅ statusLine command (event-driven + `refreshInterval` N-sec, reads files freely) | ✅ `$.ui.status/toast/panes` | ✅ TUI-plugin slots (`session_prompt_right`, footer…) + toast; reactive Solid | ✅ richer slots (`prompt.footer.status`) + toast + durable stores | ✅ `setStatus/setWidget/setFooter/notify`, resident |

## 4. Steering / queue / wake

| Cell | CDX-H | CDX-B | CC | CC-Mod | OC1 | OC2 | PI |
|---|---|---|---|---|---|---|---|
| 4a mid-turn steer | 🟡 async-hook additionalContext injected at step boundaries (running turn) | ✅ `turn/steer{expectedTurnId}` — lands as user msg at step boundary, renders in TUI | ❌ classic (keystrokes queue themselves) | 🟡 deferred-to-idle; `turn.abort` yes | 🟡 busy-prompt queues; loop re-reads ⇒ consumed next step (endpoint blocks; use promptAsync) | ✅ `delivery:"steer"` inbox-promoted at step boundary | ✅ `deliverAs:"steer"` — drained after current tool batch, before next LLM call |
| 4b queue w/o keystroke | ❌ | ✅ `thread/queue/*` (add auto-wakes loaded-idle thread; skips Interrupted) | ❌ classic | ✅ hold + submit at idle | 🟡 same busy-prompt parking | ✅ `delivery:"queue"` parked items + inbox API | ✅ `followUp` (fires at settle) / `nextTurn` (lands w/ next human prompt) |
| 4c wake idle, zero keystrokes | ❌ hooks cannot wake | ✅ **`turn/start` on any loaded root thread from any daemon connection** — runs + renders in human's TUI. Gate: daemon mode (default = embedded ⇒ ❌) | ❌ classically external; **🟡 human-armed `/loop`** re-arms itself via ScheduleWakeup ⇒ `source:loop_wakeup` turns fire our hooks; Esc cancels | ✅ `$.clock` + `$.prompt.submit` (runs classic UserPromptSubmit hooks too) | ✅ resident plugin timer + `client.session.prompt` | ✅ same + synthetic resume:true | ✅ `sendUserMessage` while idle ⇒ immediate turn (`file-trigger.ts` is this exact pattern); `followUp` auto-turns too |

## 5. Identity / lifecycle · 6. Execution · 7. Provenance · 8. Cost

| Cell | CDX-H | CDX-B | CC | OC1 | OC2 | PI |
|---|---|---|---|---|---|---|
| 5a stable id | ✅ `session_id`==thread uuid (+turn_id) | ✅ threadId; **hook id == thread id** | 🟡 **session_id CHANGES across /clear, resume, fork** — re-bind on `SessionStart{source}` | ✅ sessionID stable (SQLite); events directory-scoped | ✅ durable ids | ✅ `getSessionId()`; changes on fork/new |
| 5b resume/fork events | ✅ SessionStart{startup,resume,clear,compact,fork} | ✅ thread/resume attaches | ✅ same + SessionEnd{clear,resume} | 🟡 created/updated/deleted; fork=new id, no event | ✅ created/forked/moved | ✅ before_switch/fork/tree, reasons |
| 5c exit vs crash | SessionEnd fires (reason `"other"` always; also idle-unload) / ❌ crash | same + unload nuance | SessionEnd{5 reasons, **1.5 s budget**} / ❌ kill -9 | dispose hook / embedded-topology ❓ | cleanup / service-topology ❓ | `session_shutdown{reasons}` / ❌ crash |
| 6a execution | spawn-per-event (`$SHELL -lc`, scrubbed env) | resident external (us) | spawn-per-event; resident only via mods | **in-process server plugin** (timers legal) | same + hot reload | **in-process jiti** (timers legal; start in session_start) |
| 6b hot reload | session-refresh | n/a | ❌ config needs restart; script body re-read per fire | ❌ restart | ✅ watcher + generations | ✅ /reload, ctx.reload (invalidate-after-reload caveat) |
| 6c crash blast | 5 s default timeout; failed=non-blocking | n/a | 60 s default; hung hook **stalls prompt** (Esc cancels) | hook throw → session error, server survives; event-hook ❓ | typed-failure; raw throw ❓ | per-handler try/catch; **tool_call handler NOT wrapped** |
| 7a recognize ours | ❌ no marker (JSONL metadata `hooks.additional_context` only) | ✅ our own turnTrigger/id | ❌ markers in text **mandatory** (visible chars only — CC strips invisible unicode) | 🟡 synthetic-part marker / tracked messageID | ✅ metadata on inbox items | 🟡 custom messages: `customType` ✅; user-role injections: marker needed |
| 7b persistence of injections | ✅ both kinds hit rollout JSONL | ✅ | ❓ JSONL persistence of additionalContext — spike | ✅ all persisted | ✅ durable | ✅ JSONL entries (deferred-flush nuance) |
| 8a per-prompt cost | sync pre-sample | n/a | sync spinner path — fail-fast + fail-open | sequential await every hook | same + one clone | serial per input handler |
| 8b permission | can ANSWER (allow/deny) | receives approval requests on subscribed conns (⚠ never auto-answer) | PermissionRequest answers allow/deny; Notification{permission_prompt} | ❌ `permission.ask` hook **dead** — use HTTP reply API | ✅ `permission.hook("evaluate")` pre-ask | ❌ no permission system |
| 8c our CLI spawn | ✅ hooks run unsandboxed, no prompt | ✅ | ✅ hooks outside permission system | ✅ plugins = arbitrary code | ✅ | ✅ `pi.exec`, ungated |

## Consequences for the design (feeds v0.4)

1. **Wake (D6) feasibility per client**: OC1/OC2/PI — fully supported natively by resident
   adapters (this is a capability we *gate*, not a gap). CDX — yes **iff daemon mode**
   (`codex app-server` running; default embedded is unreachable). CC — classically no
   external wake; **human-armed `/loop` is the door**: wakeups fire real model turns
   through `UserPromptSubmit{source:"loop_wakeup"}` where our hook delivers; Esc cancels;
   mods (EA) would add full external wake but we do not build on early-access APIs.
2. **v0.3 corrections**: (a) OC1 zero-turn command via empty parts is **false** — spike
   `api.command.register` or accept throw/noReply paths (AC3a sharpened); (b) OC1
   `permission.ask` hook is dead code; (c) CC identity must not key solely on
   `session_id` (changes on /clear/resume/fork — rebind via SessionStart.source);
   (d) CC blocked prompts: text fate unspecified ⇒ spike; (e) pi ESC fields
   mirror-vs-installed drift confirmed as risk — feature-detect stays.
3. **Envelope markers must be plain visible text** (CC strips invisible unicode/tags).
4. **Permission-wait ≠ idle ≠ done** on every client — publish-on-idle must re-check
   after permission resolves; pi has no permission state at all.
5. **CC/Codex hooks are synchronous on the human keystroke path** ⇒ ai-link hook code:
   fail-open, fast exit, tiny per-prompt work (fs-append only).
6. **Never auto-answer approvals** (CC/Codex both allow hooks/clients to answer
   permission dialogs — ai-link must not; hard rule).
7. **Never wake via `claude -p --resume`** against a live session: it forks history.
8. Codex daemon mode (if enabled) also gives ai-link *free* extras: reasoning deltas,
   live transcript streaming, mid-turn steer — candidate feature tiers, out of minimal
   v1 core but recorded.

## Spike list (ordered; blocks v0.4 / adapters)

| # | Spike | Client | Blocks |
|---|---|---|---|
| S1 | v1 `api.command.register` (deprecated): registers? zero-turn? result visible? | OC1 (team's build) | AC3a, §6.3-v1 |
| S2 | v1 idle→timer→`prompt(noReply)` delivery + `prompt(wake)` e2e; capture round-trip of own injections | OC1 | v1 adapter |
| S3 | installed pi: presence of `agent_settled`/`turn_end.outcome`/`ui_prompt_*`; `sendMessage(triggerTurn:false)` render | PI | pi adapter |
| S4 | CC 2.1.x: block-text fate; additionalContext JSONL persistence + human visibility; `source` population; `/loop` wakeup firing our hook | CC | CC adapter, D6-CC |
| S5 | Codex daemon mode: `codex app-server` stable enough to document; TUI attaches to existing daemon without flags; external turn/start renders | CDX-B | D6-Codex (optional tier) |
| S6 | CC pending-permission observability duration (Notification at show only?) | CC | 2f edge |

## Top surprises

1. **The "lazy clients" assumption was half-wrong**: Codex is the *most* observable
   client of all — via app-server (reasoning deltas, tool items, approvals, steer,
   wake). We simply assumed the wrong plane (hooks) in v0.3.
2. **pi is the most autonomous**: fs.watch + timers + steer/queue/no-wake-inject +
   display-only entries — the full ai-link loop, including watch mode, is its native
   habitat (`file-trigger.ts` is a working skeleton of it).
3. **Claude Code's `source` field** (`loop_wakeup`/`schedule_wakeup`/`poll_event`) means
   the engine already distinguishes exactly the human-vs-machine distinction our pause
   rules need — but it's optional during rollout ⇒ belt (source) and braces (markers).
4. **OpenCode v1 `permission.ask` is dead code in the published types** — the only
   finding that would have silently broken a design assuming it.
5. **Two Claude doors we rejected stay rejected** (mods = early-access; -p resume =
   fork-corruption), now with citations instead of vibes.

## Change log
- v1 (2026-09-22): first full matrix from four source studies; consequences 1–8 and
  spikes S1–S6 feed the v0.4 rewrite.
