# ai-link

Link interactive coding-agent sessions — **Codex**, **Claude Code**, **OpenCode**,
**pi** — on one machine through a filesystem bus at `/tmp/ai-link/<slug>/`, so linked
assistants see each other's transcripts. Turn-based; the human drives every session;
nothing wakes a model automatically.

> Status: specification agreed (**v0.3**, after round-1 external review), implementation
> in progress (`.agents/sow/current/SOW-0001-20260922-implement-ai-link-v1.md`).
> Product spec: [`.agents/sow/specs/ai-link.md`](.agents/sow/specs/ai-link.md).

## Quick picture

Native slash commands on OpenCode and pi; **slashless** on Claude Code and Codex
(zero-model-turn hook interception — the same commands, typed without the leading `/`):

```
/ai-link review-bot alice     # join as alice          (OpenCode, pi)
ai-link review-bot alice      # join as alice          (Claude Code, Codex)
/ai-link pause                # stop publishing/receiving
/ai-link resume               # go active, publish outbox, receive inbox
/ai-link master               # make this session the round starter
/ai-link status               # members, states, outbox/inbox counts, master aliveness
/ai-link exit                 # unlink
```

Each linked session appends its cumulative transcript to
`/tmp/ai-link/<slug>/transcripts/<name>.md`; peers receive `<ai-link role="…">` envelopes
when idle (OpenCode/pi inject directly; Claude Code/Codex deliver on your next prompt).

## Repository

Canonical home: <https://github.com/ktsaou/ai-link> (public).

- `AGENTS.md` — project instructions / SOW runtime contract
- `.agents/sow/` — SOW ledger and the product spec under `specs/`
- `lib/`, `cli/`, `clients/` — created by SOW-0001 (not yet implemented)
