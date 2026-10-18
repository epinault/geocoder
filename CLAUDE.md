# CLAUDE.md

Claude Code guidance for this repository. The canonical agent instructions live in AGENTS.md.

@AGENTS.md

## Plugin Setup

```bash
claude plugin install elixir@claude-code-elixir
claude plugin install superpowers@claude-plugins-official
claude plugin install code-review@claude-plugins-official
```

## Claude-specific notes

- This repo is the epinault fork of CyrusNuevoDia/geocoder. The fork-then-upstream PR chain in AGENTS.md is mandatory: open the upstream PR as soon as the user says the fork PR is merged, without waiting to be asked. Always pass `--repo` to `gh`.
- Verification before claiming done: `mix compile --warnings-as-errors`, `mix test`, `mix format --check-formatted`.
- Useful skills: `superpowers:test-driven-development` for provider changes, `superpowers:verification-before-completion` before reporting success, `code-review:code-review` before opening a PR, `update-dependencies` for dependency bumps (mind the held httpoison/hackney note in AGENTS.md Gotchas).
