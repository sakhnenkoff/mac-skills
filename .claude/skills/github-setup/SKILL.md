---
name: github-setup
description: Use when setting up GitHub on a new Mac, adding another GitHub account (personal, work, enterprise), configuring multi-account SSH, or fixing HTTPS auth failures for gh CLI tools
---

# GitHub Setup

## When to use

- Setting up GitHub on a new Mac from scratch
- Adding another GitHub account (personal, work, enterprise)
- Installing and authenticating the GitHub CLI (`gh`)
- Fixing HTTPS authentication failures (e.g., plugin installs, `gh` commands)
- Configuring per-directory git identity for multi-account workflows

## Key concept: SSH vs `gh` CLI

SSH and `gh` CLI handle multiple accounts differently. Explain this to the user when setting up 2+ accounts on the same hostname.

| Layer | Multiple accounts on same host? | How it works |
|-------|--------------------------------|--------------|
| **SSH** (git clone/push/pull) | Simultaneous — no switching | Host aliases in `~/.ssh/config` route to correct key |
| **`gh` CLI** (PRs, issues, API, HTTPS) | One active at a time — requires `gh auth switch` | Only the active account is used for `gh` commands and HTTPS |

Different hostnames (e.g., `github.com` + `github.enterprise.com`) are always independent for both SSH and `gh`.

## Procedure

**Before starting any step, run the status check for that step. Skip steps that are already complete. Report skipped steps to the user.**

### Step 1: Install Homebrew (if needed)

**Check:** `which brew`
- If found: skip this step.

```bash
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
```

### Step 2: Install GitHub CLI (if needed)

**Check:** `gh --version`
- If found: skip this step.

```bash
brew install gh
```

### Step 3: Generate SSH key(s)

**Check:** `ls ~/.ssh/id_ed25519*.pub 2>/dev/null`
- If keys exist for all needed accounts: skip this step.
- If some keys exist but a new account is being added: generate only the missing key.

Generate one ed25519 key per GitHub account. Use the account email as the comment.

```bash
ssh-keygen -t ed25519 -C "your-email@example.com" -f ~/.ssh/id_ed25519_<label>
```

Example labels: `personal`, `work`, `sch`, `company-name`.

If you only have one GitHub account, a single key with the default path is fine:

```bash
ssh-keygen -t ed25519 -C "your-email@example.com"
```

### Step 4: Add keys to SSH agent

**Check:** `ssh-add -l`
- If all needed keys are listed: skip this step.
- If some keys are missing: add only the missing ones.

```bash
# Add each key, storing the passphrase in macOS Keychain
ssh-add --apple-use-keychain ~/.ssh/id_ed25519_<label>
```

### Step 5: Configure `~/.ssh/config`

**Check:** `cat ~/.ssh/config 2>/dev/null` — look for existing Host entries matching the needed GitHub hosts/aliases.
- If all needed hosts are configured: skip this step.
- If config exists but a new host is needed: append only the new entry.

**Always include `IdentitiesOnly yes`** to prevent SSH from trying wrong keys.

**Single account (simple):**

```
Host github.com
  HostName github.com
  User git
  IdentityFile ~/.ssh/id_ed25519
  AddKeysToAgent yes
  UseKeychain yes
  IdentitiesOnly yes
```

**Multiple accounts on the same hostname (alias-based):**

When two or more accounts use the same hostname, keep the first account on the bare host and create aliases for additional accounts:

```
# Primary account (personal)
Host github.com
  HostName github.com
  User git
  IdentityFile ~/.ssh/id_ed25519_personal
  AddKeysToAgent yes
  UseKeychain yes
  IdentitiesOnly yes

# Second account on same host (enterprise)
Host github.com-work
  HostName github.com
  User git
  IdentityFile ~/.ssh/id_ed25519_work
  AddKeysToAgent yes
  UseKeychain yes
  IdentitiesOnly yes
```

Clone using the alias: `git clone git@github.com-work:org/repo.git`

**Multiple accounts on different hostnames (e.g., github.com + GitHub Enterprise):**

No aliases needed — SSH matches by hostname automatically:

```
Host github.com
  HostName github.com
  User git
  IdentityFile ~/.ssh/id_ed25519_personal
  AddKeysToAgent yes
  UseKeychain yes
  IdentitiesOnly yes

Host github.example.com
  HostName github.example.com
  User git
  IdentityFile ~/.ssh/id_ed25519_work
  AddKeysToAgent yes
  UseKeychain yes
  IdentitiesOnly yes
```

**Mixed scenario (both same and different hostnames):**

This is the most common real-world setup. Combine the patterns:

```
# Account 1: github.com (personal)
Host github.com
  HostName github.com
  User git
  IdentityFile ~/.ssh/id_ed25519_personal
  AddKeysToAgent yes
  UseKeychain yes
  IdentitiesOnly yes

# Account 2: GitHub Enterprise (different hostname — no alias needed)
Host github.enterprise.com
  HostName github.enterprise.com
  User git
  IdentityFile ~/.ssh/id_ed25519_corp
  AddKeysToAgent yes
  UseKeychain yes
  IdentitiesOnly yes

# Account 3: github.com (enterprise — alias needed, shares hostname with Account 1)
Host github.com-ent
  HostName github.com
  User git
  IdentityFile ~/.ssh/id_ed25519_ent
  AddKeysToAgent yes
  UseKeychain yes
  IdentitiesOnly yes
```

### Step 6: Add public keys to GitHub

**Check:** `ssh -T git@<host-or-alias> 2>&1` for each host/alias.
- If `Hi <username>!` is returned: the key is already added, skip for that host.
- If `Permission denied`: the public key still needs to be added to GitHub.

Copy each public key to clipboard and instruct the user to add it in GitHub UI under **Settings > SSH and GPG keys > New SSH key**.

```bash
pbcopy < ~/.ssh/id_ed25519_<label>.pub
```

**This step requires user action.** Tell them the key is on their clipboard, where to paste it, and **wait for confirmation before proceeding** to Step 7.

### Step 7: Verify SSH connections

Run `ssh -T` for every configured host/alias. This step is never skipped — it confirms the full chain works.

```bash
ssh -T git@github.com
ssh -T git@github.enterprise.com
ssh -T git@github.com-work  # alias
```

Expected: `Hi <username>! You've successfully authenticated...`

(Exit code 1 is expected — GitHub doesn't provide shell access.)

### Step 8: Authenticate GitHub CLI

**Check:** `gh auth status`
- If all needed accounts show as authenticated with SSH protocol: skip this step.
- If authenticated but using HTTPS protocol: re-authenticate with `--git-protocol ssh`.
- If not authenticated: proceed.

**This step is interactive.** Tell the user to run:

```
! gh auth login --hostname github.com
```

Select **SSH** as the preferred protocol when prompted.

For multiple accounts on the same hostname, repeat `gh auth login` for each account. Use `gh auth switch` to change the active account:

```bash
gh auth switch --hostname github.com --user <username>
```

### Step 9: Set up HTTPS credential helper

**Check:** `git config --global credential.helper`
- If `gh auth setup-git` output or `/usr/local/bin/gh auth git-credential` is configured: skip.

Many tools (Claude Code plugins, `npm`, package managers) clone via HTTPS, not SSH. Register `gh` as the HTTPS credential helper so these work:

```bash
gh auth setup-git
```

This uses whichever `gh` account is currently active on each hostname.

### Step 10: Set per-directory git identity

**Check:** `git config --global --get-regexp includeIf` and check for existing `[user]` in `~/.gitconfig`.
- If `includeIf` rules already cover the needed directories: skip this step.
- If only one account is used: skip this step entirely.

Use `includeIf` in `~/.gitconfig` to automatically set name/email based on the repo directory:

```gitconfig
[user]
  name = Personal Name
  email = personal@example.com

[includeIf "gitdir:~/work/"]
  path = ~/.gitconfig-work
```

Then create `~/.gitconfig-work`:

```gitconfig
[user]
  name = Work Name
  email = work@company.com
```

Any repo cloned under `~/work/` will automatically use the work identity.

## Status summary

After running through all steps, print a summary table:

| Step | Status |
|------|--------|
| Homebrew | Installed / Skipped / Newly installed |
| GitHub CLI | Installed / Skipped / Newly installed |
| SSH key(s) | Found / Generated |
| SSH agent | Keys loaded / Keys added |
| SSH config | Configured / Updated / Created |
| GitHub SSH | Verified / Needs manual key upload |
| gh auth | Authenticated / Needs login |
| HTTPS credential helper | Configured / Skipped |
| Git identity | Configured / Skipped (single account) |

For multi-account setups, also print which account is active per hostname:

| Host | Active `gh` account | SSH key |
|------|---------------------|---------|
| github.com | username (via `gh auth status`) | key file |
| github.enterprise.com | username | key file |

## Checklist

- [ ] Homebrew installed
- [ ] `gh` CLI installed (`gh --version`)
- [ ] SSH key(s) generated
- [ ] Keys added to SSH agent with Keychain integration
- [ ] `~/.ssh/config` configured (with `IdentitiesOnly yes`)
- [ ] Public key(s) added to GitHub
- [ ] `ssh -T` succeeds for each host/alias
- [ ] `gh auth status` shows authenticated
- [ ] HTTPS credential helper configured (`gh auth setup-git`)
- [ ] Per-directory git identity configured (if multi-account)
