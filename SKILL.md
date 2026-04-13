---
name: sonarqube-fixer
description: Fix SonarQube issues on the current PR using the SonarQube REST API. Fetches all unresolved issues reported against the PR and applies fixes to the codebase. Use when the user runs /sonarqube-fixer or asks to fix SonarQube/SonarCloud issues on a PR.
version: 1.1.0
---

# SonarQube Fixer

Fetches all unresolved SonarQube/SonarCloud issues for a pull request via the REST API and fixes them in the codebase.

## Environment Variables

| Variable | Required | Default | Description |
|----------|----------|---------|-------------|
| `SONARQUBE_TOKEN` | Yes | — | User token for API authentication |
| `SONARQUBE_HOST` | No | `https://sonarcloud.io` | Base URL of your SonarQube instance. Set this only for self-hosted SonarQube. |
| `SONARQUBE_ORGANIZATION` | SonarCloud only | — | Your SonarCloud organization key. Not needed for self-hosted SonarQube. |
| `SONARQUBE_PROJECT_KEY` | No | Auto-detected | Override project key if auto-detection fails. |

**SonarCloud vs self-hosted:** If `SONARQUBE_HOST` is unset or is `https://sonarcloud.io`, this is SonarCloud mode and `SONARQUBE_ORGANIZATION` is required. If `SONARQUBE_HOST` points to any other URL, this is self-hosted mode and no organization key is needed.

## Workflow

### Step 1 — Resolve the PR number

If the user provided a PR number, use it directly. Otherwise run:

```bash
gh pr view --json number --jq '.number'
```

If that fails (detached HEAD, no upstream, etc.), ask the user for the PR number.

### Step 2 — Resolve and validate all configuration

Work through each item below. Collect everything before proceeding — do not fetch issues until all required values are known.

#### Token (`SONARQUBE_TOKEN`)

Check the environment variable. If missing, read `setup.md` from this skill's directory and present the relevant sections to the user, then ask them to paste their token.

#### Host (`SONARQUBE_HOST`)

Check the environment variable. If not set, default to `https://sonarcloud.io` (SonarCloud mode).

#### Organization (`SONARQUBE_ORGANIZATION`) — SonarCloud only

Skip this step entirely if in self-hosted mode.

If in SonarCloud mode, try to auto-detect from config files:

```bash
grep "sonar.organization" sonar-project.properties .sonarcloud.properties 2>/dev/null | head -1
```

If not found, check the `SONARQUBE_ORGANIZATION` environment variable. If still missing, read `setup.md` and show the relevant section, then ask the user:

> "What is your SonarCloud organization key? You can find it at sonarcloud.io → your avatar → **My Organizations** — it's the short slug in the URL (e.g. `my-company`, not the display name)."

#### Project key (`SONARQUBE_PROJECT_KEY`)

Try each of these in order, stopping at the first match:

1. `SONARQUBE_PROJECT_KEY` environment variable
2. `sonar-project.properties` → `sonar.projectKey=`
3. `.sonarcloud.properties` → `sonar.projectKey=`
4. `pom.xml` → `<sonar.projectKey>` property or `<groupId>:<artifactId>`
5. `build.gradle` or `build.gradle.kts` → `sonarqube { properties { property "sonar.projectKey", "..." } }`
6. `package.json` → `"sonar": { "projectKey": "..." }`

```bash
grep -r "sonar.projectKey\|sonar\.projectKey" \
  sonar-project.properties .sonarcloud.properties package.json \
  build.gradle build.gradle.kts pom.xml 2>/dev/null | head -5
```

If not found anywhere, ask the user:

> "What is your SonarQube project key? You can find it on the project's main page in SonarQube/SonarCloud — it's shown under the project name or in the URL."

### Step 3 — Fetch issues via REST API

#### Issues

Self-hosted:

```bash
curl -s -u "${SONARQUBE_TOKEN}:" \
  "${SONARQUBE_HOST}/api/issues/search?projectKeys=${PROJECT_KEY}&pullRequest=${PR_NUMBER}&resolved=false&ps=500"
```

SonarCloud (append `&organization=`):

```bash
curl -s -u "${SONARQUBE_TOKEN}:" \
  "${SONARQUBE_HOST}/api/issues/search?projectKeys=${PROJECT_KEY}&pullRequest=${PR_NUMBER}&resolved=false&ps=500&organization=${ORGANIZATION}"
```

If `total > 500`, fetch subsequent pages by appending `&p=2`, `&p=3`, etc. until all issues are retrieved.

#### Security hotspots

```bash
curl -s -u "${SONARQUBE_TOKEN}:" \
  "${SONARQUBE_HOST}/api/hotspots/search?projectKey=${PROJECT_KEY}&pullRequest=${PR_NUMBER}"
```

For SonarCloud, append `&organization=${ORGANIZATION}`.

#### Parsing the response

Each issue in the `issues` array contains:
- `component` — `projectKey:path/to/file.ts` — strip the `projectKey:` prefix to get the file path relative to the repo root
- `line` — line number where the issue occurs
- `message` — human-readable description
- `rule` — rule ID (e.g. `typescript:S1135`, `python:S1481`)
- `severity` — `BLOCKER`, `CRITICAL`, `MAJOR`, `MINOR`, `INFO`
- `type` — `BUG`, `VULNERABILITY`, `CODE_SMELL`

Each hotspot in the `hotspots` array contains:
- `component`, `line`, `message`, `rule` — same as above
- `status` — `TO_REVIEW`, `ACKNOWLEDGED`, `FIXED`, `SAFE`

Only act on hotspots with `status: TO_REVIEW`.

**If the API returns a 401:** The token is invalid or expired — tell the user and stop.
**If the API returns a 403:** The token lacks permission or the project/org key is wrong — tell the user both possibilities and stop.
**If the response contains no issues and no hotspots:** Tell the user the PR is clean and stop.

### Step 4 — Triage and fix

Before starting, read `rules.md` from this skill's directory for guidance on how to fix common rule IDs.

Group all issues by file so each file is read only once.

For each file:
1. Read the file.
2. Work through all issues for that file.
3. Apply fixes — do not refactor surrounding code or clean up unrelated issues.

For each individual issue:
- Understand the rule and the flagged line in context.
- Apply the minimal correct fix.
- If the issue is in a generated file (e.g. `*.generated.ts`, `dist/`, `build/`, `vendor/`), skip it and note it in the report.
- For hotspots (`TO_REVIEW`): assess whether it is a genuine vulnerability. Fix real ones. Mark false positives in the report — do not edit the code for false positives.
- If a fix requires an architectural change beyond a targeted edit, note it as "Needs manual review" in the report and skip.

### Step 5 — Report

Produce a summary table after all fixes are applied:

| File | Line | Rule | Issue | Action |
|------|------|------|-------|--------|
| `src/auth.ts` | 42 | `typescript:S2068` | Hardcoded credential | Fixed |
| `src/utils.ts` | 10 | `typescript:S3776` | Cognitive complexity too high | Fixed |
| `scripts/deploy.sh` | 7 | `shell:S2086` | Unquoted variable | Fixed |
| `Dockerfile` | 3 | `docker:S6476` | Possible root user | False positive — `:nonroot` image |
| `src/legacy.ts` | 88 | `typescript:S3776` | Cognitive complexity too high | Needs manual review — full function rewrite required |

## Notes

- Always read a file before editing it.
- Apply fixes conservatively. Do not clean up surrounding code.
- If pagination is needed (more than 500 issues), fetch all pages before starting fixes.
- Issues in `node_modules/`, `dist/`, `build/`, `vendor/`, or `*.generated.*` files should be skipped.
