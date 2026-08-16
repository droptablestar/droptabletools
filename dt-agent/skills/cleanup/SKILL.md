---
name: cleanup
description: >-
  Audit and prune Claude Code's persistent memory - CLAUDE.md and .claude/rules/ at user
  and project scope. Reports contradictory, duplicated, stale, misplaced, and redundant
  instructions, and applies only the edits the user approves. Use for any ask to clean up,
  trim, audit, prune, or reorganize memory, CLAUDE.md, rules, or agent context, including
  vaguer ones like "my CLAUDE.md has gotten out of hand" or "why does Claude keep ignoring
  my instructions".
license: MIT
allowed-tools: Read, Grep, Glob, Edit, Write, SlashCommand, Bash(git status:*), Bash(git diff:*), Bash(ls:*)
---

# Memory cleanup

Memory decays: instructions accumulate faster than they are removed, the same
rule lands at two scopes, and files keep describing a codebase that has moved on.
A stale instruction is worse than none - it is followed confidently.

Report first, edit second. The user knows why a strange-looking rule is there;
you do not. Audit both scopes unless the user names one, and change nothing until
they approve specific edits.

## Step 1: Inventory

Run `/context` first: it shows what actually loaded this session, the only ground
truth - a file can exist on disk and never load. Then `/memory` for file
locations across scopes. Check each location on disk. Loaded in full at launch:

| Scope | Location |
|---|---|
| Managed policy | macOS `/Library/Application Support/ClaudeCode/CLAUDE.md`; Linux and WSL `/etc/claude-code/CLAUDE.md`; Windows `C:\Program Files\ClaudeCode\CLAUDE.md` |
| User | `~/.claude/CLAUDE.md`, `~/.claude/rules/**/*.md` |
| Project | `./CLAUDE.md` or `./.claude/CLAUDE.md`, `./.claude/rules/**/*.md` |
| Local | `./CLAUDE.local.md` |

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

Run `git status` in the project and again in `~/.claude`, which is often a
dotfiles repo. Report uncommitted changes before touching anything - a clean tree
is the whole safety net.

Report the inventory with line counts before analyzing, so the user can flag
anything missed.

## Step 2: Analyze

Read each file completely. Look for these, in rough order of value.

**Contradictions.** Two instructions that cannot both be followed, such as "always
use pnpm" at user scope against "use npm, we pin the lockfile" at project scope.
Highest value: the user has likely already noticed the inconsistent behavior
without knowing the cause.

CLAUDE.md files are concatenated, not overridden - a project file is read after
the user file, not instead of it. Nothing resolves a conflict, so per the docs,
"if two rules contradict each other, Claude may pick one arbitrarily." Never pick
for the user: present both with file and line, and ask which survives.

**Duplicates across scope.** The same rule stated at both user and project scope.
The project copy usually wins because it travels with the repo - say why rather
than assume.

**Stale references.** Paths, filenames, scripts, commands, or services that no
longer exist. Verify each against the filesystem - `Glob` and `Grep` are cheap, a
wrong deletion is not - and mark anything unverifiable as unverified rather than
dropping it. Also flag rules for a framework the project has left, workarounds for
a version long upgraded, and anything the toolchain now enforces: linter,
formatter, type checker, pre-commit hook. If CI catches it, memory need not.

**Misplaced scope.** A personal preference in a project file, where it is imposed
on everyone who clones the repo, or a project-specific rule in the user file, where
it leaks into unrelated work. This is the most common source of "why did Claude do
that in this other repo".

**Instructions Claude already follows.** Restatements of language or framework
convention, or of default behavior: context every session, nothing bought. Flag,
do not delete.

**Bloat.** Target is under 200 lines per CLAUDE.md; longer files consume context
and reduce adherence. Cut narrative and history no instruction depends on. Prefer
relocation over deletion:

- Applies to only part of the tree -> a rule in `.claude/rules/` with a `paths:`
  key, so it loads only for matching files.
- A multi-step procedure -> a skill. Memory is for standing facts and constraints,
  not runbooks.
- Duplicates a README, a config file, or the type system -> cut it and reference
  the source.

Show the condensed version so the user can judge whether meaning survived.

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

## Suggested edits
<per file: what would be removed, moved, or condensed>

## Not touched
<anything deliberately left alone, and why>
```

Group findings by file when there are more than a handful. A flat list of thirty
items is harder to act on than five files with six items each.

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
  change is both out of scope and likely to be reverted.
- **A project `CLAUDE.md` is usually committed and shared.** Pruning it changes
  behavior for every teammate - say so, and prefer a PR to a direct edit.
- **`CLAUDE.local.md` is personal and typically gitignored.** Nothing in it is safe
  to assume a teammate has.
- **Absence of evidence is not staleness.** A rule that has not come up recently
  may be a rare case a past session got badly wrong. When you cannot tell, ask
  instead of proposing.
- Never widen the pass into editing settings, hooks, or skills. If those look
  wrong, say so and leave them alone.
