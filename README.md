# dotnet-optimizely-skills

Agent skills and orchestration agents for migrating EPiServer/Optimizely CMS 11 (.NET 4.x) to CMS 12 (.NET 8) using a fresh-build approach.

Works with Claude Code, GitHub Copilot, Gemini, Codex, and Junie.

---

## Quick Start

### Option A — Automated Workspace (recommended for full migrations)

Set up a workspace with old and new codebases side by side, and let Claude Code drive the entire migration:

```
migration-workspace/
├── old-solution/       ← your CMS 11 source (read-only)
├── new-solution/       ← your fresh .NET 8 project
│   ├── .claude/        ← copy skills + agents here (see installation)
│   └── CLAUDE.md       ← copy from this repo — drives the automation
└── audit-output/       ← JSON inventories produced automatically
```

**1. Set up the workspace:**
```bash
mkdir migration-workspace
cd migration-workspace
# copy your old solution in
cp -r /path/to/old-cms11-solution ./old-solution

# create new .NET 8 solution
dotnet new web -n MyProject
mv MyProject new-solution

# install skills
cd new-solution
git submodule add https://github.com/azeznassar/dotnet-optimizely-skills .optimizely-skills
cp -r .optimizely-skills/.claude .
cp .optimizely-skills/CLAUDE.md .
mkdir -p ../audit-output
```

**2. Run Phase 1 — Audit (fully automated):**
```bash
cd new-solution
claude "Read CLAUDE.md. We are starting Phase 1. Run all four audit steps in order and save output to ../audit-output/. Report when complete."
```

**3. Review audit output, then run Phase 2 — Build:**
```bash
# Single session (sequential)
claude "Read CLAUDE.md and ../audit-output/. We are in Phase 2. Start from Step 1 and work through all steps in order. Commit after each step."

# OR parallel sessions (faster for large codebases)
# Terminal 1 — pages
claude "Read CLAUDE.md. Build all PageData content types from ../audit-output/content-types.json into src/Models/Pages/"
# Terminal 2 — blocks  
claude "Read CLAUDE.md. Build all BlockData content types from ../audit-output/content-types.json into src/Models/Blocks/"
# Terminal 3 — infrastructure
claude "Read CLAUDE.md. Migrate all scheduled jobs and init modules from ../audit-output/modules.json"
# Terminal 4 — startup
claude "Read CLAUDE.md. Set up Program.cs, DI registration, and appsettings.json"
```

**4. Run Phase 3 — Validate:**
```bash
claude "Read CLAUDE.md. Run Phase 3 validation. Do not proceed past schema diff if any CRITICAL issues exist."
```

> **CLAUDE.md is the brain.** It persists migration state across sessions, enforces rules (never modify old-solution, always preserve GUIDs), tracks build progress, and tells Claude exactly which skill to use at each step.

---

### Option B — Manual skill-by-skill

Use individual skills on demand without the automated workspace:

```bash
# Install
git clone https://github.com/azeznassar/dotnet-optimizely-skills.git
cp -r dotnet-optimizely-skills/.claude your-solution/
```

Then invoke skills directly in Claude Code:
```
"Run the optimizely-content-type-auditor skill against src/Models/"
"Use the optimizely-content-type-builder skill to create a new ArticlePage"
"Use the optimizely-startup-wiring skill to set up Program.cs"
```

---

### Option C — curl individual skills
```bash
mkdir -p .claude/skills/optimizely-content-type-builder
curl -o .claude/skills/optimizely-content-type-builder/SKILL.md \
  https://raw.githubusercontent.com/azeznassar/dotnet-optimizely-skills/main/.claude/skills/optimizely-content-type-builder/SKILL.md
```

---

## Agents (`.claude/agents/`)

Top-level orchestration agents from [managedcode/dotnet-skills](https://github.com/managedcode/dotnet-skills):

| Agent | Purpose |
|---|---|
| `dotnet-modernization` | **Start here for ad-hoc work** — routes all upgrade, migration, and legacy modernization tasks |
| `dotnet-router` | Classifies tasks and routes to the right skill |
| `dotnet-build` | Restore, build, pack, CI, diagnostics |
| `dotnet-data` | EF Core, EF6, migrations, query issues |
| `dotnet-review` | Code review, analyzers, testing, architecture |

---

## Skills (`.claude/skills/`)

### Optimizely CMS 12 — Phase 1: Audit

| Skill | Trigger | Purpose |
|---|---|---|
| `optimizely-content-type-auditor` | "audit content types" | Inventories all PageData/BlockData/MediaData classes, GUIDs, properties |
| `optimizely-dependency-mapper` | "map NuGet packages to CMS 12" | Maps old EPiServer packages to v12 equivalents, flags removed ones |
| `optimizely-init-module-scanner` | "scan initialization modules" | Finds IInitializableModule, IInitializableHttpModule, scheduled jobs |
| `optimizely-route-auditor` | "audit routing" | Finds partial routers, custom routes, URL handlers |

### Optimizely CMS 12 — Phase 2: Build

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
| `optimizely-content-schema-diff` | "compare content types old vs new" | Diffs old vs new inventories — must be zero CRITICAL issues before DB migration |

### .NET & ASP.NET Core (from [managedcode/dotnet-skills](https://github.com/managedcode/dotnet-skills))

| Skill | Purpose |
|---|---|
| `aspnet-core` | Modern hosting, middleware, auth, routing, deployment patterns |
| `legacy-aspnet` | Reading old .NET 4.x patterns during audit (read-only reference) |
| `modern-csharp` | Records, pattern matching, nullable refs, version-aware C# |
| `minimal-apis` | Minimal API groups, TypedResults, filters |
| `web-api` | [ApiController], model binding, validation, responses |
| `worker-services` | IHostedService, BackgroundService patterns |
| `entity-framework-core` | DbContext, migrations, queries, performance |
| `entity-framework6` | EF6 patterns for audit/migration planning |
| `optimizing-ef-core-queries` | N+1 fixes, tracking modes, compiled queries |
| `dotnet` | SDK, runtime, project structure, tooling |
| `project-setup` | Solution layout, .csproj, Directory.Build.props |
| `architecture` | Clean architecture, dependency rules, layering |
| `code-review` | .NET-specific review patterns, analyzer usage |
| `thread-abort-migration` | Thread.Abort → CancellationToken |
| `migrate-nullable-references` | Enabling nullable reference types incrementally |

---

## Migration Order (manual reference)

```
── PHASE 1: AUDIT ──────────────────────────────────────────────
1.  optimizely-content-type-auditor    → audit-output/content-types.json
2.  optimizely-dependency-mapper       → audit-output/packages.json
3.  optimizely-init-module-scanner     → audit-output/modules.json
4.  optimizely-route-auditor           → audit-output/routes.json

── PHASE 2: BUILD ──────────────────────────────────────────────
5.  project-setup                      → .csproj, solution structure
6.  optimizely-startup-wiring          → Program.cs, AddCms(), pipeline
7.  legacy-ioc-to-di                   → StructureMap/Windsor → IServiceCollection
8.  optimizely-di-registration         → CMS-specific service registration
9.  webconfig-to-appsettings           → config migration
10. optimizely-content-type-builder    → all PageData types
11. optimizely-content-type-builder    → all BlockData types
12. optimizely-content-type-builder    → all MediaData types
13. optimizely-init-module-scanner     → IInitializableModule migration
14. httpmodule-to-middleware           → IHttpModule → middleware
15. globalasax-to-program             → Global.asax → Program.cs
16. optimizely-view-engine-migrator    → controllers + views
17. optimizely-scheduled-job-migrator  → scheduled jobs
18. optimizely-route-auditor           → routing migration

── PHASE 3: VALIDATE ───────────────────────────────────────────
19. optimizely-content-schema-diff     → MUST be zero CRITICAL issues
20. dotnet build                       → MUST succeed
21. DB migration on staging
22. Content spot-check in CMS 12 editor
23. Production deploy
```

---

## Credits

`.NET` skills (aspnet-core, legacy-aspnet, modern-csharp, etc.) and agents are from [managedcode/dotnet-skills](https://github.com/managedcode/dotnet-skills) (MIT License).

Optimizely CMS 12 skills are custom, built against the [Optimizely CMS 12 developer docs](https://docs.developers.optimizely.com/content-management-system/).
