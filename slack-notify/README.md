# Slack Deploy Notify

Sends an attractive Slack notification (Block Kit) about a deploy or
workflow run. Generic for any project/environment: reports status, repo,
branch, commit, author, the first line of the commit message, and a link
back to the GitHub Actions run. Supports arbitrary extra fields (image tag,
stack name, domain, etc.).

## What it does

1. Maps `status` to an emoji, default headline and Slack attachment color
   (`success` → ✅ green, `cancelled` → ⚪ grey, anything else → ❌ red).
2. Builds a title like `✅ Deploy succeeded · my-repo · PROD`.
3. Takes the first line of the commit message (falls back to
   `(manual run: <event-name>)` for manually-triggered runs with no commit,
   e.g. `workflow_dispatch`).
4. Assembles a Slack Block Kit payload: header, a fields section
   (Environment / Repo / Branch / Commit / Author + any `extra-fields`),
   the commit message, and a context line with the raw status and a link
   to the run.
5. POSTs it to your Slack Incoming Webhook with `curl`. A failed POST only
   emits a `::warning::` — it will not fail your workflow.

## Usage

### Minimal — report a job's own result

```yaml
jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - run: ./deploy.sh

  notify:
    needs: deploy
    if: always()
    runs-on: ubuntu-latest
    steps:
      - uses: alexandergv2117/actions/slack-notify@main
        with:
          webhook-url: ${{ secrets.SLACK_WEBHOOK_URL }}
          status: ${{ needs.deploy.result }}
          environment: prod
```

### With extra fields (image tag, domain, stack name)

```yaml
- uses: alexandergv2117/actions/slack-notify@main
  if: always()
  with:
    webhook-url: ${{ secrets.SLACK_WEBHOOK_URL }}
    status: ${{ job.status }}
    environment: prod
    action-name: Deploy
    extra-fields: |
      [
        {"label": "Image", "value": "${{ needs.build.outputs.image }}"},
        {"label": "Stack", "value": "${{ needs.deploy.outputs.stack-name }}"},
        {"label": "Domain", "value": "api.example.com"}
      ]
```

### Full example — build, deploy, then notify on success or failure

```yaml
jobs:
  build:
    # ...
    outputs:
      image: ${{ steps.build.outputs.image }}

  deploy:
    needs: build
    # ...

  notify:
    needs: [build, deploy]
    if: always()
    runs-on: ubuntu-latest
    steps:
      - uses: alexandergv2117/actions/slack-notify@main
        with:
          webhook-url: ${{ secrets.SLACK_WEBHOOK_URL }}
          status: ${{ needs.deploy.result }}
          environment: prod
          action-name: Deploy
          extra-fields: '[{"label":"Image","value":"${{ needs.build.outputs.image }}"}]'
```

### Overriding the status text

By default the text is derived from `status` + `action-name` (e.g. `Deploy
succeeded`, `Release failed`). Override it outright when that doesn't read
well:

```yaml
- uses: alexandergv2117/actions/slack-notify@main
  with:
    webhook-url: ${{ secrets.SLACK_WEBHOOK_URL }}
    status: success
    status-text: "Rollback to v1.2.3 completed"
```

## Inputs

| Input             | Required | Default                              | Description |
|--------------------|----------|----------------------------------------|-------------|
| `webhook-url`      | yes      | —                                       | Slack Incoming Webhook URL. **Pass as a secret.** |
| `status`           | yes      | —                                       | Overall result to report. One of `success`, `failure`, `cancelled` (anything else is treated as `failure`). Typically wired to `job.status`, `${{ job.status }}` of the current job, or `${{ needs.<job>.result }}` of a job this one depends on. |
| `environment`      | no       | `"prod"`                               | Environment the deploy targets (e.g. `dev`, `staging`, `prod`). Shown upper-cased in the title and as a field. |
| `action-name`      | no       | `"Deploy"`                              | Label for the action being reported, used in the default status text (e.g. `Deploy`, `Release`, `Build`, `Rollback`). |
| `status-text`      | no       | `""`                                    | Override the automatic status text entirely (e.g. `"Deploy succeeded"`). When empty, derived from `status` + `action-name`. |
| `repo`             | no       | `${{ github.repository }}`             | Repository in `owner/name` form. Used to build the repo link and the commit link. |
| `server-url`       | no       | `${{ github.server_url }}`             | GitHub server URL, used to build links (relevant for GitHub Enterprise Server). |
| `branch`           | no       | `${{ github.ref_name }}`               | Branch or ref name, shown as a field. |
| `sha`              | no       | `${{ github.sha }}`                    | Commit SHA. Only the first 7 chars are shown; linked to the full commit. |
| `actor`            | no       | `${{ github.actor }}`                  | User who triggered the run, shown as the Author field. |
| `commit-message`   | no       | `${{ github.event.head_commit.message }}` | Commit message; only the first line is shown. Empty on events with no head commit (e.g. `workflow_dispatch`), in which case the message falls back to `(manual run: <event-name>)`. |
| `event-name`       | no       | `${{ github.event_name }}`             | Name of the triggering event, shown when there is no commit message. |
| `run-id`           | no       | `${{ github.run_id }}`                 | Workflow run id, used to build the "View run on GitHub" link. |
| `run-attempt`      | no       | `${{ github.run_attempt }}`            | Workflow run attempt, used to build the run link (so a re-run links to the right attempt). |
| `extra-fields`     | no       | `"[]"`                                  | Extra fields to append after Author, as a JSON array of `{"label": "...", "value": "..."}` objects. Use it for things like image tag, stack name or domain. Example: `'[{"label":"Image","value":"ghcr.io/org/app:1.2.3"},{"label":"Domain","value":"api.example.com"}]'`. |

This action has no outputs.

## Message layout

```
✅ Deploy succeeded · my-repo · PROD
───────────────────────────────────────
Environment: PROD      Repo: my-repo (linked)
Branch: `main`          Commit: a1b2c3d (linked)
Author: octocat         <extra fields…>
───────────────────────────────────────
Commit: <first line of commit message>
───────────────────────────────────────
status: `success`       View run on GitHub (linked)
```

## Requirements

- `jq` and `curl` must be available on the runner — both are preinstalled
  on `ubuntu-latest`/`ubuntu-*` GitHub-hosted runners. On a self-hosted
  runner, make sure both are installed.
- Use `if: always()` on the job/step calling this action if you want it to
  fire on failure too — by default a job is skipped once an earlier step in
  the same job (or a `needs` job) fails.

## Notes

- A failed Slack POST (bad webhook URL, Slack outage, etc.) only logs a
  `::warning::` annotation — it will never fail your workflow.
- The commit message is HTML-escaped (`&`, `<`, `>`) before being embedded
  in the JSON payload's `mrkdwn` text, so messages containing those
  characters won't break Slack's rendering.
- `extra-fields` must be valid JSON — an invalid value will make the `jq`
  step fail (and thus the action step fail, independent of the Slack POST
  itself).
