---
name: smart-solution-and-project-structure
description: Common repository conventions and the ASP.NET Core hosting template of the SmartStandards landscape - currently the SmartStandards.AspNetCoreTemplate / SmartStandards.WebApiDependencies startup pattern (Program.Main, LaunchProfileHelper with "LaunchProfile=<name>", branch-specific appsettings files, SmartStandardsCheapInitializer). Use when writing or debugging Program.Main of a SmartStandards-style ASP.NET Core Web API, when a service behaves differently started from Visual Studio vs. as EXE (e.g. falls back to port 5000), or when appsettings.<Branch>.json files are involved. Snapshot of the upstream skills; escalate to the upstream repositories for library development or deep troubleshooting.
metadata:
  snapshot-date: "2026-10-19"
---

# Solution and Project Structure (Snapshot)

<!-- Layer 1 - Domain Overview -->

How SmartStandards repositories and hosts are structured and started. This snapshot currently covers only the ASP.NET Core hosting template; the general repository conventions of the domain (shared projects, multi-targeting, generated versioning artifacts, documentation layout) are not yet snapshotted - read the `most-important-details-for-*` skill of the respective repository.

- Libraries / repositories in scope: `AspNetCoreTemplate` (package `SmartStandards.WebApiDependencies`)
- Use for: `Program.Main` of a SmartStandards-style Web API, launch profiles for directly started EXEs, branch-specific configuration files
- Out of scope: UJMW, SmartAmbience and logging internals (only wired by the template) - see `smart-http-remoting-standard`, `smart-dataflow-and-ambience`; versioning - see `smart-versioning-and-nuget`

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
