# PRD — Language Sidecar (`en-sidecar` / `fr-sidecar`)

**Status:** Design settled, implementation pending
**Date:** 2026-09-27
**Project:** `ling_sidecar`
**Environment:** opencode 1.18.32, macOS

---

## 1. Goal

A pair of independent language-learning sidecars for this project:

- `en-sidecar` — turns the user's raw message into an English lesson
- `fr-sidecar` — turns the user's raw message into a French lesson

Every reply, each sidecar spawns its own worker, which appends one structured entry to that language's log. The user never sees the worker's output. The logs are the artifact.

The two sidecars are fully independent: separate skills, separate agents, separate commands, separate logs. Either can be designated on or off per project.

---

## 2. Behaviour

Each sidecar handles two cases, distinguished by the *language of the input*:

| Input language | `en-sidecar` | `fr-sidecar` |
|---|---|---|
| The target language | **correction** — fix grammar, spelling, awkward phrasing | **correction** — same, for French |
| Any other language | **lesson** — translate into natural English | **lesson** — translate into natural French |

Both cases then produce 2–3 alternative phrasings, each with a tone/nuance note.

For non-English input translated into English, add a literal gloss in parentheses when it aids comprehension. For the French worker, additionally flag noun/adjective gender and number agreement, `tu` vs `vous` register, common false friends (*actuellement*, *sensible*, *assister*), and prefer native phrasing over calques from English or Chinese.

**Activation rule:** the worker is spawned after every reply, *unless the user has said otherwise in this session*.

**Visibility rule:** the worker never surfaces output into the conversation, and the sidecar is never mentioned to the user unless they ask.

---

## 3. Names

Three namespaces, no collisions:

| Thing | English | French | Namespace |
|---|---|---|---|
| Skill directory + `name:` | `en-sidecar` | `fr-sidecar` | `skill({ name })` |
| Agent | `en-coach` | `fr-coach` | `task({ subagent_type })` |
| Command | `/en-sidecar` | `/fr-sidecar` | `/` menu |
| Log directory | `en/` | `fr/` | — |
| Log file | `en_sidecar_log.md` | `fr_sidecar_log.md` | — |
| Rotated log | `en_sidecar_log_<ts>.md` | `fr_sidecar_log_<ts>.md` | — |

`<ts>` = `date '+%Y-%m-%d_%H-%M-%S'`.

The agent is deliberately named `*-coach` rather than matching the skill, so the `@` autocomplete menu shows two distinct labels instead of two identical ones. Humans trigger a sidecar with `/en-sidecar`; the model resolves `@en-sidecar` (skill) and `@en-coach` (agent) by tool parameter.

---

## 4. Layout

```
ling_sidecar/
├── .gitignore                        # en/  fr/  .DS_Store
├── .opencode/
│   ├── .gitignore                    # node_modules  package*.json  bun.lock
│   ├── opencode.jsonc                # mode switch, agents, permissions
│   ├── commands/
│   │   ├── en-sidecar.md
│   │   └── fr-sidecar.md
│   └── skills/
│       ├── en-sidecar/
│       │   ├── SKILL.md              # single source of truth for the worker spec
│       │   └── README.md
│       └── fr-sidecar/
│           ├── SKILL.md
│           └── README.md
├── docs/
│   └── PRD.md
├── en/
│   └── en_sidecar_log.md             # gitignored
└── fr/
    └── fr_sidecar_log.md             # gitignored
```

`en/` and `fr/` are gitignored: the logs are personal study material, not source. Rotated generations are kept indefinitely and never pruned — their filenames are self-describing and sort chronologically, so no index is needed.

---

## 5. Mode switch

Mode is a pure function of the `instructions` array in `.opencode/opencode.jsonc`. Nothing else varies.

| Mode | `instructions` |
|---|---|
| **none** | `[]` |
| **en only** | `[".agents/skills/en-sidecar/SKILL.md"]` |
| **fr only** | `[".agents/skills/fr-sidecar/SKILL.md"]` |
| **both** | both paths |

**This is a soft gate, and the tradeoff is deliberate.** A skill listed in `instructions` is force-loaded every turn, so a designated sidecar fires *deterministically* — this is why the activation rule in §2 is a genuine guarantee rather than a hope. The cost: an undesignated skill is still discovered by the `skill` tool, so nothing *hard-denies* it. "En only" means *en always fires; fr is merely not instructed to fire.* A determined invocation of `/fr-sidecar` or `@fr-sidecar` will still start French.

The alternative was `permission.skill` with `"deny"` rules, which would be a hard gate — but it is bypassed by `instructions`, so using both would make `/sidecar on` fail for denied languages. Deterministic firing was judged worth more than hard denial. See §9.1.

---

## 6. Commands

`.opencode/commands/{en,fr}-sidecar.md`. Identical surface for both languages:

| Invocation | Effect |
|---|---|
| `/en-sidecar` | Load the skill via the `skill` tool; follow it for the rest of the session |
| `/en-sidecar off` | Stop spawning `en-coach`; leave other sidecars alone |
| `/en-sidecar status` | Report last-entry time and entry count for English |

Commands are **thin**. They contain no worker spec — they load the skill, which is the single source of truth (§7). This keeps the copy-paste drift class closed.

Deactivation also works in plain language ("stop the French sidecar for this session"), which is what the *"unless the user has said otherwise"* clause in §2 refers to. Toggle-by-command was rejected: it requires the model to track state it can silently lose, and a sidecar that stays on when you meant it off is worse than needing a sentence.

`/en-sidecar status` reports a single language. Read the other log directly, or the rotation filenames, for the rest.

---

## 7. Skills and agents

### 7.1 Division of responsibility

`SKILL.md` is the **only** place the worker specification exists. The agent block in `opencode.jsonc` carries **no `prompt`** — the skill passes the spec to the worker verbatim as the task prompt at spawn time.

This is a deliberate reversal of the pattern that caused the original append bug: the same instructions lived in `SKILL.md` and in the JSONC agent prompt, drifted apart (`bash >>` in one, the overwriting `write` tool in the other), and the overwrite silently truncated real history. One copy cannot drift.

### 7.2 Agent configuration

```jsonc
"en-coach": {
  "mode": "subagent",
  "description": "Compose an English lesson from the user's message: correct it if it is already English, translate it into English if it is not. Append to en/en_sidecar_log.md.",
  "permission": {
    "read": "allow", "glob": "allow", "grep": "allow",
    "edit": "allow", "write": "allow",
    "bash": "allow", "task": "deny"
  }
}
```

- **No `model` field.** opencode resolves an unset subagent model to the invoking primary agent's model. Accepted: this couples sidecar output to whatever the main session runs.
- **No `prompt` field.** Per §7.1.
- `task: "deny"` so a worker can never spawn further workers.
- Not `hidden` — both agents stay in the `@` menu for direct invocation.

### 7.3 Worker spec (lives in SKILL.md, passed verbatim to the worker)

1. **Rotate if oversized.** If the log is ≥ 983040 bytes, move it to `<log>_<timestamp>.md` and `touch` a fresh one.
2. **Skip if degenerate** (§8.1) — write nothing and stop.
3. **Decide `correction` or `lesson`** from §2.
4. **Emit the header via bash**, so the timestamp is generated, never invented:
   ```bash
   mkdir -p en && printf '\n## %s\n\n' "$(date '+%Y-%m-%d %H:%M:%S (%A)')" >> en/en_sidecar_log.md
   ```
5. **Append the body with a quoted heredoc**, so backticks, `$(...)`, or a line reading `EOF` in user text cannot break the write or execute anything:
   ```bash
   cat >> en/en_sidecar_log.md <<'SIDECAR_EOF'
   ...entry body...
   SIDECAR_EOF
   ```
6. **Never use the `write` tool for the log.** It overwrites. This is the one instruction that must not be relaxed.

Step 4 exists because the timestamp was previously requested politely in a prompt and the log ended up with five competing formats — including a literally-pasted placeholder and a fabricated `12:00:00 GMT+0000`. Generating the header in shell makes fabrication structurally impossible rather than merely discouraged.

---

## 8. Log format

### 8.1 Skip rule

Write **no entry** when the raw input is degenerate:

- trimmed length < 3 characters, **or**
- it contains no whitespace, **or**
- its lowercased trimmed value is in the acknowledgement set: `ok okay k yes no y n sure thanks thank you ty done got it cool nice hi hello hey np`

The set is load-bearing, not decorative: `"got it"` has whitespace and is 6 characters, so the first two clauses miss it. The rule is mechanical rather than semantic on purpose — semantic judgement by the model is what produced the inconsistent timestamps.

### 8.2 Entry schema

```markdown
## `2026-09-27 14:35:00 (Sunday)`

**Original:** <raw user input, verbatim>
**Type:** correction | lesson
**In English:** <corrected or translated text>

**Alternatives:**
1. <alternative 1> — <tone/nuance note>
2. <alternative 2> — <tone/nuance note>
3. <alternative 3> — <tone/nuance note>
---
```

The French log uses **`**En Français:**`** (with the cedilla) in place of `**In English:**`, and nothing else changes.

`**Type:**` distinguishes a correction from a lesson, so the log can be filtered for "only my actual mistakes" without inferring intent by comparing `Original` to the output.

`## \`<timestamp>\`` keeps the literal backticks. The timestamp is produced by `date` in shell (see §7.3 step 4) and must never be guessed.

---

## 9. Research findings

Load-bearing facts behind the design. All verified against opencode 1.18.32.

### 9.1 `permission.skill` is implemented — the SDK is just stale

The bundled SDK is `@opencode-ai/sdk@1.14.19`, four minor versions behind the 1.18.32 binary, and its `AgentConfig.permission` type has no `skill` key. The binary does: `skill` is an explicit field of the embedded `PermissionConfig` schema, there is a live enforcement call site immediately before skill files load (`ask({ permission: "skill", patterns: [name] })`, with the skill name as the match pattern so `"internal-*": "deny"` works), it appears in the permission-key registries, and it has its own settings-UI entry.

So a hard gate is genuinely available. It is not used, because it is bypassed by `instructions` (§5) and the deterministic-firing guarantee was judged the higher-value property. Revisit if that trade ever flips.

### 9.2 The configured global model is dead config

`~/.config/opencode/opencode.jsonc` sets `"model": "ollama/qwen3.5:9b-mlx"`, but:

```
$ opencode models ollama
Error: Provider not found: ollama
```

ollama is not a registered provider in 1.18.32 — it must be declared under `provider.ollama` (`npm: "@ai-sdk/openai-compatible"`, `baseURL: "http://localhost:11434/v1"`). Ollama itself runs fine and serves the model on `:11434`; opencode simply never connects. Sessions fall back to another model.

Consequence for this project: the workers inherit the primary agent's model (§7.2), which means the model grading the user's English and French is whatever the session happens to fall back to. Recorded as an accepted risk in §11. The global config is explicitly out of scope (§13).

Available to opencode: `opencode/big-pickle` and seven other free models. Installed locally but unreachable: `qwen3.5:9b-mlx`, `qwen3.5:9b`, `gemma4:e4b-mlx`.

### 9.3 Subagent availability

Docs list `general` as a built-in subagent and it is a valid config key, but the `task` tool in practice advertised only `explore` — a read-only agent that cannot append to a log. Rather than depend on `general`, the sidecars spawn `en-coach`/`fr-coach`, which this project defines and therefore guarantees.

### 9.4 Skill frontmatter

Only `name` (required), `description` (required), `license`, `compatibility`, `metadata` are recognised; unknown fields are silently ignored. `disable-model-invocation` is a Claude Code field and is **not** supported here. `name` must match its directory and match `^[a-z0-9]+(-[a-z0-9]+)*$` — `en-sidecar` and `fr-sidecar` both pass.

### 9.5 Commands

Custom commands are markdown files in `.opencode/commands/` — **plural**. The filename becomes the command name. Templates support `$ARGUMENTS` and positional `$1`/`$2`; frontmatter supports `description`, `agent`, `model`, `subtask`. The per-language commands need none of these beyond `description`.

### 9.6 Discovery

OpenCode discovers skills from `.opencode/skills/`, `~/.config/opencode/skills/`, `.claude/skills/`, and `.agents/skills/`, walking up from cwd to the git worktree. This project uses **`.agents/skills/`**, the cross-harness standard location, which OpenCode, Codex, and Pi all read. See §10.

---

## 10. Portability and distribution

### 10.1 One skill directory, three harnesses

Skills live at `.agents/skills/{en,fr}-sidecar/SKILL.md`. This is the
[Agent Skills](https://agentskills.io/specification) standard location, read by:

| Harness | Project scope | Global scope |
|---|---|---|
| OpenCode | `.agents/skills/`, walking up to the git worktree | `~/.agents/skills/` |
| Codex | `$CWD`, ancestors, and `$REPO_ROOT` `.agents/skills/` | `~/.agents/skills/` |
| Pi | `.agents/skills/`, cwd to repo root | `~/.agents/skills/` |

Moving off `.opencode/skills/` is what makes a single copy work everywhere. No
per-harness skill configuration is needed — all three read the same files.

### 10.2 What does *not* port: the worker dispatch

Skill *discovery* is universal; subagent *dispatch* is not. Each harness has an
unrelated API:

- **OpenCode** — `task({ subagent_type: "en-coach" })`, agent defined in `opencode.jsonc`
- **Codex** — agents are TOML files in `.codex/agents/*.toml`; builtins are `default`/`worker`/`explorer`
- **Pi** — agents are Markdown in `.pi/agents/`, and Pi has **no built-in subagent tool**; it needs a third-party extension

So `SKILL.md` must not hardcode one harness's API. Each skill body now specifies
the *work* and branches at execution time (D22):

```markdown
- If your harness offers a subagent or delegate tool **and** an agent named
  `en-coach` is registered, spawn it and pass the worker prompt below
  **verbatim**, with `<user input>` replaced by the user's raw message.
- Otherwise, follow the worker prompt below yourself, inline.
```

This degrades rather than breaks: OpenCode delegates to the configured subagent;
Codex and Pi do the work inline until a per-harness agent file is added. The
worker prompt block itself is byte-identical across that branch — it remains the
single source of truth (D12).

### 10.3 Adding real subagent isolation later

Optional per-harness upgrades, not required for the skill to work:

- `.codex/agents/en-coach.toml` and `fr-coach.toml` — Codex subagents are built in
- `.pi/agents/*.md` — requires installing a Pi subagent extension first

Neither changes `SKILL.md`; the branch already prefers a registered subagent
when one exists.

### 10.4 Distribution to other projects

The canonical source is this repo. A consumer enables sidecars by committing
**relative** symlinks:

```
<consumer>/.agents/skills/en-sidecar     ->  ../../<path>/ling_sidecar/.agents/skills/en-sidecar
<consumer>/.agents/skills/fr-sidecar     ->  ../../<path>/ling_sidecar/.agents/skills/fr-sidecar
<consumer>/.opencode/commands/en-sidecar.md ->  ../../<path>/ling_sidecar/.opencode/commands/en-sidecar.md
<consumer>/.opencode/commands/fr-sidecar.md ->  ../../<path>/ling_sidecar/.opencode/commands/fr-sidecar.md
```

Only the `commands/` symlinks are OpenCode-specific. The skill symlinks are
harness-neutral, so a Codex or Pi consumer ignores the command links and still
discovers the skills.

Relative symlinks are committed to the consumer's repo, so they survive either
repository being moved or cloned. Absolute paths break the moment either location
changes.

An OpenCode consumer keeps its **own** `opencode.jsonc` — its `instructions`
array *is* its mode switch and cannot be shared — and its own `en/` and `fr/`
log directories.

Because each agent block is metadata only (§7.2), there is no duplicated
specification to drift. Updating a skill in this repo updates it for every
consumer on the next pull.

### 10.5 Known harness caveats

- **Pi requires project trust.** In an untrusted project, `.agents/skills/` is
  silently ignored. If the skills do not appear in Pi, check trust before
  suspecting the layout.
- **Codex registers nested `SKILL.md` files** found beneath an installed skill
  directory ([openai/codex#22275](https://github.com/openai/codex/issues/22275)).
  These skill directories contain no nested `SKILL.md`, so nothing is
  double-registered — but do not add one without reading that issue.
- **Codex must not be sandboxed read-only**, or the log write fails.

---

## 11. Decision log

| # | Decision | Rationale |
|---|---|---|
| D1 | Skills `en-sidecar` / `fr-sidecar` | Short, groups together |
| D2 | Logs `en_sidecar_log.md` / `fr_sidecar_log.md` | Matches the skill names |
| D3 | Real skills with YAML frontmatter | Discoverable and `@`-invocable |
| D4 | Non-target-language input becomes a lesson, not a translation-only branch | Every message is practice material in both languages |
| D5 | Subagents named `en-coach` / `fr-coach` | Avoids two identical labels in the `@` menu |
| D6 | No `model` on either subagent | Inherits the primary agent's model; zero config |
| D7 | Global config untouched | Out of scope; the dead ollama line stays |
| D8 | Agents not `hidden` | Direct invocation stays available |
| D9 | Mode switch is the `instructions` array | Only gate confirmed working here; gives deterministic firing |
| D10 | Per-language commands, not one parameterised command | No argument parsing; each carries its own language |
| D11 | Command surface is on / off / status | Natural language also deactivates, so toggle state is unnecessary |
| D12 | Commands are thin; `SKILL.md` is the only spec | Closes the duplication class that caused the original append bug |
| D13 | `printf` header + temp-file body, appended by `cat` | The timestamp comes from a real `date` call, so fabrication is structurally impossible. (Corrected from an earlier quoted-heredoc design, which could not pass the user's raw input through unexpanded.) |
| D14 | Mechanical skip rule, not semantic | Semantic judgement produced five inconsistent timestamp formats |
| D15 | `**Type:**` field | Makes the log filterable for actual mistakes |
| D16 | Target-language label as the field name | Correct for both correction and lesson cases |
| D17 | No `plan` write carve-out | Reversed from the original D17. The carve-out could not help: the append needs a `bash` step, which plan mode denies, so a `write` grant alone never completes an entry. Worse, `write allow en/**` would let a read-only agent overwrite the log, which has no version-control recovery. Removed outright. |
| D18 | `{"*": "ask", ...}` explicit fallback | Retained as a config style for any future narrow carve-out; no longer used, since the D17 block is gone. |
| D19 | Logs gitignored, rotations kept forever | Personal study material; filenames self-describe, so no index needed |
| D20 | Relative symlinks for distribution | Survives moves and clones of either repo |
| D21 | Soft gate accepted over hard denial | Deterministic firing judged more valuable than hard denial (§9.1) |
| D22 | Skills moved to `.agents/skills/` | The Agent Skills standard path, read natively by OpenCode, Codex, and Pi. One committed copy serves all three with no per-harness config (§10.1) |
| D23 | `SKILL.md` branches on subagent availability instead of naming one API | Worker dispatch is the only non-portable layer. Degrading to inline execution works everywhere today; hardcoding `task`/`subagent_type` would silently no-op on Codex and Pi (§10.2) |
| D24 | Worker prompt block kept byte-identical across the branch | The single-source-of-truth rule (D12) must survive the portability refactor, or the duplication class that caused the original append bug returns |
| D25 | Canonical copy committed in-repo only, not symlinked into `~/.agents/skills/` | Avoids a global install that would make the sidecar fire in every project on the machine. Distribution stays explicit, per consumer (§10.4) |
| D26 | `plan` permission block removed | No functional benefit, real data-loss risk (D17) |

---

## 12. Implementation steps

1. `git init` + initial commit. Nothing distributes correctly without it, and it makes the symlink target a real repo.
2. `.gitignore` (`en/`, `fr/`, `.DS_Store`) and `.opencode/.gitignore`.
3. `.agents/skills/en-sidecar/SKILL.md` — frontmatter, the §7.3 worker spec, `en/en_sidecar_log.md`, `en-coach`, English-specific correction and translation rules.
4. `.agents/skills/fr-sidecar/SKILL.md` — same skeleton plus the French-specific checks in §2.
5. `.opencode/commands/en-sidecar.md` and `fr-sidecar.md` — the §6 surface.
6. `.opencode/opencode.jsonc` — `en-coach` and `fr-coach` per §7.2, `instructions` set to **both**. No `plan` block (D26).
7. README for each skill — behaviour, entry format, review tips, mode switch, command surface.
8. Import history: copy any existing `english-polish-log.md` to `en/en_sidecar_log.md`; create `fr/fr_sidecar_log.md`.
9. Verify per §13.

---

## 13. Verification

- [ ] Both skills appear in the `skill` tool's available-skills list
- [ ] Both commands appear in the `/` menu
- [ ] `en-coach` and `fr-coach` resolve as `task` subagent types
- [ ] `en-coach` and `fr-coach` are distinct labels in the `@` menu
- [ ] A test message produces one entry in each log, with a real `date`-generated timestamp
- [ ] `**Type:**` is correctly `correction` for target-language input and `lesson` otherwise
- [ ] Neither log is truncated across multiple turns (append semantics hold)
- [ ] A message containing backticks, `$(...)`, or a line `SIDECAR_EOF` is logged verbatim without executing
- [ ] Degenerate input (`ok`, `thanks`, `got it`, a single token) produces no entry
- [ ] All four modes behave correctly: none / en / fr / both
- [ ] `/en-sidecar off` stops English and leaves French running
- [ ] `/en-sidecar status` reports a real last-entry time and count
- [ ] Rotation at 983040 bytes produces `en_sidecar_log_<ts>.md` and a fresh log
- [ ] Imported history is intact in `en/en_sidecar_log.md`

Note: `[ ]` entry-format checks above are manual. There is no automated validator — status reporting catches a sidecar that stopped firing, but not one that is firing with a malformed entry.

---

## 14. Risks and accepted tradeoffs

| Item | Impact | Status |
|---|---|---|
| Global `model: ollama/qwen3.5:9b-mlx` is dead config | Workers inherit an uncontrolled fallback model; language feedback quality is not under your control | **Accepted** (D6, D7). Fixing means adding `provider.ollama` to a global file outside this project |
| Soft gate (§5) | An undesignated sidecar can still be started deliberately | **Accepted** (D21). A hard gate is available (§9.1) if this ever matters |
| No automated format validation | Silent entry-format drift after a model update would go unnoticed | **Accepted**. `/…-sidecar status` catches non-firing only |
| `plan` could write the logs | A read-only primary could overwrite the sole artifact, which has no version-control recovery | **Resolved** (D17, D26). The carve-out is removed; plan mode can no longer touch `en/` or `fr/` |
| Two workers per turn | Roughly 2x latency and compute per message in "both" mode | Known cost of the design |
| Rotations never pruned | `en/` grows without bound | **Accepted** (D19); filenames are self-describing |
| Not a git repo yet | Symlinks in consumers will not resolve until step 1 | Resolved by step 1 |
| Codex and Pi run the work inline | No context isolation on those harnesses; sidecar reasoning sits in the main conversation, so a slip could leak commentary into the reply | **Accepted** (D23). Fixable with `.codex/agents/*.toml` (built in) or `.pi/agents/*.md` (needs an extension) — §10.3 |
| Implicit skill invocation is host-dependent | Codex and Pi may not load the skill after every reply the way OpenCode's `instructions` array guarantees | **Accepted**. D22 makes the skill *discoverable* everywhere; only OpenCode has a deterministic per-turn gate (§5) |

### Future improvements

- Add `provider.ollama` to the global config and pin the workers to a fast free model
- Pin a specific model per worker to make log quality reproducible and measurable
- Add a validator that checks recent entries for header, required fields, and plausible target-language content
- Add `.codex/agents/{en,fr}-coach.toml` for real Codex subagent isolation (§10.3)
- Evaluate a Pi subagent extension before committing to `.pi/agents/*.md` (§10.3)
- Extract the rotate/skip/append steps into a `scripts/` helper so all three harnesses run identical shell, and serialisation is enforced in code rather than by model discipline
- Consolidate other projects onto this design via the symlink set in §10.4
- Consider `color` on each agent for visual distinction in the TUI
- Add further languages, following the same three-namespace pattern

---

## 15. Out of scope

- Modifying `~/.config/opencode/opencode.jsonc`
- Adding a provider for ollama or other local models
- Migrating other existing projects to this design
- Languages other than English and French
- An automated log validator
- A third namespace for worker naming (i.e. keeping agents identical to skill names)
