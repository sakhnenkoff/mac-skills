# mac-skills

Reusable [Claude Code](https://docs.anthropic.com/en/docs/claude-code) skills for setting up a new Mac. Clone this repo and point Claude Code at it — it handles the rest.

## Skills

| Skill | What it does |
|-------|-------------|
| [github-setup](.claude/skills/github-setup/SKILL.md) | Full GitHub setup from zero: Homebrew, SSH keys (single or multi-account), `~/.ssh/config`, GitHub CLI install + auth, per-directory git identity |

## Usage

### Quick start

Clone this repo anywhere on your Mac:

```bash
git clone git@github.com:sakhnenkoff/mac-skills.git ~/Developer/Personal/mac-skills
```

Then open it in Claude Code and ask for what you need:

```
> Set up GitHub on this Mac
> Add a second GitHub account for work
> Install and authenticate the gh CLI
```

Claude Code picks up the skills from `.claude/skills/` automatically.

### Use skills from another project

You can reference this repo's skills from any project by adding them to your personal Claude config at `~/.claude/skills/`. Copy or symlink the skill directories:

```bash
ln -s ~/Developer/Personal/mac-skills/.claude/skills/github-setup ~/.claude/skills/github-setup
```

## How skills work

Each skill is a `SKILL.md` file in `.claude/skills/<name>/`. The file contains:

- **Frontmatter** — `name` and `description` so Claude Code can discover and match it
- **Procedure** — Step-by-step instructions with built-in status checks (skips what's already done)
- **Checklist** — Verification items to confirm everything works

Skills are idempotent — safe to run on a fully configured Mac. Each step checks current state and only acts on what's missing.

## Adding a new skill

1. Create `.claude/skills/<skill-name>/SKILL.md`
2. Add YAML frontmatter with `name` and `description`
3. Write the procedure with status checks per step
4. Update the table in this README and in `CLAUDE.md`

## License

MIT
