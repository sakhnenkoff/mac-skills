---
name: github-email-filters
description: Set up Gmail filters to silence noisy GitHub notification emails while keeping important ones (review requests, PR author replies, @mentions).
---

# GitHub Email Notification Filters

## When to use

- Getting spammed by GitHub notification emails (every PR comment reply, repo watching noise)
- Want to keep important notifications: review requests, replies on your PRs, @mentions
- Setting up a new work email and want sane GitHub defaults from day one
- Works for any GitHub host (github.com, GitHub Enterprise)

## Background

GitHub includes a CC address in every notification email that indicates *why* you received it:

| CC address | Reason | Keep in inbox? |
|---|---|---|
| `author@noreply.<host>` | You authored the PR | Yes |
| `review_requested@noreply.<host>` | You were asked to review | Yes |
| `mention@noreply.<host>` | You were @mentioned | Yes |
| `comment@noreply.<host>` | You commented once on a thread | No - this is the main noise source |
| `subscribed@noreply.<host>` | You're watching the repo | No - repo-level noise |
| `team_mention@noreply.<host>` | Your team was mentioned | User's choice |

Filters use Gmail's `cc:` search operator to match these addresses and auto-archive the noisy ones.

## Prerequisites

- Google Workspace (Gmail) email account
- Playwright MCP server available in Claude Code

## Procedure

**Ask the user for their GitHub host before starting.** Common values:
- `github.com` (public GitHub)
- `github.schibsted.io` (Schibsted GitHub Enterprise)
- Any other GitHub Enterprise hostname

### Step 1: Open Gmail in Playwright

Navigate to Gmail and ensure the user is signed in.

```
Navigate to: https://mail.google.com
```

- If a Google sign-in page appears: ask the user to sign in with their work account in the Playwright browser, then continue.
- If Gmail loads: proceed.

### Step 2: Create filter for "comment" notifications

This filter catches the main noise source — you commented once on a PR thread and now receive an email for every subsequent reply.

1. Click the **Advanced search options** button (filter icon in search bar)
2. Fill in the search form:
   - **From:** `notifications@<github-host>`
   - **Has the words:** `cc:comment@noreply.<github-host>`
3. Click **Create filter** (not "Search Mail")
4. If a "Confirm creating filter" dialog appears warning about `cc:` — click **OK** (the `cc:` operator does work for incoming mail headers)
5. On the filter action screen:
   - Check **Skip the Inbox (Archive it)**
   - Check **Also apply filter to matching conversations**
6. Click **Create filter**

### Step 3: Create filter for "subscribed" notifications

This filter catches repo watching noise — emails sent because you're watching a repository.

1. Click **Advanced search options** again
2. Fill in:
   - **From:** `notifications@<github-host>`
   - **Has the words:** `cc:subscribed@noreply.<github-host>`
3. Click **Create filter**
4. If confirm dialog appears — click **OK**
5. On the filter action screen:
   - Check **Skip the Inbox (Archive it)**
   - Check **Also apply filter to matching conversations**
6. Click **Create filter**

### Step 4 (Optional): Create filter for "team_mention" notifications

Only if the user wants to silence team @mentions too.

1. Same process as above with:
   - **Has the words:** `cc:team_mention@noreply.<github-host>`

### Playwright notes

- The filter action checkboxes may be outside the viewport. Use `scrollIntoView()` + `click()` via `browser_evaluate` if `browser_click` or `browser_fill_form` times out with "element is outside of the viewport".
- Gmail's advanced search dialog refs change on every open — always take a fresh snapshot before interacting.
- The "Search within" dropdown may default to a label instead of "All Mail" — this doesn't affect filter creation.

## Status summary

After creating filters, print:

| Filter | CC match | Action | Status |
|--------|----------|--------|--------|
| Comment noise | `comment@noreply.<host>` | Skip Inbox | Created |
| Watching noise | `subscribed@noreply.<host>` | Skip Inbox | Created |
| Team mentions | `team_mention@noreply.<host>` | Skip Inbox | Skipped / Created |

And confirm what still reaches the inbox:
- Replies on PRs you authored
- Review requests directed at you
- @mentions
- State changes (merged, closed) on your PRs

## Checklist

- [ ] Gmail accessible in Playwright browser
- [ ] "comment" filter created and applied
- [ ] "subscribed" filter created and applied
- [ ] "team_mention" filter created (if requested)
- [ ] User understands archived emails are still searchable in Gmail
