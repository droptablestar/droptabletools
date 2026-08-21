---
name: cleanup
description: >-
  Audit and prune Claude Code's persistent memory - CLAUDE.md, .claude/rules/, and auto
  memory at user and project scope. Reports contradictory, duplicated, stale, misplaced,
  and redundant instructions, and applies only the edits the user approves. Use for any
  ask to clean up, trim, audit, prune, or reorganize memory, CLAUDE.md, rules, or agent
  context, including vaguer ones like "my CLAUDE.md has gotten out of hand" or "why does
  Claude keep ignoring my instructions".
license: MIT
allowed-tools: Read, Grep, Glob, Bash(git status:*), Bash(git diff:*), Bash(ls:*), Bash(find:*), Bash(readlink:*), Bash(stat:*), Bash(wc:*)
---

# Memory cleanup

Memory decays: instructions accumulate faster than they are removed, the same
rule lands at two scopes, and files keep describing a codebase that has moved on.
A stale instruction is worse than none - it is followed confidently.

Report first, edit second. The user knows why a strange-looking rule is there;
you do not. Audit both scopes unless the user names one, and change nothing until
they approve specific edits.

## Step 1: Inventory

Start from what actually loaded. Ask the user to run `/context` and report the
list under **Memory files**; `/memory` lists locations across scopes. If nobody
is there to run it, say so and work from disk alone.

Read each location on disk either way - the two answer different questions.
`/context` is what this session got; disk is what the next one gets, so a file
edited after launch loads stale. Where they disagree, say which you used.
Loaded in full at launch:

| Scope | Location |
|---|---|
| Managed policy | macOS `/Library/Application Support/ClaudeCode/CLAUDE.md`; Linux and WSL `/etc/claude-code/CLAUDE.md`; Windows `C:\Program Files\ClaudeCode\CLAUDE.md` |
| User | `~/.claude/CLAUDE.md`, `~/.claude/rules/**/*.md` |
| Project | `./CLAUDE.md` or `./.claude/CLAUDE.md`, `./.claude/rules/**/*.md` |
| Local | `./CLAUDE.local.md` |
| Auto memory | `~/.claude/projects/<project>/memory/MEMORY.md`, first 200 lines or 25KB |

Auto memory is what Claude wrote about this repo, not what the user wrote, so a
stale entry there is still a finding - propose it the same way. Topic files beside
`MEMORY.md` load on demand and cost nothing at launch.

Check `~/.claude/memory/` too. The per-project path can be empty while a store
sits there instead. If you cannot tell whether such a file loads, inventory it
and say so - do not assume either way.

Easy to miss:

- **Parent directories.** Claude loads every `CLAUDE.md` and `CLAUDE.local.md`
  between the working directory and the filesystem root. A stale file two levels
  up loads invisibly - walk the chain.
- **Subdirectory files.** Load only when work touches that subtree - absent most
  sessions, surprising when they appear.
- **Rules are recursive.** Every `.md` under a `rules/` directory is discovered,
  at any depth. Do not glob one level deep.
- **`@path` imports.** A `CLAUDE.md` can import other files, nesting up to four
  hops, and a file's real weight includes everything it pulls in. Paths resolve
  relative to the importing file; a path in backticks or a fenced block is not an
  import. Note any that resolve to nothing.

A rule with a `paths:` frontmatter key loads only when Claude reads a matching
file; without one it loads every session. Record which is which - it changes what
the rule costs.

Resolve `~/.claude` before recording paths. If it is a symlink, record where it
lands and whether that location is tracked - a user-scope edit is then a commit,
and may sync to other machines.

Run `git status` in the project and again in `~/.claude`, in case it is tracked.
Report uncommitted changes before touching anything - a clean tree is the whole
safety net.

Report the inventory with line counts before analyzing, so the user can flag
anything missed.

## Step 2: Analyze

Read each file completely. Look for these, in rough order of value.

**Contradictions.** Two instructions that cannot both be followed, such as "always
use pnpm" at user scope against "use npm, we pin the lockfile" at project scope.
Highest value: they produce inconsistent behavior with no visible cause.

CLAUDE.md files are concatenated, not overridden - a project file is read after
the user file, not instead of it. Nothing resolves a conflict, so per the docs,
"if two rules contradict each other, Claude may pick one arbitrarily." Never pick
for the user: present both with file and line, and ask which survives.

Reading file by file will not find these. Contradictions hide between sections
that never mention each other - a testing rule under "workflow" against another
under "efficiency". Collect every directive on one subject across all files and
sections first, then compare them as a set. Subjects worth a pass of their own:
running tests, committing, asking before acting, verbosity, tool choice.

**Duplicates across scope.** The same rule stated at both user and project scope.
The project copy usually wins because it travels with the repo - say why rather
than assume.

**Stale references.** Anything memory asserts about the codebase that no longer
holds - paths, scripts, services, versions, ports. Verify each against the repo -
`Glob` and `Grep` are cheap, a wrong deletion is not - and mark anything
unverifiable as unverified rather than dropping it. Also flag rules for a
framework the project has left, workarounds for a version long upgraded, and
anything the toolchain now enforces: linter, formatter, type checker, pre-commit
hook. If CI catches it, memory need not.

**Misplaced scope.** A personal preference in a project file, where it is imposed
on everyone who clones the repo, or a project-specific rule in the user file, where
it leaks into unrelated work - the usual explanation for "why did Claude do that
in this other repo".

**Instructions Claude already follows.** Restatements of language or framework
convention, or of default behavior: context every session, nothing bought. Flag,
do not delete.

**Bloat.** Target is under 200 lines per CLAUDE.md; longer files consume context
and reduce adherence. Prefer relocation over deletion: applies to part of the
tree -> a `paths:`-keyed rule in `.claude/rules/`; a multi-step procedure -> a
skill; duplicates a README, a config, or the type system -> a reference to the
source. Show the condensed version so the user can judge what survived.

**Vagueness.** An instruction that cannot be verified will not be followed
reliably. "Format code properly" and "test your changes" are noise; "use 2-space
indentation" and "run `npm test` before committing" are not. Propose the concrete
rewrite or propose the cut. Do not leave a vague line alone because it looks
harmless - it dilutes everything around it.

## Step 3: Report

```
## Inventory
<file> - <line count>, <"clean" | N uncommitted changes>

## Findings
### Contradictions
1. <file>:<line> says X; <file>:<line> says Y. Which should win?

### Duplicates across scope
### Stale references
### Misplaced scope
### Instructions Claude already follows
### Bloat
### Vagueness

## Not touched
<anything deliberately left alone, and why>
```

Print only the categories that have findings. Close with a single line naming
the empty ones - "Nothing found: duplicates, bloat, vagueness" - rather than
seven headings saying none. A small clean target should produce a short report.

Every finding carries its own proposed edit - cut, move, or rewrite, with the
replacement shown - so there is no second section restating them. Group by file
when there are more than a handful; a flat list of thirty items is harder to act
on than five files with six items each.

## Step 4: Apply

Only for the items the user approved. Silence is not approval, and approving the
report in outline is not approving every edit in it - confirm per file. If they
asked for a report only, stop here.

Apply one file at a time, showing each diff. Preserve formatting and surrounding
content: this is a pruning pass, not a rewrite, and a user who finds their whole
file reflowed will not run this skill again.

When an edit moves an instruction between scopes, make the removal and the addition
in the same pass, so no window exists where the rule is in neither file.

Finish by suggesting a commit for whichever repo was modified, with a message
naming what was pruned. Do not run it.

## Cautions

- **Managed policy files are org controlled.** Report them for visibility and do
  not edit them. On a work machine they may be centrally deployed, so a local
  change is both out of scope and likely to be reverted. Managed content can also
  arrive as a `claudeMd` key inside `managed-settings.json`, with no file to find.
- **A project `CLAUDE.md` is usually committed and shared.** Pruning it changes
  behavior for every teammate - say so, and prefer a PR to a direct edit.
- **`CLAUDE.local.md` is personal, and the docs tell you to gitignore it.** Nothing
  in it is safe to assume a teammate has.
- **Absence of evidence is not staleness.** A rule that has not come up recently
  may be a rare case a past session got badly wrong. When you cannot tell, ask
  instead of proposing.
- Never widen the pass into editing settings, hooks, or skills. If those look
  wrong, say so and leave them alone.
