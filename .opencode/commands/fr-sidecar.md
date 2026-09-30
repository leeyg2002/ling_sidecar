---
description: Activate, deactivate, or check the French sidecar.
---

# /fr-sidecar

Argument received: `$ARGUMENTS`

Handle exactly one of these three cases.

## Case `off`

If `$ARGUMENTS` is `off`, stop spawning the `fr-coach` worker after each reply
for the rest of this session. Do not load the skill. Do not change any other
sidecar. Confirm in one short sentence that French is off.

## Case `status`

If `$ARGUMENTS` is `status`, report the state of `fr/fr_sidecar_log.md`:

```bash
LOG="fr/fr_sidecar_log.md"
if [ ! -f "$LOG" ]; then
  echo "no log yet"
else
  N=$(grep -c '^## ' "$LOG" || true)
  echo "entries: $N"
  echo "bytes:   $(wc -c < "$LOG" | tr -d ' ')"
  if [ "$N" -gt 0 ]; then
    echo "last:    $(grep '^## ' "$LOG" | tail -1)"
  else
    echo "last:    none (log is empty)"
  fi
fi
```

Then report those numbers in one or two plain sentences. If the log exists but
has no entries, say so explicitly rather than reporting a blank last-entry time.
Do not spawn a worker. Do not attempt to validate the format of any entry.

## Case activate (default)

For an empty argument, `on`, or anything else: load the `fr-sidecar` skill with
the `skill` tool and follow it for the rest of this session. Do not restate the
skill's contents here and do not inline its worker prompt. Confirm in one short
sentence that French is on.
