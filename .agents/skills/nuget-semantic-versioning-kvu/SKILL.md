---
name: nuget-semantic-versioning-kvu
description: Changelog-driven semantic versioning with KVU (KornSW VersioningUtil) - changelog.md as release database, versioninfo.json, change classification (patch, New Feature, Breaking Change, 0.x rule, **MVP**), git fallback, pre-releases from branch names, versions in .csproj / AssemblyInfo / package.json / .nuspec, initial changelog.md / versioninfo.json for new repositories, the package design (one multi-target package per library, standalone .nuspec template), the .nuspec pre-processor (versioningTwins, dependencySync, constraintLevel), the kvu command line (DoMagic, ...), the Azure DevOps pipeline procedure (DoMagic before the build, commit back "-> VERSIONING [skip ci]"), and a GitHub Actions variant for WordPress plugins (Python release script, self-update). Load when reading or writing changelog.md or versioninfo.json, computing a version, editing .nuspec dependencies or the KVU comment, setting up or debugging a KVU pipeline, or adding automatic versioning / self-update to a WordPress plugin repository.
metadata:
  version: "2026-10-23"
---

# NuGet Semantic Versioning with KVU

KVU (`kvu.exe`, repository `VersioningUtil`) computes semantic versions from a `changelog.md` and patches them into project, package and `.nuspec` files. This skill is the compact, cached working knowledge. The leading sources are upstream (section 9): `kvu-versioning-rules` (normative, technology-agnostic rules) and `most-important-details-for-kvu` (command line, `DoMagic`, `.nuspec` pre-processor, pipeline, agent rules). New knowledge goes there first, then into this skill.

Status: synchronized 2026-10-21 against `VersioningUtil` @ `732e6a4` (`kvu-versioning-rules` v1.1, `most-important-details-for-kvu` 2026-10-21, README, source code); `versioninfo.json` keys, `DoMagic` nuspec handling and group matching re-checked in the source on 2026-10-23 (still `732e6a4`). Apply the staleness thresholds of `smart-skill-maintaining-guidelines` (90 / 180 days) and do a quick upstream check before relying on details after that.

## 1. Core Model

- **`changelog.md` is the leading release database**, not just documentation: last official version, pending changes, pre-release state. Developers never assign version numbers up front; a build run materializes exactly one version.
- **`versioninfo.json`** sits next to it and is the machine-readable extract of the same calculation (`currentVersionWithSuffix`, `releaseType`, `currentVersion`, `currentMajor` / `currentMinor` / `currentFix`, `previousVersion`, `changeGrade`, `versionDateInfo`, `versionTimeInfo`, `versionNotes`). Never let both disagree.
- **`currentFix`, not `currentPatch`**: current KVU serializes its `VersionInfo` class, whose property is `currentFix` (verified 2026-10-23 at `732e6a4`). `currentPatch` came from an older, hand-written serializer (now commented out in `KVersioningHelper.cs`); files that still contain it were produced by an outdated `kvu.exe` (e.g. a copy bundled in `/doc`). Current KVU ignores the key when reading and writes `currentFix` on its next run. Do not "correct" such files by hand.
- Location in the landscape: `/doc/changelog.md` and `/doc/versioninfo.json` (older repositories: a `vers` folder).
- **New repositories** contain both files from the first commit on: the changelog with the standard header (section 2), `## Upcoming Changes` with `*(none)*` and no release block, plus this `versioninfo.json` (key order as written by KVU):

```json
{
  "currentVersionWithSuffix": "0.0.0",
  "releaseType": "",
  "currentVersion": "0.0.0",
  "currentMajor": 0,
  "currentMinor": 0,
  "currentFix": 0,
  "previousVersion": "0.0.0",
  "changeGrade": "initial",
  "versionTimeInfo": "00:00:00",
  "versionDateInfo": "1900-01-01",
  "versionNotes": ""
}
```

## 2. changelog.md Rules (Essentials)

```markdown
# Change log
This files contains a version history including all changes relevant for semantic versioning...
*(it is automatically maintained using the ['KornSW-VersioningUtil'](https://github.com/KornSW/VersioningUtil))*

## Upcoming Changes

*(none)*

## v 1.4.0
released **2026-09-04**, including:
 - **New Feature:** Added export
```

| Entry in `Upcoming Changes` | Effect from `1.x` | Effect during `0.x` |
| --- | --- | --- |
| any text (`- **Bug Fix:** ...` is only readable sugar) | patch | patch |
| contains `New Feature` (case-insensitive substring) | minor | minor |
| contains `Breaking Change` (case-insensitive substring) | major | **minor** (no automatic 1.0.0) |
| contains exactly `**MVP**` (bold is part of the trigger) | no special meaning | **1.0.0** |

- Highest impact wins; several entries never add up. Lower digits reset on a bump.
- Canonical writing: `- **New Feature:** ...`, `- **Breaking Change:** ...`, `- **MVP** reached`. Avoid the trigger words in free text.
- Release blocks in the history are never rewritten or re-sorted.
- **Git fallback**: only if `Upcoming Changes` holds no entries. Commit subjects since the latest merged version tag (else since repository start, max. 20 commits) become normal entries and are persisted in the changelog. Subjects containing `VERSIONING` are ignored; `[skip ci]` is stripped.
- **Pre-releases**: the caller passes a semantic (usually the branch name). It is trimmed, lower-cased and appended (`1.5.0-feature-x`) - no further sanitizing, so branch names with `/` are a caller problem. Patterns in the ignore list (default `master;main;rel-*`, wildcards, case-insensitive) produce regular releases. The pre-release state is stored in the header (`## Upcoming Changes (1.5.0-feature-x)`); repeated builds of the same line advance the patch (`1.5.1-feature-x`, ...). A later regular build may continue at `1.5.1` instead of `1.5.0` (status-quo behavior, do not "fix" it). `official` is a special semantic; see upstream.

### Agent Rules for Communicating Changes

- Never guess or raise version numbers manually. In `changelog.md`, never change anything outside `## Upcoming Changes` (no history blocks, no header, no markers).
- **Preferred: commit messages.** Pipelines running current KVU (`DoMagic` with the pre-processor) take the git commit subjects into the changelog automatically. Use the magic prefixes deliberately in the commit subject: `Fix: ...` (or no prefix), `New Feature: ...`, `Breaking Change: ...`, `**MVP** ...`. This avoids merge conflicts, because the pipeline itself commits into `changelog.md`.
- **For more detailed descriptions:** add conformant entries (section 2) directly to `changelog.md`, **only below `## Upcoming Changes`**, replacing `*(none)*`.
- Note: the commit subjects are taken over only when `Upcoming Changes` contains no entries (git fallback). As soon as one entry stands there, the commit subjects of that release are ignored - then everything relevant (including the impact) must be in `Upcoming Changes`.
- Commits are only created when the user asks for them (general agent rule); the prefix applies to the commit message proposed or used then.

## 3. Where Versions Are Written

- `.csproj` / `.vbproj` (SDK style): `Version`, `AssemblyVersion`, `FileVersion`; classic projects: `AssemblyInfo.cs` / `.vb`; `package.json` (UTF-8 without BOM); `.nuspec` `<version>`.
- Plain text files can carry placeholders like `{currentVersion}` or `{currentVersionWithSuffix}` (all keys of `versioninfo.json`), resolved by `ReplaceVersionPlaceholders`.
- In the repository the versions only change through the pipeline's commit back (section 6); manual edits cause merge conflicts and are exactly what KVU exists to avoid.

## 4. .nuspec Files and Pre-Processor Configuration

### Package Design (Landscape Convention, 2026-10-23)

- Every multi-targeted library is published as **one** NuGet package containing all its targets.
- A classic, standalone `<Package>.nuspec` lives **next to the solution** (`dotnet/`) and is a solution item (see `smart-solution-and-project-structure`, `Solution Layout (.NET)`). No `dotnet pack` of the `.csproj`, no `$id$` / `$version$` or other replacement tokens: everything is written out. `<version>` is written by KVU.
- Root element **with** the nuspec namespace: `<package xmlns="http://schemas.microsoft.com/packaging/2012/06/nuspec.xsd">`, XML declaration with `encoding="utf-8"`.
- Metadata: `id` (= assembly name, no vendor prefix), `version`, `title`, `authors`, `owners`, `copyright`, `description`, `tags`, license, `requireLicenseAcceptance` false, `icon` `PackageIcon.png` (file next to the `.nuspec`), `projectUrl`.
- License: SPDX comment plus `<license type="expression">` for licenses with an SPDX identifier (e.g. `GPL-3.0-only`). The landscape default for new repositories, MIT + Commons Clause, has none (and nuget.org accepts expressions only for OSI / FSF-approved licenses): use `<license type="file">LICENSE</license>` and pack the root `LICENSE` file. Not yet verified with a real `nuget pack`; check on first use.
- `dependencies`: one `<group targetFramework="...">` per target (`.NETFramework4.8`, `net8.0`, `net10.0`) with `SEM-SAFE` ranges `[x.y.z, (x+1).0)`, maintained by KVU.
- `files`: the icon (and the license file, if used), then per target a commented block with `dist\Shared\<tfm>\<Assembly>.dll`, `.xml` and `.pdb` -> `lib/<tfm>`.
- The **KVU comment is mandatory** (see below): `dependencySync` with one source per target shell (`"targetFramework":"="`; the `net48` shell needs the explicit `".NETFramework4.8"`, because KVU compares the project's raw `TargetFramework` value with the group name as plain text, verified 2026-10-23), `constraintLevel` `SEM-SAFE`, `addMissing` / `updateExisting` / `allowDowngrades` true, `removeOrphaned` false; `versioningTwins` lists the same-repository packages referenced via `ProjectReference`.
- Applications, tools and test projects are not packaged (`IsPackable` false, no `.nuspec`).

```xml
<?xml version="1.0" encoding="utf-8"?>
<package xmlns="http://schemas.microsoft.com/packaging/2012/06/nuspec.xsd">
  <metadata>
    <id>Example.Product</id>
    <version>0.0.0</version>
    <title>Example Product</title>
    <authors>ExampleVendor</authors>
    <owners>ExampleVendor</owners>
    <copyright>ExampleVendor</copyright>
    <description>What the package does</description>
    <tags>Example Product</tags>
    <!-- https://spdx.org/licenses/ (MIT + Commons Clause has no SPDX identifier, therefore the license file) -->
    <license type="file">LICENSE</license>
    <requireLicenseAcceptance>false</requireLicenseAcceptance>
    <icon>PackageIcon.png</icon>
    <projectUrl>https://github.com/ExampleVendor/Example.Product</projectUrl>
    <dependencies>
      <!-- (configuration for the Versioning-Util, which is not part of the xml-schema, because it's a pre-processor)
      "KVU": {
        "versioningTwins":[ ],
        "dependencySync": {
          "addMissing":true, "updateExisting":true, "allowDowngrades":true, "removeOrphaned":false,
          "constraintLevel":"SEM-SAFE",
          "sources": [
            { "origin":"src\\Example.Product.net48\\Example.Product.net48.csproj", "targetFramework":".NETFramework4.8" },
            { "origin":"src\\Example.Product.net8.0\\Example.Product.net8.0.csproj", "targetFramework":"=" },
            { "origin":"src\\Example.Product.net10.0\\Example.Product.net10.0.csproj", "targetFramework":"=" }
          ]
        }
      } -->
      <group targetFramework=".NETFramework4.8">
      </group>
      <group targetFramework="net8.0">
      </group>
      <group targetFramework="net10.0">
      </group>
    </dependencies>
  </metadata>
  <files>
    <file src="PackageIcon.png" target="" />
    <file src="..\LICENSE" target="" />
    <!-- .NET 4.8 -->
    <file src="dist\Shared\net48\Example.Product.dll" target="lib/net48" />
    <file src="dist\Shared\net48\Example.Product.xml" target="lib/net48" />
    <file src="dist\Shared\net48\Example.Product.pdb" target="lib/net48" />
    <!-- .NET 8.0 -->
    <file src="dist\Shared\net8.0\Example.Product.dll" target="lib/net8.0" />
    <file src="dist\Shared\net8.0\Example.Product.xml" target="lib/net8.0" />
    <file src="dist\Shared\net8.0\Example.Product.pdb" target="lib/net8.0" />
    <!-- .NET 10.0 -->
    <file src="dist\Shared\net10.0\Example.Product.dll" target="lib/net10.0" />
    <file src="dist\Shared\net10.0\Example.Product.xml" target="lib/net10.0" />
    <file src="dist\Shared\net10.0\Example.Product.pdb" target="lib/net10.0" />
  </files>
</package>
```

Known debt in the reference solution `ComponentDiscovery` @ `a159ddb`: its `.nuspec` files have no KVU comment and no namespace. They still received new versions, most likely because that repository's pipeline runs an outdated KVU flow (`kvu.exe` bundled in `/doc`, separate version steps; the pipeline definition is not in the repository). Under `DoMagic` they would freeze. The skill is correct; the files are wrong (recorded in that repository's `doc/work/tech-debt.md`).

### Pre-Processor Configuration

KVU reads **exactly one** JSON configuration from an XML comment in the `.nuspec` (the root may be wrapped as `"KVU": {...}`; comments and trailing commas allowed). More than one fails. **No configuration: `DoMagic` skips the file completely - not even the version is written** (verified 2026-10-23 in `ProcessNuspecFiles`; `DoMagic`'s `ImportVersion` pattern excludes `*.nuspec`). Therefore every `.nuspec` must contain a KVU comment, at least the minimal form `"KVU": { }` (open design debt in KVU's `doc/work/tech-debt.md`). The first `{...}` of **every** XML comment is parsed (no `KVU` marker check): keep braces out of other comments.

```xml
<!-- (configuration for the Versioning-Util, which is not part of the xml-schema, because it's a pre-processor)
"KVU": { } -->
```

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

Processing order per `.nuspec`: write version -> dependency sync -> versioning twins.

- **`versioningTwins`**: packages built from the **same repository** that this package references as `ProjectReference`. They are written as exact `[<currentVersionWithSuffix>]` dependencies into every group and excluded from sync and orphan removal. List only same-repository siblings.
- **`dependencySync.sources`**: project files (paths relative to the `.nuspec`) whose `PackageReference`s are copied into the `.nuspec`. `targetFramework`: `"="` = into the group whose name equals the source's raw `TargetFramework` text (default when the `.nuspec` has groups; `net48` does not match `.NETFramework4.8` - a new group would be created), `"*"` = all groups, an explicit TFM = only that group; empty without groups = flat list. Once groups exist, they are authoritative.
- Flags (global, overridable per source): `addMissing` (true), `updateExisting` (true), `allowDowngrades` (true), `removeOrphaned` (false); `whitelist` / `blacklist` of package IDs.
- **`constraintLevel`**: `KEEP` (default, constraint copied as is), `EXACT` (`[x.y.z]`), `MIN` (`x.y.z` = minimum), any other value, conventionally `SEM-SAFE` = non-breaking range `[x.y.z, (x+1).0)`; for `0.x` the upper bound is `2.0` to allow the regular move to `1.0`.

## 5. Command Line

`kvu <Command> <positional arguments...>` - the command name is case-insensitive; quote arguments with spaces; file patterns are minimatch lists separated by `;` (`!` excludes). Without arguments `kvu` starts an interactive console (F1 lists commands). The first token is the command itself: do not prefix it with `/` or `-` (those mark named arguments for the parser).

| Command | Arguments (defaults) | Purpose |
| --- | --- | --- |
| `DoMagic` | `entryDirectory` (current), `preReleaseSemantic` (""), `ignoreSemantic` (`master;main;rel-*`) | Full workflow, see section 6. |
| `CreateNewVersionOnChangelog` | `targetFile` (`versioninfo.json`), `changeLogFile` (`changelog.md`), `preReleaseSemantic`, `ignoreSemantic` | Compute version, transform changelog, write target. |
| `ImportVersion` | `targetFilesToProcess`, `metaDataSourceFile` (`versioninfo.json`) | Patch the version into matching files. |
| `ProcessNuspecFiles` | `targetFilesToProcess`, `metaDataSourceFile` | Section 4 for matching `.nuspec` files. |
| `ReplaceVersionPlaceholders` | `targetFilesToProcess`, `metaDataSourceFile` | Resolve `{...}` placeholders. |
| `SetVersion` / `IncreaseVersion` / `SetPrerelease` | version / majority (1-3) / suffix, `targetFilesToProcess` | Manual tooling, not for the normal flow. |
| `ReadVersion`, `ListDependencies`, `ListFiles` | file / pattern | Diagnostics (test minimatch patterns with `ListFiles`). |
| `InjectIntoAzureDevOpsBuildNumber`, `InjectIntoAzureDevOpsVariables` | `metaDataSourceFile` | Emit `##vso[...]` commands. |
| `ImportGitCommentsIntoChangelog`, `UpdatePackageReference`, `CopyDependencyEntries`, `CopyVersionToDependencyEntry`, `SetVersionToDependencyEntry` | see source | Special tooling. |

## 6. Build Pipeline Procedure

The landscape builds with Azure DevOps; a reference YAML pipeline is not documented yet. Run versioning **after checkout and before restore / build**, so every later step uses the same version.

1. **Checkout** with full history and tags (no shallow clone) and credentials that may push to the source branch.
2. **Install KVU**: latest release from GitHub (`doc/install-latest-kvu.ps1` of `VersioningUtil`: downloads `kvu.exe` with its side files to `C:\kvu` and puts it on the PATH; Windows agent) or the Azure DevOps extension `kVersioningUtil` (publisher `KornSW`).
3. **Version**: `kvu DoMagic "<repository root>" "<branch name>" "master;main;rel-*"` (Azure DevOps: `$(Build.SourceBranchName)`). `DoMagic`:
   1. finds exactly **one** `changelog.md` below the entry directory (ignoring `bin`, `obj`, `packages`, `node_modules`) - several fail; then start from a more specific directory;
   2. computes the version (with git fallback) and writes `changelog.md` and the neighboring `versioninfo.json`;
   3. sets the Azure DevOps build number and the variables `CurrentVersion`, `CurrentVersionWithSuffix`, `PreviousVersion`, `ChangeGrade`, `ReleaseType`, `PreReleaseSuffix`, `CurrentMajor`, `CurrentMinor`, `CurrentFix`, `VersionDateInfo`, `VersionTimeInfo`, `VersionNotes` (other CI systems: read `versioninfo.json`);
   4. patches `*.csproj`, `*.vbproj`, `package.json`, `AssemblyInfo.cs` / `.vb`;
   5. processes all `*.nuspec` files with a KVU configuration (section 4).
4. **Build** the solution and **pack** the `.nuspec` files (they reference `dist\Shared\<tfm>\` outputs, see `dotnet-sdk-style-project-files`); publish packages.
5. **Commit back** (usually after the package push) all files changed by KVU (changelog, `versioninfo.json`, project files, `AssemblyInfo`, `.nuspec`) with the message `<subject of the triggering commit> -> VERSIONING [skip ci]` and push to the source branch. `VERSIONING` keeps the commit out of the next git fallback; the skip-CI marker is **mandatory** - without it the push restarts the pipeline and produces versions in an endless loop.

Older pipelines use separate steps (`CreateNewVersionOnChangelog ".\doc\versioninfo.json" ".\doc\changelog.md" <branch> <primary branch>`, then `ImportVersion` per file type). They do not process the `.nuspec` pre-processor configuration; prefer `DoMagic`.

### Special Case: WordPress Plugins (GitHub Actions)

WordPress plugin repositories on GitHub use a minimal Python re-implementation of this flow instead of `kvu.exe`: `.kvu/release.py` versions the repository, retrofits a GitHub-based self-update into each plugin and publishes a GitHub release with a plugin ZIP; `.github/workflows/release.yml` runs it on every push to `master`. Deviations: no pre-releases (`master` only), and the current script has a regex defect that never recognizes `New Feature` / `Breaking Change` (every release becomes a patch; see the reference). Write prefixes in the canonical bold form. To introduce it, follow [references/wordpress-plugins.md](references/wordpress-plugins.md) and copy the files from [assets/wordpress/](assets/wordpress/).

## 7. Quick Troubleshooting

| Symptom | Likely cause | Quick check |
| --- | --- | --- |
| Pipeline runs in a loop, versions explode | Commit back without skip-CI marker | Commit message of the bot commits |
| `kvu /DoMagic ...` fails (unknown command or exception) | Leading `/` turns the command into a named argument | Call `kvu DoMagic ...` |
| `.nuspec` keeps old version | No KVU configuration comment in it | Search the `.nuspec` for `"KVU"`; add at least `"KVU": { }` |
| `.nuspec` without KVU comment still gets versions | Outdated pipeline (separate `ImportVersion` steps or an old bundled `kvu.exe`) | Pipeline steps; add the KVU comment before switching to `DoMagic` |
| `versioninfo.json` contains `currentPatch` | Written by an outdated `kvu.exe` | Which `kvu.exe` the pipeline runs (section 1) |
| "more than one KVU pre-processor configuration" / "failed to parse it" | Another XML comment contains braces | Comments in the file |
| "Found more than one changelog.md" | Additional changelog below the entry directory | Pass a more specific entry directory |
| Sibling package referenced with a wrong version | Missing in `versioningTwins` | `ProjectReference`s of the source projects |
| "KVU source ... does not exist" | `origin` path stale after a project rename | Paths relative to the `.nuspec` |
| Unexpected version jump | Trigger word in free text, or git fallback imported commits | Upcoming entries; were they empty? |
| Pre-release suffix on a release branch | Branch not in the ignore list | Third `DoMagic` argument |
| Version not as on the last pre-release | Known status-quo behavior (section 2) | Header `## Upcoming Changes (...)` |

## 8. Neighbors

- `refactoring-processes` - checks `.nuspec` paths and KVU configuration when a solution is brought up to date.
- `dotnet-sdk-style-project-files` - `dist\Shared\<tfm>\` output layout the `.nuspec` files point to (role-based layout: `smart-solution-and-project-structure`, `Build Output Layout (dist)`).
- `package-version-preferences-dot-net` - which external package versions are wanted (KVU only synchronizes them).

## 9. Upstream Sources

| Repository | URL | Skills | Docs |
| --- | --- | --- | --- |
| `VersioningUtil` (KVU) | https://github.com/KornSW/VersioningUtil | `.agents/skills/kvu-versioning-rules/` (normative rules incl. conformance tests and pseudocode), `.agents/skills/most-important-details-for-kvu/` (implementation, pre-processor, pipeline, agent rules) | `readme.md`, `doc/` (install script, Azure DevOps task group exports) |
| `WP-AutoUpdate-G2` (WordPress variant, leading template) | https://github.com/KornSW/WP-AutoUpdate-G2 | - | `doc/Update-Logic-Dokumentation.md`; assets copied into this skill |

Escalate to upstream (local clone preferred) when changing KVU itself, when exact parser or calculation details matter (pre-release edge cases, `official`, git tag boundaries), when observed behavior contradicts this skill, or after the staleness thresholds. Upstream wins on conflict.
