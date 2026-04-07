<p align="center">
  <strong>SWEny Triage</strong>
</p>

<p align="center">
  Watch your observability provider for new alerts, investigate the root cause, file tickets, and (optionally) open fix PRs — all from a single GitHub Action.
</p>

---

This is the **triage preset** for [SWEny](https://github.com/swenyai/sweny). It runs the bundled `triage` workflow with all the observability and issue-tracker plumbing pre-wired, so you can drop it into a scheduled job and get autonomous SRE triage on your repo.

For the generic engine wrapper that runs any workflow YAML, see [`swenyai/sweny@v5`](https://github.com/swenyai/sweny). For agentic E2E browser tests, see [`swenyai/e2e@v1`](https://github.com/swenyai/e2e).

## Usage

```yaml
name: Triage

on:
  schedule:
    - cron: "*/15 * * * *"
  workflow_dispatch:

jobs:
  triage:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: swenyai/triage@v1
        with:
          claude-oauth-token: ${{ secrets.CLAUDE_CODE_OAUTH_TOKEN }}

          observability-provider: datadog
          dd-api-key: ${{ secrets.DD_API_KEY }}
          dd-app-key: ${{ secrets.DD_APP_KEY }}

          issue-tracker-provider: linear
          linear-api-key: ${{ secrets.LINEAR_API_KEY }}
          linear-team-id: ${{ vars.LINEAR_TEAM_ID }}
```

That's the minimum setup. The action picks up new errors from Datadog, classifies them, files novel ones in Linear (skipping duplicates), and posts a summary to the GitHub Actions job summary by default.

## What it does

```
┌─ Load rules & context (optional Linear docs / URLs)
│
├─ Gather: pull errors, stack traces, logs, recent commits, similar issues
│
├─ Investigate: identify root cause, severity, fix complexity, novelty
│
├─ ┌─ Novel + actionable → create issue → implement fix → open PR → notify
│  └─ Duplicate / low priority → +1 existing issue → notify
```

The full DAG lives in [`packages/core/src/workflows/triage.yml`](https://github.com/swenyai/sweny/blob/main/packages/core/src/workflows/triage.yml) in the SWEny repo.

## Inputs

### Auth (one required)

| Input | Description |
|---|---|
| `claude-oauth-token` | Claude Code OAuth token. Doesn't consume API credit. |
| `anthropic-api-key` | Anthropic API key for Claude. |

### Observability provider

| Input | Default | Description |
|---|---|---|
| `observability-provider` | `datadog` | One of: `datadog`, `sentry`, `betterstack`, `cloudwatch`, `splunk`, `elastic`, `newrelic`, `loki`, `prometheus`, `honeycomb`, `axiom`, `pagerduty`, `heroku`, `opsgenie`, `vercel`, `supabase`, `netlify`, `fly`, `render`, `file`. |
| `dd-api-key`, `dd-app-key`, `dd-site` | — / — / `datadoghq.com` | Datadog credentials. |
| `sentry-auth-token`, `sentry-org`, `sentry-project`, `sentry-base-url` | — / — / — / `https://sentry.io` | Sentry credentials. |
| `betterstack-api-token`, `betterstack-source-id`, `betterstack-table-name` | — | Better Stack credentials. |

For providers without first-class inputs (CloudWatch, Splunk, Vercel, etc.), pass credentials via the calling job's `env:` block — the SWEny CLI reads them directly from the environment. See the [SWEny providers reference](https://docs.sweny.ai/providers/) for env var names.

### Issue tracker

| Input | Default | Description |
|---|---|---|
| `issue-tracker-provider` | `github-issues` | One of: `github-issues`, `linear`, `jira`. |
| `linear-api-key` | — | Linear API key. |
| `linear-team-id` | — | Linear team UUID. |
| `linear-bug-label-id`, `linear-triage-label-id` | — | Linear label UUIDs. |
| `linear-state-backlog`, `linear-state-in-progress`, `linear-state-peer-review` | — | Linear workflow state UUIDs (used when transitioning issues during the implement phase). |
| `jira-base-url`, `jira-email`, `jira-api-token` | — | Jira credentials. |

### Source control

| Input | Default | Description |
|---|---|---|
| `source-control-provider` | `github` | One of: `github`, `gitlab`. |
| `github-token` | `${{ github.token }}` | Used for API access and PR creation. |
| `bot-token` | — | Optional elevated token for cross-repo dispatch or pushing to protected branches. |
| `gitlab-token`, `gitlab-project-id`, `gitlab-base-url` | — / — / `https://gitlab.com` | GitLab credentials. |

### Notification

| Input | Default | Description |
|---|---|---|
| `notification-provider` | `github-summary` | One of: `console`, `github-summary`, `slack`, `teams`, `discord`, `email`, `webhook`, `file`. |
| `notification-webhook-url` | — | Webhook URL for Slack, Teams, Discord, or generic webhook notifications. |

### Investigation knobs

| Input | Default | Description |
|---|---|---|
| `time-range` | `24h` | Time range to analyze (`1h`, `6h`, `24h`, `7d`, …). |
| `severity-focus` | `errors` | One of: `errors`, `warnings`, `all`. |
| `service-filter` | `*` | Glob to filter which services are in scope. |
| `investigation-depth` | `standard` | One of: `quick`, `standard`, `thorough`. |
| `max-investigate-turns` | `50` | Max Claude turns for the investigation phase. |
| `max-implement-turns` | `30` | Max Claude turns for the implementation phase. |

### PR / branch settings

| Input | Default | Description |
|---|---|---|
| `base-branch` | `main` | Target branch for PRs. |
| `pr-labels` | `agent,triage,needs-review` | Comma-separated labels applied to created PRs. |
| `review-mode` | `review` | `auto` (enable GitHub auto-merge when CI passes — automatically suppressed for high-risk changes such as migrations, auth, lockfiles, or >20 files) or `review` (open PR and wait). |
| `dry-run` | `false` | Analyze only — don't create issues or PRs. |
| `novelty-mode` | `true` | Only report novel issues; skip ones already tracked. |
| `additional-instructions` | — | Free-form extra guidance for the agent. |
| `service-map-path` | `.github/service-map.yml` | Path to your service ownership map. |

### Setup

| Input | Default | Description |
|---|---|---|
| `cli-version` | `latest` | Version of `@sweny-ai/core` to install. |
| `node-version` | `24` | Node.js version. |
| `working-directory` | `.` | Working directory to run from. |

## How auth & credentials flow

The composite action does two things with your inputs:

1. **Secrets** (API keys, tokens) are set as **environment variables** before invoking `sweny triage`. The CLI reads them directly — you don't see them on the command line, so they never end up in step logs.
2. **Behavior knobs** (provider names, time ranges, mode flags) are mapped to **CLI flags** via env-routed bash arrays — never inline-templated into the script, so caller input can't escape its quoting.

You can also pass anything else through the calling job's `env:` block — the SWEny CLI reads its full env, including any provider credentials this action doesn't expose as first-class inputs.

```yaml
- uses: swenyai/triage@v1
  with:
    claude-oauth-token: ${{ secrets.CLAUDE_CODE_OAUTH_TOKEN }}
    observability-provider: prometheus
  env:
    PROMETHEUS_URL: https://prometheus.internal:9090
    PROMETHEUS_TOKEN: ${{ secrets.PROMETHEUS_TOKEN }}
```

## See also

- **[`swenyai/sweny@v5`](https://github.com/swenyai/sweny)** — generic engine wrapper. Run any SWEny workflow YAML on CI.
- **[`swenyai/e2e@v1`](https://github.com/swenyai/e2e)** — agentic end-to-end browser tests.
- **[docs.sweny.ai](https://docs.sweny.ai)** — full documentation.
- **[marketplace.sweny.ai](https://marketplace.sweny.ai)** — community workflow catalog.

## License

[MIT](LICENSE)
