# AI-SKILLS-SMART

Consolidated snapshot skills for the official, public SmartStandards and UShell open-source landscape (including selected public KornSW repositories that belong to it). Only knowledge traceable to public upstream repositories is admitted, so this repository can be cloned legitimately as a knowledge source in any organizational context.

The libraries are developed in decentralized, domain-driven repositories, each with its own authoritative AI skills. This repository does not duplicate them. It provides one compact **snapshot skill per problem domain**: a precise description, cached essentials for everyday use, a snapshot date with staleness alarm, and references to the upstream repositories for library development and deep troubleshooting.

How snapshot skills are built, refreshed and retired is defined in [smart-skill-maintaining-guidelines](.agents/skills/smart-skill-maintaining-guidelines/SKILL.md). Agents working in this repository must load it first.

## Snapshot Index

| Skill | Problem domain | Upstream repositories | Snapshot date |
| --- | --- | --- | --- |
| [smart-skill-maintaining-guidelines](.agents/skills/smart-skill-maintaining-guidelines/SKILL.md) | Rules of this repository (not a snapshot), incl. the recurring instruction scope cleanup | - | 2026-10-22 |
| [smart-standards-library-ecosystem](.agents/skills/smart-standards-library-ecosystem/SKILL.md) | Index of all repositories of the landscape (not a snapshot, not copied) | all | - |
| [include-additional-mandatory-skills](.agents/skills/include-additional-mandatory-skills/SKILL.md) | Template skill (leading source here, may be duplicated) | - | - |
| [master-architect](.agents/skills/master-architect/SKILL.md) | General skill: capability-first architecture and wire-up reasoning (not a snapshot) | - | - |
| [additional-fields-via-smart-extensiondata](.agents/skills/additional-fields-via-smart-extensiondata/SKILL.md) | General skill: unknown JSON fields via SmartExtensionBag (not a snapshot) | - | - |
| [dotnet-t4-source-embedding](.agents/skills/dotnet-t4-source-embedding/SKILL.md) | General skill: embedding foreign C# sources via T4 (not a snapshot) | - | - |
| [asp-net-core-development-guidelines](.agents/skills/asp-net-core-development-guidelines/SKILL.md) | General skill: designed-state application bootstrap (canonical Program.Main, partial hooks) and explicit controller exposure in ASP.NET Core (not a snapshot) | - | - |
| [csharp-coding-standards](.agents/skills/csharp-coding-standards/SKILL.md) | General skill: conservative, explicit C# coding standards and AI implementation rules (not a snapshot) | - | - |
| [package-version-preferences-dot-net](.agents/skills/package-version-preferences-dot-net/SKILL.md) | General skill: preferred NuGet version ranges, pins, competing technologies, reference solutions (not a snapshot) | - | - |
| [dotnet-sdk-style-project-files](.agents/skills/dotnet-sdk-style-project-files/SKILL.md) | General skill: designed state of SDK-style .csproj files (section order, canonical BUILD, project-type modules) (not a snapshot) | - | - |
| [refactoring-processes](.agents/skills/refactoring-processes/SKILL.md) | General skill: standardized high-level refactoring processes; currently bringing an existing solution up to date (analysis, plan, confirmation; target frameworks, renames, nuspec / KVU, bootstrap, conventions, design debt) (not a snapshot) | - | - |
| [wordpress-plugin-development-guidelines](.agents/skills/wordpress-plugin-development-guidelines/SKILL.md) | General skill: uniform conventions for WordPress plugins (repository layout, entry file, naming and prefixing, admin UI, storage, security, testing) (not a snapshot) | - | - |
| [nuget-semantic-versioning-kvu](.agents/skills/nuget-semantic-versioning-kvu/SKILL.md) | General skill: changelog-driven semantic versioning with KVU, .nuspec pre-processor configuration, kvu command line and build pipeline procedure (not a snapshot) | `VersioningUtil` | 2026-10-21 |
| [smart-solution-and-project-structure](.agents/skills/smart-solution-and-project-structure/SKILL.md) | Repository and /doc layout of standalone solutions (landscape convention) and the ASP.NET Core hosting template | `AspNetCoreTemplate` | 2026-10-19 |
