# SOW-0001 - Implement ai-link v1 (core, CLI, four client adapters)

## Status

Status: open

Sub-state: spec agreed to v0.2; implementation not started. Gate drafted; blocked on user
approval of the draft gate (see Open decisions).

## Requirements

### Purpose

Link interactive coding-agent sessions (Codex, Claude Code, OpenCode, pi) on one machine
through a filesystem-mediated bus at `/tmp/ai-link/<slug>/`, so linked assistants see each
other's turn-based transcripts with the human in control of every turn.

### User Request

- Slash commands: `/ai-link <slug> <name>`, `exit|unlink|off|stop|leave`, `pause`,
  `resume`, `master`.
- Per-slug directory under `/tmp/ai-link/<slug>/`; each member's output written to
  `<name>.md`; on turn completion, signal linked sessions (except self) to receive the
  transcript when idle.
- Envelope accumulates assistant text + user-typed turns since last publish, formatted
  `<ai-link role="<name>">…<assistant>…</assistant><user>…</user>…</ai-link>`.
- ESC stop or user typing → automatic pause with visible TUI notice; `/ai-link resume` to
  continue.
- Later (v0.2 review): `role` attribute confirmed; `/ai-link status` + `list` in scope;
  shared `transcripts/` directory as history for newcomers; join welcome message naming
  the transcripts directory; first joiner sees solo-member history when second joins.
- "Absolutely minimal implementation to deliver the complete functionality" (2026-09-22).
- "1 implementation with X interfaces, not X implementations — exactly the same
  implementation for all ai clients with specific adapters each client provides"
  (2026-09-22).

### Assistant Understanding

Facts:

- `.agents/sow/specs/ai-link.md` (v0.2) records all agreed mechanics, per-client evidence
  with upstream commits, and the decisions log.
- Capability audit per client (spec §6): OpenCode and pi support real commands, exact
  turn/interrupt events, and zero-turn injection; Claude Code and Codex rely on hooks with
  next-prompt piggyback delivery and slashless command interception (Codex has no user
  slash commands at all).

Inferences:

- A single minimal JavaScript library + CLI with thin per-client adapters minimizes drift;
  the in-process adapters (OpenCode/pi) import the library directly.
- Fake-client harness (driving the library through the adapter contract §6.5) can test all
  lifecycle behavior without installing the four real clients; real-client validation is
  then a manual milestone gate (M2–M4).

Unknowns:

- None blocking: remaining unknowns are enumerated as numbered open decisions below and in
  spec §9 (accepted gaps).

### Acceptance Criteria

- AC1: `ai-link` CLI + lib pass tests for: join/leave/name-collision/welcome queueing,
  pause/resume/auto-pause state machine, publish/round/release semantics, envelope
  encode/decode incl. base64 escape, transcript append format, inbox cursors/dedup,
  crash-safe atomic writes under lock. Verified by `node --test`.
- AC0: one-implementation invariant (spec §6.6): fake clients prove all four adapters
  drive the same `lib` code paths (push-capable and non-push adapters differ only in
  capability declaration); grep test proves no client-name branches in `lib/`/`cli/`.
  Verified by `node --test`.
- AC2: pi adapter e2e: two pi sessions on one slug exchange envelopes per spec; ESC pauses
  with immediate status; `resume` flushes; welcome message names the transcripts dir.
  Verified by scripted manual run, evidence summarized in this SOW.
- AC3: OpenCode adapter e2e same as AC2 (v2 API; v1-compat path smoke-tested).
- AC4: Claude Code adapter e2e: piggyback delivery on next prompt; slashless command
  interception blocks with systemMessage; statusline snippet.
- AC5: Codex adapter e2e: same as AC4 plus `Interrupt` hook auto-pause.
- AC6: Spec §10 decisions log and §6 client surfaces match shipped code; docs per client
  installed under `docs/clients/`.

## Analysis

Sources checked:

- `.agents/sow/specs/ai-link.md` v0.2 (includes evidence table of mirrored upstream repos
  and commits gathered during spec work).
- Current repo state: bootstrap complete (AGENTS.md, SOW ledger, symlinks, specs).

Current state:

- No implementation exists; repo contains spec, SOW system, and pointer stub.

Risks:

- Client APIs move fast (all four repos are active upstreams): adapter breakage. Mitigate
  by pinning tested versions in docs and keeping adapters thin over core.
- Codex hook behavior is newest-surface of the four; the whole Codex command surface rides
  on `UserPromptSubmit` blocking + systemMessage. Mitigated by M4 ordering (last milestone,
  after core is proven) and documented fallback (CLI-driven operation).
- Filesystem races on shared slug dirs: mitigated by locking + journal-first writes (AC1).

## Pre-Implementation Gate

Status: needs-user-decision

Problem / root-cause model:

- Greenfield implementation: build the v1 system defined by the spec. Root design model =
  filesystem bus + per-client adapter contract; no alternative architecture under
  consideration (daemon/auto-wake rejected by user, recorded in spec §10).

Evidence reviewed:

- Spec §6 evidence table (client capabilities with commits):
  - `openai/codex @ e4dba902abb7` — `codex-rs/hooks/src/lib.rs` (hook events incl.
    `Interrupt`), `codex-rs/tui/src/slash_command.rs` (closed enum),
    `codex-rs/core/src/hook_runtime.rs` (`additionalContext` injection).
  - `anthropics/claude-code @ b486776a2eef` — `mods/types/claude-code.d.ts`
    (hook payloads, `additionalContext`, `PostToolUseFailure.is_interrupt`),
    `plugins/plugin-dev/` (command/hook formats).
  - `anomalyco/opencode @ fe3f3a41f79a` (v1: `packages/plugin/src/index.ts` hooks,
    SDK `noReply`) and `@ 1814dd97996a` (v2: `packages/plugin/src/promise/*`,
    `packages/plugin/src/tui/context.ts` slots, `session.synthetic`).
  - `badlogic/pi-mono @ 1a584a7a56eb` — `packages/coding-agent/src/core/extensions/types.ts`
    (`registerCommand`, `agent_settled`, `sendMessage`, `setStatus`),
    `packages/coding-agent/examples/extensions/send-user-message.ts`, `file-trigger.ts`.
- Spec §9 accepted gaps.

Affected contracts and surfaces:

- New public artifacts: envelope format (spec §5.1), on-disk layout (§3.1), CLI
  (`ai-link <subcommand> …`, §6.5 adapter contract), npm package(s), four client plugin
  bundles under `clients/`, docs.
- No existing contracts to change (greenfield).

Existing patterns to reuse:

- pi examples `send-user-message.ts` / `file-trigger.ts` for injection + watcher shape.
- Claude plugin `ralph-wiggum` (bash-expansion command + stop-hook loop) and `hookify`
  (hooks.json wrapper format).
- Codex hook fixtures (`hooks/src/schema.rs`) for payload field names.
- OpenCode built-in `init`/`review` command definitions for zero-prompt `execute`.

Risk and blast radius:

- Blast radius outside this repo: writes only under `/tmp/ai-link/` and the user's own
  client config when they install adapters. No network. Hook/extension code executes with
  user privileges in their sessions — keep CLI surface minimal and auditable.
- Data-loss risk: transcript loss only via `/tmp` cleanup (accepted, spec §8/§11).

Sensitive data handling plan:

- Transcripts in tests/fixtures must be synthetic; never commit real conversation content
  from live linked sessions (AGENTS.md hazard rule). No secrets involved in the protocol
  itself. Manual e2e evidence summarized, not pasted verbatim, when it may contain
  real conversation.

Implementation plan:

0. Constraint (user, 2026-09-22): **absolutely minimal** implementation delivering the
   complete spec — zero runtime deps, no build step, plain ESM JS, single package,
   smallest file tree (spec §10/§12). Every layer must earn its place.
1. M1 — `lib/` + `cli/ai-link.js` + fake-client harness tests on `node:test`
   (join/welcome, state machine, publish/round/release, envelope codec, transcript
   writer, locking/journal). `package.json` with `bin` only; no tooling beyond node.
2. M2 — pi adapter + OpenCode v2 adapter (direct `lib/` imports) + scripted e2e; v1-OpenCode
   compat only if it fits in the same file without abstraction cost.
3. M3 — Claude Code plugin (hooks.json, command md, statusline snippet) driven by CLI.
4. M4 — Codex plugin bundle (hooks.json) driven by CLI; skill sugar only if zero-cost.
5. M5 — install story (README, `docs/clients/*.md`), spec conformance pass.

Validation plan:

- AC1 unit/integration tests (fake clients, temp-dir roots so tests never touch `/tmp/ai-link`).
- AC2–AC5 scripted manual e2e on the workstation with real clients; findings summarized in
  Execution Log/Validation here (synthetic conversation content only).
- Same-failure search before close: grep SOW + tests for deferred failure modes.
- Reviewer pass (user may request external review at milestone boundaries).

Artifact impact plan:

- AGENTS.md: add Project-specific commands (build/test) at M1; skill index at first
  project skill.
- Runtime project skills: create `project-adapter-testing` (fake-client harness usage) once
  M1 harness exists — concrete reusable knowledge, replacing the "none yet" note.
- Specs: `.agents/sow/specs/ai-link.md` updated at each milestone to shipped reality;
  draft-state removed at M1 (CLI names finalized).
- End-user/operator docs: README (M1) + `docs/clients/*` (M2–M5).
- End-user/operator skills: none needed — adapters are shipped as client plugin bundles,
  not skills; reason: no operator skill artifact exists yet; revisit at M5 if an
  ai-link-operator guide as a skill is wanted.
- SOW lifecycle: single SOW for v1 with milestone execution log; completed only when all
  ACs verified; close committed with final work.

Open-source reference evidence:

- Yes — mirrored upstreams checked during spec work; cited above as `owner/repo @ commit`
  (commits recorded at check time 2026-09-22).

Open decisions:

1. ~~**Package manager/test runner**~~ — **resolved 2026-09-22**: the minimalism constraint
   (user) selects plain ESM JS + `node:test` + bare `package.json`; no package manager
   beyond npm-defaults, no runner dependency.
2. ~~**Package publishing**~~ — **resolved 2026-09-22**: single npm package `ai-link`
   (repo-as-package) as canonical distribution, plus `ai-link install <client>` git-clone
   fallback; per-client vendored bundles rejected. Recorded as spec §12.1.
3. **Repo hosting**: this local git repo — publish to GitHub now or keep local until M5.
   Does not block.

## Implications And Decisions

1. v0.1→v0.2 spec amendments (user decisions 2026-09-22): `role` attr; status+list added;
   join welcome + shared `transcripts/` history; solo publish still records history so the
   second joiner receives it; systemMessage feedback approved; runtime question (5) retired
   — Node ≥ 20 chosen, no bun/deno requirement stated by user.

## Plan

1. Await user resolution of Open decisions 1–2 (3 optional).
2. M1 → M2 → M3 → M4 → M5 as in gate (one milestone sequence, single SOW).
3. Per milestone: update spec §6/§10 to shipped reality, log in Execution Log, run tests.

## Execution Log

### 2026-09-22

- Repo bootstrapped (git init, AGENTS.md from SOW template, spec v0.2 in specs/, SOW
  created pending gate approval).
- Minimalism constraint recorded (zero deps, no build, single package); open decision 1
  resolved by it.
- One-implementation/X-interfaces invariant recorded as spec §6.6 + AC0; capability
  declarations added to the adapter contract model.

## Validation

Pending.

## Outcome

Pending.

## Lessons Extracted

Pending.

## Followup

None yet.

## Regression Log

None yet.

Append regression entries here only after this SOW was completed or closed and later testing or use found broken behavior. Use a dated `## Regression - YYYY-MM-DD` heading at the end of the file. Never prepend regression content above the original SOW narrative.
