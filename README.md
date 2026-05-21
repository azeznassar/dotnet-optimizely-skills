# optimizely-cms12-skills

Agent skills and orchestration agents for migrating legacy EPiServer/Optimizely CMS 11 (.NET 4.x) to Optimizely CMS 12 (.NET 8) using a fresh-build approach.

Works with Claude Code, GitHub Copilot, Gemini, Codex, and Junie.

---

## Installation

### Option A — Copy into your project (recommended)
```bash
git clone https://github.com/yourname/optimizely-cms12-skills.git
cp -r optimizely-cms12-skills/.claude your-solution/
```

### Option B — Git submodule
```bash
git submodule add https://github.com/yourname/optimizely-cms12-skills.git .optimizely-skills
cp -r .optimizely-skills/.claude .
```

### Option C — curl individual skills
```bash
mkdir -p .claude/skills/optimizely-content-type-auditor
curl -o .claude/skills/optimizely-content-type-auditor/SKILL.md \
  https://raw.githubusercontent.com/yourname/optimizely-cms12-skills/main/.claude/skills/optimizely-content-type-auditor/SKILL.md
```

---

## Agents (`.claude/agents/`)

Top-level orchestration agents from [managedcode/dotnet-skills](https://github.com/managedcode/dotnet-skills):

| Agent | Purpose |
|---|---|
| `dotnet-modernization` | **Start here** — routes all upgrade, migration, and legacy modernization work |
| `dotnet-router` | Classifies tasks and routes to the right skill |
| `dotnet-build` | Restore, build, pack, CI, diagnostics |
| `dotnet-data` | EF Core, EF6, migrations, query issues |
| `dotnet-review` | Code review, analyzers, testing, architecture |

---

## Skills (`.claude/skills/`)

### Optimizely CMS 12 — Phase 1: Audit (run against OLD codebase first)

| Skill | Trigger | Purpose |
|---|---|---|
| `optimizely-content-type-auditor` | "audit content types" | Inventories all PageData/BlockData/MediaData classes, GUIDs, properties |
| `optimizely-dependency-mapper` | "map NuGet packages to CMS 12" | Maps old EPiServer packages to v12 equivalents, flags removed ones |
| `optimizely-init-module-scanner` | "scan initialization modules" | Finds IInitializableModule, IInitializableHttpModule, scheduled jobs |
| `optimizely-route-auditor` | "audit routing" | Finds partial routers, custom routes, URL handlers |

### Optimizely CMS 12 — Phase 2: Build (run against NEW project)

| Skill | Trigger | Purpose |
|---|---|---|
| `optimizely-content-type-builder` | "create a content type" | Correct CMS 12 PageData/BlockData/MediaData patterns and attributes |
| `optimizely-startup-wiring` | "set up Program.cs for Optimizely" | AddCms(), middleware pipeline, appsettings, IInitializableModule |
| `optimizely-scheduled-job-migrator` | "create a scheduled job" | ScheduledJobBase, stop signals, GUID, breaking changes from v11 |
| `optimizely-di-registration` | "register a service in CMS 12" | IServiceCollection, IConfigurableModule, ServiceLocator rules |
| `optimizely-view-engine-migrator` | "migrate views" | PageController, BlockComponent, ViewComponent, tag helpers |
| `webconfig-to-appsettings` | "convert web.config" | web.config → appsettings.json, connection strings, custom sections |
| `globalasax-to-program` | "migrate Global.asax" | Application_Start/Error/BeginRequest → Program.cs equivalents |
| `httpmodule-to-middleware` | "convert IHttpModule" | IHttpModule/IHttpHandler → ASP.NET Core middleware and minimal APIs |
| `legacy-ioc-to-di` | "migrate StructureMap" | StructureMap/Windsor/Autofac → IServiceCollection |

### Optimizely CMS 12 — Phase 3: Validation

| Skill | Trigger | Purpose |
|---|---|---|
| `optimizely-content-schema-diff` | "compare content types old vs new" | Diffs old vs new inventories before DB migration — must be zero CRITICAL issues |

---

### .NET & ASP.NET Core (from [managedcode/dotnet-skills](https://github.com/managedcode/dotnet-skills))

| Skill | Trigger | Purpose |
|---|---|---|
| `aspnet-core` | Any ASP.NET Core task | Modern hosting, middleware, auth, routing, deployment patterns |
| `legacy-aspnet` | Reading old .NET 4.x code | Web Forms, legacy MVC, classic EPiServer patterns (audit/read only) |
| `modern-csharp` | Writing any C# | Records, pattern matching, nullable refs, version-aware C# |
| `minimal-apis` | New lightweight endpoints | Minimal API groups, TypedResults, filters |
| `web-api` | REST API controllers | [ApiController], model binding, validation, responses |
| `worker-services` | Background processing | IHostedService, BackgroundService patterns |
| `entity-framework-core` | EF Core data access | DbContext, migrations, queries, performance |
| `entity-framework6` | Reading old EF6 code | EF6 patterns for audit/migration planning |
| `optimizing-ef-core-queries` | Slow EF queries | N+1 fixes, tracking modes, compiled queries |
| `dotnet` | General .NET platform | SDK, runtime, project structure, tooling |
| `project-setup` | New project creation | Solution layout, .csproj, Directory.Build.props |
| `architecture` | Solution design | Clean architecture, dependency rules, layering |
| `code-review` | PR reviews | .NET-specific review patterns, analyzer usage |
| `thread-abort-migration` | Thread.Abort usage | Migrating Thread.Abort to CancellationToken |
| `migrate-nullable-references` | NRT enablement | Enabling nullable reference types incrementally |

---

## Migration Order

```
1.  dotnet-modernization agent  → routes all work automatically
2.  optimizely-content-type-auditor    → inventory JSON from old codebase
3.  optimizely-dependency-mapper       → package migration plan
4.  optimizely-init-module-scanner     → module + job migration notes
5.  optimizely-route-auditor           → routing migration notes
    ── NEW .NET 8 SOLUTION ──
6.  project-setup                      → solution/project structure
7.  optimizely-startup-wiring          → Program.cs, AddCms(), pipeline
8.  optimizely-content-type-builder    → rebuild all content types (use audit inventory)
9.  legacy-ioc-to-di                   → StructureMap/Windsor → IServiceCollection
10. optimizely-di-registration         → CMS-specific service registration
11. webconfig-to-appsettings           → config migration
12. globalasax-to-program              → bootstrap migration
13. httpmodule-to-middleware           → HTTP module migration
14. optimizely-view-engine-migrator    → views + block components
15. optimizely-scheduled-job-migrator  → scheduled jobs
16. optimizing-ef-core-queries         → data access tuning
17. migrate-nullable-references        → clean up nullability
18. optimizely-content-schema-diff     → MUST be zero CRITICAL before DB migration
19. DB migration on staging → content spot-check → production
```

---

## Credits

`.NET` skills in `.claude/skills/` (aspnet-core, legacy-aspnet, modern-csharp, etc.) and agents are copied verbatim from [managedcode/dotnet-skills](https://github.com/managedcode/dotnet-skills) (MIT License).

Optimizely CMS 12 skills are custom, built against the [Optimizely developer docs](https://docs.developers.optimizely.com/content-management-system/).
