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

Check the environment variable. If missing, show the **First-time Setup** guide (see below) and then ask the user to paste their token.

#### Host (`SONARQUBE_HOST`)

Check the environment variable. If not set, default to `https://sonarcloud.io` (SonarCloud mode).

#### Organization (`SONARQUBE_ORGANIZATION`) — SonarCloud only

Skip this step entirely if in self-hosted mode.

If in SonarCloud mode, try to auto-detect from config files:

```bash
# sonar-project.properties or .sonarcloud.properties
grep "sonar.organization" sonar-project.properties .sonarcloud.properties 2>/dev/null | head -1
```

If not found, check the `SONARQUBE_ORGANIZATION` environment variable. If still missing, ask the user:

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
# Quick scan across common locations
grep -r "sonar.projectKey\|sonar\.projectKey" \
  sonar-project.properties .sonarcloud.properties package.json \
  build.gradle build.gradle.kts pom.xml 2>/dev/null | head -5
```

If not found anywhere, ask the user:

> "What is your SonarQube project key? You can find it on the project's main page in SonarQube/SonarCloud — it's shown under the project name or in the URL."

### Step 3 — Fetch issues via REST API

#### Issues

```bash
curl -s -u "${SONARQUBE_TOKEN}:" \
  "${SONARQUBE_HOST}/api/issues/search?projectKeys=${PROJECT_KEY}&pullRequest=${PR_NUMBER}&resolved=false&ps=500"
```

For SonarCloud, append `&organization=${ORGANIZATION}`:

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
- If a fix requires an architectural change that is beyond a targeted edit (e.g. refactoring an entire service), note it as "Needs manual review" in the report and skip.

### Step 5 — Report

Produce a summary table after all fixes are applied:

| File | Line | Rule | Issue | Action |
|------|------|------|-------|--------|
| `src/auth.ts` | 42 | `typescript:S2068` | Hardcoded credential | Fixed |
| `src/utils.ts` | 10 | `typescript:S3776` | Cognitive complexity too high | Fixed |
| `scripts/deploy.sh` | 7 | `shell:S2086` | Unquoted variable | Fixed |
| `Dockerfile` | 3 | `docker:S6476` | Possible root user | False positive — `:nonroot` image |
| `src/legacy.ts` | 88 | `typescript:S3776` | Cognitive complexity too high | Needs manual review — full function rewrite required |

---

## First-time Setup Guide

Show this when any required configuration is missing. Tailor it to what is actually missing — don't show the full guide if only the organization key is needed.

### Generating a token

**SonarCloud:**
1. Go to [sonarcloud.io](https://sonarcloud.io) and sign in.
2. Click your avatar (top right) → **My Account** → **Security** tab.
3. Under "Generate Tokens", enter a name (e.g. `claude-code`) and click **Generate**.
4. Copy the token — it will not be shown again.
5. Use a **User Token** (not a Project Analysis Token). Project tokens cannot call the issues API on behalf of a user.

**Self-hosted SonarQube:**
1. Go to your SonarQube instance and sign in.
2. Click your avatar → **My Account** → **Security**.
3. Generate a **User Token** and copy it.

### Storing configuration permanently

**Option A — Shell profile** (available to all tools, not just Claude Code):

Add to `~/.zshrc`, `~/.bashrc`, or `~/.zprofile` (use whichever your shell loads):

```bash
export SONARQUBE_TOKEN="your-token-here"
export SONARQUBE_ORGANIZATION="your-org-key"   # SonarCloud only
# export SONARQUBE_HOST="https://sonar.yourcompany.com"  # self-hosted only
```

Then reload: `source ~/.zshrc` (or open a new terminal).

**Option B — Claude Code settings** (only available inside Claude Code sessions):

Edit `~/.claude/settings.json` and add an `env` block:

```json
{
  "env": {
    "SONARQUBE_TOKEN": "your-token-here",
    "SONARQUBE_ORGANIZATION": "your-org-key"
  }
}
```

If `~/.claude/settings.json` does not exist, create it with just the content above.

**Recommendation:** Option A (shell profile) makes the token available everywhere. Option B is fine if you only use this skill inside Claude Code.

---

## Common SonarQube Rule Fixes

**Code Smells:**
- `S1135` (TODO comment) — Remove or resolve the TODO
- `S3776` (cognitive complexity) — Extract inner logic into named helper functions
- `S107` (too many parameters) — Introduce an options/config object
- `S1066` (collapsible if) — Merge nested `if` into a single condition with `&&`
- `S1854` (useless assignment) — Remove the dead assignment or use the value
- `S2814` (variable re-declaration) — Replace `var` with `const`/`let`
- `S1192` (duplicated string literal) — Extract to a named constant

**Bugs:**
- `S2201` (return value ignored) — Use or explicitly discard the return value
- `S905` (non-boolean in boolean context) — Add an explicit boolean comparison
- `S3516` (function always returns same value) — Fix the logic or simplify the function

**Security / Vulnerabilities:**
- `S2068` (hardcoded credential) — Move value to an environment variable
- `S5042` (zip slip) — Validate that extracted paths stay within the target directory
- `S6096` (path traversal) — Sanitize and validate user-supplied file paths
- `S4790` (weak hash) — Replace MD5/SHA-1 with SHA-256 or stronger

**Shell scripts:**
- `S2086` / `SC2086` — Quote variables: `"$var"` not `$var`
- `S2155` / `SC2155` — Separate `declare`/`local` from assignment to capture exit codes
- `SC2034` — Remove unused variables
- `SC2046` — Quote command substitutions: `"$(cmd)"` not `$(cmd)`
- Use `[[ ]]` instead of `[ ]` for bash conditionals

**Java / Kotlin:**
- `S2095` (resource leak) — Use try-with-resources
- `S1612` (lambda can be method ref) — Replace `x -> foo(x)` with `Foo::foo`

**Python:**
- `S1481` (unused variable) — Remove or prefix with `_`
- `S5754` (bare except) — Catch specific exception types

## Notes

- Always read a file before editing it.
- Apply fixes conservatively. Do not clean up surrounding code.
- If pagination is needed (more than 500 issues), fetch all pages before starting fixes.
- Issues in `node_modules/`, `dist/`, `build/`, `vendor/`, or `*.generated.*` files should be skipped — these are not part of the source code.
