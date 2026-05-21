# dotnet-optimizely-skills

Agent skills and orchestration agents for migrating EPiServer/Optimizely CMS 11 (.NET 4.x) to CMS 12 (.NET 8).

Works with Claude Code, GitHub Copilot, Gemini, Codex, and Junie.

---

## Automated Workspace Setup

Skills live in the **workspace root** so Claude sees both codebases from one directory.

```
migration-workspace/
├── .claude/            ← skills + agents installed here
├── CLAUDE.md           ← migration brain, lives here too
├── old-solution/       ← your CMS 11 source (read-only)
├── new-solution/       ← CMS 12 Alloy-based fresh build
└── audit-output/       ← JSON inventories (auto-generated)
```

**1. Create the workspace:**
```bash
mkdir migration-workspace && cd migration-workspace

# old solution (read-only reference)
cp -r /path/to/old-cms11-solution ./old-solution

# new solution from official Alloy CMS 12 template
dotnet new install EPiServer.Templates
dotnet tool install EPiServer.Net.Cli --global \
  --add-source https://nuget.optimizely.com/feed/packages.svc/
dotnet new epi-alloy-mvc --name MyProject --output new-solution

# install skills at workspace root
git init
git submodule add https://github.com/azeznassar/dotnet-optimizely-skills .optimizely-skills
cp -r .optimizely-skills/.claude .
cp .optimizely-skills/CLAUDE.md .
mkdir audit-output

git add . && git commit -m "chore: migration workspace setup"
```

**2. Run from the workspace root:**
```bash
# Phase 1 — Audit
claude "Read CLAUDE.md. Run all Phase 1 audit steps and save output to audit-output/."

# Phase 2 — Build (single session)
claude "Read CLAUDE.md. Run Phase 2. Commit after each step."

# Phase 2 — Build (parallel terminals, faster for large codebases)
claude "Read CLAUDE.md. Build all PageData types from audit-output/content-types.json into new-solution/Models/Pages/"
claude "Read CLAUDE.md. Build all BlockData types from audit-output/content-types.json into new-solution/Models/Blocks/"
claude "Read CLAUDE.md. Migrate scheduled jobs and init modules from audit-output/modules.json"

# Phase 3 — Validate
claude "Read CLAUDE.md. Run Phase 3 validation. Stop if any CRITICAL issues exist."
```

---

## Manual Usage

To use individual skills without the automated workspace:

```bash
git clone https://github.com/azeznassar/dotnet-optimizely-skills.git
cp -r dotnet-optimizely-skills/.claude your-project/
```

---

## Agents (`.claude/agents/`)

| Agent | Purpose |
|---|---|
| `dotnet-modernization` | Routes all migration and legacy modernization work |
| `dotnet-router` | Classifies tasks and routes to the right skill |
| `dotnet-build` | Restore, build, pack, CI, diagnostics |
| `dotnet-data` | EF Core, EF6, migrations, query issues |
| `dotnet-review` | Code review, analyzers, testing, architecture |

---

## Skills (`.claude/skills/`)

### Optimizely CMS 12 — Phase 1: Audit

| Skill | Purpose |
|---|---|
| `optimizely-content-type-auditor` | Inventories all PageData/BlockData/MediaData classes, GUIDs, properties |
| `optimizely-dependency-mapper` | Maps old EPiServer packages to v12 equivalents, flags removed ones |
| `optimizely-init-module-scanner` | Finds IInitializableModule, IInitializableHttpModule, scheduled jobs |
| `optimizely-route-auditor` | Finds partial routers, custom routes, URL handlers |

### Optimizely CMS 12 — Phase 2: Build

| Skill | Purpose |
|---|---|
| `optimizely-content-type-builder` | Correct CMS 12 PageData/BlockData/MediaData patterns and attributes |
| `optimizely-startup-wiring` | AddCms(), middleware pipeline, appsettings, IInitializableModule |
| `optimizely-scheduled-job-migrator` | ScheduledJobBase, stop signals, GUID, breaking changes from v11 |
| `optimizely-di-registration` | IServiceCollection, IConfigurableModule, ServiceLocator rules |
| `optimizely-view-engine-migrator` | PageController, BlockComponent, ViewComponent, tag helpers |
| `webconfig-to-appsettings` | web.config → appsettings.json, connection strings, custom sections |
| `globalasax-to-program` | Application_Start/Error/BeginRequest → Program.cs equivalents |
| `httpmodule-to-middleware` | IHttpModule/IHttpHandler → ASP.NET Core middleware |
| `legacy-ioc-to-di` | StructureMap/Windsor/Autofac → IServiceCollection |

### Optimizely CMS 12 — Phase 3: Validation

| Skill | Purpose |
|---|---|
| `optimizely-content-schema-diff` | Diffs old vs new content type inventories — must be zero CRITICAL before DB migration |

### .NET & ASP.NET Core (from [managedcode/dotnet-skills](https://github.com/managedcode/dotnet-skills))

| Skill | Purpose |
|---|---|
| `aspnet-core` | Modern hosting, middleware, auth, routing, deployment |
| `legacy-aspnet` | Reading old .NET 4.x patterns during audit |
| `modern-csharp` | Records, pattern matching, nullable refs, version-aware C# |
| `minimal-apis` | Minimal API groups, TypedResults, filters |
| `web-api` | [ApiController], model binding, validation |
| `worker-services` | IHostedService, BackgroundService |
| `entity-framework-core` | DbContext, migrations, queries, performance |
| `entity-framework6` | EF6 patterns for audit/migration planning |
| `optimizing-ef-core-queries` | N+1 fixes, tracking modes, compiled queries |
| `dotnet` | SDK, runtime, project structure, tooling |
| `project-setup` | Solution layout, .csproj, Directory.Build.props |
| `architecture` | Clean architecture, dependency rules, layering |
| `code-review` | .NET-specific review patterns |
| `thread-abort-migration` | Thread.Abort → CancellationToken |
| `migrate-nullable-references` | Enabling nullable reference types incrementally |

---

## Credits

`.NET` skills and agents copied verbatim from [managedcode/dotnet-skills](https://github.com/managedcode/dotnet-skills) (MIT License).
Optimizely CMS 12 skills built against the [Optimizely CMS 12 developer docs](https://docs.developers.optimizely.com/content-management-system/).
