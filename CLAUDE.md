# Optimizely CMS 11 → CMS 12 Migration

## Workspace Layout

```
migration-workspace/
├── old-solution/       ← CMS 11 source (READ ONLY)
├── new-solution/       ← CMS 12 Alloy-based fresh build


└── audit-output/       ← JSON inventories produced in Phase 1
```

## Absolute Rules

- **NEVER modify anything in `old-solution/`** — read-only reference only
- **NEVER copy code from old-solution directly** — always rewrite using skill patterns
- **ALWAYS preserve GUIDs exactly** from `audit-output/content-types.json`
- **ALWAYS check `audit-output/`** for existing inventory before re-scanning
- **ALWAYS check what Alloy already provides** before creating any content type, controller, or view
- **ALWAYS use item templates** (`dotnet new epi-cms-contenttype` etc.) to scaffold new files — never write from scratch
- **NEVER duplicate Alloy's existing content types** — extend or use them as base classes where appropriate
- **ALWAYS commit after each completed unit** (one content type, one module, one job)
- **NEVER proceed past Phase 3 validation** if any CRITICAL issues exist in the schema diff

## Current Migration State

```
CURRENT_PHASE=SETUP
# Valid values: SETUP | AUDIT | BUILD | VALIDATE
```

```
ALLOY_BASELINE_COMPLETE=false
# Set to true once you have read and understood what Alloy already provides
```

```
AUDIT_COMPLETE=false
# Set to true once all audit-output JSON files exist and have been reviewed
```

```
BUILD_PROGRESS=
# e.g. "startup,pages,blocks-teaser,blocks-hero,scheduled-jobs"
```

## Alloy Template — What It Already Provides

The new-solution is based on `dotnet new epi-alloy-mvc`. Alloy is Optimizely's official CMS 12 demo site. **Before building anything, understand what is already there.**

### Already wired up (DO NOT recreate)
- `Program.cs` with `AddCms()`, `AddCmsAspNetUi()`, `AddTinyMce()`, correct middleware order
- `appsettings.json` with CMS connection string placeholders
- ASP.NET Core Identity authentication setup
- `nuget.config` pointing to Optimizely NuGet feed
- `SiteViewLocationExpander` — adds `/Views/Shared/PagePartials` and `/Views/Shared/Blocks` as view lookup paths
- `TemplateCoordinator` — registers partial views to content types without requiring a controller

### Alloy existing content types (scan `Models/` on session start)
Alloy ships with example pages and blocks. Common ones include:
- Pages: `StartPage`, `ArticlePage`, `ProductPage`, `SearchPage`, `LandingPage`, `NotFoundPage`
- Blocks: `TeaserBlock`, `VideoBlock`, `PageListBlock`, `ButtonBlock`, `FormContainerBlock`
- Media: `ImageFile`, `VideoFile`

**Check what actually exists in `Models/` — the list above may vary by template version.**

### Alloy folder conventions (follow these for all new code)
```
new-solution/
├── Models/
│   ├── Pages/          ← PageData subclasses
│   ├── Blocks/         ← BlockData subclasses
│   └── Media/          ← MediaData, ImageData, VideoData subclasses
├── Controllers/        ← PageController<T> classes
├── Views/
│   ├── {PageTypeName}/ ← Index.cshtml per page type
│   ├── Shared/
│   │   ├── Blocks/     ← Partial views for blocks (no controller needed for simple blocks)
│   │   └── PagePartials/
├── Business/           ← Services, helpers, ContentLoader wrappers
└── Infrastructure/
    ├── Initialization/ ← IInitializableModule implementations
    └── ScheduledJobs/  ← ScheduledJobBase implementations
```

### Item templates available (USE THESE to scaffold new files)
```bash
dotnet new epi-cms-contenttype       # new content type class
dotnet new epi-cms-pagecontroller    # new PageController<T>
dotnet new epi-cms-initializationmodule  # new IInitializableModule
dotnet new epi-cms-job               # new ScheduledJobBase
dotnet new epi-cms-razorpage         # new Razor page
```

Always scaffold with item templates first, then fill in properties — never write a content type class from scratch.

## On Session Start — Always Do This First

1. Read this `CLAUDE.md` in full
2. Check `CURRENT_PHASE` and `BUILD_PROGRESS`
3. Check `audit-output/` for existing inventory files
4. List `Models/Pages/`, `Models/Blocks/`, `Models/Media/` in new-solution to see what is already built
5. Report current status before taking any action
6. Ask which task to work on next, or proceed if given permission

---

## Phase 0 — Setup

**Run once before anything else. Skip if new-solution already exists.**

```bash
# From migration-workspace/ root

# Install Optimizely templates and CLI
dotnet new install EPiServer.Templates
dotnet tool install EPiServer.Net.Cli --global --add-source https://nuget.optimizely.com/feed/packages.svc/

# Create Alloy as the new solution
dotnet new epi-alloy-mvc --name MyProject --output new-solution

# Install skills at workspace root (not inside new-solution)
# Claude runs from here and can see both codebases directly
git init
git submodule add https://github.com/azeznassar/dotnet-optimizely-skills .optimizely-skills
cp -r .optimizely-skills/.claude .
cp .optimizely-skills/CLAUDE.md .
mkdir -p audit-output

git add . && git commit -m "chore: migration workspace setup"
```

**After setup:**
- Update `CURRENT_PHASE=AUDIT`
- Update `ALLOY_BASELINE_COMPLETE=true` once you have read Models/ and noted what Alloy provides

---

## Phase 1 — Audit

**Goal:** Produce structured JSON inventories of the old codebase. No new code yet.

### Step 1 — Alloy Baseline Inventory
Before auditing old-solution, document what Alloy already provides:
```
List all files in Models/, Controllers/, Infrastructure/.
For each content type found, record: className, baseClass, GUID, properties.
Save to audit-output/alloy-baseline.json
This will be used in Phase 2 to avoid duplicating existing types.
```

### Step 2 — Content Types
Use skill: `optimizely-content-type-auditor`
```
Scan old-solution for all [ContentType] classes.
Record className, baseClass, GUID, all properties and their types.
Flag deprecated types (UrlBuilder, PageType property type, etc).
Cross-reference with audit-output/alloy-baseline.json —
  mark any type that is equivalent to an Alloy type as "alloy-equivalent".
Save to audit-output/content-types.json
```

### Step 3 — Init Modules & Scheduled Jobs
Use skill: `optimizely-init-module-scanner`
```
Scan old-solution for IInitializableModule, IConfigurableModule,
IInitializableHttpModule, and ScheduledJobBase.
Flag IInitializableHttpModule (removed in CMS 12).
Save to audit-output/modules.json
```

### Step 4 — NuGet Packages
Use skill: `optimizely-dependency-mapper`
```
Read old-solution packages.config and .csproj files.
Map every EPiServer.* package to CMS 12 equivalent.
Flag any package Alloy already includes (no need to add again).
Save to audit-output/packages.json
```

### Step 5 — Routing
Use skill: `optimizely-route-auditor`
```
Scan old-solution for RouteTable.Routes registrations,
IPartialRouter implementations, custom URL handlers.
Save to audit-output/routes.json
```

**When audit is complete:**
- Update `AUDIT_COMPLETE=true`
- Update `CURRENT_PHASE=BUILD`
- Print a human-readable summary: total content types (net new vs alloy-equivalent), breaking changes, modules needing rewrite

---

## Phase 2 — Build

**Goal:** Add your content types and infrastructure on top of Alloy.
Alloy's existing code stays intact — you are adding to it, not replacing it.

### Step 1 — Adapt Startup (if needed)
Skill: `optimizely-startup-wiring`, `webconfig-to-appsettings`
```
Alloy already has a working Program.cs — DO NOT recreate it.
Only add what is missing from old-solution that Alloy does not cover:
  - Additional services registered in old Global.asax or IoC config
  - Connection strings and app settings from old-solution web.config
  - Any custom TinyMCE configuration
  - SchedulerOptions if scheduled jobs are present
Update appsettings.json with migrated values only.
Commit: "feat: startup adaptations from legacy config"
```

### Step 2 — DI Registration
Skills: `legacy-ioc-to-di`, `optimizely-di-registration`
```
Read audit-output/modules.json for IConfigurableModule registrations.
Read old-solution for StructureMap/Windsor/Autofac container setup.
Add new registrations to Program.cs or a new IConfigurableModule.
Do NOT touch Alloy's existing DI setup.
Commit: "feat: DI registration migrated from legacy IoC"
```

### Step 3 — Content Types (Pages)
Skill: `optimizely-content-type-builder`
```
Read audit-output/content-types.json — process pages where baseClass == "PageData".
For each page type:
  - If marked "alloy-equivalent": skip or add only the missing properties to the Alloy type
  - If net-new: scaffold with `dotnet new epi-cms-contenttype -n {TypeName} -o Models/Pages/`
    then add properties, GUID, [Display] attributes from audit
  - Preserve GUID exactly
  - Replace UrlBuilder with Url
Commit after every 5 types: "feat: add {TypeName} content type"
```

### Step 4 — Content Types (Blocks)
Skill: `optimizely-content-type-builder`
```
Same as Step 3 but for baseClass == "BlockData".
Scaffold: dotnet new epi-cms-contenttype -n {TypeName} -o Models/Blocks/
```

### Step 5 — Content Types (Media)
Skill: `optimizely-content-type-builder`
```
Same as Step 3 but for MediaData, ImageData, VideoData.
Output to Models/Media/
```

### Step 6 — Init Modules
Skill: `optimizely-init-module-scanner`
```
Read audit-output/modules.json.
For each IInitializableModule:
  Scaffold: dotnet new epi-cms-initializationmodule -n {Name} -o Infrastructure/Initialization/
  Then implement Initialize/Uninitialize from the old-solution equivalent.
For each IInitializableHttpModule: rewrite as IStartupFilter or middleware.
Commit: "feat: initialization modules migrated"
```

### Step 7 — Scheduled Jobs
Skill: `optimizely-scheduled-job-migrator`
```
Read audit-output/modules.json for ScheduledJobBase entries.
For each job:
  Scaffold: dotnet new epi-cms-job -n {JobName} -o Infrastructure/ScheduledJobs/
  Implement Execute() from the old-solution equivalent.
  Preserve ScheduledPlugIn GUID exactly.
  Add IsStoppable = true and _stopSignaled cooperative check.
  Remove HttpContext.Current and PrincipalInfo.CurrentPrincipal usage.
Add SchedulerOptions to Program.cs if not already present.
Commit: "feat: scheduled jobs migrated"
```

### Step 8 — Routing
Skill: `optimizely-route-auditor`
```
Read audit-output/routes.json.
Alloy handles most standard routing — only add custom routes that are not covered.
Register IPartialRouter via builder.Services.AddSingleton.
Commit: "feat: custom routing migrated"
```

### Step 9 — Controllers & Views
Skills: `optimizely-view-engine-migrator`, `aspnet-core`
```
For each net-new PageData type:
  Scaffold: dotnet new epi-cms-pagecontroller -n {TypeName}Controller -o Controllers/
  Implement Index action.
  Create Views/{TypeName}/Index.cshtml following Alloy view conventions.
For each BlockData type needing complex rendering:
  Create ViewComponent in Components/ or partial view in Views/Shared/Blocks/
  Simple blocks: just a partial view in Views/Shared/Blocks/ — no controller needed
    (Alloy's TemplateCoordinator will pick it up automatically by name convention)
Commit per group: "feat: {section} controllers and views"
```

### Step 10 — Global.asax / HTTP Modules
Skills: `globalasax-to-program`, `httpmodule-to-middleware`
```
Read old-solution Global.asax and HttpModule implementations.
Migrate remaining application lifecycle logic to Program.cs middleware or IHostedService.
Alloy already handles the core Optimizely bootstrap — only add custom logic.
Commit: "feat: application bootstrap migrated"
```

**When build phase is complete:**
- Update `CURRENT_PHASE=VALIDATE`

---

## Phase 3 — Validate

**Goal:** Verify nothing was missed before touching the database.

### Step 1 — Schema Diff
Skill: `optimizely-content-schema-diff`
```
Compare audit-output/content-types.json against Models/ in new-solution.
For alloy-equivalent types: verify any added properties are present.
For net-new types: verify all properties and GUIDs match exactly.
STOP if any CRITICAL issues exist. Do not proceed.
Save to audit-output/schema-diff.json
```

### Step 2 — Build Verification
```
Run: dotnet build
Fix all compiler errors.
Run: dotnet test (if tests exist)
Commit: "chore: validation fixes"
```

### Step 3 — Pre-DB-Migration Checklist
- [ ] Zero CRITICAL issues in schema-diff.json
- [ ] `dotnet build` passes with zero errors
- [ ] All GUIDs verified
- [ ] Staging DB migration tested
- [ ] Content spot-checked in CMS 12 editor on staging

---

## Parallel Agent Mode

For large codebases, split Phase 2 across terminals (non-overlapping directories):

| Terminal | Task | Writes To |
|---|---|---|
| 1 | Pages content types | Models/Pages/ |
| 2 | Blocks content types | Models/Blocks/ |
| 3 | Scheduled jobs + init modules | Infrastructure/ |
| 4 | Startup adaptations + DI | Program.cs, appsettings.json |

---

## Skill Reference

| Task | Skill |
|---|---|
| Audit content types | `optimizely-content-type-auditor` |
| Audit packages | `optimizely-dependency-mapper` |
| Audit modules/jobs | `optimizely-init-module-scanner` |
| Audit routing | `optimizely-route-auditor` |
| Build content types | `optimizely-content-type-builder` |
| Adapt startup/Program.cs | `optimizely-startup-wiring` |
| Build scheduled jobs | `optimizely-scheduled-job-migrator` |
| Build DI registration | `optimizely-di-registration` |
| Build views/controllers | `optimizely-view-engine-migrator` |
| Migrate web.config | `webconfig-to-appsettings` |
| Migrate Global.asax | `globalasax-to-program` |
| Migrate IHttpModule | `httpmodule-to-middleware` |
| Migrate IoC container | `legacy-ioc-to-di` |
| Validate schema | `optimizely-content-schema-diff` |
| .NET 8 patterns | `aspnet-core`, `modern-csharp` |
| EF Core | `optimizing-ef-core-queries` |
