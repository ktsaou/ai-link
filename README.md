# ai-link

Link interactive coding-agent sessions — **Codex**, **Claude Code**, **OpenCode**,
**pi** — on one machine through a filesystem bus at `/tmp/ai-link/<slug>/`, so linked
assistants see each other's transcripts. Turn-based; the human drives every session;
nothing wakes a model automatically.

> Status: specification agreed (v0.2), implementation pending
> (`.agents/sow/pending/SOW-0001-20260922-implement-ai-link-v1.md`).
> Product spec: [`.agents/sow/specs/ai-link.md`](.agents/sow/specs/ai-link.md).

## Quick picture

```
/ai-link review-bot alice     # link this session as alice
/ai-link pause                # stop publishing/receiving
/ai-link resume               # flush and rejoin the round
/ai-link master               # make this session the round starter
/ai-link status               # who's linked, states, pending envelopes
/ai-link exit                 # unlink
```

Each linked session appends its cumulative transcript to
`/tmp/ai-link/<slug>/transcripts/<name>.md`; peers receive `<ai-link role="…">` envelopes
when idle (OpenCode/pi inject directly; Claude Code/Codex deliver on your next prompt).

## Repository

- `AGENTS.md` — project instructions / SOW runtime contract
- `.agents/sow/` — SOW ledger and the product spec under `specs/`
- `packages/`, `clients/` — created by SOW-0001 (not yet implemented)
