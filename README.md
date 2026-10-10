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

| | English | French |
| --- | --- | --- |
| Skill | [`en-sidecar`](.agents/skills/en-sidecar/SKILL.md) | [`fr-sidecar`](.agents/skills/fr-sidecar/SKILL.md) |
| Log | `en/en_sidecar_log.md` | `fr/fr_sidecar_log.md` |
| What it does | Corrects your English; translates anything else into natural English | Corrects your French; translates anything else into natural French |

When both are active, each eligible message produces one entry per language
if logging succeeds. Turn one off for a session and the other remains active.

They are [Agent Skills](https://agentskills.io/specification), so the same files
work on OpenCode, Codex, and Pi with no per-harness setup.

## Turning them on

Which sidecars fire is a pure function of the `instructions` array in
`.opencode/opencode.jsonc` — empty for none, one path for one language, both paths for
both:

```jsonc
"instructions": [
  ".agents/skills/en-sidecar/SKILL.md",
  ".agents/skills/fr-sidecar/SKILL.md"
]
```

On OpenCode this fires them every turn. On Codex and Pi the skills are discovered
and chosen from their descriptions, but discovery alone does not guarantee execution or log creation.

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

`en/` and `fr/` are gitignored and never pushed. Before appending, logs are
renamed as timestamped archives only when their size is greater than 983040
bytes (960 KiB). A log exactly at the limit is not rotated. Archives are kept
indefinitely, and existing archives are never overwritten.

Entries are append-only, and timestamps come from a clock tool or system clock
in the user's timezone when known. The instructions are plain language and do
not require Bash, Git Bash, WSL, or a particular script. They do require safe
append, file-size inspection, and clock access, plus safe move support when
rotation is needed. Missing capabilities prevent logging and are reported.

The OpenCode-only `status` commands still use Bash for log inspection; see the
full reference for that remaining command-specific dependency.

## Further reading

- [`.agents/skills/README.md`](.agents/skills/README.md) — full behaviour, entry
  format, the corrections-only filter, and cross-harness portability
- [`docs/PRD.md`](docs/PRD.md) — design rationale, decision log, risks
