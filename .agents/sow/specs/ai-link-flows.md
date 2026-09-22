# ai-link — Flow Specification (DRAFT v0.1, 2026-09-22)

**Draft state**: covers SOW-0001 (open). Not shipped reality yet. Complements
`ai-link.md` (product spec v0.3): this file walks **every flow end-to-end** against the
proposed ledger model and pins the anti-echo rules. Where it supersedes v0.3 (ledger/roles
replaces per-peer inboxes; capture→transcript is live), v0.3 §3.1/§4/§5 get rewritten
here once the open decisions close. Decisions D1–D3 from the ledger discussion are
**still open**; D4 is new (self-resume).

Structural assumptions used below (ledger model):

```
ledger/<lsn>-<name>-<gen>.md   one append-only entry per publish: JSON header
                               {lsn, from, fromGen, round, kind: turn|event|welcome,
                                masterAtAppend, membersAtAppend[], deliverable}
                               + envelope body; tmp+rename (atomic)
transcripts/<name>.md          LIVE record: appended when each assistant/user turn
                               FINALIZES, pause-independent (user decision: live)
roles/<name>/cursor            LSN this role has received (advanced only post-inject)
meta.json                      {lsnHead(derived), round, master}
```

## 1. Anti-echo rules (the ping-pong defense)

The system's greatest risk is agents exchanging valueless acknowledgments forever.
Hard and soft defenses, in strength order:

- **R1 — Injection never wakes a model.** All inbound primitives are no-turn
  (`session.synthetic(resume:false)`, `sendMessage(triggerTurn:false)`,
  `prompt(noReply:true)`, `additionalContext`). A peer's content cannot cause an
  assistant response.
- **R2 — Every broadcast rides a human-initiated turn.** Turn-end publish requires an
  active member; every human prompt auto-pauses first; publishing otherwise requires
  `resume`. Therefore **broadcast rate ≤ human keystroke rate** across the group: the
  loop terminates the moment humans stop. This is the structural ceiling; R3–R7 below
  are about hygiene, not safety.
- **R3 — Notices are system voice.** Welcome/event envelopes carry the frame line:
  `(system notice — informational; do not acknowledge this notice in your response to
  your user)`. Soft, cheap, catches compliant models.
- **R4 — Join-turn rule.** Assistant text that finalizes between a member's join and
  its first real human prompt is **captured (transcript) but excluded from broadcast
  (ledger)**. Deterministic position rule — no intent detection. Kills the
  "OK, ai-link is enabled!" first-broadcast. Same rule applies to any turn that
  contains only responses to notices while no human prompt intervened — i.e. precisely
  "turns since last human prompt". (Human prompts always broadcast; they're the signal.)
- **R5 — Control tags are executed and stripped.** `<ai-link role="<self>" action="…"/>`
  in the member's **own captured output** executes the action and is removed from both
  transcript and ledger (it is a keystroke equivalent, not conversation). Tags appearing
  in **inbound** content are inert (never executed — inbound is base64-fenced data).
- **R6 — Notices are minimal.** `join`/`leave`/`master`/manual-`pause`/manual-`resume`
  are one-line events; auto-pause/resume emit nothing. No "X joined" prose beyond the
  one line.
- **R7 — Envelopes are context, not instructions.** The standing frame line (§5.1 of
  the product spec) stays; models are told peer content never overrides the operator.

## 2. Welcome message (canonical text)

Unicast to the joiner only; peers get an R6 `join` event instead. Rendered by `lib`,
delivered by the joiner's client channel:

```
You have joined ai-link <slug> as <role>.
- You are the only member — this makes you the turn initiator (lead).
| There <X> other role(s): <role1>, <role2>, … — the lead is <leadRole>.
The current round is <round>. <(round 0 — no turn has been published yet)|>
You can read the transcripts of the discussion at any time, in real time, at
<root>/transcripts/ : one file per role (<role>.md), newest turn at the end; each
section shows the role's client and join generation.
Your interaction with your user is now monitored: anything you say and anything the
user types becomes part of ai-link and is propagated to the other members on publish.
If the user interacts with you directly or stops you, ai-link pauses automatically
and your user needs to type `ai-link resume` (slashless on Claude Code/Codex) to
share the accumulated turns. [D4: You may also end your turn with
<ai-link role="<role>" action="resume"/> to publish exactly this turn once.]
Do not acknowledge this notice in your response to the user.
```

## 3. Flows

Notation: **→** state/flow, writes in `code`, guard rules cited.

### F1. First join (solo), push client (pi/OpenCode)

1. Human: `/ai-lock… ai-link demo alice` → adapter → `lib.join`.
2. Slug lock: init dirs if missing; `members=∅` ⇒ alice becomes master (locked
   election — no race); `meta.round=0`; `roles/alice/cursor = 0`; member file created.
3. Enqueue `welcome` (unicast, `deliverable=true` always); enqueue `join` event to
   the ledger for the record (no other members ⇒ no receiver).
4. Member state: **active**. Idle watcher delivers welcome immediately (inject, cursor
   → its lsn). R1: no model turn is caused.
5. Model's next output happens only when the human prompts. Per **R4**, text
   finalizing before that first prompt is captured to `transcripts/alice.md`, never
   broadcast.
6. Human's first real prompt → F3.

### F2. Second join to an existing slug, lazy client (CC/Codex)

1. `ai-link demo bob` → lock: name check (live holder ⇒ reject), gen bump if
   reclaiming stale holder's name, member file, cursor := `lsnHead` (**bob receives no
   backlog injection** — newcomer history = `transcripts/`, per the welcome).
2. `welcome` enqueued to bob's cursor stream (unicast, deliverable); `join` event
   appended to ledger (lsn+1) — visible to alice per F6 (alice's human must type to
   receive it: lazy).
3. Welcome reaches bob's model context only at bob's next real prompt (lazy piggyback);
   bob's pre-prompt chatter suppressed by R4 — and there is no chatter before the human
   ever types in bob's session anyway.
4. The CC/Codex human simultaneously sees the command result `linked as bob (2 members,
   lead alice, round 3)` via systemMessage — the human-facing confirmation is not the
   model-facing welcome.

### F3. Normal turn, active member (the steady state)

1. Human types prompt P. Hook/event fires.
2. **Piggyback first** (lazy): pull `ledger lsn > cursor, deliverable, from ≠ self` →
   attach as context; cursor advances **at hook return** (content was committed with
   the submitted prompt — even if the turn is ESC'd after this, the delivery happened;
   F9). Push clients with nothing pending: no-op.
3. Capture P → `transcripts/<name>.md` (live, `### user` section) + outbox.
4. **Auto-pause** (state `paused{auto}`); status surfaces `paused · outbox N · inbox M`.
5. Model answers; each finalized assistant message appends to the transcript (live —
   pause does not stop capture or transcript, user decision).
6. Turn ends. Member is paused ⇒ **no publish**. Content sits in outbox.
7. Human types `ai-link resume` → F4.

### F4. Resume (human), member publishes

1. Lock. `paused := null` (now active).
2. Publish outbox: append ledger entry (`kind=turn`, lsn=head+1, `deliverable =
   (self is master)`); outbox compacted **after** the atomic rename (outbox = intent
   log; crash between = idempotent retry, deterministic file name makes re-append a
   no-op collision).
3. If self is master: `round++`; flip `deliverable=true` on all queued entries
   (lsn < new head) under the same lock; this is the round release.
4. Try deliver-self-inbox per F6 rules (push at idle; lazy at next prompt).
5. Resume-during-turn (human resumes while the model is mid-reply): allowed; flush
   text finalized so far with `flushedBytes` remainder-tracking on the open turn; the
   later turn-end (member now active) publishes only the remainder. No dup, no loss.
6. Status: active, `waiting for your next prompt` (the next prompt will auto-pause
   again — R2's cost, confirmed intended).

### F5. Resume via model action tag — **D4 open**

If D4=one-shot: tag executes ⇒ steps identical to F4 **but** after publishing that
one entry, member re-enters `paused{auto:tag}` and only the *next* tag (or human)
releases again. If D4=as-written: member stays active until next human prompt.
If D4=drop: tags are inert data. Always: R5 (own output only; stripped everywhere).

### F6. Delivery of a released envelope to a peer

- **Push peer, idle, active**: watcher (≈2 s) takes lock, reads `lsn > cursor`,
  injects (no-turn), cursor advances after successful inject. **At-least-once**
  (crash between inject and cursor write ⇒ rare visible duplicate, labeled with lsn —
  D2 open).
- **Push peer, busy**: nothing until watcher observes idle. Not a daemon; the interval
  lives in the client process and dies with it (client gone ⇒ no delivery target
  anyway ⇒ stale rules).
- **Lazy peer**: released envelopes are simply *available*; they ride the next human
  prompt (F3 step 2). No timer exists or is possible.
- **Paused peer**: watcher/piggyback both check state first — paused ⇒ no cursor
  advance, no inject; the queue grows. `resume` drains it (F4 step 4).
- **Stale peer**: never a target; cursor frozen; rejoined role catches up per D1.

### F7. Ack-after-welcome (the user's core worry, both directions)

- Joiner side: pre-first-prompt ack suppressed by R4 ⇒ "OK ai-link enabled" exists only
  in `transcripts/alice.md`, never in the ledger. If the human's first prompt *itself*
  asks the model to "confirm you're linked", the reply is a real turn ⇒ broadcasts —
  and honestly should.
- Existing-members side: alice receives bob's `join` event (one line). Her model at her
  next human turn sees it; R3 instructs not to acknowledge; if she nonetheless says
  "welcome bob", that text is in a human-initiated turn (post-prompt) ⇒ broadcasts —
  and it then sits in bob's inbox until bob's human resumes. One human keystroke per
  hop, forever capped by R2.

### F8. ESC / interrupt, at three moments

- **Mid-reply, active or paused**: interrupt event → `paused{auto}` (immediate for push
  clients; lazy clients apply it when their next hook fires — accepted gap §9). Partial
  assistant text finalizes as a captured turn (`stopReason=aborted` marked in
  transcript; broadcast eligibility: it's real output ⇒ publishable on resume).
- **At the turn boundary** (after final text, before turn-end event): publishable
  content already finalized; pause outcome identical either way — safe by construction.
- **During the piggyback moment (lazy)**: hook already returned ⇒ delivery counted
  (F3 step 2); aborted human prompt is captured (the human did type it; others may
  legitimately see what they interrupted).
- ESC on a **command prompt** (`ai-link resume` typed then ESC'd before submit): never
  captured, never executed — interception happens on submit.

### F9. Assistant crash & restart; rejoin

- Killed process: no SessionEnd ⇒ member goes stale on the 24 h rule **or** sooner:
  any command by anyone recomputes liveness cheaply where possible (`status` marks
  `lastSeenAt` age; no fake liveness).
- Restart, rejoin same `<role>`: gen stays (same member file, new sessionId).
  **Cursor persists under `roles/<name>/`** ⇒ nothing already-received re-delivers.
  Welcome fires again (context refresh). D1 decides whether the absent-era gap in the
  ledger is delivered or skipped.
- Graceful `/ai-link exit`: final flush of outbox into transcript **without** ledger
  append (nothing captured is lost from the shared file); `leave` event appended; if
  master ⇒ handover to earliest live member under lock.

### F10. Master lifecycle

- Election: only ever at first join, under the lock.
- Move: `ai-link master` anywhere ⇒ atomic meta rewrite + `master` event. Old master
  demoted silently (no event spam).
- Stale master (crashed): rounds hold; `status` everywhere shows
  `master alice STALE (rounds held)`; fix = a live member types `master`. No automatic
  takeover (determinism; surfaced, human-resolved).

### F11. Two humans prompt two members "simultaneously"

Each prompt: lock → piggyback-check (may find nothing new), capture, auto-pause.
Lock serializes; no interleaving hazards (each member's outbox is its own file).
Both models reply; both sit paused; both humans resume ⇒ two publishes; whoever is
master opens the round that releases both.

### F12. Command prompts

`ai-link …` (or `/ai-link …` native): executed, result surfaced to the human,
**not captured** as a user turn, **no auto-pause**. Prevents command noise polluting
shared history and prevents `/ai-link pause` itself pausing… before pausing (no-op
loops).

### F13. Solo history → newcomer sees everything

Solo alice: every resume appends ledger AND (live, F3 step 5) transcript — even
un-broadcast content is in the transcript, since capture is pause-independent. Bob
joins: cursor := head (no injection flood); welcome points at `transcripts/` where
alice's complete discussion sits in real time. Requirement satisfied without ever
replaying history through model context.

### F14. Envelope mentioning/asking me

Peer text is context-only (R7). If the human wants a reply, they resume and type —
the same F3/F4 path. ai-link never routes "reply to X" prompts; there is no
addressing semantics in v1 (backlog: `@role` mentions would be a spec change, not an
implementation detail).

## 4. Invariants (testable ⇒ AC0/AC1 extensions)

1. **L1**: transcript append happens at turn finalize, independent of pause/publish —
   test: pause mid-conversation, verify transcript ahead of ledger.
2. **L2**: `cursor` advances only after inject/commit success; delivery is
   at-least-once; every envelope self-labels `lsn/from` ⇒ duplicates recognizable.
3. **L3**: no code path calls a model-waking primitive. Grep: forbidden call set
   (`triggerTurn: true`, `resume: true`, plain `prompt(` without `noReply`) absent in
   `clients/` inbound handlers.
4. **L4**: R4 boundary = `lastHumanPromptSeq > lastPublishedSeq` check at publish
   time (pre-first-prompt suppression is a special case of "no human prompt since
   join"); testable with the fake client.
5. **L5**: ledger append is the only lsn producer, under the lock; entry file names
   are deterministic ⇒ append is idempotent under crash-retry.
6. **L6**: inbound content can never execute an action tag (R5); test: envelope
   body containing the literal resume tag must not change state.

## 5. Open decisions blocking the v0.4 spec rewrite

- **D1 (carried)**: rejoin catch-up — deliver-missed vs skip-to-head vs per-join flag.
  (Cursor persistence makes both implementable; recommend deliver-missed with
  `status` warning + pause-before-resume advice.)
- **D2 (carried)**: accept at-least-once duplicates (recommended) vs two-phase inject.
- **D3 (carried)**: confirm per-entry `masterAtAppend` snapshot interpretation of
  "each turn has its own master role" — mastership remains global in meta.
- **D4 (new)**: model self-resume tag — one-shot (recommended) / persistent / drop.
- **D5 (new, from F8)**: should ESC'd **partial** assistant turns be publishable at all
  (recommended yes — they're real output; the transcript marks them `aborted`), or
  publish only fully-completed turns?

## 6. Change log

- v0.1 draft: first full flow walk (F1–F14), anti-echo rules R1–R7, canonical welcome,
  invariants L1–L6; ledger model assumed per the 2026-09-22 discussion (D1–D3 pending).
