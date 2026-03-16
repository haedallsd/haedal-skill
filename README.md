# haedal-skills

Haedal Protocol agent skills for **any AI coding agent** — interact with haSUI, haWAL, VeHaedal.

## Quick Start

```bash
# Install all Haedal skills (any agent)
npx skills add haedallsd/haedal-skill

# List available skills
npx skills add haedallsd/haedal-skill --list

# Install specific skills only
npx skills add haedallsd/haedal-skill --skill haedal-hasui --skill haedal-hawal --skill haedal-vehaedal

# Install globally (all projects)
npx skills add haedallsd/haedal-skill -g

# Install to specific agents
npx skills add haedallsd/haedal-skill -a cursor -a claude-code
```

## Available Skills

| Skill | Description |
|-------|-------------|
| `haedal-hasui` | haSUI — stake, withdraw, instant withdraw, claim |
| `haedal-hawal` | haWAL — stake, withdraw, instant withdraw, claim |
| `haedal-vehaedal` | VeHaedal — add stake, extend lock, claim rewards, decay control |

## Supported Agents

Skills are installed via the open [Agent Skills CLI](https://github.com/vercel-labs/add-skill) and work with **40+ agents**, including:

- **Cursor** — `.agents/skills/`
- **Claude Code** — `.claude/skills/`
- **Codex** — `.codex/skills/`
- **OpenCode** — `.agents/skills/`
- **GitHub Copilot** — `.agents/skills/`
- And many more — the CLI auto-detects your installed agents.

## Usage

After installation, restart your coding agent. It will automatically detect and use these skills when you mention related tasks:

- "Stake 100 SUI into haSUI"
- "Redeem 50 haWAL"
- "Stake haedal locked for 52 weeks"

## Managing Skills

```bash
# List installed skills
npx skills list

# Check for updates
npx skills check

# Update all skills
npx skills update

# Remove skills
npx skills remove haedal-hasui
```

## License

MIT
