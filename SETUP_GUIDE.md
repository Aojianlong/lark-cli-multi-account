# lark-cli Multi-Account Setup Guide (for AI Agents)

You are an AI agent helping a user set up multi-account support for lark-cli (Feishu/Lark CLI).

## Why this is needed

lark-cli v1.0.0 only supports one active app at a time. `config init --new` wipes all previous keychain entries, and `RequireConfig()` hardcodes `Apps[0]`. Users with multiple Feishu tenants (e.g., enterprise + personal) cannot switch between them natively.

This guide uses the undocumented `LARKSUITE_CLI_CONFIG_DIR` env var to create fully isolated config directories per account.

## Prerequisites

Run these checks first:

```bash
# Check if lark-cli is installed
lark-cli --version

# If not installed:
npm install -g @larksuite/cli
# NOTE: On Windows, if npm postinstall fails (PowerShell Expand-Archive error),
# edit the install script to use `unzip` instead, then run `node scripts/install.js` manually.

# Install AI agent skills (optional but recommended)
npx skills add larksuite/cli -y -g
```

## Step 1: Collect account info from user

Ask the user:
- How many Feishu accounts do you have?
- For each account: a short name (e.g., "work", "personal") and which tenant it belongs to

Example: two accounts named "work" (enterprise) and "personal" (personal edition).

## Step 2: Create isolated config directories

For each account, create a separate directory:

```bash
mkdir -p ~/.lark-cli-work
mkdir -p ~/.lark-cli-personal
```

## Step 3: Initialize each account (requires user interaction)

For EACH account, run the following. The command will output a browser URL — send it to the user and tell them which account to use in the browser.

**IMPORTANT:** Do these one at a time. Each `config init --new` only affects its own directory thanks to the env var.

```bash
# Account 1: work
LARKSUITE_CLI_CONFIG_DIR=~/.lark-cli-work lark-cli config init --new
# Tell user: "Open the URL in browser and complete setup with your WORK/ENTERPRISE Feishu account"

# After config init succeeds:
LARKSUITE_CLI_CONFIG_DIR=~/.lark-cli-work lark-cli auth login --recommend
# Tell user: "Open the URL in browser and authorize with your WORK/ENTERPRISE Feishu account"
```

```bash
# Account 2: personal
LARKSUITE_CLI_CONFIG_DIR=~/.lark-cli-personal lark-cli config init --new
# Tell user: "Open the URL in browser and complete setup with your PERSONAL Feishu account"

# After config init succeeds:
LARKSUITE_CLI_CONFIG_DIR=~/.lark-cli-personal lark-cli auth login --recommend
# Tell user: "Open the URL in browser and authorize with your PERSONAL Feishu account"
```

## Step 4: Create wrapper scripts

Detect the user's shell and create scripts accordingly.

### For bash/zsh (macOS, Linux, Git Bash on Windows):

```bash
# Find a directory in PATH for scripts
# Common choices: ~/bin, ~/.local/bin, /usr/local/bin

mkdir -p ~/bin

# Create wrapper for each account
cat > ~/bin/lark-work << 'EOF'
#!/bin/bash
export LARKSUITE_CLI_CONFIG_DIR="$HOME/.lark-cli-work"
exec lark-cli "$@"
EOF

cat > ~/bin/lark-personal << 'EOF'
#!/bin/bash
export LARKSUITE_CLI_CONFIG_DIR="$HOME/.lark-cli-personal"
exec lark-cli "$@"
EOF

chmod +x ~/bin/lark-work ~/bin/lark-personal
```

Make sure `~/bin` is in PATH. If not, add to `~/.bashrc` or `~/.zshrc`:
```bash
export PATH="$HOME/bin:$PATH"
```

### For PowerShell (Windows native):

```powershell
# Create wrapper functions in PowerShell profile
$profileDir = Split-Path $PROFILE
New-Item -ItemType Directory -Force -Path $profileDir | Out-Null

Add-Content $PROFILE @'

function lark-work {
    $env:LARKSUITE_CLI_CONFIG_DIR = "$env:USERPROFILE\.lark-cli-work"
    & lark-cli @args
    Remove-Item Env:\LARKSUITE_CLI_CONFIG_DIR
}

function lark-personal {
    $env:LARKSUITE_CLI_CONFIG_DIR = "$env:USERPROFILE\.lark-cli-personal"
    & lark-cli @args
    Remove-Item Env:\LARKSUITE_CLI_CONFIG_DIR
}
'@
```

## Step 5: Verify

Test both accounts:

```bash
lark-work auth status
# Should show the work/enterprise account

lark-personal auth status
# Should show the personal account
```

If both return valid status with different usernames, the setup is complete.

## Step 6: Configure your agent (optional)

Add this to your project's CLAUDE.md, .cursorrules, or equivalent agent config:

```markdown
## Feishu CLI (lark-cli)

Two Feishu accounts available:

| Command | Account | Tenant |
|---------|---------|--------|
| `lark-work` | <work_username> | Enterprise |
| `lark-personal` | <personal_username> | Personal |

Rules:
- All Feishu operations use lark-cli via these wrapper commands
- If the user specifies an account name or type, use the corresponding command
- If a URL contains `<company>.feishu.cn` (enterprise subdomain), use `lark-work`
- Otherwise, ask the user which account to use before proceeding
- Fetch documents: `lark-work docs +fetch --doc "<URL>" --format pretty`
```

## Troubleshooting

**Q: `auth login` fails with "keychain entry not found"**
A: Make sure you set `LARKSUITE_CLI_CONFIG_DIR` before running `config init --new`. Each directory must have its own `config init` first.

**Q: npm install fails on Windows**
A: The postinstall script uses PowerShell `Expand-Archive` which may fail. Fix: `npm install -g @larksuite/cli --ignore-scripts`, then edit `scripts/install.js` to use `unzip` instead of PowerShell, and run `node scripts/install.js` manually.

**Q: Can I add more than 2 accounts?**
A: Yes. Repeat Steps 2-4 for each account with a new directory name and wrapper script name.

## How it works

lark-cli reads `LARKSUITE_CLI_CONFIG_DIR` to determine where to look for `config.json` and store keychain entries. By pointing each wrapper to a different directory, each account gets its own isolated config and credentials. No keychain conflicts, no overwrites.

This workaround is needed because lark-cli v1.0.0 has two limitations:
1. `RequireConfig()` in `internal/core/config.go` hardcodes `Apps[0]`
2. `cleanupOldConfig()` in `cmd/config/init.go` wipes other apps' keychain entries

Tracking issue: https://github.com/larksuite/cli/issues/29
