---
name: cleanup
description: Audit and prune Claude Code's persistent memory — CLAUDE.md files and .claude/rules/ at user and project scope. Use when memory has grown long or stale, when instructions contradict each other, when Claude seems to ignore a rule that is written down, or when someone asks what is actually loaded into context and whether it is still earning its place.
license: MIT
---

# Memory cleanup

Persistent memory decays. Rules get added after a bad session and never
removed, two files end up disagreeing, and instructions pile up until the ones
that matter are buried. This skill inventories what is loaded, finds what is no
longer pulling its weight, and proposes cuts.

**Never edit or delete a memory file without showing the user the exact change
first.** These files are the user's accumulated instructions to Claude. Losing
one silently is worse than leaving ten stale lines in place.

## Step 1 — inventory

Run `/context` first. It reports what actually loaded this session, which is
the only reliable ground truth; a file can exist and not be loaded. Then
`/memory` lists memory file locations across scopes and opens them.

Then check each location on disk. Loaded in full at launch:

| Scope | Location |
|---|---|
| Managed policy | macOS `/Library/Application Support/ClaudeCode/CLAUDE.md` · Linux/WSL `/etc/claude-code/CLAUDE.md` · Windows `C:\Program Files\ClaudeCode\CLAUDE.md` |
| User | `~/.claude/CLAUDE.md`, `~/.claude/rules/**/*.md` |
| Project | `./CLAUDE.md` or `./.claude/CLAUDE.md`, `./.claude/rules/**/*.md` |
| Local | `./CLAUDE.local.md` |

Two things people forget to look for:

- **Parent directories.** Claude walks up from the working directory to the
  filesystem root, loading every `CLAUDE.md` and `CLAUDE.local.md` on the way.
  A stale file two levels up is loaded and invisible. Check the whole chain.
- **`@` imports.** A `CLAUDE.md` can pull in other files with `@path/to/file`,
  and those files can import further, up to four hops. Follow the chain — the
  real size of a memory file includes everything it imports. Paths resolve
  relative to the importing file. A path inside backticks or a code fence is
  not an import.

Rules files in subdirectories are discovered recursively. Rules with a `paths:`
frontmatter key load only when Claude reads a matching file; rules without one
load at launch, every session.

Report the inventory as a table with line counts before proposing anything.

## Step 2 — find the problems

### Contradictions

The highest-value finding. CLAUDE.md files are **concatenated into context,
not overridden** — a project file does not replace a user file, it is simply
read after it. So two files that disagree do not resolve; per the docs, "if two
rules contradict each other, Claude may pick one arbitrarily."

Look for direct conflicts and for near-conflicts: the same topic covered at two
scopes with different detail, one file saying "always" and another carving out
an exception, a rule and its replacement both still present. Report both
locations and ask which wins.

### Staleness

- Instructions naming a file, directory, command, script, or tool that no
  longer exists. **Verify each one against the filesystem before flagging it.**
- Rules about a framework, service, or workflow the project has migrated off.
- Fixes for a bug that was fixed, or workarounds for a version long upgraded.
- Anything the code now enforces on its own — a lint rule, a formatter config,
  a pre-commit hook, a type checker. If CI catches it, memory need not.

### Bloat

Target is under 200 lines per CLAUDE.md; longer files consume context and
reduce adherence. When a file is over, the fix is usually relocation, not
deletion:

- Instructions that only apply to part of the tree → a rule in `.claude/rules/`
  with a `paths:` frontmatter key, so it loads only for matching files.
- A multi-step procedure → a skill. Memory is for standing facts and
  constraints, not runbooks.
- Content duplicating the README, a config file, or the type system → cut it
  and reference the source.

### Vagueness

An instruction that cannot be verified will not be followed reliably. "Format
code properly" and "test your changes" are noise; "use 2-space indentation" and
"run `npm test` before committing" are not. Propose the concrete rewrite or
propose the cut — do not leave a vague line untouched just because it is
harmless. It is not harmless; it dilutes everything around it.

### Misplaced scope

- Personal preference sitting in a project `CLAUDE.md`, where it is imposed on
  the whole team → move to `~/.claude/CLAUDE.md` or `./CLAUDE.local.md`.
- A project-specific fact in `~/.claude/CLAUDE.md`, where it leaks into every
  unrelated repo → move to the project.

## Step 3 — propose

Group findings by file. For each proposed change give the exact lines, the
category, and one sentence of reasoning. Order by scope, most disruptive first.

Sort every finding into: **cut**, **move** (say where), **rewrite** (show the
replacement), or **keep** (say why it looked stale but is not).

Then stop and let the user decide. Do not apply the pass wholesale because it
was approved in outline — confirm per file.

## Cautions

- **Managed policy files are org-controlled.** Report them for visibility, and
  do not edit them. On a work machine they may be centrally deployed, and a
  local change is both out of scope and likely to be reverted.
- **A project `CLAUDE.md` is usually committed and shared.** Pruning it changes
  behavior for every teammate. Flag that explicitly and prefer a PR to a direct
  edit.
- **`CLAUDE.local.md` is personal and typically gitignored.** Nothing there is
  safe to assume a teammate has.
- **Absence of evidence is not staleness.** A rule that has not come up
  recently is not thereby obsolete — it may just be a rare case that a past
  session got badly wrong. When you cannot tell, ask instead of proposing.
- Never widen the pass into editing settings, hooks, or skills. If those look
  wrong, say so and leave them alone.
