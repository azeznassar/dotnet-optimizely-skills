# Optimizely CMS 11 → CMS 12 Migration

## Workspace Layout

```
migration-workspace/
├── old-solution/       ← CMS 11 source — READ ONLY, never modify
├── new-solution/       ← CMS 12 fresh build — all new code goes here
│   ├── .claude/        ← skills + agents from dotnet-optimizely-skills
│   └── CLAUDE.md       ← this file
└── audit-output/       ← JSON inventories produced in Phase 1
```

## Absolute Rules

- **NEVER modify anything in `../old-solution/`** — it is a read-only reference
- **NEVER copy code from old-solution directly** — always rewrite using the skill patterns
- **ALWAYS preserve GUIDs exactly** from `../audit-output/content-types.json` when building content types
- **ALWAYS check `../audit-output/`** for existing inventory before re-scanning old-solution
- **ALWAYS commit after each completed unit of work** (one content type, one module, one job)
- **NEVER proceed past Phase 3 validation** if any CRITICAL issues exist in the schema diff report
- All content type properties MUST be `public virtual`
- Use constructor injection everywhere — no ServiceLocator except where explicitly documented

## Current Migration State

```
CURRENT_PHASE=AUDIT
# Valid values: AUDIT | BUILD | VALIDATE
# Update this line manually as you progress through phases
```

```
AUDIT_COMPLETE=false
# Set to true once all audit-output JSON files exist and have been reviewed
```

```
BUILD_PROGRESS=
# Track completed sections e.g: "startup,content-types-pages,content-types-blocks"
```

## On Session Start — Always Do This First

1. Read `CLAUDE.md` (this file) to understand current phase and rules
2. Check which audit files exist in `../audit-output/`:
   - `content-types.json` — content type inventory
   - `modules.json` — init modules and scheduled jobs inventory
   - `packages.json` — NuGet package mapping
   - `routes.json` — routing inventory
   - `schema-diff.json` — validation report (Phase 3 only)
3. Check `BUILD_PROGRESS` above to see what is already done
4. Check `src/Models/` in the new solution to see what content types are already built
5. Report current status before taking any action — "X of Y content types built, currently on Z"
6. Ask which task to work on next, or proceed with the next logical item if given permission to continue

## Phase 1 — Audit

**Goal:** Produce structured JSON inventories of the old codebase. Do not write any new code.

### Run in order:

**Step 1 — Content Types**
Use skill: `optimizely-content-type-auditor`
```
Scan ../old-solution for all classes decorated with [ContentType].
Classify each by base class (PageData, BlockData, MediaData, ImageData, VideoData).
Extract all public virtual properties, their types, and all attributes.
Record [ContentType(GUID=...)] values exactly — these must be preserved in CMS 12.
Flag any deprecated property types (UrlBuilder, PageType, etc).
Save full structured output to ../audit-output/content-types.json
Print a summary: total types, pages, blocks, media, flagged items.
```

**Step 2 — Init Modules & Scheduled Jobs**
Use skill: `optimizely-init-module-scanner`
```
Scan ../old-solution for all IInitializableModule, IConfigurableModule,
IInitializableHttpModule implementations and ScheduledJobBase subclasses.
Flag anything using HttpContext.Current, PrincipalInfo.CurrentPrincipal,
IInitializableHttpModule (removed in CMS 12).
Save output to ../audit-output/modules.json
```

**Step 3 — NuGet Packages**
Use skill: `optimizely-dependency-mapper`
```
Read ../old-solution/**/packages.config and **/*.csproj.
Map every EPiServer.* package to its CMS 12 equivalent.
Flag packages with no CMS 12 replacement.
Save output to ../audit-output/packages.json
```

**Step 4 — Routing**
Use skill: `optimizely-route-auditor`
```
Scan ../old-solution for RouteTable.Routes registrations,
IPartialRouter implementations, and custom URL handlers.
Save output to ../audit-output/routes.json
```

**When audit is complete:**
- Update `AUDIT_COMPLETE=true` in this file
- Update `CURRENT_PHASE=BUILD`
- Print a human-readable audit summary covering: total content types, breaking changes found, packages needing replacement, modules needing rewrite

---

## Phase 2 — Build

**Goal:** Construct the CMS 12 solution from scratch using audit output as the blueprint.
Work in the order below. Commit after each step.

### Step 1 — Project & Startup
Skills: `project-setup`, `optimizely-startup-wiring`, `webconfig-to-appsettings`
```
1. Verify new-solution .csproj targets net8.0 with correct CMS 12 package references
   (cross-reference ../audit-output/packages.json for required packages)
2. Build Program.cs with AddCms(), AddCmsAspNetUi(), AddTinyMce(), correct middleware order
3. Build appsettings.json from ../old-solution web.config connection strings and appSettings
4. Build appsettings.Development.json and appsettings.Production.json skeletons
Commit: "feat: project setup and CMS 12 startup wiring"
```

### Step 2 — DI Registration
Skills: `legacy-ioc-to-di`, `optimizely-di-registration`
```
Read ../audit-output/modules.json for IConfigurableModule registrations.
Migrate all StructureMap/Windsor/Autofac registrations to IServiceCollection.
Create DependencyRegistration.cs or equivalent.
Commit: "feat: DI registration migrated from legacy IoC"
```

### Step 3 — Content Types (Pages)
Skill: `optimizely-content-type-builder`
```
Read ../audit-output/content-types.json.
For each item where baseClass == "PageData":
  - Create the class in src/Models/Pages/
  - Preserve GUID exactly from audit
  - Rewrite all properties as public virtual with correct CMS 12 types
  - Add [Display] attributes for all editor-facing properties
  - Flag any UrlBuilder properties replaced with Url
Commit after every 5 page types: "feat: add [PageTypeName] content type"
```

### Step 4 — Content Types (Blocks)
Skill: `optimizely-content-type-builder`
```
Same as Step 3 but for baseClass == "BlockData".
Output to src/Models/Blocks/
Commit after every 5 block types.
```

### Step 5 — Content Types (Media)
Skill: `optimizely-content-type-builder`
```
Same as Step 3 but for MediaData, ImageData, VideoData.
Output to src/Models/Media/
```

### Step 6 — Init Modules
Skill: `optimizely-init-module-scanner` (migration guidance), `optimizely-startup-wiring`
```
Read ../audit-output/modules.json.
For each IInitializableModule: migrate to CMS 12 pattern (keep IInitializableModule)
For each IInitializableHttpModule: rewrite as IStartupFilter or middleware
For each HttpApplication event handler: rewrite as middleware in Program.cs
Output to src/Infrastructure/Initialization/
Commit: "feat: initialization modules migrated"
```

### Step 7 — Scheduled Jobs
Skill: `optimizely-scheduled-job-migrator`
```
Read ../audit-output/modules.json for ScheduledJobBase entries.
Rebuild each job in src/Infrastructure/ScheduledJobs/
Preserve ScheduledPlugIn GUID exactly.
Ensure IsStoppable = true and cooperative stop signal (_stopSignaled check in loop).
Remove any HttpContext.Current or PrincipalInfo.CurrentPrincipal usage.
Commit: "feat: scheduled jobs migrated"
```

### Step 8 — Routing
Skill: `optimizely-route-auditor` (migration guidance)
```
Read ../audit-output/routes.json.
Migrate MapRoute registrations to Program.cs MapControllerRoute.
Migrate IPartialRouter implementations — register via builder.Services.AddSingleton.
Commit: "feat: routing migrated"
```

### Step 9 — Views & Controllers
Skills: `optimizely-view-engine-migrator`, `aspnet-core`
```
For each PageData type: create PageController<T> in src/Controllers/
For each BlockData type: create BlockComponent<T> ViewComponent in src/Components/
Migrate Views/ folder structure
Commit per controller group: "feat: [section] controllers and views"
```

### Step 10 — Config & Bootstrap
Skills: `globalasax-to-program`, `httpmodule-to-middleware`
```
Migrate remaining Global.asax logic to Program.cs / IHostedService
Migrate remaining IHttpModule implementations to middleware
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
Compare ../audit-output/content-types.json against built models in src/Models/.
Check:
  - Every content type from audit exists in new solution
  - Every GUID matches exactly
  - Every property name matches exactly (case-sensitive)
  - No CRITICAL issues
Save report to ../audit-output/schema-diff.json
STOP and report all issues before proceeding.
DO NOT proceed to database migration if any CRITICAL issues exist.
```

### Step 2 — Build Verification
```
Run: dotnet build
Fix all compiler errors before proceeding.
Run: dotnet test (if tests exist)
Commit: "chore: validation fixes"
```

### Step 3 — Pre-DB-Migration Checklist
Before running any database migration commands, confirm ALL of the following:
- [ ] Zero CRITICAL issues in ../audit-output/schema-diff.json
- [ ] `dotnet build` succeeds with zero errors
- [ ] All content type GUIDs verified
- [ ] Staging database migration tested
- [ ] Content spot-checked in CMS 12 editor on staging

---

## Parallel Agent Mode

For large codebases, run these steps in parallel terminals (each works on a non-overlapping area):

| Terminal | Task | Output Directory |
|---|---|---|
| 1 | Pages content types | src/Models/Pages/ |
| 2 | Blocks content types | src/Models/Blocks/ |
| 3 | Scheduled jobs + init modules | src/Infrastructure/ |
| 4 | Startup wiring + DI | Program.cs, src/Configuration/ |

Each terminal operates on a different directory — no conflicts.

---

## Skill Reference

| Task | Skill to Use |
|---|---|
| Audit content types | `optimizely-content-type-auditor` |
| Audit packages | `optimizely-dependency-mapper` |
| Audit modules/jobs | `optimizely-init-module-scanner` |
| Audit routing | `optimizely-route-auditor` |
| Build content types | `optimizely-content-type-builder` |
| Build startup/Program.cs | `optimizely-startup-wiring` |
| Build scheduled jobs | `optimizely-scheduled-job-migrator` |
| Build DI registration | `optimizely-di-registration` |
| Build views/controllers | `optimizely-view-engine-migrator` |
| Migrate web.config | `webconfig-to-appsettings` |
| Migrate Global.asax | `globalasax-to-program` |
| Migrate IHttpModule | `httpmodule-to-middleware` |
| Migrate IoC container | `legacy-ioc-to-di` |
| Validate schema | `optimizely-content-schema-diff` |
| .NET 8 patterns | `aspnet-core`, `modern-csharp` |
| EF Core queries | `optimizing-ef-core-queries` |
