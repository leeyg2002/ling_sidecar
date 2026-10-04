# ling_sidecar

A sidecar is a quiet companion that runs alongside your chat. The conversation is
never interrupted, but everything you type becomes practice material.

There are two of them today, one for English and one for French. Adding a third
follows the same pattern: a skill, a subagent, and a log.

After every reply, each sidecar spawns a subagent that turns your raw message into
a lesson — a correction if you already wrote in that language, a translation if
you didn't — then offers 2–3 alternative phrasings with a note on tone and nuance.
Every lesson lands in a local log.

| | English | French |
| --- | --- | --- |
| Skill | [`en-sidecar`](.agents/skills/en-sidecar/SKILL.md) | [`fr-sidecar`](.agents/skills/fr-sidecar/SKILL.md) |
| Log | `en/en_sidecar_log.md` | `fr/fr_sidecar_log.md` |
| What it does | Corrects your English; translates anything else into natural English | Corrects your French; translates anything else into natural French |

Every language runs at once, so each message produces one entry per language. Turn
one off for a session and the rest keep going.

They are [Agent Skills](https://agentskills.io/specification), so the same files
work on OpenCode, Codex, and Pi with no per-harness setup.

## Turning them on

Which sidecars fire is a pure function of the `instructions` array in
`opencode.jsonc` — empty for none, one path for one language, both paths for
both:

```jsonc
"instructions": [
  ".agents/skills/en-sidecar/SKILL.md",
  ".agents/skills/fr-sidecar/SKILL.md"
]
```

On OpenCode this fires them every turn. On Codex and Pi the skills are discovered
and chosen from their descriptions, which is reliable but not a guarantee.

Per session, in any harness:

| Action | OpenCode | Codex | Pi |
| --- | --- | --- | --- |
| activate | `/en-sidecar` | `$en-sidecar` | `/skill:en-sidecar` |
| off | `/en-sidecar off` | `$en-sidecar off` | `/skill:en-sidecar off` |
| status | `/en-sidecar status` | `$en-sidecar status` | `/skill:en-sidecar status` |

Plain language works too ("turn the French sidecar off"). `status` reports entry
count, byte size, and last-entry time for that language only.

## What an entry looks like

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

`correction` when the message is already in the target language, `lesson` when it
is not. That field is what makes the log filterable. The French log is identical
in shape, with `**En Français:**` in place of `**In English:**`.

## Your logs stay local

`en/` and `fr/` are gitignored and never pushed. Logs rotate at 983040 bytes and
the old generations are kept forever, with self-describing filenames.

Entries are append-only: the `write` tool never touches a log directly, and the
timestamp always comes from a real `date` call, so it cannot be fabricated.

Requires a POSIX shell (`sh`/`bash`). On Windows that means Git Bash or WSL.

## Further reading

- [`.agents/skills/README.md`](.agents/skills/README.md) — full behaviour, entry
  format, the corrections-only filter, and cross-harness portability
- [`docs/PRD.md`](docs/PRD.md) — design rationale, decision log, risks
