---
name: package-version-preferences-dot-net
description: Preferred version ranges of well-known .NET NuGet packages (legacy lower bound and medium-term upper bound), pinned versions with their reasons (e.g. MSTest 3.11.1), the status of competing technologies (preferred / tolerated / banned, e.g. Newtonsoft.Json preferred, System.Text.Json tolerated) and the reference solutions that represent the leading versions (AspNetCoreTemplate = state of the art, UniversalBFF = bleeding edge). Load before adding, updating, downgrading or choosing a NuGet package or test framework, when a package version looks unusual, or when choosing between competing libraries. Includes a cyclic review rule and the duty to actively question grossly outdated or much too new packages with the user.
metadata:
  version: "2026-10-22"
---

# Package Version Preferences (.NET)

This general skill records which versions of well-known NuGet packages are preferred and which of competing technologies are wanted. It describes preferences, not the actual versions of a given repository; the `.csproj` / `.nuspec` files of the repository remain the truth for what is used.

Package changes are never a side effect: do not add, upgrade or downgrade packages as part of unrelated work. If a referenced version lies outside the preferred range, report it and ask. When a whole solution is brought up to date, this check is aspect A of `refactoring-processes`.

## 1. Reference Solutions

These solutions represent the decisions about leading versions. When a range below is unclear or missing, compare with them first.

| Solution | Repository | Represents |
| --- | --- | --- |
| `AspNetCoreTemplate` | https://github.com/SmartStandards/AspNetCoreTemplate | **State of the art** - the versions a normal, current host should use. |
| `UniversalBFF` | https://github.com/KornSW/UniversalBFF | **Bleeding edge** - the newest versions in deliberate use; indicates where the upper bounds are heading. |

A version used in `UniversalBFF` but not in `AspNetCoreTemplate` is not automatically approved for normal hosts.

## 2. Well-Known Packages

Columns:

- **Legacy lower bound**: oldest version still accepted in existing (legacy) code.
- **Upper bound**: the version we want to reach in the medium term; do not go beyond it.
- **Pinned**: keep exactly this version for now; neither upgrade nor downgrade.
- **Observed**: versions found in the reference solutions at the given date (information only, not a decision).

| Package | Legacy lower bound | Upper bound | Pinned | Reason / note | Observed (2026-10-21) |
| --- | --- | --- | --- | --- | --- |
| `MSTest` | ? | 3.11.1 | **3.11.1** | Stay on 3.11.1 for compatibility reasons; do not upgrade (e.g. to 4.x) until this entry is changed. | `AspNetCoreTemplate`: 3.11.1 |
| `Newtonsoft.Json` | ? | ? | - | Preferred JSON library (see section 3). | `AspNetCoreTemplate`: 13.0.3, 13.0.4; `UniversalBFF`: 13.0.4 |
| `Swashbuckle.AspNetCore` | ? | ? | - | Several major versions in parallel per target framework. | `AspNetCoreTemplate`: 6.5.0, 10.1.7; `UniversalBFF`: 6.5.0, 8.0.0 |
| `Microsoft.EntityFrameworkCore` (and `.SqlServer`, `.Design`, `.Tools`, `.Proxies`) | ? | ? | - | | `UniversalBFF`: 10.0.9 |
| `Microsoft.Data.SqlClient` | ? | ? | - | | `UniversalBFF`: 7.0.1 |

`?` = not yet decided. Do not fill in a bound on your own; ask the user (section 4).

SmartStandards / UShell / KornSW packages of the landscape (e.g. `UJMW.*`, `SmartAmbience*`, `SmartStandards.*`, `ComponentDiscovery`, `FUSE-fx.*`) follow their own versioning rules; see `smart-standards-library-ecosystem` and the respective upstream skills. They are added here only when a deliberate range decision exists.

## 3. Competing Technologies

| Concern | Preferred (state of the art) | Tolerated | Banned | Note |
| --- | --- | --- | --- | --- |
| JSON serialization | `Newtonsoft.Json` | `System.Text.Json` | - | Do not introduce `System.Text.Json` without a concrete technical requirement; do not mix both in one project without a strong reason (see `csharp-coding-standards`, section 15). |
| Unit test framework | `MSTest` | ? | ? | Use MSTest for unit test projects; version pin see section 2. |

Status meaning:

- **Preferred**: use for new code.
- **Tolerated**: may stay where it already exists or where a concrete requirement demands it; do not spread it.
- **Banned**: do not introduce; propose replacement when touching affected code.

## 4. Cyclic Review

The table ages. Agents keep it alive:

- When this skill is loaded and `metadata.version` is older than 90 days, ask the user once in the session whether the ranges, pins and technology statuses are still valid. Mention the pins explicitly (e.g. "Bleiben wir bei MSTest 3.11.1?").
- Whenever a package without an entry is about to be added or updated, or a bound is `?`, ask for the intended range and record the answer here.
- Update the observed versions of the reference solutions when they are compared (record the date).
- After every confirmed change, update the table and raise `metadata.version`. Without an actual confirmation by the user, do not raise the date.

## 5. Gross Violations

Agents actively question gross violations with the user as soon as they discover them - also when the package is not part of the current task. Gross violations are:

- **totally outdated** packages: far below the legacy lower bound, several major versions behind the reference solutions, out of vendor support, or with known security vulnerabilities;
- **much too new** packages: beyond the upper bound, newer than even the bleeding-edge reference solution (`UniversalBFF`), or preview / pre-release versions in production code;
- violations of a pin (e.g. `MSTest` other than 3.11.1) or use of a banned technology.

Report them briefly (package, found version, preferred range, location) and ask how to proceed. Do not change the version on your own; minor deviations within the same major version need no question.

## 6. Agent Checklist

- [ ] No package change outside an explicit request.
- [ ] Gross violations discovered on the way questioned with the user (section 5).
- [ ] Version within the preferred range; pins respected.
- [ ] No banned technology introduced; tolerated technology not spread.
- [ ] Unclear or missing entries asked and recorded.
- [ ] Review due? Then ask (section 4).
