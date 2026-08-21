# droptabletools

A free, open marketplace of Claude skills. Add it once and you keep getting new
and updated skills as they land — no account, no payment, no gating.

## Install

In Claude Code, add the marketplace:

```
/plugin marketplace add droptablestar/droptabletools
```

Then install a plugin from it:

```
/plugin install dt-agent@droptabletools
```

The same plugins work in the Claude web and desktop apps once installed.

## What's here

| Plugin | Skills | What it does |
|---|---|---|
| `dt-agent` | `cleanup` | Audits Claude Code's persistent memory — `CLAUDE.md` files and `.claude/rules/` at user and project scope — and proposes cuts. Finds contradictions between scopes, instructions pointing at things that no longer exist, and rules that belong in a path-scoped rule or a skill instead. |
| `dt-agent` | `unstick` | Breaks out of a debugging session that has stopped converging — fix-and-recheck rounds that keep finding new errors instead of shrinking the list. Writes an oracle-driven ledger, rules out hypotheses for good, and knows when to revert and escalate instead of retrying. |

## Using a skill

Skills load on their own when the work calls for them. You can also ask for one
by name:

```
run cleanup on this repo's memory
```

`cleanup` proposes changes and stops — it never edits a memory file without
showing you the exact change first.

## Contributing

Open an issue or a PR. A skill is a directory with a `SKILL.md` inside a
plugin's `skills/` directory. Keep frontmatter to the Agent Skills spec fields
(`name`, `description`, `license`, `compatibility`, `allowed-tools`,
`metadata`) so skills stay portable across Claude Code and the Claude apps.

## License

MIT.
