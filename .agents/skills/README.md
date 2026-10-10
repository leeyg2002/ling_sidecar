# ling_sidecar

A sidecar is a quiet companion that runs alongside your chat. Your messages become practice material; skipped inputs produce no entry, and
logging failures are reported briefly.

There are two of them today, one for English and one for French. Adding a third
follows the same pattern: a skill, an optional coach subagent, and a log.

After each reply, an active sidecar follows its worker instructions, using a
registered coach subagent when available or working inline otherwise. It turns
your raw message into a correction or translation, then offers 2–3 alternative
phrasings with tone and nuance notes. Eligible messages are appended to a local
log when the required tools and permissions are available.

Both are [Agent Skills](https://agentskills.io/specification), so the same files
work on OpenCode, Codex, and Pi with no per-harness setup.

| Skill                               | Log                    | Target language |
| ----------------------------------- | ---------------------- | --------------- |
| [`en-sidecar`](en-sidecar/SKILL.md) | `en/en_sidecar_log.md` | English         |
| [`fr-sidecar`](fr-sidecar/SKILL.md) | `fr/fr_sidecar_log.md` | French          |

When both are active, each eligible message produces one entry per language
if logging succeeds. Turn
either off for a session without touching the other.

## Behaviour

| Input              | Type         | What happens                                            |
| ------------------ | ------------ | ------------------------------------------------------- |
| Target language    | `correction` | grammar, spelling, awkward phrasing fixed               |
| Any other language | `lesson`     | translated naturally, with a literal gloss where useful |

Both cases get 2–3 alternatives, each with a tone/nuance note. To review
corrections only, ask the agent to read the log and return complete entries
whose `**Type:**` field is `correction`, without changing the file. No shell
filter is required.

French additionally checks gender and number agreement, chooses `tu` or `vous`
deliberately (stating which and why), and flags false friends such as
*actuellement*, *sensible*, *assister*, *librairie*, *bureau*. Calques are
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

| Harness  | Automatic                            | Explicit                                 |
| -------- | ------------------------------------ | ---------------------------------------- |
| OpenCode | yes, via `instructions`              | `/en-sidecar`, `/fr-sidecar`             |
| Codex    | implicit, if the description matches | `$en-sidecar`, `$fr-sidecar`             |
| Pi       | implicit, if the description matches | `/skill:en-sidecar`, `/skill:fr-sidecar` |

Explicit arguments: bare name activates, `off` stops for the session, `status`
reports entry count, byte size, and last-entry time. On OpenCode these are
commands in `.opencode/commands/`; elsewhere plain language works equally well
("turn the French sidecar off").

The existing OpenCode `status` command files still use Bash for read-only log
inspection and need a POSIX environment on Windows. This command-specific
dependency does not apply to lesson generation or appending entries.

Note that only OpenCode fires the sidecar deterministically every turn, via its
`instructions` array. Codex and Pi choose implicitly from the skill description,
so discovery alone does not guarantee execution or log creation.

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

`../../docs/PRD.md` §10 covers this in full, including the Pi project-trust caveat
and a known Codex issue about nested `SKILL.md` files.

## How entries get written

The worker instructions are plain language; no particular shell or script is
required. Each worker:

1. Applies the skip rules before changing files.
2. Confirms clock access, byte-size inspection, and safe append support. Safe
   move support is also required if rotation is needed. Missing capabilities
   leave the log unchanged and produce a brief failure notice.
3. Obtains the current time from a clock tool or system clock, using the user's
   timezone when known and otherwise the system timezone. Headers use
   `YYYY-MM-DD HH:mm:ss (Weekday)` with an English weekday name.
4. Composes a complete entry, preserving raw input verbatim as literal data.
5. Before appending, renames a log greater than 983040 bytes (960 KiB) to
   `<language>_sidecar_log_<YYYY-MM-DD_HH-mm-ss>.md`. Exactly 983040 bytes does
   not trigger rotation. A unique suffix prevents archive collisions, and all
   archives are retained.
6. Appends the complete entry with an operation explicitly supporting append.
   Whole-file replacement and read-and-rewrite approaches are prohibited.
7. Verifies the saved entry and preservation of existing content. An uncertain
   result is inspected before retrying, so retries do not duplicate entries.
   Failures and any completed rotation are reported briefly.

Neither a fixed temporary file nor a Bash command is part of the required
workflow. The operation chosen by the agent must still preserve existing logs.

## Skip rule

Nothing is written for input under 3 characters, input with no whitespace, or
pure acknowledgements. English skips `ok`, `thanks`, `got it`, `np` and similar;
French additionally skips `merci`, `oui`, `super`, `d'accord` and similar. The
rule is mechanical on purpose — see the decision log in `../../docs/PRD.md`.

## Logs are local

`en/` and `fr/` are gitignored and never pushed. Your practice material stays on
your machine.
