# First-time Setup Guide

Show this when any required configuration is missing. Tailor it to what is actually missing — don't show the full guide if only the organization key is needed.

## Generating a token

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

## Storing configuration permanently

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
