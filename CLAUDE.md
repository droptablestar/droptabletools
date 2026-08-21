# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repo is

A public, free Claude Code plugin marketplace (`droptabletools`, owned by
`droptablestar`). It has no build, no tests, no runtime — it is JSON and
Markdown config consumed by the Claude Code plugin system. See
`README.md` for install instructions and `NOTES-handoff.md` (gitignored,
local only) for the research and design decisions behind the layout.

## Structure

Three nested layers — do not conflate them:

- **Marketplace** — `.claude-plugin/marketplace.json` at repo root. Lists
  plugins by `name` + `source` (a relative path in this repo).
- **Plugin** — a directory (e.g. `dt-agent/`) with its own
  `.claude-plugin/plugin.json`, bundling one or more skills.
- **Skill** — `<plugin>/skills/<name>/SKILL.md`, one capability.

A marketplace entry points at a plugin directory, never directly at a skill.

## Hard constraint: skill frontmatter

Every `SKILL.md` frontmatter is restricted to the Agent Skills spec fields
only: `name`, `description`, `license`, `compatibility`, `allowed-tools`,
`metadata`. No Claude Code-specific fields (`disable-model-invocation`,
`user-invocable`, `context`, `agent`, `paths`, `argument-hint`, `arguments`,
etc.) — adding one breaks portability to the Claude web/desktop apps, which
validate against the plain spec. If a CLI-only field ever seems necessary,
raise it rather than adding it silently.

## Validating changes

- `claude plugin validate ./<plugin-dir>` — validate a plugin directory
  before publishing.
- Add the marketplace from a local path to confirm `marketplace.json`
  resolves before pushing.

## Conventions

- New skills should be small but genuinely useful, not placeholders — see
  `dt-agent/skills/cleanup/SKILL.md` for the target shape and depth.
