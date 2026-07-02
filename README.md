# CI Failure Inspector

Automated CI failure inspection tool — queries Jenkins builds, groups failing tests, and creates GitHub issues with environment context.

## Overview

The inspector runs as a GitHub Actions workflow (or locally via Node.js). It talks directly to the Jenkins REST API and GitHub API with zero external dependencies.

```
Jenkins REST API
  └── find batch anchor → collect failing tests per build

GitHub API
  ├── fetch all existing issues upfront (deduplication)
  ├── create new issues with environments table
  ├── reopen closed issues (regression detection)
  └── add to Projects v2 board (Backlog column)
```

### Daily Builds Inspected (community env matrix)

| Version | User |
|---------|------|
| `head` | `@adminUser` |
| `head` | `@standardUser` |
| `v2.14-head` | `@adminUser` |
| `v2.14-head` | `@standardUser` |
| `v2.13-head` | `@adminUser` |
| `v2.13-head` | `@standardUser` |

Batch anchor: the most recent completed build with description exactly `head · community · @adminUser`.

### Issue Behavior

| Condition | Action |
|-----------|--------|
| No existing issue | Create new issue with environments table, add to project Backlog |
| Open issue exists | Add a comment listing the environment(s) where it failed |
| Closed issue exists | Reopen + add regression comment + move back to Backlog |

> **Prerequisite:** Enable the GitHub Projects workflow to auto-close issues when moved to Done (`Projects → Workflows → Auto-close issue`). This ensures closed/open state reflects the board.

---

## GitHub Actions Workflow

The workflow runs automatically Tue–Sat at 11:00 UTC (4:00 AM PST, after daily Jenkins runs complete) and can also be triggered manually via `Actions → Jenkins CI Failure Inspection → Run workflow`.

### Required Secrets

| Secret | Description |
|--------|-------------|
| `INSPECTOR_GITHUB_TOKEN` | GitHub PAT with `repo` and `project` scopes |
| `INSPECTOR_JENKINS_AUTH` | Base64-encoded `username:token` for Jenkins REST API |
| `INSPECTOR_JENKINS_BASE_URL` | Jenkins instance base URL |

### Optional Variables

| Variable | Description | Default |
|----------|-------------|---------|
| `INSPECTOR_JENKINS_JOB_PATH` | Jenkins job path | `rancher_qa/ui-automation-ansible-job` |
| `INSPECTOR_GITHUB_ORG` | Org or user owning the target repo | `yonasberhe23` |
| `INSPECTOR_GITHUB_REPO` | Repo to create issues in | `fleet-test` |
| `INSPECTOR_PROJECT_NUMBER` | Projects v2 board number | `1` |
| `INSPECTOR_PROJECT_OWNER` | `user` or `org` | `user` |
| `INSPECTOR_ISSUE_LABELS` | Comma-separated labels | `area/automation-test-ui,kind/flaky-test` |
| `INSPECTOR_STATUS_FIELD_ID` | ProjectV2 Status field node ID | — |
| `INSPECTOR_BACKLOG_OPTION_ID` | Status option ID for the Backlog column | — |

---

## Local Setup

Create a `.env` file at the repo root with the required variables:

```bash
GITHUB_TOKEN=your_pat
JENKINS_AUTH=base64(username:token)
JENKINS_BASE_URL=https://your-jenkins-instance
```

Then run:

```bash
node src/inspect.js
```

### Dry Run

```bash
DRY_RUN=true node src/inspect.js
```

---

## Source Files

| File | Description |
|------|-------------|
| `src/inspect.js` | Main entrypoint — orchestrates the full inspection run |
| `src/jenkins-client.js` | Jenkins REST API client — batch detection and test result fetching |
| `src/github-client.js` | GitHub API client — issue creation, deduplication, project board |

