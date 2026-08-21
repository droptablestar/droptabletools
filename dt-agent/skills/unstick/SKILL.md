---
name: unstick
description: >-
  Break out of a debugging session that has stopped converging - the cycle where
  a check finds errors, the errors get fixed, the next check finds new errors,
  and it repeats. Use when two fix-and-recheck rounds have passed without the
  failure list shrinking, when a fixed symptom reappears, or when a hypothesis
  is about to be retried. Applies to non-code debugging too: documents, configs,
  data, infrastructure.
---

# Debug Loop Breaker

## Trigger

Any one of these means the loop is active. Start at Step 1.

- Two fix-and-recheck rounds have passed and the failure list is not smaller.
- A recheck found a problem the previous fix caused.
- A hypothesis already recorded as ruled out is about to be retried.

## 1. Halt and write the ledger

Stop editing. State in one line which trigger fired.

Create or update `DEBUG-LOG.md` in the working directory (untracked). Capture
Pre-existing changes before anything else, so a later revert never touches
them. Fill every field before resuming work:

```markdown
# <one-line symptom>
Oracle: <exact command or checklist>  |  Passes when: <observable outcome>
Last known-good: <commit SHA, file copy, or "none">
Pre-existing changes: <diff, file list, or "clean">   (never reverted - Steps 2, 4)
In scope: <files or sections>   (anything else goes to Deferred, unfixed)

| # | Hypothesis | Change | Oracle result | Verdict |
|---|------------|--------|---------------|---------|

Deferred: <findings not fixed this session>
```

The ledger is on disk because context gets compacted and the ruled-out list is
the first thing lost. Read it before every attempt; it is authoritative.

The `Oracle` field is the load-bearing one. Most non-converging loops are caused
by leaving it undefined: "double check this" always finds something, so the
finding list is never empty. Constraints on it:

- A single command, or a fixed list of at most seven yes/no items.
- Identical every round. Never rewritten to make a change pass.
- If no oracle can be stated, stop and ask the user for one.
- If it turns out to be the wrong oracle, say so, ask the user for the right
  one, and restart at Step 1 with the counter in Step 3 reset to zero.

## 2. Name the loop

Apply the matching row's Exit once, before starting Step 3. Step 3's rules
then govern every round after that - do not repeat a Step 2 Exit mid-loop.

| Signature | Exit |
|---|---|
| Each recheck reports different, unrelated problems | Recheck against the oracle only; everything else is Deferred |
| Edits were made before a cause was confirmed | Produce evidence tying cause to symptom before the next edit |
| Each fix breaks something that worked | One time: apply Step 4's revert procedure, then reapply the accumulated intent as one minimal change. Step 3 governs every round from there |
| The same hypothesis keeps returning | Ledger wins: ruled out stays ruled out without new evidence |
| Fixes are correct but the symptom persists | The cause is outside the searched layer - environment, data, dependency version, or the spec. Test that assumption instead of patching |

## 3. Resume under rules

1. Write the hypothesis to the ledger before making the change.
2. One change, then run the oracle. Record the verbatim result.
3. If the oracle fails with a different failure than the one-line symptom
   recorded in Step 1, keep the change, record the new failure under
   Deferred per rule 5, and continue. If it fails with the same recorded
   symptom, revert the change.
4. After three oracle runs without a pass, go to Step 4.
5. Anything deferred under rule 3 stays deferred - do not revisit it before
   Step 4.

## 4. Escalate

Revert only the changes recorded in this session's ledger, restoring exactly
those files or sections to Last known-good. Never touch Pre-existing changes.

If a debugging edit and a Pre-existing change land in the same file or
section, a clean revert of one without the other is not possible - do not
attempt it. Leave that file as-is, note the overlap under `Needed from you`,
and revert only the files/sections where the two do not overlap.

If Last known-good is "none", leave the work in place and list every change
made so the user can undo them.

Then report and stop:

```markdown
## Stuck: <symptom>
Oracle: <command> - fails with <verbatim output>
Ruled out: <hypothesis> - <evidence>
Best remaining: <hypothesis> - blocked by <missing information>
State: reverted tracked changes to <ref>  OR  not reverted; changes made: <list>
Needed from you: <specific input>
Deferred: <findings>
```

Stopping with an accurate ledger is a better result than continuing with a tree
of unverified edits. Report it plainly rather than apologizing, and do not
resume because the wait feels long - another round is what the loop is made of.
