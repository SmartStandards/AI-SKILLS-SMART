---
name: smart-solution-and-project-structure
description: Repository conventions and the ASP.NET Core hosting template of the SmartStandards landscape - default branch master, default license for new open-source repositories (MIT + Commons Clause), repository layout (README, LICENSE, AGENTS.md, CLAUDE.md, .editorconfig mirror, .agents/skills, /doc with requirements, architecture, quickstart, ecosystem.md, decisions/, epics/, work/, changelog.md), .NET names without vendor prefix, .sln folder layout (Solution Items with changelog.md and .nuspec files, src, src/targets, tests), the SmartStandards.AspNetCoreTemplate startup pattern (Program.Main, LaunchProfileHelper, branch-specific appsettings) and the launch profile design (Development, Infra, Infra-<Qualifier>; profile name = environment name). Use when creating a repository or solution, reviewing root files, /doc or .sln structure, writing Program.Main or launchSettings.json, or when a service behaves differently from Visual Studio vs. as EXE. Snapshot; escalate upstream for library work.
metadata:
  snapshot-date: "2026-10-19"
---

# Solution and Project Structure (Snapshot)

<!-- Layer 1 - Domain Overview -->

How SmartStandards repositories and hosts are structured and started. This skill covers the basic repository conventions (default branch, license, naming; see `Repository Conventions`), the repository and documentation layout of standalone solutions (see `Repository Layout of Standalone Solutions`), the layout of .NET solutions (see `Solution Layout (.NET)`), the ASP.NET Core hosting template and the design of launch profiles and configuration files (see `Launch Profile and Configuration Design`). All sections except the hosting template are landscape conventions, not snapshots of an upstream repository. Code-level details of shared projects are not covered here - read the `most-important-details-for-*` skill of the respective repository.

- Libraries / repositories in scope: `AspNetCoreTemplate` (package `SmartStandards.WebApiDependencies`); reference solution for the solution layout: `ComponentDiscovery`
- Use for: creating a new repository or solution, reviewing the root files, the `/doc` folder or the `.sln` structure, `Program.Main` of a SmartStandards-style Web API, launch profiles for directly started EXEs, the designed state of `launchSettings.json` and the appsettings files, branch-specific configuration files
- Out of scope: UJMW, SmartAmbience and logging internals (only wired by the template) - see `smart-http-remoting-standard`, `smart-dataflow-and-ambience`; project file contents - see `dotnet-sdk-style-project-files`; versioning and `.nuspec` contents - see `nuget-semantic-versioning-kvu`
- Bringing a whole solution up to date (target frameworks, renames, nuspec, conformance check against this skill): follow the general skill `refactoring-processes`

<!-- Layer 2 - Cached Essentials -->

## Snapshot Status

- Snapshot date: 2026-10-19
- Synchronized against: AspNetCoreTemplate @ 1c97a14 (package 10.0.14)
- Staleness thresholds: fresh < 90 days, aging 90-180 days, stale > 180 days
- Volatility: low

Apply the alarm logic from `smart-skill-maintaining-guidelines` before relying on this snapshot.

## Core Concepts

- A directly started EXE ignores `Properties/launchSettings.json` and falls back to ASP.NET Core defaults (`http://localhost:5000`, environment `Production`). `LaunchProfileHelper` applies a profile itself when the EXE gets `LaunchProfile=<name>` (name or zero-based index; unknown name or index throws).
- Only two profile values are applied: the environment (`ASPNETCORE_ENVIRONMENT`, fallback `DOTNET_ENVIRONMENT`) and the URLs (`applicationUrl`, split at `;`). Other `environmentVariables`, `commandName`, `launchBrowser`, `launchUrl`, `iisSettings` are ignored.
- `launchSettings.json` is searched relative to the helper assembly (`./`, `./Properties/`, `../Properties`, `../../Properties`); copy `Properties/launchSettings.json` to the output directory.
- `AddBranchSpecificConfigurationFiles()` adds optional `appsettings.<Branch>.json` and `appsettings.<Environment>.<Branch>.json` (branch via `git rev-parse --abbrev-ref HEAD`). They are appended after `CreateBuilder` and therefore override environment variables and command-line arguments. Without Git (normal on servers) nothing is added.
- `SmartStandardsCheapInitializer.OnConfigureServices` / `OnRunApplication`: default wiring of logging, controllers, UJMW, SmartStandards-flavored Swagger, static files, `UseAmbientFieldAdapterMiddleware`, developer exception page unless `ProdMode` is `true`, HTTPS redirection, permissive CORS, authorization, `MapControllers`.

## Typical Usage

Order: environment before `CreateBuilder`, branch files before the configuration is consumed.

```csharp
string explicitLaunchProfile = null;
LaunchProfileHelper launchProfileHelper;
if (LaunchProfileHelper.TryPickLaunchProfileFromCommandlineArgs(args, out explicitLaunchProfile)) {
  launchProfileHelper = LaunchProfileHelper.CreateForCurrentLaunchsettingsJson();
}
else {
  launchProfileHelper = LaunchProfileHelper.CreateEmpty();
}
WebApplicationBuilder builder;
if (launchProfileHelper.TryGetAspEnvironmentNameFromLaunchProfile(explicitLaunchProfile, out string aspEnvironmentName)) {
  builder = WebApplication.CreateBuilder(new WebApplicationOptions { Args = args, EnvironmentName = aspEnvironmentName });
}
else {
  builder = WebApplication.CreateBuilder(args);
}
builder.AddBranchSpecificConfigurationFiles();
if (launchProfileHelper.TryGetUrlsFromLaunchProfile(explicitLaunchProfile, out string[] explicitUrls)) {
  builder.WebHost.UseUrls(explicitUrls);
}
```

## Quick Troubleshooting

| Symptom | Likely cause | Quick check |
| --- | --- | --- |
| EXE listens on port 5000 / runs as `Production` | No `LaunchProfile=` argument, or helper not used in `Program.Main` | Command line; startup pattern above |
| Startup throws "no launchSettings.json file found" | File not copied to output directory | Copy-to-output setting of `Properties/launchSettings.json` |
| Throws although `throwIfNotFound: false` | Known defect in 10.0.14 (internal lookup throws) | Fix upstream; do not work around in consuming code |
| Environment variable / CLI override seems ignored | A branch file overrides it | Look for `appsettings.<Branch>.json` in the checkout |
| Branch file not applied | Branch name contains `/`, detached HEAD, or no Git | `git rev-parse --abbrev-ref HEAD` |
| With profile `Infra`: Swagger, user secrets, developer exception page or static web assets missing (present with `Development`) | Code or framework defaults depend on the environment name `Development` | Search for `IsDevelopment()`, `IsEnvironment(`, `IsProduction()`; see `Launch Profile and Configuration Design` |

## Launch Profile and Configuration Design

Landscape convention (user decision 2026-10-21), not a snapshot of an upstream repository; it applies to every SmartStandards-style ASP.NET Core host and is part of bringing a solution up to date (`refactoring-processes`, aspect D).

### Two Scenarios, Two Profiles

Local development knows only two scenarios. Branch-dependent differences are not a scenario; they live in the branch-specific files.

| Profile | `ASPNETCORE_ENVIRONMENT` | Purpose |
| --- | --- | --- |
| `Development` | `Development` | Fully local development against local databases and services. Deliberately the ASP.NET Core standard name, so that the framework defaults bound to it (`IsDevelopment()`, automatic user secrets, DI validation, static web assets, template code such as Swagger in development) apply without extra code. |
| `Infra` | `Infra` | Local development against an already deployed, shared development or integration infrastructure. The deployed state **must match the current code state**. Never meant for production. |
| `Infra-<Qualifier>` | `Infra-<Qualifier>` | Optional, one per further infrastructure installation (e.g. a separate silo or client group), only if its granularity is not already covered by the branches. |

`Development` and `Infra` are the minimum standard of every Web API host. `Development` names the scenario "fully local", not a branch or a deployment stage.

### Rules for `Properties/launchSettings.json`

- **Profile name = environment name**, exactly. The profile `Infra` sets `ASPNETCORE_ENVIRONMENT` to `Infra` and therefore selects `appsettings.Infra.json`; no mapping table is needed.
- **Only `commandName: "Project"` profiles.** No IIS Express profile, no `iisSettings` block, no profiles named after the project or service (e.g. `DemoWebService`). Other launch targets (Docker, WSL) only when the solution really uses them - ask.
- **No branch words** in profile or environment names: no `Develop`, `Dev`, `Main`, `Master`, `Release`, `Feature`, or similar (`Development` is the scenario name above, not a branch). Branch-dependent settings go into `appsettings.<Branch>.json` / `appsettings.<Environment>.<Branch>.json`.
- **Order:** `Development` first (default for `dotnet run` and fresh checkouts), then `Infra`, then `Infra-<Qualifier>` profiles.
- **URLs:** keep the existing ports of the service; all profiles of a service normally use the same `applicationUrl`.
- **`environmentVariables`** contains only `ASPNETCORE_ENVIRONMENT`. Further variables are ignored by `LaunchProfileHelper` (see `Core Concepts`) and would make a Visual Studio start differ from an EXE start; put such values into the appsettings files. `launchBrowser` / `launchUrl` are allowed (Visual Studio only).
- **Starting the EXE outside Visual Studio** (launcher tools, scripts): pass `LaunchProfile=<name>` by name, not by index, so reordering profiles cannot switch the environment.

```json
{
  "profiles": {
    "Development": {
      "commandName": "Project",
      "launchBrowser": true,
      "launchUrl": "swagger",
      "applicationUrl": "https://localhost:44301;http://localhost:44300",
      "environmentVariables": { "ASPNETCORE_ENVIRONMENT": "Development" }
    },
    "Infra": {
      "commandName": "Project",
      "launchBrowser": true,
      "launchUrl": "swagger",
      "applicationUrl": "https://localhost:44301;http://localhost:44300",
      "environmentVariables": { "ASPNETCORE_ENVIRONMENT": "Infra" }
    }
  }
}
```

### Configuration Files

| File | Content |
| --- | --- |
| `appsettings.json` | Base configuration for deployment. **No local connection strings**: if a connection string is missing during development, startup must fail instead of silently using a deployment value. |
| `appsettings.Development.json` / `appsettings.Infra.json` / `appsettings.Infra-<Qualifier>.json` | Values of the scenario, e.g. local database connections vs. endpoints of the shared infrastructure. |
| `appsettings.<Branch>.json` / `appsettings.<Environment>.<Branch>.json` | Optional, branch-dependent values (e.g. one database per branch); override everything above (see `Core Concepts`). Preferred over additional profiles. |

- Add a further `Infra-<Qualifier>` profile only when a different infrastructure must be addressed whose granularity is not already covered by the branches.
- Deployed hosts are not configured via launch profiles; the hosting environment (e.g. `web.config`, environment variables) sets the environment, and branch files are not applied without Git.
- `IsDevelopment()` is `true` only for `Development`; with `Infra` / `Infra-<Qualifier>` it is `false`, so the framework defaults bound to `Development` are off there. Do not add more `IsDevelopment()` checks for behavior that is also needed with `Infra` (e.g. Swagger, diagnostics); switch it via explicit settings (e.g. `ProdMode` of `SmartStandardsCheapInitializer`) instead of further environment names.

## Repository Conventions

Landscape convention (user decision 2026-10-23); applies to every repository of the landscape.

### Default Branch

- **`master`** is the default branch of every repository. Agents work on the repository's existing default branch and never create `main`.
- **First push of an empty repository goes to `master`.** GitHub makes the first pushed branch the default branch, so pushing `main` (or a feature branch) first silently changes the default. Agent environments often cannot change repository settings afterwards (e.g. cloud sessions); pushing `master` first makes that unnecessary.
- KVU's ignore list `master;main;rel-*` only tolerates `main`; it is no recommendation (see `nuget-semantic-versioning-kvu`).

### License

- **Default for new open-source repositories: MIT + Commons Clause** (`LICENSE` = MIT license text followed by the Commons Clause condition). Name it in the README `License` section.
- Existing repositories keep their license. Never change a license, and never deviate from the default for a new repository, without asking.
- In `.nuspec` files, MIT + Commons Clause has no SPDX expression; ship the license file instead (see `nuget-semantic-versioning-kvu`, section 4).

### No Vendor Prefix in .NET Names

- The vendor (GitHub owner or publisher, e.g. `KornSW`) never appears in solution names, project folders, project file names, assembly names, root namespaces or package ids: `src/Example.Product.net10.0/Example.Product.net10.0.csproj`, not `Vendor.Example.Product.net10.0.csproj`.
- The vendor appears only in package metadata (`authors`, `owners`, `copyright`) and in the repository URL. Package id = assembly name (see `dotnet-sdk-style-project-files`, section 3).
- **`SmartStandards.*`** is the established product-family prefix of the packages in repositories owned by `SmartStandards` (e.g. `SmartStandards.AiSupport`, `SmartStandards.Logging`). Existing names stay. Never introduce it, or any other organization name, in repositories of other owners. Whether new `SmartStandards` repositories keep the prefix is not decided yet; ask.
- Renaming existing assemblies, namespaces or package ids is breaking; record it in `doc/work/tech-debt.md` instead (see `refactoring-processes`).

## Repository Layout of Standalone Solutions

Landscape convention (user decision 2026-10-21), not a snapshot of an upstream repository; it applies to every Git repository that holds one standalone solution. It supersedes the older numbered layout (`1-requirements.md`, `2-architecture.md`, `3-quickstart.md`, `ideas.md`).

```text
/
├── README.md                     # mandatory entry point
├── LICENSE                       # mandatory in open-source repositories
├── AGENTS.md                     # mandatory, prepared for agents
├── CLAUDE.md                     # mandatory, only "@AGENTS.md"
├── .editorconfig                 # mandatory, identical mirror of the leading version
├── .agents/skills/
│   ├── include-additional-mandatory-skills/   # copy (or the organization twin)
│   └── most-important-details-for-<repo>/     # project skill
├── doc/
│   ├── requirements.md           # goals, audiences, requirements, non-goals
│   ├── architecture.md           # high-level structure, components, data flows
│   ├── quickstart.md             # task-oriented getting started
│   ├── ecosystem.md              # relevant incoming / outgoing dependencies
│   ├── changelog.md              # KVU, see below
│   ├── versioninfo.json          # KVU
│   ├── <topic>.md                # optional further specifications / topic documents
│   ├── assets/                   # optional images for the documents
│   ├── decisions/
│   │   └── decision-<title>.md   # one decision with its rationale
│   ├── epics/
│   │   └── epic-<title>.md       # new ideas, one file per epic
│   └── work/
│       ├── backlog.md            # planned to-dos
│       └── tech-debt.md          # known technical / design debt
└── <source folders, e.g. dotnet/>
```

Rules:

- **Repositories without code** (e.g. pure documentation or skill repositories): `/doc` and the files inside it are optional, as are `.editorconfig` and the KVU files. `README.md`, `LICENSE` (open source), `AGENTS.md`, `CLAUDE.md` and `.agents/skills/` stay mandatory. Whatever exists must follow this structure.
- **No numeric prefixes** in `/doc`. Rename `1-requirements.md` -> `requirements.md`, `2-architecture.md` -> `architecture.md`, `3-quickstart.md` -> `quickstart.md` and update all links (README, other documents, project skill) in the same change.
- **Only documents with clear semantics** in `/doc`; no `README.*` there (merge its content into `architecture.md` or a topic document). Further documents are allowed when they are clear specifications or topic documents (e.g. `api.md`, `configuration.md`, `oauth-server-pkce.md`); they are named in English, lower case with hyphens, and are linked from the README.
- **Images and similar media** for the documents live in `doc/assets/`.
- **Not in `/doc` as loose files:**
  - AI skill content (e.g. `[AI-Skill] *.md`) -> `.agents/skills/<name>/SKILL.md` in Agent Skills format;
  - hand-over notes, refactoring plans and other working notes -> `epics/`, `work/backlog.md` or `decisions/`, depending on their content;
  - drafts, parallel versions and backups (`*.v1.md`, `*OLD*`, `*(backup)*`, `*DRAFT*`) -> merge into one current document (the history lives in Git); undecided drafts -> `epics/`;
  - one-off submission drafts or issue texts -> remove once obsolete, or keep the substance in a clear specification.
- **Generated release artifacts** may live in `/doc` next to `changelog.md` and `versioninfo.json` when tooling requires the location (e.g. `<slug>.update.json` and `<slug>.updater-prefix.txt` of the WordPress release flow, see `nuget-semantic-versioning-kvu`).
- **Other technical artifacts** currently found in `/doc` (e.g. tool binaries, pipeline exports, install scripts) are tolerated for now and left alone; they are not moved as part of a refactoring until a rule exists.
- **`.github/`**: workflows as needed; a `copilot-instructions.md` contains only a pointer to `AGENTS.md` (like `CLAUDE.md`); issue / pull request templates are optional.
- **`changelog.md` is left alone** apart from the KVU rules: agents may only add conformant entries below `## Upcoming Changes` (see `nuget-semantic-versioning-kvu`).
- **New repositories with code** contain `doc/changelog.md` (standard header, `## Upcoming Changes` with `*(none)*`, no release block) and a matching `doc/versioninfo.json` (`0.0.0`) from the first commit on; templates in `nuget-semantic-versioning-kvu`, section 1.
- **`ecosystem.md`** describes outgoing dependencies (libraries / services this repository builds on) and incoming ones (known consumers), but only
  1. entries that really matter for understanding or classifying this repository, and
  2. entries whose visibility is not narrower than this repository's: open-source repositories name only open-source dependencies and consumers; in an organization context, internal consumers may be named as well.
- **`epics/epic-<title>.md`**: new ideas and larger concepts that are not active requirements yet (replaces `ideas.md`; lower-case, hyphenated title). Nothing in it is implemented without explicit commitment; accepted parts move into `requirements.md` and `work/backlog.md`.
- **`decisions/decision-<title>.md`**: one significant decision per file (architecture, technology, deliberate deviations such as "not renamed because ..."): context, decision, alternatives considered, consequences, date. `architecture.md` describes the resulting structure and links the relevant decisions; a decision that gets replaced stays and links its successor.
- **`work/backlog.md`**: clearly planned to-do list; entries may link to epics. **`work/tech-debt.md`**: known technical and design debt (deviation, why not fixed, proposed resolution); this is also where structural deviations found during refactorings are recorded.
- **Root `README.md`** (mandatory), default outline - omit sections without value, never copy the explanations:

  ```markdown
  # <Project name>
  <one or two paragraphs: what it is and does>
  ## Motivation          <!-- problem it solves -->
  ## Examples            <!-- a few concise usage examples -->
  ## Differentiation     <!-- only if relevant alternatives exist -->
  ## Documentation       <!-- relative links to every file directly in /doc, plus decisions/, epics/ and work/ -->
  ## License             <!-- open source: name the license, link LICENSE -->
  ```

  The documentation section links every file that lives directly in `/doc` (relative links), in the order requirements, architecture, quickstart, ecosystem, further documents, changelog, then the `decisions/`, `epics/` and `work/` folders.
- **`LICENSE`** in the root of every open-source repository; default and rules in `Repository Conventions`.
- **`AGENTS.md`** in the root, short: points agents to `.agents/skills/include-additional-mandatory-skills/SKILL.md` (or the organization twin) as the mandatory entry, to the project skill `most-important-details-for-<repo>` and to `/doc`. **`.agents/skills/`** contains the copy of `include-additional-mandatory-skills` (organization repositories: only the twin) and the project skill.
- **`CLAUDE.md`** in the root contains only the import line `@AGENTS.md`, so tools reading `CLAUDE.md` and tools reading `AGENTS.md` get the same instructions. Content goes into `AGENTS.md`, never into `CLAUDE.md`.
- **`.editorconfig`** in the root is an identical mirror of the leading version in `csharp-coding-standards` (`assets/.editorconfig`, header with version). Never edit the mirror; repository-specific deviations go into a nested `.editorconfig` without `root = true`. Older copies in sub-folders (e.g. `dotnet\.editorconfig`) are replaced by the root mirror; their deviating rules are reviewed and either dropped or kept in a nested file.
- `README.md` and everything in `/doc` are written in English.

## Solution Layout (.NET)

Landscape convention (user decision 2026-10-23). Reference solution: `ComponentDiscovery` @ `a159ddb` (`dotnet/ComponentDiscovery.sln`); its `.nuspec` files carry known debt (see `nuget-semantic-versioning-kvu`, section 4).

Virtual solution folders (names lower case, except `Solution Items`):

```text
Solution Items      ..\doc\changelog.md, <Package>.nuspec (one per package)
src                 shared projects (.shproj) and all non-target projects (applications, hosts, tools)
  targets           target shells per framework (<Name>.net48 / .net8.0 / .net10.0)
tests               test projects
```

Physical layout:

```text
dotnet/
├── <Solution>.sln
├── <Package>.nuspec              # one per package, standalone, see nuget-semantic-versioning-kvu
├── PackageIcon.png
├── src/
│   ├── <Name>/                   # shared project (.shproj + .projitems) with the code
│   ├── <Name>.net48/             # target shells, flat next to the shared project
│   ├── <Name>.net8.0/
│   ├── <Name>.net10.0/
│   └── <Application>/            # hosts, tools
├── tests/
│   └── <Name>.Tests/
└── dist/                         # build output, see Build Output Layout (dist)
    ├── Shared/<tfm>/             # packaged / shared libraries
    └── <ArtifactName>/<tfm>/     # runnable bundle per Web API / EXE
```

- **`changelog.md` is always a solution item** (`..\doc\changelog.md`), so the pending changes are visible in the IDE. Every `.nuspec` is a solution item, too.
- **`targets`** exists whenever a library is multi-targeted via a shared project plus target-framework shells. The shells carry no own code; nesting them in `targets` keeps the solution focused on the shared project. `targets` is **only virtual**: physically the shells stay flat next to the shared project in `src/`.
- **`tests`** is the folder name, physically and virtually. Older repositories using `test/` (e.g. `AiSupport`) are a tolerated deviation; renaming the physical folder is a separate plan item of `refactoring-processes`.
- Applications, tools and test projects are not packaged (`IsPackable` false, no `.nuspec`); every multi-targeted library is published as one package containing all targets (see `nuget-semantic-versioning-kvu`, section 4).

### Build Output Layout (dist)

Landscape convention (user decision 2026-10-24). The build output is cut by the role of the project, not by one shared folder for everything. Project file implementation: see `dotnet-sdk-style-project-files`, section 4.

```text
dist/
├── Shared/
│   ├── net8.0/                   # -> NuGet (lib/net8.0)
│   └── net10.0/                  # -> NuGet (lib/net10.0)
└── <ArtifactName>/
    ├── net8.0/                   # runnable bundle incl. all references
    └── net10.0/
```

| Role | Output | Content |
| --- | --- | --- |
| Shared library - a) packaged as NuGet, b) consumed by other solutions in the same build, c) test support / test content | `dist/Shared/<tfm>/` | The official build output of libraries that are no runnable artifact. Only the library's own files (`.dll`, `.xml`, `.pdb`, own content). For .NET (Core) no peer dependencies are copied in - nothing is executed from here. |
| Runnable artifact (Web API, Windows EXE) | `dist/<ArtifactName>/<tfm>/` | Cherry-picked, clean bundle of one operable artifact with all its references and content. |
| Test project | own `bin` of the test project | Nothing in `dist`. |

Rules:

- **Bundles are produced by MSBuild itself**: redirection of the build output (`OutputPath`), copy-local of references and content items (`CopyToOutputDirectory`). No make- / batch-like scripts that assemble the bundle.
- `AfterBuild` / custom copy targets only in absolute special cases; with .NET (Core) they should not be necessary. Document the reason in the project file.
- `<ArtifactName>` is set explicitly in the `OutputPath` of the host project (normally the product or assembly name of the artifact). Every runnable artifact has its own folder; artifacts never share a folder with each other or with `Shared`.
- Libraries referenced by an artifact reach the bundle via the `ProjectReference` / `PackageReference` copy-local of the host - not by copying from `dist/Shared`.
- `.nuspec` files reference only `dist\Shared\<tfm>\` (see `nuget-semantic-versioning-kvu`, section 4).
- `dist` is lower case (folder naming convention); `Shared` and `<ArtifactName>` keep their casing.
- .NET Framework targets (e.g. `net48`) use the same folder structure; whether peer dependencies end up in `dist/Shared/net48/` follows the classic copy-local behavior and is not normative.

<!-- Layer 3 - Upstream Escalation -->

## Upstream Sources

| Repository | URL | Skills | Docs | Packages |
| --- | --- | --- | --- | --- |
| `AspNetCoreTemplate` | https://github.com/SmartStandards/AspNetCoreTemplate | `.agents/skills/most-important-details-for-smartstandards-aspnetcore-template/` | `doc/` | `SmartStandards.WebApiDependencies` |

### Escalate to upstream when

- developing, extending or fixing the library itself;
- deep troubleshooting needs implementation details not covered above;
- this snapshot is stale or its alarm fires and the task depends on the affected details;
- observed behavior or the consumed package version contradicts this snapshot.

Then load the upstream skills (local clone preferred, otherwise online) and treat them as authoritative.
