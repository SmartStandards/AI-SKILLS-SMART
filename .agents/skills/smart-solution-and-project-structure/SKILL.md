---
name: smart-solution-and-project-structure
description: Common repository conventions and the ASP.NET Core hosting template of the SmartStandards landscape - the repository layout of standalone solutions (root README, LICENSE, AGENTS.md, CLAUDE.md, .editorconfig mirror, .agents/skills, /doc with requirements, architecture, quickstart, ecosystem.md, decisions/, epics/, work/backlog.md and tech-debt.md, changelog.md) and the SmartStandards.AspNetCoreTemplate / SmartStandards.WebApiDependencies startup pattern (Program.Main, LaunchProfileHelper with "LaunchProfile=<name>", branch-specific appsettings files, SmartStandardsCheapInitializer). Use when writing or debugging Program.Main of a SmartStandards-style ASP.NET Core Web API, when a service behaves differently started from Visual Studio vs. as EXE (e.g. falls back to port 5000), or when appsettings.<Branch>.json files are involved. Snapshot of the upstream skills; escalate to the upstream repositories for library development or deep troubleshooting.
metadata:
  snapshot-date: "2026-10-19"
---

# Solution and Project Structure (Snapshot)

<!-- Layer 1 - Domain Overview -->

How SmartStandards repositories and hosts are structured and started. This skill covers the repository and documentation layout of standalone solutions (a landscape convention, see `Repository Layout of Standalone Solutions`) and the ASP.NET Core hosting template. Further conventions of the domain (shared projects, multi-targeting) are not yet covered here - read the `most-important-details-for-*` skill of the respective repository.

- Libraries / repositories in scope: `AspNetCoreTemplate` (package `SmartStandards.WebApiDependencies`)
- Use for: creating or reviewing the root files and the `/doc` folder of a repository, `Program.Main` of a SmartStandards-style Web API, launch profiles for directly started EXEs, branch-specific configuration files
- Out of scope: UJMW, SmartAmbience and logging internals (only wired by the template) - see `smart-http-remoting-standard`, `smart-dataflow-and-ambience`; versioning - see `nuget-semantic-versioning-kvu`
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
- **`LICENSE`** in the root of every open-source repository. Never choose or change a license on your own; ask.
- **`AGENTS.md`** in the root, short: points agents to `.agents/skills/include-additional-mandatory-skills/SKILL.md` (or the organization twin) as the mandatory entry, to the project skill `most-important-details-for-<repo>` and to `/doc`. **`.agents/skills/`** contains the copy of `include-additional-mandatory-skills` (organization repositories: only the twin) and the project skill.
- **`CLAUDE.md`** in the root contains only the import line `@AGENTS.md`, so tools reading `CLAUDE.md` and tools reading `AGENTS.md` get the same instructions. Content goes into `AGENTS.md`, never into `CLAUDE.md`.
- **`.editorconfig`** in the root is an identical mirror of the leading version in `csharp-coding-standards` (`assets/.editorconfig`, header with version). Never edit the mirror; repository-specific deviations go into a nested `.editorconfig` without `root = true`. Older copies in sub-folders (e.g. `dotnet\.editorconfig`) are replaced by the root mirror; their deviating rules are reviewed and either dropped or kept in a nested file.
- `README.md` and everything in `/doc` are written in English.

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
