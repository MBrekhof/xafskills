# XAF Skills for Claude Code

Hard-learned lessons from 15+ DevExpress XAF projects, distilled into reusable [Claude Code skills](https://docs.anthropic.com/en/docs/claude-code).

These skills prevent AI coding agents from hitting the silent gotchas that make XAF + EF Core development painful.

## Skills

| Skill | What it covers |
|---|---|
| **xaf-efcore-entities** | Entity authoring: virtual properties, no OwnsOne, BaseObjectInt, ObservableCollection, decimal precision, PostgreSQL DateTime trap, GCRecord indexes, ValueConverters |
| **xaf-blazor-startup** | Startup.cs configuration: service registration order, middleware pipeline, JWT auth, OData/Web API, module lifecycle, non-persistent objects |
| **xaf-security** | Permissions: type/object/member-level security, role export/import, PermissionsReloadMode, background job auth, CurrentUserIdOperator |
| **xaf-reporting** | ReportsV2: parameter objects, Visible=false gotcha, GetCriteria() vs FilterString, PredefinedReportsUpdater |

## Installation

Copy the skill folders to your Claude Code skills directory:

```bash
# Global (all projects)
cp -r xaf-efcore-entities ~/.claude/skills/
cp -r xaf-blazor-startup ~/.claude/skills/
cp -r xaf-security ~/.claude/skills/
cp -r xaf-reporting ~/.claude/skills/

# Or per-project
cp -r xaf-efcore-entities /path/to/your/project/.claude/skills/
```

Skills trigger automatically when Claude Code works on matching tasks.

## Requirements

- DevExpress XAF v25.2+ with EF Core
- .NET 8.0 or .NET 9.0

## Source

Mined from real production projects covering: dynamic assembly loading, runtime entity creation, Hangfire integration, Elsa workflows, PostgreSQL partitioning, role management, navigation hubs, AI chat integration, report parameter generation, and more.

## License

MIT
