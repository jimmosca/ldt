# Issue tracker: GitHub

Issues and PRDs for this repo live as GitHub issues in `jimmosca/ldt`. Use the `gh` CLI for all operations.

## The RPIV loop's intake/output adapter

This is where the RPIV loop (see [`CLAUDE.md`](../../CLAUDE.md)) touches GitHub — it bookends the loop.

- **Intake (loop opens).** Pick up issues labelled `ready-for-agent` (fully specified, AFK-ready — see [`triage-labels.md`](triage-labels.md)); fetch with `gh issue view <n> --comments`. As part of Research, identify the hard invariant(s) the change touches.
- **Close-out (loop exits).** Post the Verify evidence via `gh issue comment <n>`, then relabel/close. Evidence differs by mode: a **builder** change → the offline test/lint results; an **operator** run (`ldt pull`) → the `reports/<run_id>.md` summary + the manifest masking proof (applied-per-column claim + adapter `generated_sql`).
- **Label state-machine.** `needs-triage` → (`needs-info` if blocked on the reporter) → `ready-for-agent` (agent picks up) → `ready-for-human` (offline-green but needs a human for the gated-live tier or a security-surface review) → close. `wontfix` is terminal.
- **ADR-conflict escape hatch.** If the change would contradict an Accepted ADR (new masking semantics, a port-shape or manifest-core change, treating PRs as intake), do **not** close — propose a new ADR on the issue instead. See [`domain.md`](domain.md) ("Flag ADR conflicts").

## Conventions

- **Create an issue**: `gh issue create --title "..." --body "..."`. Use a heredoc for multi-line bodies.
- **Read an issue**: `gh issue view <number> --comments`, filtering comments by `jq` and also fetching labels.
- **List issues**: `gh issue list --state open --json number,title,body,labels,comments --jq '[.[] | {number, title, body, labels: [.labels[].name], comments: [.comments[].body]}]'` with appropriate `--label` and `--state` filters.
- **Comment on an issue**: `gh issue comment <number> --body "..."`
- **Apply / remove labels**: `gh issue edit <number> --add-label "..."` / `--remove-label "..."`
- **Close**: `gh issue close <number> --comment "..."`

Infer the repo from `git remote -v` — `gh` does this automatically when run inside a clone.

## Pull requests as a triage surface

**PRs as a request surface: no.** _(Set to `yes` if this repo treats external PRs as feature requests; `/triage` reads this flag.)_

When set to `yes`, PRs run through the same labels and states as issues, using the `gh pr` equivalents:

- **Read a PR**: `gh pr view <number> --comments` and `gh pr diff <number>` for the diff.
- **List external PRs for triage**: `gh pr list --state open --json number,title,body,labels,author,authorAssociation,comments` then keep only `authorAssociation` of `CONTRIBUTOR`, `FIRST_TIME_CONTRIBUTOR`, or `NONE` (drop `OWNER`/`MEMBER`/`COLLABORATOR`).
- **Comment / label / close**: `gh pr comment`, `gh pr edit --add-label`/`--remove-label`, `gh pr close`.

GitHub shares one number space across issues and PRs, so a bare `#42` may be either — resolve with `gh pr view 42` and fall back to `gh issue view 42`.

## When a skill says "publish to the issue tracker"

Create a GitHub issue.

## When a skill says "fetch the relevant ticket"

Run `gh issue view <number> --comments`.
