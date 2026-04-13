# SonarQube Fixer

Fetches all unresolved SonarQube/SonarCloud issues on your pull request and fixes them — using the REST API, no MCP server required.

## What It Does

Run `/sonarqube-fixer` and Claude will:

- Detect the open PR from your current branch (or accept a PR number)
- Fetch all unresolved issues and security hotspots via the SonarQube REST API
- Read and fix each affected file with targeted, minimal changes
- Report every issue — fixed, skipped, or flagged as a false positive

## Usage

```
/sonarqube-fixer
/sonarqube-fixer 123
"Fix the SonarQube issues on this PR"
"Fix any sonar issues"
```

## Configuration

The skill uses environment variables. On first run, Claude will walk you through anything that is missing.

| Variable | Required | Default | Description |
|----------|----------|---------|-------------|
| `SONARQUBE_TOKEN` | Yes | — | User token (not a Project Analysis Token) |
| `SONARQUBE_HOST` | No | `https://sonarcloud.io` | Set only for self-hosted SonarQube |
| `SONARQUBE_ORGANIZATION` | SonarCloud only | — | Your org slug — not needed for self-hosted |
| `SONARQUBE_PROJECT_KEY` | No | Auto-detected | Override if auto-detection fails |

Store credentials in your shell profile or `~/.claude/settings.json` so you are not prompted each session.

## Authors

Leo Mosley (leo@leomosley.com)
