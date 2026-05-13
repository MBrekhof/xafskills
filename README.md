# XAF Skills for Claude Code

![XAF Skills Overview](xafskills-overview.png)

Hard-learned lessons from 15+ DevExpress XAF projects, distilled into reusable [Claude Code skills](https://docs.anthropic.com/en/docs/claude-code).

These skills prevent AI coding agents from hitting the silent gotchas that make XAF + EF Core development painful.

## Skills

### Core

| Skill | What it covers |
|---|---|
| **xaf-efcore-entities** | Entity authoring: virtual properties, no OwnsOne, BaseObjectInt, ObservableCollection, decimal precision, PostgreSQL DateTime trap, GCRecord indexes, ValueConverters |
| **xaf-blazor-startup** | Startup.cs configuration: service registration order, middleware pipeline, JWT auth, OData/Web API, module lifecycle, non-persistent objects |
| **xaf-security** | Permissions: type/object/member-level security, role export/import, PermissionsReloadMode, background job auth, CurrentUserIdOperator |
| **xaf-reporting** | ReportsV2: parameter objects, Visible=false gotcha, GetCriteria() vs FilterString, PredefinedReportsUpdater |
| **devexpress-xaf-docker** | Containerizing XAF Blazor: SkiaSharp native deps, DevExpress version pinning, PostgreSQL/MySQL provider setup, schema initialization |

### Patterns

| Skill | What it covers |
|---|---|
| **xaf-hangfire-jobs** | Command/Handler jobs with zero XAF dependency, HangfireJobDispatcher vs DirectJobDispatcher, JobDefinition entity, XafJobScopeInitializer service-account auth, JobSyncService startup reconciliation |
| **xaf-search-panels** | Configurable advanced-search popups with generated `[DomainComponent]` DTOs — and why runtime Roslyn compilation is incompatible with AddSecuredEFCore |
| **xaf-conditional-appearance** | Data-driven appearance rules from the database via AppearanceController.CollectAppearanceRules + IAppearanceRuleProperties adapter, with immediate-effect cache invalidation |
| **xaf-navigation-hub** | Card-based DashboardView launchpad as startup view: IModelNavigationHub, permission-filtered tiles via ShowNavigationItemController, per-user pinned favorites |
| **xaf-environment-auth** | SSO vs password authentication switched by ASPNETCORE_ENVIRONMENT (not #if DEBUG), with the HangfireJob service-account carve-out |
| **xaf-playwright-testing** | E2E testing XAF Blazor with Playwright + NUnit: AuthenticatedTestBase, multi-fallback selectors, screenshot-on-failure, NetworkIdle timing |

## Installation

Copy the skill folders to your Claude Code skills directory:

```bash
# Global (all projects) — pick the skills you want
cp -r xaf-efcore-entities ~/.claude/skills/
cp -r xaf-blazor-startup ~/.claude/skills/
cp -r xaf-security ~/.claude/skills/
cp -r xaf-reporting ~/.claude/skills/
cp -r devexpress-xaf-docker ~/.claude/skills/
cp -r xaf-hangfire-jobs ~/.claude/skills/
cp -r xaf-search-panels ~/.claude/skills/
cp -r xaf-conditional-appearance ~/.claude/skills/
cp -r xaf-navigation-hub ~/.claude/skills/
cp -r xaf-environment-auth ~/.claude/skills/
cp -r xaf-playwright-testing ~/.claude/skills/

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
