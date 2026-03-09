---
name: github-setup
description: Complete GitHub setup on a new Mac — SSH key generation, multi-account SSH config, GitHub CLI installation and authentication.
---

# GitHub Setup

## When to use

- Setting up GitHub on a new Mac from scratch
- Adding another GitHub account (personal, work, enterprise)
- Installing and authenticating the GitHub CLI (`gh`)
- Configuring per-directory git identity for multi-account workflows

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
# For each account:
ssh-keygen -t ed25519 -C "your-email@example.com" -f ~/.ssh/id_ed25519_<label>
```

Example labels: `personal`, `work`, `company-name`.

If you only have one GitHub account, a single key with the default path is fine:

```bash
ssh-keygen -t ed25519 -C "your-email@example.com"
```

### Step 4: Add keys to SSH agent

**Check:** `ssh-add -l`
- If all needed keys are listed: skip this step.
- If some keys are missing: add only the missing ones.

```bash
# Start the agent (usually already running on macOS)
eval "$(ssh-agent -s)"

# Add each key, storing the passphrase in macOS Keychain
ssh-add --apple-use-keychain ~/.ssh/id_ed25519_<label>
```

### Step 5: Configure `~/.ssh/config`

**Check:** `cat ~/.ssh/config 2>/dev/null` — look for existing Host entries matching the needed GitHub hosts/aliases.
- If all needed hosts are configured: skip this step.
- If config exists but a new host is needed: append only the new entry.

**Single account (simple):**

```
Host github.com
  AddKeysToAgent yes
  UseKeychain yes
  IdentityFile ~/.ssh/id_ed25519
```

**Multiple accounts on github.com (alias-based):**

When two or more accounts use the same `github.com`, create host aliases:

```
# Personal account
Host github.com-personal
  HostName github.com
  User git
  AddKeysToAgent yes
  UseKeychain yes
  IdentityFile ~/.ssh/id_ed25519_personal

# Work account
Host github.com-work
  HostName github.com
  User git
  AddKeysToAgent yes
  UseKeychain yes
  IdentityFile ~/.ssh/id_ed25519_work
```

Then clone using the alias as the host:

```bash
git clone git@github.com-work:org/repo.git
```

**Multiple accounts on different hostnames (e.g., github.com + GitHub Enterprise):**

```
Host github.com
  AddKeysToAgent yes
  UseKeychain yes
  IdentityFile ~/.ssh/id_ed25519_personal

Host github.example.com
  AddKeysToAgent yes
  UseKeychain yes
  IdentityFile ~/.ssh/id_ed25519_work
```

No aliases needed — SSH matches by hostname automatically.

### Step 6: Add public keys to GitHub

**Check:** `ssh -T git@<host> 2>&1` for each host/alias.
- If `Hi <username>!` is returned: the key is already added, skip for that host.
- If `Permission denied`: the public key still needs to be added to GitHub.

Copy each public key and add it in GitHub UI under **Settings > SSH and GPG keys > New SSH key**.

```bash
pbcopy < ~/.ssh/id_ed25519_<label>.pub
```

### Step 7: Verify SSH connections

Run `ssh -T` for every configured host/alias. This step is never skipped — it confirms the full chain works.

```bash
ssh -T git@github.com
# or for aliases:
ssh -T git@github.com-personal
ssh -T git@github.com-work
```

Expected: `Hi <username>! You've successfully authenticated...`

### Step 8: Authenticate GitHub CLI

**Check:** `gh auth status`
- If all needed accounts show as authenticated with SSH protocol: skip this step.
- If authenticated but using HTTPS protocol: re-authenticate with `--git-protocol ssh`.
- If not authenticated: proceed.

For a single account:

```bash
gh auth login
```

Select **SSH** as the preferred protocol when prompted.

For multiple accounts:

```bash
# Log in to each account
gh auth login --hostname github.com
gh auth login --hostname github.example.com

# Switch between accounts on the same host
gh auth switch
```

### Step 9: Set per-directory git identity

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
| Git identity | Configured / Skipped (single account) |

## Checklist

- [ ] Homebrew installed
- [ ] `gh` CLI installed (`gh --version`)
- [ ] SSH key(s) generated
- [ ] Keys added to SSH agent with Keychain integration
- [ ] `~/.ssh/config` configured
- [ ] Public key(s) added to GitHub
- [ ] `ssh -T` succeeds for each host/alias
- [ ] `gh auth status` shows authenticated
- [ ] Per-directory git identity configured (if multi-account)
