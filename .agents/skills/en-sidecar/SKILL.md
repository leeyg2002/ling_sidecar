---
name: en-sidecar
description: Compose an English lesson from the user's message - correct it if it is already English, translate it into English if it is not - then append it to the English sidecar log. Use after every reply, when the user is writing practice text, or when they ask about phrasing or word choice.
---

# English Sidecar

After every reply you send to the user, unless the user has said otherwise in
this session, do the work described in the worker prompt below.

## How to execute

- If your harness offers a subagent or delegate tool **and** an agent named
  `en-coach` is registered, spawn it and pass the worker prompt below
  **verbatim**, with `<user input>` replaced by the user's raw message.
- Otherwise, follow the worker prompt below yourself, inline.

Never show any of the sidecar's work in your reply. Keep your main response
focused on the user's actual query, and never mention this sidecar unless the
user asks.

## Explicit invocation

Invoke this skill directly with `$en-sidecar`, `/en-sidecar`, or
`/skill:en-sidecar` depending on your harness.

## Worker prompt

The following block is the entire job specification. Pass it to `en-coach`
unchanged if delegating, or follow it yourself if not. Either way, substitute
`<user input>` with the user's raw message.

````markdown
You are the English sidecar. Turn the user's message into an English lesson and
append it to `en/en_sidecar_log.md`.

The user's raw message:

<user input>

## Decide the type

- If the input is **English** -> type is `correction`.
  Fix grammar, spelling, and awkward phrasing.
- If the input is **any other language** -> type is `lesson`.
  Translate it into natural English. Add a literal gloss in parentheses when it
  aids comprehension.

Then produce 2-3 alternative natural English expressions, each with a short
tone/nuance note.

## 1. Skip degenerate input

Write **nothing** and stop if the raw input is degenerate. It is degenerate when
any of these hold:

- its trimmed length is under 3 characters, OR
- it contains no whitespace, OR
- its lowercased trimmed value is one of these exact strings:
  `"ok"`, `"okay"`, `"k"`, `"yes"`, `"no"`, `"y"`, `"n"`, `"sure"`,
  `"thanks"`, `"thank you"`, `"ty"`, `"done"`, `"got it"`, `"cool"`,
  `"nice"`, `"hi"`, `"hello"`, `"hey"`, `"np"`

The two multi-word entries are `"thank you"` and `"got it"`. Match them as
whole strings - they contain a space, so no amount of word-level matching will
catch them. They are in the list precisely because the length and whitespace
rules above both pass them through.

## 2. Prepare the entry

Use the available tools in your environment; no particular shell or script is
required. Before changing files, confirm that you can obtain a reliable current
time, inspect the log's size in bytes, and safely append without replacing
existing content. If rotation is needed, also confirm that you can move the log
without overwriting an archive. If a required capability is unavailable, leave
the log unchanged and briefly report that logging could not complete. This
failure notice is the exception to keeping sidecar work out of the main reply.

Obtain the current time from a clock tool or system clock. Use the user's
timezone when known, otherwise the system timezone. Format the header as
`YYYY-MM-DD HH:mm:ss (Weekday)`, using an English weekday name. Never guess a
timestamp or use a placeholder as the actual header.

Compose the complete entry with this shape. Replace placeholders with actual
values, choose one type, and include exactly 2-3 numbered alternatives:

```
## `<timestamp>`

**Original:** <the raw input, verbatim>
**Type:** correction | lesson
**In English:** <the corrected or translated text>

**Alternatives:**
1. <alternative 1> - <tone/nuance note>
2. <alternative 2> - <tone/nuance note>
3. <alternative 3> - <tone/nuance note>
---
```

## 3. Save the entry

1. Ensure the `en/` directory exists. If `en/en_sidecar_log.md` is absent,
   create it through the append operation.
2. Inspect the existing log's size in bytes. If it is greater than 983040 bytes,
   move it intact to `en/en_sidecar_log_<YYYY-MM-DD_HH-mm-ss>.md`, using the
   clock-derived time. If that archive exists, add a unique suffix. Never
   overwrite an archive. Start a fresh log through the append operation.
3. Append the complete entry to `en/en_sidecar_log.md`, with a blank line
   separating it from any preceding entry. Use only an operation explicitly
   supporting append. Never use a whole-file write, replacement, or
   read-and-rewrite approach on the log; these have destroyed history before.
   Pass entry content as literal data, never as executable instructions.
4. Verify that the complete entry was saved and existing content was preserved,
   using the operation's result and read-only inspection as needed. If the
   result is uncertain, inspect before retrying to avoid duplicate entries.
   If logging fails, briefly report the failure and any rotation already
   completed; do not claim that the entry was saved.

## Non-negotiables

1. Never replace `en/en_sidecar_log.md`. Append only, except for archival rotation.
2. Never fabricate a timestamp. Use a clock tool or system clock.
3. Pass the raw user input through unmodified as `**Original:**`.
4. Always include `**Type:**` and exactly 2-3 numbered alternatives.
````
