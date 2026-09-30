# en-sidecar

The English half of the sidecar pair. After every reply you send, the user's raw
message is turned into an English lesson and appended to
`en/en_sidecar_log.md`. The user never sees the work.

## Works on OpenCode, Codex, and Pi

The skill lives at `.agents/skills/en-sidecar/SKILL.md` — the
[Agent Skills](https://agentskills.io/specification) standard location, read
natively by all three harnesses:

| Harness | How to invoke |
|---|---|
| OpenCode | `/en-sidecar`, or fires automatically via `instructions` |
| Codex | `$en-sidecar` |
| Pi | `/skill:en-sidecar` |

Worker *dispatch* differs per harness, so the skill body branches: if a
registered `en-coach` subagent exists it is delegated to, otherwise the work is
done inline. Both paths follow the identical worker prompt. On Codex and Pi that
means no context isolation until you add a per-harness agent file — see
`docs/PRD.md` §10.

## Behaviour

| Input | Type | What happens |
|---|---|---|
| English | `correction` | grammar, spelling, awkward phrasing fixed |
| Anything else | `lesson` | translated into natural English, with a literal gloss where useful |

Both then get 2-3 alternatives, each with a tone note.

## Entry format

```markdown
## `2026-09-27 14:35:00 (Sunday)`

**Original:** I go to the market yesterday to buy some bread.
**Type:** correction
**In English:** I went to the market yesterday to buy some bread.

**Alternatives:**
1. I went to the market yesterday to pick up some bread. - slightly more casual
2. I was at the market yesterday buying some bread. - shifts the focus to being there
3. Yesterday I made a trip to the market for bread. - more deliberate, a little formal
---
```

The `**Type:**` field is what makes the log filterable. To review only real
mistakes, keep `correction` entries. `lesson` entries are translations of
practice input, useful but not errors.

## Skip rule

Nothing is written for input under 3 characters, input with no whitespace, or
pure acknowledgements (`ok`, `thanks`, `got it`, ...). The rule is mechanical on
purpose — see the decision log in `docs/PRD.md`.

## Commands

```
/en-sidecar          activate for this session
/en-sidecar on       same
/en-sidecar off      stop spawning en-coach
/en-sidecar status   entry count, byte size, and last-entry timestamp
```

Plain language works too — "turn the English sidecar off" is equivalent to
`/en-sidecar off`.

## How entries get written

Deliberately append-only, in two shell steps:

1. The `write` tool creates a temp file with the entry body.
2. `printf` appends a header using a real `date` call, then `cat` appends the
   body, then the temp file is removed.

The `write` tool is never pointed at the log itself — it overwrites, and doing
so destroyed real history in an earlier version of this system. The timestamp
is always produced by `date`, never typed by the model. Rotated generations
appear as `en_sidecar_log_<timestamp>.md` at 983040 bytes and are kept forever.

## Reviewing

The log is gitignored, so it stays on your machine. To find the corrections
worth revisiting:

```bash
awk '/^\*\*Type:\*\* correction$/{f=1} f' en/en_sidecar_log.md
```

To see recent entries, `tail -60 en/en_sidecar_log.md`.

## Related

- `docs/PRD.md` — full design, decision log, and known risks
- `../fr-sidecar/` — the French counterpart
