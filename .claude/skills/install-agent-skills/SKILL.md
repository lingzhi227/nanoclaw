---
name: install-agent-skills
description: Install agent skills from a GitHub repository into NanoClaw. Clones the repo, detects required API keys, adds them to .env and the secrets whitelist, and syncs to running sessions. Use when the user wants to give Claw new capabilities via a skill pack.
argument-hint: [github-url]
---

# Install Agent Skills

Install a GitHub skill repository so container agents can use the skills.

## Overview

Agent skills live in `~/.agents/skills/` and are symlinked to `~/.claude/skills/`.
On startup, `syncSharedSkills()` copies `~/.claude/skills/` (dereferenced) into `data/shared-skills/`,
which is bind-mounted read-only into every container at `/home/node/.claude/skills/`.
After installation, restart the service so `syncSharedSkills()` picks up the new skills.

## Steps

### 1. Clone the repository

```bash
REPO_URL="$ARGUMENTS"
REPO_NAME=$(basename "$REPO_URL" .git)
AGENTS_DIR="$HOME/.agents/skills"
mkdir -p "$AGENTS_DIR"

# Clone or update
if [ -d "$AGENTS_DIR/$REPO_NAME" ]; then
  git -C "$AGENTS_DIR/$REPO_NAME" pull
else
  git clone "$REPO_URL" "$AGENTS_DIR/$REPO_NAME"
fi
```

### 2. Discover skills in the repo

A skill is any directory containing a `SKILL.md` file.

```bash
find "$AGENTS_DIR/$REPO_NAME" -name "SKILL.md" | sort
```

For each discovered skill `<repo>/<skill-name>/SKILL.md`:

### 3. Symlink skills to `~/.claude/skills/`

```bash
CLAUDE_SKILLS="$HOME/.claude/skills"
mkdir -p "$CLAUDE_SKILLS"

# For each skill directory <repo>/<skill-name>:
ln -sfn "$AGENTS_DIR/$REPO_NAME/<skill-name>" "$CLAUDE_SKILLS/<skill-name>"
```

If a skill already exists at `~/.claude/skills/<skill-name>`, check if it's from this repo
(a symlink pointing into `$AGENTS_DIR/$REPO_NAME`) — update it. Otherwise ask the user
whether to overwrite.

### 4. Detect required API keys

Scan all SKILL.md files in the repo for env var patterns:

```bash
# Patterns to look for: $VAR_NAME, ${VAR_NAME}, "VAR_NAME" env var, env var VAR_NAME
grep -hoE '\$\{?[A-Z][A-Z0-9_]+\}?' "$AGENTS_DIR/$REPO_NAME"/*/SKILL.md 2>/dev/null \
  | sed 's/[${}]//g' | sort -u
```

Filter out common non-secret vars: `HOME`, `PATH`, `PWD`, `USER`, `ARGUMENTS`, `SHELL`, etc.

### 5. Check which vars are missing from `.env`

Read `/path/to/nanoclaw/.env` and list which detected vars are absent.

For each missing var:
- Show what it's needed for (which skill uses it)
- Ask the user to provide the value
- Add it to `.env`:

```bash
echo "VARNAME=value" >> /path/to/nanoclaw/.env
```

### 6. Update the secrets whitelist

The whitelist at `/path/to/nanoclaw/.claude/agent-env.json` tells `container-runner.ts`
which extra env vars to pass into containers. Add all detected vars (even if already in .env).

Read the file (or start with `[]`), merge in new var names, write back:

```json
["S2_API_KEY", "NEW_VAR_NAME", "ANOTHER_KEY"]
```

```bash
# Read existing list, add new keys, deduplicate
node -e "
  const fs = require('fs');
  const p = '.claude/agent-env.json';
  const existing = fs.existsSync(p) ? JSON.parse(fs.readFileSync(p,'utf8')) : [];
  const newKeys = ['NEW_VAR_NAME'];   // replace with actual detected keys
  const merged = [...new Set([...existing, ...newKeys])];
  fs.writeFileSync(p, JSON.stringify(merged, null, 2) + '\n');
  console.log('Updated:', merged);
"
```

### 7. Restart the NanoClaw service

```bash
launchctl unload ~/Library/LaunchAgents/com.nanoclaw.plist 2>/dev/null || true
launchctl load ~/Library/LaunchAgents/com.nanoclaw.plist
```

Or if running in dev mode: the operator should restart `npm run dev`.

### 8. Verify installation

Confirm the skills were installed:

```bash
ls ~/.claude/skills/ | grep -E "$(ls "$AGENTS_DIR/$REPO_NAME" | tr '\n' '|' | sed 's/|$//')"
```

Tell the user which skills were installed and what commands to use them with (e.g., `/deep-research`, `/literature-search`).

## Path Normalization (automatic)

`container-runner.ts` automatically normalizes paths in `SKILL.md` files when syncing to
containers. Specifically:
- `${homeDir}/` (e.g. `/Users/alice/`) → `~/`
- `.//` (double-slash typo) → `./`

So skill authors do not need to avoid absolute paths — they will be fixed automatically.

## Example

```
/install-agent-skills https://github.com/lingzhi227/agent-research-skills
```

1. Clones to `~/.agents/skills/agent-research-skills/`
2. Finds: `deep-research/`, `literature-search/`, `citation-management/`, etc.
3. Detects: `$S2_API_KEY` in multiple SKILL.md files
4. Checks: is `S2_API_KEY` in `.env`? If not, prompts for it.
5. Adds to `.claude/agent-env.json`
6. Symlinks all skill dirs to `~/.claude/skills/`
7. Restarts service

## Notes

- Skills from `~/.claude/skills/` override project-level skills from `container/skills/`
- Symlinks are dereferenced when copied to containers (`cpSync` with `dereference: true`)
- Path normalization happens at sync time (not at install time), so re-cloning is safe
- To uninstall: remove the symlinks from `~/.claude/skills/` and the keys from `.claude/agent-env.json`
