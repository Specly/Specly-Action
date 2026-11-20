# Specly GitHub Action

A GitHub Action that turns a vague issue into a spec ready for a coding agent. Add the `specly` label to an issue (or comment `/specly`) and Specly reads the issue, checks out your repo, and posts **one** collapsed comment with a spec grounded in your code: relevant files, constraints, acceptance criteria and files not to touch. Copy it into Claude Code, Cursor, Codex, Copilot or Gemini.

If the issue is too vague, Specly posts a short checklist of questions instead. Once they're answered, it runs again and replaces the questions with the spec.

Specly is a hosted service. The action uses your **Specly access token**. You don't need a model provider key.

## Setup

1. Create an access token in your Specly account.
2. In your repo, add it as a secret named `SPECLY_TOKEN` (Settings > Secrets and variables > Actions > New repository secret).
3. Create a label named `specly` (or set `trigger-label` to the label you want).
4. Copy [`examples/specly-issue.yml`](examples/specly-issue.yml) to `.github/workflows/specly-issue.yml`:

```yaml
name: Specly issue to spec

on:
  issues:
    types: [labeled, edited]
  issue_comment:
    types: [created, edited]

permissions:
  contents: read
  issues: write

concurrency:
  group: specly-issue-${{ github.event.issue.number }}
  cancel-in-progress: true

jobs:
  specly:
    if: >-
      ${{
        !github.event.issue.pull_request &&
        github.event.sender.type != 'Bot' &&
        (
          github.event_name == 'issues' ||
          (github.event.action == 'created' && startsWith(github.event.comment.body, '/specly')) ||
          (github.event.action == 'edited' && contains(github.event.comment.body, '<!-- specly:issue-to-spec -->'))
        )
      }}
    runs-on: ubuntu-latest
    timeout-minutes: 10
    steps:
      - uses: actions/checkout@v7
        with:
          persist-credentials: false
      - uses: Specly/Specly-Action@main
        with:
          specly-token: ${{ secrets.SPECLY_TOKEN }}
          target: claude-code
```

Pin `@main` to a release tag or commit SHA once there is one.

## How it runs

| Trigger | Who | What happens |
| --- | --- | --- |
| The `specly` label is added | Anyone who can label (triage access) | Builds the spec, or asks questions |
| `/specly` comment | Owners, members and collaborators on any issue. The issue's author once the issue has the label or a Specly comment. | Same, using any answers in the comment |
| `/specly skip` comment | Same as above | Builds the spec now and lists the open questions inside it |
| The issue title or body is edited | Only on issues that have the label or a Specly comment | Rebuilds the spec |
| A suggestion is ticked in Specly's questions comment | People with write access (GitHub only lets them edit the comment) | Rebuilds with the ticked answers |

**Answering questions.** Tick a suggestion in Specly's comment, or reply:

```
/specly
1. The login page
2. Tests pass
```

With one question you can just write the answer after `/specly`. Answers are saved in Specly's comment and reused on later runs.

**One comment.** Specly finds its earlier comment by a hidden marker and only trusts that marker on comments written by a bot account. It then updates that comment and never adds a second one. If a later run fails, a good spec is kept and the problem goes to the workflow log.

## Inputs

| Input | Default | Description |
| --- | --- | --- |
| `specly-token` | (none) | Your Specly access token. Pass `${{ secrets.SPECLY_TOKEN }}`. If it's missing, Specly comments on the issue explaining how to add it and the run fails. |
| `target` | `.specly.json`, else `claude-code` | Tool the spec is formatted for: `claude-code`, `cursor`, `codex`, `vscode`, `gemini` or `generic`. |
| `model` | `.specly.json` | The model your coding agent runs, recorded in the spec. |
| `trigger-label` | `specly` | The label that runs Specly. |
| `command` | `/specly` | The comment command. |
| `allowed-associations` | `OWNER,MEMBER,COLLABORATOR` | Who may run the command on any issue. |
| `path` | `.` | Folder to scan, relative to the checkout. |
| `github-token` | `${{ github.token }}` | Token for reading and writing the comment. Needs `issues: write`. |

Outputs: `status` (`skipped`, `refused`, `spec`, `questions`, `missing-token` or `failed`) and `comment-url`.

## Security

- **The token stays secret.** It's masked in logs (`setSecret`) and removed from anything Specly posts or logs.
- **Least privilege.** The workflow needs `contents: read` and `issues: write` and nothing else.
- **No pull requests.** The action handles only `issues` and `issue_comment` events, and it skips comments on pull requests. It never runs on `pull_request` or `pull_request_target`, so code from a fork's PR never runs next to your secret. The checkout is your default branch, with `persist-credentials: false`.
- **No spoofing.** People without repo access can't trigger runs on issues a maintainer hasn't opted in, and the issue author can only answer once an issue is opted in. A planted marker comment is ignored.
- **What leaves the runner.** The issue text and a small repo-context bundle go to the hosted Specly service. Files your `.gitignore` or `.specly.json` `ignore` exclude, and files that look like secrets, are never read.

## About this repository

This repository holds the published action: `action.yml` and the bundled `dist/main/index.mjs`. It is built from Specly's own source and updated with each release, so please open issues here rather than pull requests against `dist/`.
