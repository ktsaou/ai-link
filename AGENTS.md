# ai-link

## Goals

ai-link is a per-machine plugin that links interactive coding-agent sessions across four
clients — Codex, Claude Code, OpenCode, and pi — so linked sessions can see each other's
turn-based transcripts. The product is filesystem-mediated (no daemon, no network): each
link group `<slug>` owns `/tmp/ai-link/<slug>/`, members publish cumulative transcripts
under `transcripts/`, and deliveries reach peers through each client's own extension
surface (hook piggyback for Claude Code/Codex, zero-turn injection for OpenCode/pi). The
human stays in the loop: nothing in v1 ever wakes a model — every assistant turn is
human-initiated.

Success means: one core library + CLI, four adapters, and the behavior defined in
`.agents/sow/specs/ai-link.md` works across the four clients with no patches to any client.

## SOW System

This project uses a local Statement of Work system.

The SOW system is self-contained in this repository. Normal SOW work must not depend on
`~/.agents`, `~/.AGENTS.md`, global skills, global templates, or global scripts. Use this
`AGENTS.md`, project-local SOW files, project-local specs, project-local skills, and the
active SOW.

### Roles

- **User responsibilities:** purpose, scope decisions, design forks, risk acceptance,
  destructive approvals, and final product judgment.
- **Assistant responsibilities:** investigation, implementation, tests or equivalent
  validation, reviews, documentation, memory updates, and concise reporting.

### Required First Checks

Before creating a SOW or starting non-trivial implementation:

1. Confirm the user has requested implementation.
2. Inspect code/docs/data to establish whether a change is needed.
3. Read pending/current SOWs for overlap, contradictions, and existing decisions.
4. Read `.agents/sow/specs/ai-link.md` (product spec) and any other relevant specs.
5. Inspect `.agents/skills/project-*/SKILL.md` and load every runtime project skill whose
   trigger matches the work.
6. Ask the user only for irreducible product/design/risk decisions.

### Git Worktrees

Assistants must not create git worktrees on their own. Create a git worktree only when the
user explicitly asks for it or approves it.

### Sensitive Data In Durable Artifacts

SOWs, specs, documentation, project skills, agent instructions, and code comments are
commit-ready artifacts. Treat them as public unless a repository-specific policy explicitly
says otherwise.

CRITICAL: Never write raw sensitive data to durable artifacts. This includes passwords,
API keys, bearer tokens, SNMP communities, private keys, connection strings with embedded
credentials, session cookies, community member names, customer names, customer identifiers,
personal data, non-private IP addresses that can identify customers, private endpoints,
account IDs, and proprietary incident details.

ai-link-specific hazard: linked transcripts contain real conversation text from user
sessions. Never paste sample transcripts, envelope payloads, or `/tmp/ai-link` fixtures
containing real conversation content into SOWs, specs, docs, tests committed here, or code
comments — author synthetic fixtures with obviously fake content instead.

Write only sanitized evidence:

- use placeholders such as `[REDACTED_SECRET]`, `[CUSTOMER]`, `[ACCOUNT]`, `[PRIVATE_ENDPOINT]`;
- use stable aliases such as `customer-a` only when the real mapping is not stored in the repository;
- cite file paths, line numbers, command names, schema fields, or error classes instead of
  copying sensitive values;
- summarize logs and traces; include only minimal redacted snippets.

If sensitive data is required to continue, stop and ask the user for a secure handling
path. If sensitive data is found in a durable artifact, sanitize it before any commit. If
sensitive data was already committed, tell the user and do not rewrite history without
explicit approval.

### Open-Source Reference Evidence

When a SOW uses external open-source repositories as evidence, record the upstream
repository identity and checked commit, not the workstation mirror path.

For local mirrored or cloned open-source repositories, cite evidence in this form:

```text
owner/repo @ commit
relative/path/inside/repo:line
```

Rules:

- Never use workstation absolute paths for external open-source evidence in SOWs.
- Resolve `owner/repo` from the repository remote, not only from the local directory name.
- Record the commit with `git -C <repo> rev-parse --short=12 HEAD` or the full hash when
  precision matters.
- Use paths relative to the upstream repository root after the `owner/repo @ commit` line.
- If multiple repositories were checked, list each repository and commit separately.

The client extension surfaces this project builds on were researched in mirrored sources;
the baseline commits are recorded in §6 of `.agents/sow/specs/ai-link.md`. Re-record
commits whenever evidence is gathered anew.

### Pre-Implementation Gate

Implementation covered by a SOW must not begin until the SOW contains a concrete
`## Pre-Implementation Gate` section. Before moving a SOW from `pending/open` to
`current/in-progress`, or before continuing implementation in an existing current SOW that
lacks this section, fill the gate.

The gate must record:

- Problem / root-cause model: what is happening, why it is happening, and what evidence
  supports that model.
- Evidence reviewed: specs, code, docs, tests, logs, traces, prior SOWs, issues, or external
  references checked. Open-source references from local mirrors or clones must be cited as
  `owner/repo @ commit` plus repository-relative paths, never as workstation absolute paths.
- Affected contracts and surfaces: APIs, schemas, files, commands, UI, docs, specs, skills,
  tests, integrations, operators, users.
- Existing patterns to reuse: local modules, helpers, conventions, tests, and docs that
  should shape the implementation.
- Risk and blast radius: regressions, compatibility, performance, security, data loss,
  migration, rollout, and operational risks.
- Sensitive data handling plan: whether the work may expose sensitive data (including real
  conversation content in transcript fixtures); how evidence will be redacted in SOWs,
  specs, docs, skills, instructions, and code comments.
- Implementation plan: ordered chunks with scope, dependencies, and files or modules likely
  to change.
- Validation plan: tests, fixtures, manual checks, real-use evidence, review passes, and
  same-failure searches.
- Artifact impact plan: expected updates to `AGENTS.md`, runtime project skills, specs,
  end-user/operator docs, end-user/operator skills, and SOW lifecycle.
- Open decisions: resolved decisions or numbered options for the user; unresolved decisions
  block implementation.

Generic placeholders such as `TBD`, `N/A`, or "to be checked later" are invalid unless the
SOW explains why the item truly does not apply. If the gate exposes an unknown that cannot
be resolved by investigation, stop and ask the user before implementation.

### When A SOW Is Required

Create or reuse a SOW only after the user requests implementation and preliminary analysis
confirms a non-trivial change is needed.

Questions, discussions, reviews, status reports, and read-only investigation do not need a
SOW. Trivial implementation such as typo or formatting-only fixes does not need one.

When unsure whether a change is needed, investigate first. When an authorized change has
unclear risk, treat it as non-trivial.

### SOW Locations

- Pending: `.agents/sow/pending/`
- Current: `.agents/sow/current/`
- Done: `.agents/sow/done/`
- Specs: `.agents/sow/specs/`
- Template for new SOWs: `.agents/sow/SOW.template.md`
- Local audit: `.agents/sow/audit.sh`

Create new SOW files from `.agents/sow/SOW.template.md`. The template is project-local and
may be customized for this repository.

Empty SOW directories must contain `.gitkeep` or `.keep` so the committed repository
preserves the full SOW layout after clone/checkout.

Filename:

```text
SOW-NNNN-YYYYMMDD-{slug}.md
```

Status and directory must agree:

- `open` lives in `pending/`
- `in-progress` lives in `current/`
- `paused` lives in `current/`
- `completed` lives in `done/`
- `closed` lives in `done/`

### SOW Completion And Commit

The successful terminal SOW status is `completed`. `done` is a directory name, not a status
value. Never write `Status: done` or `Status: complete`.

When a SOW's work is ready to close:

1. Finish implementation, docs, specs, skills, validation, and follow-up mapping.
2. Update the SOW to `Status: completed`.
3. Move the SOW file to `.agents/sow/done/`.
4. Commit the work, artifact updates, SOW status change, and SOW move together as one
   commit, unless the user explicitly requested a different commit split.

Do not create a separate commit just to mark or move the SOW. Do not claim a SOW is
completed while the implementation and the SOW lifecycle change live in separate
uncommitted or separately committed states.

### One SOW At A Time

Never execute multiple SOWs as one batch.

If work overlaps:

- merge or consolidate before implementation; or
- split into separate SOWs and complete one before starting the next.

Progress reports are not stop points. Once a SOW is in progress, continue until it is
delivered, failed with evidence, blocked on a real user decision/approval, or superseded by
newer user instructions.

### User Decisions

When user decisions are needed:

1. Present concrete evidence with files/lines or source references.
2. Provide numbered options.
3. Explain pros, cons, implications, and risks.
4. Recommend one option with reasoning.
5. Record the user's decision in the SOW before implementation.

### Followup Discipline

"Deferred" is not a terminal outcome.

Before a SOW can close, every valid deferred item must be:

- implemented in the current SOW; or
- explicitly rejected as not worth doing, with evidence; or
- represented by a real pending/current SOW file.

Pre-close, search the SOW for:

```text
defer|later|follow-up|future|TODO|pending
```

Map every remaining item to implemented, rejected, or tracked.

### Regressions

A regression is discovered after a SOW was considered completed or closed, later testing or
use finds broken behavior, and the original SOW's claimed outcome is no longer true.

When behavior that a completed SOW claimed working stops working:

1. Find the original SOW in `done/`.
2. Move it back to `current/`.
3. Mark it `in-progress` with a regression note in `## Status`.
4. Append a new dated `## Regression - YYYY-MM-DD` section at the end of the file, after
   the original outcome, lessons, and follow-up content.
5. In that appended section, record what broke, evidence, why previous validation missed
   it, the repair plan, validation, and updates needed to specs, skills, docs, audits, or
   follow-up SOWs.
6. Fix and validate there.

Never prepend regression content above the original SOW narrative. The original
requirements, analysis, plan, validation, outcome, lessons, and follow-up must remain
readable first. Do not create a new SOW for a true regression.

### Validation Gate

A SOW cannot be completed until Validation records:

- acceptance criteria evidence;
- tests or equivalent validation;
- real-use evidence when a runnable path exists;
- reviewer findings and how they were handled;
- same-failure search results;
- sensitive data gate: durable artifacts contain no raw secrets or sensitive data, and no
  real conversation content from live linked sessions (see ai-link-specific hazard above);
- artifact maintenance gate for `AGENTS.md`, runtime project skills, specs,
  end-user/operator docs, end-user/operator skills, and SOW lifecycle;
- SOW status/directory consistency;
- spec update or specific reason no spec update was needed;
- project skill update or specific reason no skill update was needed;
- end-user/operator docs update or evidence-backed reason none were affected;
- end-user/operator skills update or evidence-backed reason none were affected by
  docs/spec changes;
- lessons extracted or specific reason there were none;
- follow-up mapping.

Generic "N/A" is invalid.

### Artifact Maintenance Gate

Every SOW close must explicitly record whether each durable artifact class was updated or
why no update was needed:

- `AGENTS.md` - workflow, responsibility, local framework, project-wide guardrails.
- Runtime project skills - `.agents/skills/project-*/SKILL.md` for HOW to work here.
- Specs - `.agents/sow/specs/` for WHAT the project does.
- End-user/operator docs - README, docs site, runbooks, published guides, help text, or
  other human-facing documentation.
- End-user/operator skills - output/reference skills copied or consumed outside normal
  repo work.
- SOW lifecycle - split, merge, status, directory, deferred work, regression reopening,
  and follow-up mapping.

This is an assistant responsibility. If a SOW changes behavior, docs, specs, commands,
schemas, defaults, workflows, examples, or operating procedure, the assistant must update
every affected artifact in the same SOW, or record the evidence-backed reason an artifact
is unaffected.

### Specs

Specs are memory of WHAT this project does.

`.agents/sow/specs/ai-link.md` is the product specification (wire format, lifecycle,
per-client surfaces, decisions log). It is authoritative for shipped behavior; the
implementation must agree with it, and when they disagree, record the discrepancy in the
active SOW and resolve or track it.

Update specs when shipped work changes:

- product behavior;
- public contracts (envelope format, on-disk layout, CLI surface, adapter contract);
- data formats;
- UX rules (per-client command/status surfaces);
- business logic (pause/master/round state machine);
- operational guarantees;
- known edge cases.

Specs describe current reality, not aspiration. Draft sections are allowed only while the
covering SOW is open, and must be marked as such.

### Project Skills

Project skills are memory of HOW to work here.

Runtime input project skills should live under `.agents/skills/project-*/SKILL.md`. The
`project-` prefix is the generic hook meaning "agents working in this repo must consider
this skill." Before non-trivial implementation, inspect those skill descriptions and load
every matching runtime skill. Skill descriptions are mandatory hooks, not suggestions.

Do not create generic `project-*` skills only to make the framework look complete. This
project has no runtime project skills yet; the decision to grow them incrementally is
recorded in `.agents/sow/current/SOW-0001-20260922-implement-ai-link-v1.md` (as the
implementation SOW progresses, concrete reusable knowledge — e.g. client-adapter testing
harness patterns — becomes `project-*` skills there).

Output/reference skills may also use `project-*` when that name is part of the exported
artifact semantics. Do not rename, shorten, or change their frontmatter descriptions only
to satisfy runtime discovery. Instead, list them separately below and exclude them from
default runtime guidance unless editing or validating those artifacts.

Non-`project-*` skills under `.agents/skills/` are not automatically runtime instructions.
If they are runtime input skills, rename them or add `project-*` wrappers. If the user
explicitly defers conversion, preserve them under `Legacy runtime skills` below and track
the unresolved alignment with a real SOW. If they are output/reference skills for end
users, operators, or downstream assistants, list them separately below with their intended
consumer.

Output/reference skills are part of the documentation/specification surface, not just
internal agent memory. When docs, specs, schemas, commands, defaults, examples, or
public/operator-facing workflows change, update every affected output/reference skill in
the same SOW, or record the evidence-backed reason none are affected.

Skills must be updated during retrospection when:

- the user corrects the assistant's workflow;
- a reviewer finds a repeated mistake;
- a validation misses a failure mode;
- a new command or workflow becomes canonical;
- a new project hazard is discovered;
- a new best or bad practice is learned;
- an output/reference skill would otherwise become stale after a docs/spec/product change.

### Project Skills Index

Runtime input skills: none yet (see Project Skills above; growth tracked by the active SOW).

Output/reference skills:

- (intended eventual artifact) per-client operator guides shipped under `docs/clients/`.
  Consumer: humans installing ai-link into Codex/Claude Code/OpenCode/pi.
  Update when: per-client integration surfaces or install story change. Not created yet;
  created together with their milestones in SOW-0001.

### Project-specific commands

Toolchain fixed by SOW-0001 (minimalism constraint): plain ESM JavaScript, `node` ≥ 20,
tests via `node --test`, no build step, zero runtime dependencies. Canonical commands will
be recorded here when M1 lands the `package.json` (`npm test` wrapping `node --test test/`).

### Project-specific overrides

- The product spec lives at `.agents/sow/specs/ai-link.md`; the repo-root `SPEC.md` is a
  pointer stub, not a second source of truth.
- Design constraints that override convenience:
  - No daemon, no network, no patches to any client — if a proposed implementation needs
    one of those, stop and ask the user.
  - **Absolutely minimal implementation sufficient to deliver the complete specified
    functionality** (user constraint, 2026-09-22): zero runtime dependencies, no build
    step, one package, smallest file tree that delivers the spec. Every abstraction,
    config knob, helper layer, and "nice to have" must earn its place against the spec —
    if removing it still delivers specified behavior, remove it. When minimal and elegant
    conflict, minimal wins; record any exception in the active SOW.

### Preservation Notes

- Repo created 2026-09-22; bootstrapped from empty. The original draft spec `SPEC.md`
  (v0.1) was moved verbatim-with-amendments into `.agents/sow/specs/ai-link.md` (v0.2,
  adding the `role` attribute, join welcome, `transcripts/` shared-history directory, and
  status/list commands per user decisions; v0.3 after round-1 external review); `SPEC.md`
  remains as a redirect stub so the draft's history and any inbound references still
  resolve.

Project SOW status: initialized

Active SOW: `.agents/sow/current/SOW-0001-20260922-implement-ai-link-v1.md`
(in-progress). The line above is the framework-installation marker checked by
`.agents/sow/audit.sh`; do not repurpose it for lifecycle state.
