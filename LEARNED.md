# LEARNED.md

Sharp edges, gotchas, and non-obvious behaviour you've hit while building this project.
Things that aren't obvious from the commit history or the ADRs — the kind of
"here's what bit us, here's what to do" notes that make future sessions faster.

This file is not a journal of what you did (`git log` covers that) and not a
contract (`CLAUDE.md` covers that). It's a list of **surprising things future-you
or a future contributor would otherwise re-learn the hard way.**

## When to add an entry

Add an entry when you've just spent more than ~15 minutes diagnosing something
that, in hindsight, has a one-line description. If the answer would have saved
you the hour, it goes here.

## When to remove an entry

When the underlying cause has been fixed at the source (vendor changed
behaviour, library upgraded past it, CLAUDE.md or an ADR now codifies the
guidance). Mark `RESOLVED <date>: <reason>` and remove in a follow-up cleanup
when the file gets long.

## Format

```
## <date> — <area>: <one-line summary>

**What bit us:** the specific symptom.

**Why it happened:** the underlying cause — vendor quirk, library
behaviour, type-system limitation, whatever.

**The fix / workaround:** how to avoid it next time.

**Trigger to re-evaluate:** what would make this no longer matter
(e.g. library upgrade, vendor change).
```

---

`<First entry lands when you hit your first non-obvious surprise. Until then, this file is intentionally empty.>`
