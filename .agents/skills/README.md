# ling_sidecar

Two language sidecars. After every reply, the user's raw message is turned into
a lesson for the target language and appended to a local log. The user never sees
the work.

| Skill | Log | Target language |
|---|---|---|
| [`en-sidecar/`](./en-sidecar/SKILL.md) | `en/en_sidecar_log.md` | English |
| [`fr-sidecar/`](./fr-sidecar/SKILL.md) | `fr/fr_sidecar_log.md` | French |

Both are active at once, so each message produces one entry per language. Turn
either off for a session without touching the other.

## Behaviour

| Input | Type | What happens |
|---|---|---|
| Target language | `correction` | grammar, spelling, awkward phrasing fixed |
| Any other language | `lesson` | translated naturally, with a literal gloss where useful |

Both cases then get 2–3 alternatives, each with a tone/nuance note. The `**Type:**`
field is what makes a log filterable — to review only real mistakes:

```bash
awk '/^\*\*Type:\*\* correction$/{f=1} f' en/en_sidecar_log.md
```

French additionally checks gender and number agreement, chooses `tu` or `vous`
deliberately (stating which and why), and flags false friends such as
*actuellement*, *sensible*, *assister*, *librairie*, *bureau*. Calenques are
avoided in both.

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

The French log is identical in shape, with `**En Français:**` (cedilla) in place
of `**In English:**`.

## Invoking

| Harness | Automatic | Explicit |
|---|---|---|
| OpenCode | yes, via `instructions` | `/en-sidecar`, `/fr-sidecar` |
| Codex | implicit, if the description matches | `$en-sidecar`, `$fr-sidecar` |
| Pi | implicit, if the description matches | `/skill:en-sidecar`, `/skill:fr-sidecar` |

Explicit arguments: bare name activates, `off` stops for the session, `status`
reports entry count, byte size, and last-entry time. On OpenCode these are
commands in `.opencode/commands/`; elsewhere plain language works equally well
("turn the French sidecar off").

Note that only OpenCode fires the sidecar deterministically every turn, via its
`instructions` array. Codex and Pi choose implicitly from the skill description,
which is reliable but not a hard guarantee.

## Portability

These skills live at `.agents/skills/`, the
[Agent Skills](https://agentskills.io/specification) standard path, read natively
by all three harnesses — so one committed copy serves every agent with no
per-harness configuration.

Worker *dispatch* is not universal, so each `SKILL.md` branches at execution time:
if the harness has a registered `en-coach`/`fr-coach` subagent it is delegated to,
otherwise the same worker prompt is followed inline. The worker prompt block is
byte-identical across that branch, so `SKILL.md` remains the single source of
truth. Today only OpenCode defines those subagents; Codex subagents are TOML
files in `.codex/agents/`, and Pi needs a third-party extension first. Until then
Codex and Pi run the work inline, without context isolation.

`docs/PRD.md` §10 covers this in full, including the Pi project-trust caveat and a
known Codex issue about nested `SKILL.md` files.

## How entries get written

Deliberately append-only, in two steps:

1. The `write` tool creates a temp file with the entry body.
2. `printf` appends a header using a real `date` call, then `cat` appends the
   body, then the temp file is removed.

The `write` tool is never pointed at the log itself — it overwrites, and doing so
destroyed real history in an earlier version of this system. The timestamp always
comes from `date`, never typed by the model. User input is passed through
verbatim, so shell metacharacters in a message are logged literally rather than
executed.

Rotations happen at 983040 bytes and are kept forever as
`en_sidecar_log_<timestamp>.md`.

## Skip rule

Nothing is written for input under 3 characters, input with no whitespace, or
pure acknowledgements. English skips `ok`, `thanks`, `got it`, `np` and similar;
French additionally skips `merci`, `oui`, `super`, `d'accord` and similar. The
rule is mechanical on purpose — see the decision log in `docs/PRD.md`.

## Logs are local

`en/` and `fr/` are gitignored and never pushed. Your practice material stays on
your machine.