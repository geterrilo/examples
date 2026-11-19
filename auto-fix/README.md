# Errilo auto-fix

A GitHub workflow that fixes production errors on its own. When [Errilo](https://errilo.com) reports a new issue or a regression, Claude Code reads it through the Errilo MCP server, reproduces it with a failing test and fixes it. The workflow then opens a pull request and marks the issue "resolved in the next release" in Errilo.

The work is done by the [Errilo Auto-fix action](https://github.com/geterrilo/autofix) (`geterrilo/autofix@v1`) and by [`anthropics/claude-code-action`](https://github.com/anthropics/claude-code-action). This workflow only wires them in three jobs, so that the agent never holds a token that can write to your repository.

## Use it

1. Copy [`errilo-auto-fix.yml`](errilo-auto-fix.yml) to `.github/workflows/errilo-auto-fix.yml` on your **default branch**.
2. Replace the setup steps of the `fix` job (`setup-node`, `npm ci`) with your project's, so that the agent can run your tests.
3. Add the secrets `ANTHROPIC_API_KEY` and `ERRILO_TOKEN`, the variable `ERRILO_ORG`, and preferably `ERRILO_AUTOFIX_GITHUB_TOKEN`.
4. Connect the Errilo alert to the repository (channel **GitHub**, triggers **New issue** and **Regression**).

Step-by-step setup, safeguards, cost and limits: [geterrilo/autofix](https://github.com/geterrilo/autofix#setup). Guide with a sequence diagram: [Auto-fix with Claude Code and GitHub Actions](https://www.errilo.com/en/auto-fix/) ([en français](https://www.errilo.com/correction-automatique/)).
