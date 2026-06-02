# claude-skills

Claude Code skills for Extend merchant implementation and operations.

## Skills

| Skill | Purpose |
|---|---|
| `add-merchant` | Scaffold a new merchant project file from full research sweep |
| `agenda-builder` | Build a researched meeting agenda for any merchant call |
| `ccerp` | End-to-end CCERP ticket creation workflow |
| `custom-requirements` | Track and update custom product requirements per merchant |
| `go-live-check` | Go-live readiness check for a named merchant |
| `merchant-sync` | Lightweight end-of-week merchant project file sync |
| `project-pulse` | Fast health snapshot across all active merchants |
| `raid-log` | Capture and update RAID log items per merchant |
| `refresh-merchant` | Full re-sync of an existing merchant project file |

## Usage

Install this marketplace in Claude Code:

```
/plugin add-marketplace samirextend/claude-skills
```

Then install the plugin:

```
/plugin install extend-ops@samirextend/claude-skills
```

## Updating a skill

1. Edit the relevant `SKILL.md` file in `plugins/extend-ops/skills/<skill-name>/`
2. Commit and push
3. Anyone using this marketplace runs `/plugin update` to get the latest
