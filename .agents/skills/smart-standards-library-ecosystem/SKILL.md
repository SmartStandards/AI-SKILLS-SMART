---
name: smart-standards-library-ecosystem
description: Index and routing authority for the SmartStandards / UShell open-source library landscape (incl. admitted KornSW repositories) - which repositories exist, their GitHub owner and clone URL, local folder name, role, problem domain, own upstream skills, matching snapshot skill, and which repositories are excluded (legacy, proof of concept, private). Use when an unknown open-source package, repository or component appears; when looking for the repository, clone URL or skills of a library such as FUSE-fx, UJMW, SmartAmbience, ComponentDiscovery, AuthTokenHandling, KVU, UniversalBFF or UShell; before writing helper or cross-cutting code that a library of this landscape may already own; or when asked about the role or successor of a library.
metadata:
  version: "2026-10-21"
---

# SmartStandards Library Ecosystem (Index)

This skill is the phone book of the open-source landscape. It lives only in `AI-SKILLS-SMART` and is never copied into other repositories; the copyable template `include-additional-mandatory-skills` references it.

## Purpose and Scope

This skill decides **where** a topic of the open-source landscape lives: which repository owns it, how to obtain it, and which skills to load for it.

In scope: all admitted public repositories of the GitHub owners `SmartStandards`, `ProjectUShell` and the admitted `KornSW` repositories (see `Owners and Admission`).

Out of scope (refer elsewhere):

- How to work with a problem domain in everyday use -> the domain's snapshot skill (`smart-*`) in this repository.
- Library development and implementation details -> the upstream repository's own skills (`.agents/skills/`).
- Rules for maintaining this repository -> `smart-skill-maintaining-guidelines`.
- Organization-specific usage and overrides -> the organization's own skill collection (never referenced from here).

## Rule: No Redundancy

Each cross-cutting concern and each problem domain has exactly **one** owning repository. Competing implementations of the same concern are not intended.

Before writing helper, extension or cross-cutting code (e.g. logging, IDs, ambient context, token handling, HTTP remoting, data access, composition, versioning), look up the concern in `Appendix: Repository Reference`. If an owning repository exists, use or extend that library instead of re-implementing it in a consumer. Never resolve a routing question by duplicating functionality.

## Owners and Admission

Clone URL convention for all rows below: `https://github.com/<Owner>/<Repository>.git`.

| Owner | Layer / content | Admission |
| --- | --- | --- |
| `SmartStandards` | Base standards and libraries: contracts, cross-cutting concerns, FUSE-fx persistence, remoting. | Public repositories are admitted. |
| `ProjectUShell` | UShell: the universal application shell (same vendor as SmartStandards, a higher layer with its own project scope). Central docs and skills in `UShell.Docs`. | Public repositories are admitted. |
| `KornSW` | Author account hosting further libraries of the landscape, but also unrelated public repositories. | Only repositories listed in the appendix. Ask the user before admitting any other `KornSW` repository, then add it. |

Private repositories are never admitted. Verify visibility via `https://api.github.com/repos/<owner>/<repo>` (`"visibility": "public"`; HTTP 404 = private or non-existent).

## Problem Domains

The problem domains are the cut for snapshot skills. Their names follow the cut rule in `smart-skill-maintaining-guidelines` (aligned 1:1 with the established domain names of consuming skill collections).

| Snapshot skill (planned) | Problem domain | Repositories |
| --- | --- | --- |
| `smart-composition-di-componentdiscovery` | Runtime composition, assembly/type discovery, instance resolution | `ComponentDiscovery`, `DynamicProxy` (?) |
| `smart-dataflow-and-ambience` | Ambient context, distributed dataflow, scoped singletons | `SmartAmbience`, `AmbientScoping` |
| `smart-http-remoting-standard` | HTTP remoting protocol and framework | `UnifiedJsonMessageWrapper` |
| `smart-service-ecosystem` | Endpoint resolution, service registry, cyclic triggering of services | `AddressResolving`, `CyclicTriggering` (?) |
| `smart-authentication` | Access tokens: issuing, obtaining, caching, validating, forwarding | `AuthTokenHandling` |
| `smart-persistence-entityframework-fusefx` | Data access via FUSE-fx and Entity Framework | `FUSE-fx.RepositoryContract`, `FUSE-fx.ModelDescription`, `FUSE-fx.ModelReader`, `FUSE-fx.EntityAnnotations`, `FUSE-fx.ModelDesigner`, `FUSE-fx.DynDB-EF`, `FUSE-fx.DynDB-JSONfs`, `FUSE-fx.DynDB-XLS`, `FUSE-fx.DynDB-SQL`, `FUSE-fx.WinForms`, `json-flatfile-datastore.FUSE-fx`, `EntityFrameworkCore.DataAccessFilter` |
| `nuget-semantic-versioning-kvu` (exists; general skill, named by user decision 2026-10-21) | Changelog-driven semantic versioning, packaging | `VersioningUtil` (KVU) |
| `smart-solution-and-project-structure` (exists; template only so far) | Common repository conventions and the ASP.NET Core hosting template: shared projects, multi-targeting (net48/net8/net10), generated versioning artifacts, documentation layout | `AspNetCoreTemplate`; conventions of all .NET repositories (see their `most-important-details-for-*` skills) |
| `smart-ushell-and-universalbff` | UShell frontend shell and the UniversalBFF backend-for-frontend | `UShell.*` (all `ProjectUShell` repositories), `UniversalBFF` |

Not yet assigned to a domain (no established counterpart; see `Refinement Needed`): `Logging`, `Snowflake44`, `Essentials`, `AbstractFileSystem`, `AiSupport`, `UniversalWorkItem.Contract`, `CefSharp.Wrapped`, `KnowledgeManagement`.

## Growing the Reference

Breadth of this index is critical. When an agent encounters a repository or package of the landscape that is not listed:

1. Check visibility and owner (see `Owners and Admission`).
2. Ask the user for its role (short form) and whether it is admitted; for `KornSW` repositories admission is mandatory to ask.
3. Verify the exact repository name via the GitHub API; do not record guessed spellings.
4. Add a row to the appendix (or to `Excluded Repositories`) and raise `metadata.version`.

If the agent cannot edit this skill (e.g. no local clone of `AI-SKILLS-SMART`), tell the user what should be added.

## Appendix: Repository Reference

Keep rows short. "Local folder" is only filled where developers commonly use a folder name that differs from the repository name. "Upstream skills" lists the skills in the repository's `.agents/skills/` (besides the template `include-additional-mandatory-skills`). "Snapshot" names the snapshot skill in this repository once it exists. `?` means unconfirmed.

### SmartStandards

| Repository | Local folder | Role (short) | Upstream skills | Snapshot | Note |
| --- | --- | --- | --- | --- | --- |
| `AI-SKILLS-SMART` | | Snapshot skills and index of this landscape; leading source of `include-additional-mandatory-skills`. | (this repository) | - | |
| `AddressResolving` | `EndpointDiscovery` | Endpoint / address resolving (service registry). | `most-important-details-for-endpoint-discovery` | - | Formerly `KornSW/KswServiceRegistry`; old spelling `AddressRessolving` redirects. |
| `AiSupport` | | Provider-neutral .NET contracts for AI-enabled applications, standard implementations, ASP.NET Core integration, AI harness. | `ai-harness-development`, `most-important-details-for-smartstandards-aisupport` | - | |
| `AspNetCoreTemplate` | `SmartStandards.AspNetCoreTemplate` | ASP.NET Core startup and hosting template; package `SmartStandards.WebApiDependencies`. | `most-important-details-for-smartstandards-aspnetcore-template` | `smart-solution-and-project-structure` | |
| `AuthTokenHandling` | | Issuing, obtaining, caching, validating and forwarding access tokens. | `auth-token-handling`, `most-important-details-for-auth-token-handling` | - | Successor of `KornSW/KornSW.TokenValidation`. |
| `CefSharp.Wrapped` | | C# convenience wrapper for CefSharp (extracted from `W3bstract`). | - | - | |
| `CyclicTriggering` | | `ICyclicTriggerReceiver` abstraction and .NET implementation for cyclic triggers. | `most-important-details-for-cyclic-triggering` | - | |
| `Essentials` | `SmartStandards.Essentials` | Collection of SmartStandards best-practice patterns. | - | - | |
| `FUSE-fx.DynDB-EF` | | Entity Framework-backed FUSE repository implementation. | `most-important-details-for-fusefx-dyndb-ef` | - | |
| `FUSE-fx.DynDB-JSONfs` | | JSON filesystem-backed FUSE repository implementation. | `most-important-details-for-fusefx-dyndb-jsonfs` | - | |
| `FUSE-fx.DynDB-SQL` | | PHP/MySQL gateway. | `most-important-details-for-fusefx-dyndb-sql` | - | Outside the active FUSE-fx core map. |
| `FUSE-fx.DynDB-XLS` | | Excel/ClosedXML-backed FUSE repository implementation. | `most-important-details-for-fusefx-dyndb-xls` | - | |
| `FUSE-fx.EntityAnnotations` | | FUSE entity metadata annotations. | `most-important-details-for-fusefx-entity-annotations` | - | v1.x published as package `EntityAnnotations`. |
| `FUSE-fx.ModelDescription` | | Schema DOM for FUSE entity models. | `most-important-details-for-fusefx-model-description` | - | |
| `FUSE-fx.ModelDesigner` | | Model / schema designer tooling. | `most-important-details-for-fusefx-model-designer` | - | |
| `FUSE-fx.ModelReader` | | Reflection bridge from annotated entity models to FUSE model descriptions. | `most-important-details-for-fusefx-model-reader` | - | |
| `FUSE-fx.RepositoryContract` | | Central FUSE repository contract (`IRepository<TEntity, TKey>`), stores, EF support. | `fuse-fx`, `most-important-details-for-fusefx-repository-contract` | - | Entry point for FUSE-fx. |
| `FUSE-fx.WinForms` | | Windows UI controls and presenters for FUSE repository data. | `most-important-details-for-fusefx-winforms` | - | |
| `json-flatfile-datastore.FUSE-fx` | | JSON flat file data store (FUSE-adjacent fork). | `most-important-details-for-fusefx-json-flatfile-datastore` | - | Outside the active FUSE-fx core map. |
| `KnowledgeManagement` | | ? | - | - | |
| `Logging` | `SmartStandards.Logging` | Lightweight logging facade without dependencies. | `smartstandards-logging`, `most-important-details-for-smartstandards-logging` | - | |
| `SmartAmbience` | | Ambient context and distributed dataflow (AmbientField, context adapters, AmbienceHub, FlowingContracts). | `smart-ambience` | - | |
| `Snowflake44` | | Time-sortable Int64 UID scheme. | `snowflake44` | - | |
| `UnifiedJsonMessageWrapper` | | UJMW protocol and .NET framework for HTTP remoting. | `ujmw` | - | |
| `UniversalWorkItem.Contract` | | Extendable model of a universal todo / work item for boards. | - | - | |

### ProjectUShell

| Repository | Local folder | Role (short) | Upstream skills | Snapshot | Note |
| --- | --- | --- | --- | --- | --- |
| `UShell.Docs` | | Central UShell documentation and skills; consolidation repository of the UShell layer. | `ushell-architecture` (mandatory root entry), `ushell-*` (12 more, incl. `ushell-design-debt`) | - | Verified 2026-10-19. |
| `UShell.ReactFrontend` | | Productive leading React runtime of UShell. | `ushell-frontend-development` | - | Verified 2026-10-19. |
| `UShell.CommonComponents.JS` | | Generic reusable components / widgets (npm `ushell-common-components`). | `ushell-common-components-development` | - | Verified 2026-10-19. |
| `UShell.ModuleBase.JS` | | JS/TS module and widget integration contract. | `ushell-modulebase-development` | - | Verified 2026-10-19. |
| `UShell.PortfolioDescription` | | Portfolio / module / workspace / use-case description contract. | `most-important-details-for-ushell-portfolio-description` | - | |
| `UShell.ServerCommands.Contract` | | Server-side command contract and ASP.NET Core integration. | `most-important-details-for-ushell-server-commands-contract` | - | |
| `UShell.DemoApp` | | Sample module / demo application. | `most-important-details-for-ushell-demo-app` | - | |
| `UShell.ReactFrontend.CefWrapper` | | Windows / CEF offline wrapper. | `most-important-details-for-ushell-react-frontend-cef-wrapper` | - | Concept stage. |
| `UShell.FrontendBundle` | | .NET / ASP.NET Core packaging and hosting of the frontend bundle. | `most-important-details-for-ushell-frontend-bundle` | - | Temporarily private: belongs to the landscape, do not snapshot until public. |

### KornSW (admitted)

| Repository | Local folder | Role (short) | Upstream skills | Snapshot | Note |
| --- | --- | --- | --- | --- | --- |
| `ComponentDiscovery` | | Assembly resolving, type discovery, taxonomic classification / approval, InstanceDiscovery. | `component-discovery`, `most-important-details-for-component-discovery` | - | |
| `UniversalBFF` | | ASP.NET Core backend-for-frontend composition layer for UShell and UJMW-based API gateways. | `most-important-details-for-universalbff` | - | |
| `AmbientScoping` | | Scoped singletons via ambient binding identifiers (incl. distributed dataflow). | ? | - | |
| `VersioningUtil` | `KVU` | KVU: semantic versioning driven by `changelog.md`. | `kvu-versioning-rules`, `most-important-details-for-kvu` | `nuget-semantic-versioning-kvu` | |
| `WP-AutoUpdate-G2` | | Template: KVU-like versioning and GitHub-based self-update for WordPress plugins (Python script + GitHub Actions). | - | `nuget-semantic-versioning-kvu` | Admitted by user decision 2026-10-21. |
| `WP-AuthTokenHandling` | | WordPress plugin using the `WP-AutoUpdate-G2` release template unchanged (reference consumer). | - | `nuget-semantic-versioning-kvu` | Admitted by user decision 2026-10-21 (release mechanism only). |
| `AbstractFileSystem` | | Abstract file system contracts. | `most-important-details-for-abstract-file-system` | - | Suspended 2019, restarted 2024. |
| `EntityFrameworkCore.DataAccessFilter` | | Data access filters for EF Core. | `most-important-details-for-efcore-data-access-filter` | - | |
| `DynamicProxy` | | Generates interface implementations via Emit. | - | - | |

### Excluded Repositories

Not part of this landscape's index; do not snapshot or route to them.

| Repository | Reason | Successor |
| --- | --- | --- |
| `KornSW/KornSW.AccessControl` | Legacy | ? |
| `KornSW/KornSW.TokenValidation` | Legacy | `AuthTokenHandling` |
| `KornSW/T4Framework` | Legacy | ? |
| `KornSW/kGEN` | Legacy | ? |
| `KornSW/W3bstract`, `KornSW/W3bstract.Core` | Legacy | ? (`CefSharp.Wrapped` was extracted from it) |
| `KornSW/EntityFrameworkCore.ModelFirst` | Legacy | ? |
| `KornSW/LiteDB.FUSE-fx` | Proof of concept | - |

All other public `KornSW` repositories (e.g. other WordPress plugins, home automation, forks) are unrelated to this landscape.

## Refinement Needed

- Roles of `KnowledgeManagement`, `Essentials`, `AmbientScoping` (skills unknown, not cloned during creation), `AbstractFileSystem` and `EntityFrameworkCore.DataAccessFilter` to be confirmed from their READMEs.
- Domain assignment, to be decided one repository at a time: the repositories marked `(?)` and the not yet assigned repositories (`Logging`, `Snowflake44`, `Essentials`, `AbstractFileSystem`, `AiSupport`, `UniversalWorkItem.Contract`, `CefSharp.Wrapped`, `KnowledgeManagement`).
- Lifecycle status per repository (e.g. maintained, emerging, concept) is not yet recorded except where noted.
- Successors of the excluded legacy repositories.
