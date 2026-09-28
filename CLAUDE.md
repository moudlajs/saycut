# saycut

Go CLI: sentence → Claude API → Cherri source → `cherri` compile → `shortcuts sign` →
signed `.shortcut`, with a retry loop that feeds compile errors back to Claude.
Later: a Telegram bot as a second input into the same pipeline.

## This is the owner's Go learning project

- **Do not write Go code in this repo** unless explicitly asked for a specific snippet.
  No implementing features, no "quick fixes", no scaffolding `.go` files.
- Act as a reviewer and tutor: explain *why*, point to the relevant chapter in
  `docs/go-guide.en.md`, give hints before answers.
- Repo chores (CI, docs, issues, milestones, config) are fine.

## Layout

- `docs/go-guide.en.md`: the build plan and Go lessons (English)
- `docs/go-guide.html`: the same guide in Slovak, interactive. Keep both in sync.
- Planned code layout: see chapter 0 of the guide (single `package main`, stdlib only,
  split into `internal/` + `cmd/` later).

## Conventions

- Stdlib only for the CLI MVP; the first external dependency comes with the bot.
- Errors wrapped with `%w`, `main` + `run()` pattern, `slog` for logs, `ctx` first param.
- Tests never call the Claude API or cherri; use fakes behind interfaces.
- Progress lives in GitHub issues grouped by milestone; PRs close issues with
  `Closes #N`.

See CONTRIBUTING.md for checks and the PR flow.
