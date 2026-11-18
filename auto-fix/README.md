# Errilo auto-fix

A GitHub Actions workflow that fixes production errors on its own. When [Errilo](https://errilo.com) reports a new issue or a regression, Claude Code reads it through the Errilo MCP server, reproduces it with a failing test and fixes it. The workflow then opens a pull request and marks the issue "resolved in the next release" in Errilo.

```
Errilo alert ──repository_dispatch──▶ guard ──▶ fix (Claude Code) ──▶ pull-request ──▶ Errilo issue
                                       │          read-only token       pushes errilo/fix-<id>   comment +
                                       │          commits locally       opens the PR             resolved in next release
                                       └─ skips: test alert, spike, branch exists, daily cap,
                                          issue not unresolved, dependency-only stack trace
```

If the error comes back once the fix is deployed, Errilo reopens the issue as a regression and alerts again, which starts a new run.

There is no code of ours to install: one workflow file that uses [`anthropics/claude-code-action`](https://github.com/anthropics/claude-code-action), maintained by Anthropic, and the [Errilo MCP server](https://app.errilo.com/mcp). Copy it, read it, change it.

Step-by-step guide, with a sequence diagram of what Errilo, GitHub and Claude Code send each other: [Auto-fix with Claude Code and GitHub Actions](https://www.errilo.com/en/auto-fix/) ([en français](https://www.errilo.com/correction-automatique/)).

## Setup

You need an Errilo project that receives errors from your app, admin access to the GitHub repository, and either an [Anthropic API key](https://console.anthropic.com/settings/keys) or a Claude subscription (Pro or Max).

### 1. Send releases with their commit

The agent fixes the code at the version that crashed. Errilo knows that version when your SDK sends a `release` and the release has a commit:

- a release that is a commit SHA (`release: process.env.GIT_SHA`) is enough;
- otherwise, run `sentry-cli releases set-commits <release> --auto` against Errilo when you deploy.

Without a release, the agent works from the default branch, and Errilo cannot mark the issue "resolved in the next release" (the pull request is still opened).

### 2. Create an Errilo token for the project

In Errilo, **Settings → API tokens → New token**: limited to the project of this repository, scopes `issues:read` and `issues:write`.

The token only sees this project. The workflow reads the issue with it, the agent reads the stack trace and events, and the workflow writes the comment and the status.

### 3. Add the workflow

Copy [`errilo-auto-fix.yml`](errilo-auto-fix.yml) to `.github/workflows/errilo-auto-fix.yml` on your **default branch**: GitHub only starts `repository_dispatch` workflows from there.

Then add your project's setup to the `fix` job (the commented block after the checkout), so that the agent can install dependencies and run your tests:

```yaml
      - uses: actions/setup-node@v6
        with:
          node-version: 24
      - run: npm ci
```

### 4. Secrets and variables

In the repository, **Settings → Secrets and variables → Actions**:

| Name                          | Kind     | Value                                                                                             |
| ----------------------------- | -------- | ------------------------------------------------------------------------------------------------- |
| `ANTHROPIC_API_KEY`           | Secret   | Your Anthropic API key. Or, instead:                                                              |
| `CLAUDE_CODE_OAUTH_TOKEN`     | Secret   | A Claude subscription token, created with `claude setup-token`.                                   |
| `ERRILO_TOKEN`                | Secret   | The token from step 2.                                                                            |
| `ERRILO_ORG`                  | Variable | Your organization slug: the first path segment of your Errilo URLs (`app.errilo.com/<slug>/...`). |
| `ERRILO_MAX_PRS_PER_DAY`      | Variable | Optional. Cap on pull requests opened in 24 hours, `3` by default.                                |
| `ERRILO_URL`                  | Variable | Optional. `https://app.errilo.com` by default.                                                    |
| `ERRILO_AUTOFIX_GITHUB_TOKEN` | Secret   | Recommended. See "Pull request token" below.                                                      |

Also turn on **Settings → General → Automatically delete head branches**. The workflow skips an issue while its `errilo/fix-<id>` branch exists, so a branch left after a merge would block the next attempt on a regression.

#### Pull request token

By default, the pull request job uses the workflow's `GITHUB_TOKEN`. It has two limits:

- GitHub refuses to let it open pull requests ("GitHub Actions is not permitted to create or approve pull requests") unless **Settings → Actions → General → Allow GitHub Actions to create and approve pull requests** is on, in the repository and, for an organization, in the organization's settings first;
- GitHub does not start workflows for a pull request it opened, so your CI does not run on the fix.

Add an `ERRILO_AUTOFIX_GITHUB_TOKEN` secret to avoid both: a fine-grained token on this repository only, with **Contents** and **Pull requests** in read and write, or a GitHub App token. Only the pull request job receives it, never the agent.

### 5. Connect the Errilo alert

Create a fine-grained personal access token on GitHub: **only this repository**, permission **Contents: read and write** (required by GitHub to send a `repository_dispatch`), short expiry. For a repository of an organization, choose the organization as **Resource owner**, and check that an organization admin approved the token if the organization requires it. Otherwise GitHub answers 404 and Errilo shows "The service responded 404".

In Errilo, open the project, then **Alerts → Create a rule**, channel **GitHub**: the repository (`owner/name`), the token, and the triggers **New issue** and **Regression**. Spikes are ignored by the workflow.

Click **Send a test alert**: a run appears in the Actions tab and stops at once with "test alert received". The wiring works.

### 6. Try it on one issue

**Actions → Errilo auto-fix → Run workflow**, with the id of an issue (the number at the end of its Errilo URL). The run opens a pull request named `fix: Errilo issue <id>`, and the issue shows a comment with the link and the status "resolved in the next release".

## Safeguards

- **Pull request only.** The agent runs with a read-only GitHub token: it cannot push or open a pull request. It commits on a local branch; the workflow exports the commits as a patch, and a separate job without agent pushes them to `errilo/fix-<id>` (never with `--force`, never to the default branch) and opens the pull request. Protect your default branch as well: the workflow does not depend on it, your team should.
- **No change to workflows.** A fix that touches `.github/` is refused.
- **Minimal permissions.** None at the workflow level; `contents: read` for the agent; `contents: write` and `pull-requests: write` only for the job that opens the pull request.
- **One run per issue.** Errilo delivers alerts "at least once", so the same issue can arrive twice. Runs are grouped by issue id (`concurrency`), and a run stops if the issue's branch already exists: a duplicate never opens a second pull request.
- **Daily cap.** At most `ERRILO_MAX_PRS_PER_DAY` pull requests (3 by default) in 24 hours, checked before the agent starts and again before the pull request is opened.
- **Untrusted payload.** The alert's `client_payload` contains text from production events (the issue title, the release). The workflow never writes it into a script with `${{ }}`: it reads the payload through environment variables, and only uses the issue id, after checking that it is a number. A malicious title cannot run a command on the runner.
- **Untrusted event content.** Error messages, breadcrumbs and tags come from your users, possibly from an attacker. The Errilo MCP server marks them as untrusted data, and the `fix_issue` method tells the agent never to follow instructions found in them. The read-only token and the review of the pull request limit what an injection could achieve.
- **Only your code.** No fix is attempted when the stack trace only has frames from `node_modules` or Node.js internals, and the agent reports a dependency bug instead of patching it.
- **Only real issues.** Test alerts, spikes, and issues that are no longer unresolved (resolved, ignored) are skipped.
- **The agent can say no.** If it cannot reproduce the error with a failing test, or the fix is uncertain, no pull request is opened; the reason is in the run summary.

## Cost

Errilo charges nothing for the MCP server or for this workflow, on every plan. The agent runs on your Anthropic account. With an API key, each run is billed by Anthropic according to the tokens used: reading the issue, exploring the code, running the tests; follow it in the [Anthropic Console](https://console.anthropic.com/usage). With a subscription token, runs count toward your plan's usage limits. For scale: on a small demo bug (an unknown discount code that crashes a cart function), a run took 7 agent turns, 30 seconds, and the action reported $0.08. A real code base costs more: the agent reads more code and runs a longer test suite.

Three settings bound the cost: the daily cap on pull requests, `--max-turns` in the `claude_args` of the `fix` job (80 by default), and `timeout-minutes` (45). Runs also use GitHub Actions minutes.

## Limits

- **Review every pull request.** The agent writes a test that reproduces the error, but a test that passes does not prove the fix is right.
- **Run your CI on the pull requests.** Without `ERRILO_AUTOFIX_GITHUB_TOKEN` (see "Pull request token"), close and reopen the pull request to start the checks.
- **The agent needs a working test setup.** Without the setup step, it cannot reproduce the error and stops.
- **Bots cannot trigger the agent.** `claude-code-action` only runs for a human GitHub user: the dispatch token must belong to a person, not a GitHub App.
- **One issue per run.** An alert that groups several issues sends one dispatch per issue (10 at most); each one starts its own run.
- **GitHub.com only.** Errilo sends `repository_dispatch` to `api.github.com`; GitHub Enterprise Server is not supported.

## Change the agent

The `fix` job runs the `fix_issue` prompt of the Errilo MCP server (`/mcp__errilo__fix_issue <id>`), plus CI instructions written by the "Configure the agent" step. You can:

- choose the model with `--model` in `claude_args`;
- add tools the agent may use, with `--allowedTools`;
- replace Claude Code with another agent that supports MCP: give it the same MCP configuration, the same instructions, and keep the read-only token. The `guard` and `pull-request` jobs do not depend on the agent.
