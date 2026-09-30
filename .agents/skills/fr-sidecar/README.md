# fr-sidecar

The French half of the sidecar pair. After every reply you send, the user's raw
message is turned into a French lesson and appended to
`fr/fr_sidecar_log.md`. The user never sees the work.

## Works on OpenCode, Codex, and Pi

The skill lives at `.agents/skills/fr-sidecar/SKILL.md` — the
[Agent Skills](https://agentskills.io/specification) standard location, read
natively by all three harnesses:

| Harness | How to invoke |
|---|---|
| OpenCode | `/fr-sidecar`, or fires automatically via `instructions` |
| Codex | `$fr-sidecar` |
| Pi | `/skill:fr-sidecar` |

Worker *dispatch* differs per harness, so the skill body branches: if a
registered `fr-coach` subagent exists it is delegated to, otherwise the work is
done inline. Both paths follow the identical worker prompt. On Codex and Pi that
means no context isolation until you add a per-harness agent file — see
`docs/PRD.md` §10.

## Behaviour

| Input | Type | What happens |
|---|---|---|
| French | `correction` | grammar, spelling, awkward phrasing fixed |
| Anything else | `lesson` | translated into natural French |

Both then get 2-3 alternatives, each with a register note.

## French-specific checks

- **Gender and number** — noun and adjective agreement, including across a
  series of adjectives
- **Register** — `tu` vs `vous` chosen deliberately, and stated every time
- **False friends** — *actuellement*, *sensible*, *assister*, *librairie*,
  *bureau*, *faux*, *morceau*, *sympathique*
- **Calanques** — native French phrasing preferred over word-for-word rendering
  from English or Chinese

## Entry format

```markdown
## `2026-09-27 14:35:00 (Sunday)`

**Original:** Je vais au marché hier pour acheter du pain.
**Type:** correction
**En Français:** Je suis allé au marché hier pour acheter du pain.

**Alternatives:**
1. Je suis allé au marché hier acheter du pain. - plus familier, la deuxième infinitive est très courante
2. Hier, je me suis rendu au marché pour prendre du pain. - plus soigné
3. Je suis passé au marché hier pour du pain. - bref, très oral
---
```

Note the field name is `**En Français:**` (with the cedilla), not
`**En Francais:**`. The French log is otherwise identical in shape to the
English one.

## Skip rule

Nothing is written for input under 3 characters, input with no whitespace, or
pure acknowledgements (`ok`, `merci`, `thanks`, `got it`, ...). The rule is
mechanical on purpose — see the decision log in `docs/PRD.md`.

## Commands

```
/fr-sidecar          activate for this session
/fr-sidecar on       same
/fr-sidecar off      stop spawning fr-coach
/fr-sidecar status   entry count, byte size, and last-entry timestamp
```

Plain language works too — "turn the French sidecar off" is equivalent to
`/fr-sidecar off`.

## How entries get written

Deliberately append-only, in two shell steps:

1. The `write` tool creates a temp file with the entry body.
2. `printf` appends a header using a real `date` call, then `cat` appends the
   body, then the temp file is removed.

The `write` tool is never pointed at the log itself — it overwrites, and doing
so destroyed real history in an earlier version of this system. The timestamp
is always produced by `date`, never typed by the model. Rotated generations
appear as `fr_sidecar_log_<timestamp>.md` at 983040 bytes and are kept forever.

## Reviewing

The log is gitignored, so it stays on your machine. To find the corrections
worth revisiting:

```bash
awk '/^\*\*Type:\*\* correction$/{f=1} f' fr/fr_sidecar_log.md
```

To see recent entries, `tail -60 fr/fr_sidecar_log.md`.

## Related

- `docs/PRD.md` — full design, decision log, and known risks
- `../en-sidecar/` — the English counterpart
