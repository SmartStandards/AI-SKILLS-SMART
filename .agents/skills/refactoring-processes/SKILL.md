---
name: refactoring-processes
description: Standardized high-level refactoring processes for existing .NET solutions of the SmartStandards / UShell landscape - analyze first, present a plan, execute only after confirmation, never commit. Currently - bringing a solution "up to date" ("auf den aktuellen Stand bringen", modernize): package versions, target frameworks (Web APIs net10.0 only; contracts net4x + net8.0 + net10.0, no net6.0) incl. project renames and .sln re-wiring, project files, .nuspec files (dist paths, KVU configuration, versioningTwins, namespace), ASP.NET Core bootstrap review, launch profile clean-up (Development / Infra), repository, .sln folder and /doc layout (README, LICENSE, AGENTS.md, Solution Items, src/targets, tests, ecosystem.md, epics, work) - no breaking renames of assembly names, namespaces or vendor prefixes; deviations go to doc/work/tech-debt.md. Load when a user asks to bring a solution, repository or project up to date, or for another larger structural refactoring.
metadata:
  version: "2026-10-23"
---

# Refactoring Processes

This general skill collects standardized processes for larger, high-level refactorings of existing solutions. Each process orchestrates other skills; it does not repeat their rules. Load each referenced skill before analyzing its aspect.

| Process | Trigger |
| --- | --- |
| Bring a Solution Up to Date (below) | The user asks to bring a solution "to the current state" (German: "auf den aktuellen Stand bringen", "modernisieren", "aktualisieren", or similar). |

New processes are added as further `# Process: ...` sections with their own trigger row. Ground rules that turn out to be shared move into a common section once a second process needs them.

# Process: Bring a Solution Up to Date

## 1. Ground Rules

1. **Analyze first, act later.** Phase 1 is read-only. Present the planned actions (phase 2) and wait for an explicit confirmation. The user may confirm the plan partially.
2. **Never commit or push.** Leave all changes in the working tree for review. Use `git mv` for renames so history survives.
3. **No coding-rules sweep.** Syntactic coding rules (`csharp-coding-standards`) are not a topic of this procedure; they would blow up a high-level task. They still apply to code that has to be changed anyway.
4. **No breaking renames.** Do not rename assembly names, package IDs, root namespaces, namespaces, public types or routes, even if they violate the conventions. Record such deviations as design debt (phase 1, aspect E).
5. **No silent package changes.** Package versions are reported and changed only item by item after confirmation (`package-version-preferences-dot-net`).
6. **Preserve semantics.** Structural work never changes runtime behavior, ordering or security-relevant configuration on its own; ask when unsure.
7. **Changelog only via KVU rules.** If the result is visible to package consumers (e.g. a dropped target framework), tell the user and propose the impact prefix for the commit message (e.g. `Breaking Change: ...`) or, for a detailed description, a conformant entry below `## Upcoming Changes` - nothing else in `changelog.md` is touched (`nuget-semantic-versioning-kvu`).

## 2. Phase 0 - Orientation

- Load `include-additional-mandatory-skills` (and an organization twin if the repository belongs to one), the repository's `/doc/*.md` and its `most-important-details-for-*` skill.
- Check `git status`: start from a clean working tree, or tell the user what is already pending.
- Inventory: solution file(s) (`.sln` / `.slnx`), all projects with SDK, target framework, role (web host, contract, library, test, tool), shared projects (`.shproj` / `.projitems`), linked files, `.nuspec` files, pipelines / scripts referencing paths.

## 3. Phase 1 - Analysis (read-only)

Analyze every aspect below and collect findings. Do not fix anything yet.

### A. Package Versions

Compare every `PackageReference`, `packages.config` entry and `.nuspec` dependency with `package-version-preferences-dot-net`: ranges, pins, competing technologies, gross violations. Compare with the reference solutions when an entry is missing.

### B. Target Framework Structure

Target matrix (current standard):

| Project role | Target frameworks |
| --- | --- |
| Web API / ASP.NET Core host | `net10.0` only |
| Contract | the existing .NET Framework target (`net4x`, e.g. `net48`) + `net8.0` + `net10.0`; **no** `net6.0` |
| Other libraries, tests, tools | not fixed by this procedure; report the current state and ask (propose the contract matrix for libraries consumed by .NET Framework hosts) |

Layout: one project per target framework, side by side, sharing their code via a shared project (`.projitems`) or linked files. The target framework is a suffix of the project file name **and** its folder: `src\Example.Contract.net8.0\Example.Contract.net8.0.csproj`. `AssemblyName` and `PackageId` do not contain the suffix and are identical across the variants. An existing single project with `<TargetFrameworks>` (plural) is reported, not converted, unless the user asks.

When a project changes its target framework, a variant is added or a variant is dropped, the plan must contain all of these follow-ups:

- rename folder and project file to the new suffix (`git mv`);
- `.sln`: update display name and relative path of the project entry; keep the project GUID, solution folder nesting and configuration mappings (`.slnx`: update the path);
- all `ProjectReference` paths pointing to the renamed project (also from test projects and other solutions in the repository);
- linked `Compile Include="..\<OtherVariant>\..."` items pointing into renamed folders;
- `TargetFramework`, custom compiler symbols and the canonical BUILD block (`dotnet-sdk-style-project-files`); search `#if` blocks for symbols of a dropped framework and report them;
- `.nuspec`: dependency groups, `files` entries and KVU `sources` (aspect C);
- pipelines, scripts and `/doc` mentioning the old path or framework.

A new variant (e.g. a missing `net10.0` contract) is created by copying the closest existing variant and applying `dotnet-sdk-style-project-files`. Dropping a variant (e.g. `net6.0`) removes the project from the `.sln` and the `.nuspec` and is visible to consumers (rule 7).

Bring touched project files into the designed state of `dotnet-sdk-style-project-files` (section order, canonical BUILD with the role-based `OutputPath` - `..\..\dist\Shared` for libraries, `..\..\dist\<ArtifactName>` for runnable artifacts, own `bin` for tests - and SDK target-framework suffix). Untouched project files with gross deviations are reported as separate, optional plan items.

### C. NuSpec Files

**a) Paths into `dist`.** For every `.nuspec`:

- each `<file src>` resolves (relative to the `.nuspec`) to an output of the canonical BUILD, i.e. `dist\Shared\<tfm>\<AssemblyName>.dll` / `.xml` / `.pdb` - not legacy `dist\bin-net8.0\` or flat `dist\<tfm>\` paths;
- every packaged target framework has its `dll`, `xml` and `pdb` entries and a matching `lib/<tfm>` target; no entries for dropped frameworks;
- the `dependencies` contain one `<group targetFramework="...">` per packaged framework;
- the form matches the package design of `nuget-semantic-versioning-kvu` (section 4): root element with the nuspec namespace, all metadata written out (no `$...$` tokens), one package per multi-targeted library; deviations are plan items.

**b) KVU pre-processor configuration.** KVU reads exactly one JSON configuration from an XML comment inside the `.nuspec` (more than one fails the build; none means `DoMagic` skips the file completely, not even the version is written). Details: `nuget-semantic-versioning-kvu`. Check:

- every `.nuspec` has a KVU comment; a missing one is a defect - add at least the minimal form `"KVU": { }` (plan item);
- no other XML comment of the `.nuspec` contains braces (they would be parsed as configuration);

- `versioningTwins` lists exactly the packages built **from the same repository** that this package references via `ProjectReference`. KVU writes them as exact `[<version>]` dependencies into every group and excludes them from synchronization and orphan removal. Missing twin: the package would reference a stale or wrong version of its sibling. External package listed as twin: it would be forced to this package's version.
- `dependencySync.sources` contains one entry per target framework project of the package, with `"targetFramework":"="` when the `.nuspec` uses groups (the `net48` shell: explicit `".NETFramework4.8"`, see `nuget-semantic-versioning-kvu`); every `origin` path (relative to the `.nuspec`) resolves - re-check after renames.
- `constraintLevel` and the flags (`addMissing`, `updateExisting`, `allowDowngrades`, `removeOrphaned`) are set deliberately; compare with sibling `.nuspec` files of the repository and report inconsistencies instead of unifying them silently.

```xml
<dependencies>
  <!-- (configuration for the Versioning-Util, which is not part of the xml-schema, because it's a pre-processor)
  "KVU": {
    "versioningTwins":[ /* forced same [version] as our package and never treated as orphaned */
      "Example.Product"
    ],
    "dependencySync": {
      "addMissing":true, "updateExisting":true, "allowDowngrades":true, "removeOrphaned":false,
      "constraintLevel":"SEM-SAFE",
      "sources": [
        { "origin":"src\\Example.Product.AspNetCore.net8.0\\Example.Product.AspNetCore.net8.0.csproj", "targetFramework":"=" },
        { "origin":"src\\Example.Product.AspNetCore.net10.0\\Example.Product.AspNetCore.net10.0.csproj", "targetFramework":"=" }
      ]
    }
  } -->
  <group targetFramework="net8.0"> ... </group>
  <group targetFramework="net10.0"> ... </group>
</dependencies>
```

The authoritative implementation is KVU (`VersioningUtil`, see `smart-standards-library-ecosystem`); escalate to it when the configuration behaves unexpectedly.

### D. ASP.NET Core Bootstrap

For every web host, review `Program.cs` / `Startup.cs` against `asp-net-core-development-guidelines` and its `references/application-bootstrap.md` (canonical `Program.Main`, the two hooks, `UseAmbientFieldAdapterMiddleware` first, no static configuration) and the launch-profile handling of `smart-solution-and-project-structure`. A migration into the designed state is a separate plan item with its own confirmation; follow the migration rules of that reference.

**Launch profiles and configuration files.** For every web host, compare `Properties/launchSettings.json` and the `appsettings*.json` files with section `Launch Profile and Configuration Design` of `smart-solution-and-project-structure` (profiles `Development` and `Infra`, optional `Infra-<Qualifier>`; profile name = environment name; no IIS Express, project-named or branch-named profiles; no local connection strings in `appsettings.json`). The clean-up is one plan item per host, risk `needs decision`, because renaming an environment changes which files and code paths are active. The plan item lists:

- profiles to remove (IIS Express, `iisSettings`, project-named profiles), to rename and to add; the existing ports are kept; which old profile becomes `Development` and which `Infra` is decided by the user when it is not evident from the connection strings;
- appsettings files to rename along with their environment (e.g. `appsettings.Dev.json` -> `appsettings.Development.json`, values for the shared infrastructure -> `appsettings.Infra.json`), with the split of their values where one file served both scenarios; branch-dependent values found in profile or environment files are proposed as `appsettings.<Branch>.json` entries;
- local connection strings in `appsettings.json` to move into `appsettings.Development.json` / `appsettings.Infra.json` - only after the user has confirmed that deployment does not rely on them;
- code depending on environment names (`IsDevelopment()`, `IsEnvironment(...)`, `IsProduction()`, string comparisons with environment names), especially `IsDevelopment()` checks whose behavior is also needed with `Infra` (e.g. Swagger) and checks for removed environment names; each one is reported with a proposal (e.g. an explicit setting), never silently rewritten (rule 6);
- every reference to old profile names or `LaunchProfile=<index>` arguments in launcher configurations, scripts, pipelines, `/doc` and the project skill.

### E. Repository Conventions

Check conformance with `smart-solution-and-project-structure` and the conventions visible in comparable repositories of the landscape (e.g. `dotnet\` with solution, `.nuspec` files, `src\`, `tests\`, `dist\`).

**Repository conventions and solution layout** (sections `Repository Conventions` and `Solution Layout (.NET)` of `smart-solution-and-project-structure`) - check and plan:

- default branch is `master` (report a deviation; never change repository settings or create `main`);
- vendor prefix in solution, project, assembly, namespace or package names: breaking, record in `doc/work/tech-debt.md`;
- `.sln` virtual folders: `Solution Items` with `..\doc\changelog.md` and every `.nuspec`, `src` with shared and non-target projects, `src/targets` with the target shells, `tests` with the test projects - non-breaking, fix in the plan;
- physical test folder `test\` -> `tests\`: separate plan item (paths in `.sln`, pipelines and scripts change along);
- applications, tools and tests `IsPackable=false` and without `.nuspec`.

**Repository layout** (section `Repository Layout of Standalone Solutions` of `smart-solution-and-project-structure`; in repositories without code `/doc` and its files are optional, but what exists must conform) - check and plan:

- root: `README.md` present, follows the default outline and links every file directly in `/doc` plus `decisions/`, `epics/` and `work/`; `LICENSE` present in open-source repositories (missing: report and ask which license - never pick one); `AGENTS.md` present and pointing to the mandatory skill, the project skill and `/doc`; `CLAUDE.md` containing only `@AGENTS.md`; `.editorconfig` in the root identical to the leading version of `csharp-coding-standards` (older copies in sub-folders replaced, deviating rules reviewed and either dropped or moved to a nested file; introducing it never triggers a reformatting of existing code, see rule 3);
- `.agents/skills/`: current copy of `include-additional-mandatory-skills` (or only the organization twin) and a project skill `most-important-details-for-<repo>`;
- `/doc`: numbered files renamed (`1-requirements.md` -> `requirements.md`, ...) with all links updated; no `README.*`; only documents with clear semantics; `ecosystem.md` present and limited to relevant dependencies / consumers whose visibility is not narrower than the repository's; `ideas.md` split into `epics/epic-<title>.md`; loose files sorted per the rules for further documents (clear specifications stay with a lower-case hyphenated English name, AI skill content moves to `.agents/skills/`, working notes to `epics/` / `work/` / `decisions/`, drafts and parallel versions are merged; technical artifacts such as tool binaries or pipeline exports are left alone); `decisions/` for significant decisions (deliberate deviations found in this process are good candidates); `work/backlog.md` and `work/tech-debt.md` present (legacy `design-debt.md` or similar moved into `work/tech-debt.md`); `changelog.md` untouched.

The migration renames and re-sorts; it never loses information: every statement of an old document (numbered files, `ideas.md`, a `README.*` inside `/doc`, ad-hoc notes) ends up in its new place, e.g. ideas -> `epics/epic-<title>.md`, open to-dos -> `work/backlog.md`, known shortcuts -> `work/tech-debt.md`. Use `git mv` where a file keeps its role. Content for new documents is derived from the code and existing docs; where intent is unknown (e.g. requirements, motivation), create a short skeleton and list the open questions instead of inventing content.

Fix only what is non-breaking and structural. Everything that would require renaming assembly names, package IDs or namespaces, or another breaking change visible outside the repository, goes into `/doc/work/tech-debt.md` of the repository (create it if missing; not into `changelog.md`):

```markdown
## <short title>

- Found: YYYY-MM-DD
- Deviation: <what deviates from which convention / skill>
- Why not fixed: <breaking change for consumers, ...>
- Proposed resolution: <e.g. at the next major version>
```

## 4. Phase 2 - Plan

Present the plan before changing anything, grouped by aspect A-E. For each item: action, affected files, risk (`non-breaking` / `visible to consumers` / `needs decision`), and open questions. Separate clearly:

- items proposed for execution,
- items only reported (e.g. gross package violations without a decision, untouched project files),
- tech-debt entries to be written (`doc/work/tech-debt.md`),
- questions the user must answer first.

Then stop and wait for confirmation.

## 5. Phase 3 - Execution (after confirmation only)

1. Execute only the confirmed items, in this order: target framework structure and renames (B) -> project files (B) -> `.nuspec` files (C) -> confirmed package changes (A) -> bootstrap migration (D) -> repository layout and documentation, including tech-debt entries (E).
2. Build the solution when tooling is available (e.g. `dotnet build <solution>`) and check that the `dist\Shared\<tfm>\` outputs referenced by the `.nuspec` files exist. If not possible, say so.
3. Report: what was changed, what was verified, what was not, open questions, proposed changelog entries. Do not commit.

## 6. Checklist

- [ ] Analysis complete for A-E before any change; plan presented; confirmation received.
- [ ] Web hosts `net10.0` only; contracts `net4x` + `net8.0` + `net10.0`, no `net6.0`.
- [ ] Renamed projects: folder, file, `.sln` entry, `ProjectReference`s, linked files, `.nuspec` sources and paths, pipelines updated.
- [ ] `AssemblyName`, `PackageId`, namespaces unchanged; deviations recorded in `/doc/work/tech-debt.md`.
- [ ] Root: `README.md` (links to all `/doc` files), `LICENSE` (open source), `AGENTS.md`, `CLAUDE.md` (`@AGENTS.md`), `.editorconfig` mirror; `.agents/skills/` with mandatory skill copy and project skill.
- [ ] `.sln`: `Solution Items` (changelog, `.nuspec` files), `src` / `src/targets`, `tests`; vendor prefixes only reported.
- [ ] `/doc`: no numeric prefixes, no `README.*`, `ecosystem.md`, `decisions/`, `epics/`, `work/backlog.md`, `work/tech-debt.md`.
- [ ] `.nuspec` paths match `dist\Shared\<tfm>\`; runnable artifacts build into `dist\<ArtifactName>\<tfm>\`, tests into their own `bin`; exactly one KVU comment in every `.nuspec` (at least `"KVU": { }`), no braces in other comments; `versioningTwins` = same-repository siblings; sources resolve.
- [ ] Bootstrap reviewed; migration only if confirmed.
- [ ] Launch profiles: `Development` + `Infra` (+ `Infra-<Qualifier>`), profile name = environment name, no IIS Express / project / branch names, ports kept; appsettings files renamed along; no local connection strings in `appsettings.json` (if confirmed); environment-name checks reported; launcher / script references updated.
- [ ] No unconfirmed package changes; no coding-rules sweep.
- [ ] `changelog.md` changed at most below `## Upcoming Changes`; consumer-visible changes reported with a proposed commit prefix or entry.
- [ ] Build result reported honestly; nothing committed.
