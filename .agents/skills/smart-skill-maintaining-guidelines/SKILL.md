---
name: smart-skill-maintaining-guidelines
description: MANDATORY and AUTHORITATIVE for ALL work inside this repository (AI-SKILLS-SMART) - load FIRST, before creating, editing, reviewing, refreshing, renaming or deleting any skill here. Defines how the official, public SmartStandards and UShell open-source landscape is consolidated into snapshot skills (cached essentials, snapshot date with staleness alarm, escalation edge to the upstream repositories and their skills) and how new knowledge is written back upstream. Also load when a snapshot's age, staleness alarm or escalation is in question, and for the recurring local instruction scope cleanup - scanning repository- and user-level agent instruction files (Copilot, Claude, AGENTS.md, Cursor, ...) for misplaced general rules and moving them into skills.
metadata:
  snapshot-date: "2026-10-22"
---

# Skill Maintaining Guidelines (AI-SKILLS-SMART)

This skill is the rulebook of this repository. Every skill in `.agents/skills/` other than this one is a **snapshot skill** and must follow the rules below, unless it is listed as a template, index or general skill.

### Index Skills

Index skills are not snapshots and are never copied into other repositories. They carry the `smart-` prefix but not the three layers or the snapshot status block; they are versioned via `metadata.version`.

- `smart-standards-library-ecosystem` - the phone book of the landscape: all admitted and excluded repositories, owners, clone URLs, roles, problem domains, upstream skills and snapshot skills. Every new snapshot skill is registered there.

### Template Skills

Template skills are not snapshots. This repository is their leading source; they may be used directly from here and may also be duplicated into other repositories. Copies are downstream: improve the template here first, then roll it out. Template skills follow the admission and neutrality rules, but not the snapshot naming prefix, the three layers or the snapshot status block.

- `include-additional-mandatory-skills`

### General Skills

General skills are not snapshots of one repository but cross-cutting, publishable knowledge about working with the landscape. This repository is their leading source. They follow the admission and neutrality rules, but not the snapshot naming prefix, the three layers or the snapshot status block.

- `master-architect` - capability-first architecture and wire-up reasoning.
- `additional-fields-via-smart-extensiondata` - round-tripping unknown JSON fields via the SmartExtensionBag pattern.
- `dotnet-t4-source-embedding` - embedding foreign C# sources via T4 templates instead of package references.
- `asp-net-core-development-guidelines` - designed-state application bootstrap (canonical `Program.Main`, partial hooks) and explicit controller exposure in ASP.NET Core.
- `csharp-coding-standards` - conservative, explicit C# coding standards and AI implementation rules ("syntactic salt over syntactic sugar").
- `package-version-preferences-dot-net` - preferred NuGet version ranges, pins, status of competing technologies and reference solutions; reviewed cyclically with the user.
- `dotnet-sdk-style-project-files` - designed state of SDK-style `.csproj` files: section order, identity, canonical BUILD, project-type modules, feature templates.
- `nuget-semantic-versioning-kvu` - changelog-driven semantic versioning with KVU: changelog rules, `.nuspec` pre-processor configuration, command line, build pipeline procedure.
- `refactoring-processes` - standardized high-level refactoring processes; currently bringing an existing solution up to date (analysis, plan, confirmation; target frameworks, renames, nuspec / KVU, bootstrap, conventions, design debt).

## Purpose of This Repository

The SmartStandards and UShell open-source landscape is developed in decentralized, domain-driven repositories. Each of them owns its documentation and its own AI skills. Those upstream skills are the source of truth.

This repository is a **consolidator, not a redundancy**. It gives agents fast, broad access to the whole landscape from one place:

- Each snapshot skill covers one problem domain and has a precise description, so agents find it.
- It carries a compressed, cached essence of how to work with the domain.
- It hands over quickly to the upstream repositories where the full, authoritative skills live.

### Admission Rule

This repository is meant to be cloned officially and legitimately by developers in any organizational context as a knowledge source. Therefore it contains only knowledge about:

- open-source projects published officially and publicly under the SmartStandards or UShell umbrella,
- whose upstream repositories are publicly reachable and carry an open-source license.

Admitted GitHub owners:

| Owner | Rule |
| --- | --- |
| `SmartStandards` | Public repositories are admitted. |
| `ProjectUShell` | Public repositories are admitted. `UShell.FrontendBundle` belongs to the landscape but is temporarily private: do not snapshot it until it is public. |
| `KornSW` | Exception: only public repositories that belong to the SmartStandards / UShell development landscape. This owner also hosts unrelated public repositories (e.g. WordPress plugins, home automation, forks); these are not admitted. The admitted and excluded repositories are listed in `smart-standards-library-ecosystem`. Ask the user before admitting any other `KornSW` repository, then add it there. |

Private repositories are never admitted, regardless of owner. Verify visibility before creating or refreshing a snapshot, e.g. via the public GitHub API without authentication: `https://api.github.com/repos/<owner>/<repo>` returns `"visibility": "public"` for public repositories and HTTP 404 for private or non-existent ones. If a referenced repository has become private, remove it from the snapshot and tell the user.

Not admitted: private or unpublished repositories, personal working notes, preferences or local setups of individual developers, drafts from other skill collections, and anything that cannot be traced to a public upstream source. If the admission of a project is unclear, ask the user before creating a snapshot.

### Neutrality Rule

Never mention which organization, company, customer, team or product consumes this consolidation. Do not write company names, internal system names, internal hostnames, internal repository URLs or customer-specific overrides. Everything here must be publishable as-is. Consumer-specific usage and overrides belong in separate, consumer-owned skill collections, which may reference skills from this repository (never the other way round).

### Path and Repository Reference Rules

Applies to every skill and document in this repository:

- `C:\GIT\` and every folder matching `C:\GIT-*\` (Windows) as well as `~/GIT/` and `~/GIT-*/` (Linux) are equivalent Git roots. Never hard-code a concrete root matching `GIT-*`; use the pattern.
- Every repository folder matching `AI-SKILLS` or `AI-SKILLS-*` in a Git root is to be indexed as a skill consolidation repository. Repository folders matching `*-Docs` or `*.Docs` are indexed as well, but only if already cloned locally. Hard-code a concrete `AI-SKILLS-*` folder name only after explicit user confirmation. Confirmed: `AI-SKILLS-SMART`.
- When another consolidation repository is referenced, document its Git clone URL. When running locally and the repository is missing, ask the user to clone it; the user chooses the Git root.
- `AI-SKILLS-SMART` (`https://github.com/SmartStandards/AI-SKILLS-SMART.git`) and other listed consolidation repositories (e.g. `UShell.Docs`) may have organization-internal forks (mirrors). Organization-internal skills reference their internal mirror as clone URL instead. Never name such organizations or mirrors here.
- The details for agents are defined in the template skill `include-additional-mandatory-skills`; keep both consistent.

## The Three Layers of a Snapshot Skill

Every snapshot skill has exactly these three layers, in this order:

| Layer | Content | Answers |
| --- | --- | --- |
| 1. Domain Overview | Description (frontmatter) and a short overview: what the problem domain is, which libraries/repositories belong to it, when to use it, what is out of scope. | "Am I in the right place?" |
| 2. Cached Essentials | Compressed, cached basics: core concepts, typical usage and setup, conventions, quick troubleshooting. Includes the snapshot status block with date and staleness alarm. | "How do I work with this domain in everyday use?" |
| 3. Upstream Escalation | References to the concrete upstream Git repositories and their skills, plus the explicit escalation edge (see below). | "Where are the details and the authoritative truth?" |

### Layer 1 - Domain Overview

- Cut and name problem domains 1:1 like the established domain skills of consuming organizations' skill collections: where such a collection has `<prefix>-<domain>`, the snapshot skill is `smart-<domain>`, so that a consumer's skill and the snapshot pair up. Drop organization- or product-specific name parts; never name the organization. The planned domains are listed in `smart-standards-library-ecosystem`. For repositories without an established counterpart, ask the user before inventing a new domain.
- Cut along broad problem domains, not along single packages. One snapshot skill may cover several repositories. Avoid micro-skills; the collection should stay small (well below a three-digit number of skills).
- The `description` states the domain, the main library and package names as keywords, and the trigger situations. Keep it at most 1024 characters (hard limit of the Agent Skills specification), aim well below.
- State exclusions and name neighboring snapshot skills instead of duplicating their content.

### Layer 2 - Cached Essentials

- Compress hard. Target: the whole `SKILL.md` stays below ~200 lines. Include only what an agent needs for typical consumption of the libraries: concepts, minimal setup, key APIs and conventions, common pitfalls, and a short troubleshooting checklist (symptom -> likely cause -> quick check).
- Do not copy implementation details, full API listings, or content that changes often. Summarize and link upstream instead.
- Every statement must be derived from an upstream source at the snapshot date. Do not invent or extrapolate behavior.
- The snapshot status block (see below) is mandatory and sits at the top of this layer.

### Layer 3 - Upstream Escalation

List every relevant upstream source:

- Repository name and public URL (e.g. GitHub), plus the path to its skills (usually `.agents/skills/`) and to its docs.
- Package identifiers (e.g. NuGet / npm) if helpful for version comparison.
- Reference only public upstream sources. Do not reference other skill collections, and do not write absolute or machine-specific local paths. If the agent finds a local clone of an upstream repository, it may prefer it over the online version.

**The escalation edge** - an agent must load the upstream skills and sources (local clone preferred, otherwise online) when:

1. the task is to develop, extend, fix or refactor the library itself (work inside the upstream repository);
2. deep troubleshooting requires implementation details that the cached essentials do not cover;
3. the snapshot is stale or the staleness alarm fires (see below) and the task depends on the affected details;
4. observed behavior, the consumed package version, or the code contradicts the snapshot.

For plain consumption of the libraries (using them in an application, standard setup, typical troubleshooting) the snapshot alone should be sufficient.

## Snapshot Status and Staleness Alarm

Each snapshot skill contains this block as the first section of layer 2 and mirrors the date in the frontmatter (`metadata.snapshot-date`):

```markdown
## Snapshot Status

- Snapshot date: YYYY-MM-DD
- Synchronized against: <repo> @ <tag | version | commit>, <repo> @ ...
- Staleness thresholds: fresh < 90 days, aging 90-180 days, stale > 180 days
- Volatility: low | medium | high
```

Default thresholds are 90/180 days. A skill may shorten them for volatile domains (`Volatility: high`, e.g. 30/90); it states the overridden values in its block.

### Alarm Logic (for agents consuming a snapshot)

Compute the age from the snapshot date and the current date, then act:

| State | Agent behavior |
| --- | --- |
| fresh | Use the cached essentials directly. |
| aging | Use the essentials, but mention the age to the user. If the task depends on details likely to change, do a quick upstream check first. |
| stale | Do not rely on the essentials for details without a quick upstream check. Tell the user and propose a snapshot refresh. |

Independent of age, raise the alarm when the consumed package version is newer than the synchronized version, or when observed behavior contradicts the snapshot.

**Quick upstream check** (cheap, before a full escalation): look at the upstream commit log, release tags, changelog and the upstream skills' change history since the snapshot date. If nothing relevant changed, continue with the snapshot and mention that the check was done. If relevant changes exist, escalate to the upstream skills for the affected topic and propose a refresh.

If upstream sources are not reachable, say so, continue with the snapshot only where acceptable, and mark statements as unverified. Never claim a check that was not performed.

## Creating, Refreshing and Retiring Snapshot Skills

### Naming and Location

- Path: `.agents/skills/<skill-name>/SKILL.md`, directory name equals `name`, lower case with hyphens (Agent Skills specification: https://agentskills.io/specification.md).
- Prefix every snapshot skill with `smart-` (e.g. `smart-http-remoting-standard`). This avoids name collisions with upstream skills that may be loaded in the same session.
- Write all skill content in English. User communication may use any language.
- Start new skills from [assets/snapshot-skill-template.md](assets/snapshot-skill-template.md).

### Create

1. Check existing snapshot skills first; extend one if the domain fits instead of creating an overlapping skill.
2. Check the admission rule and the visibility of every repository in scope. Read the upstream skills, README and docs of every repository in scope. Never write a snapshot from memory alone.
3. Make sure every statement the snapshot will contain exists upstream first. If knowledge originated outside the upstream repositories (e.g. from analyzing source code, a predecessor skill, or a pasted fragment) and the repository has no skill covering it yet, create or extend its `most-important-details-for-*` skill first (see `Writing Knowledge Back Upstream`). The upstream repository is the leading home of the knowledge, even when the snapshot was conceived first.
4. Fill the three layers, set the snapshot date to today and record the synchronized versions.
5. Add the skill to the index in the repository `README.md` and to the `Snapshot` column of `smart-standards-library-ecosystem`.

### Refresh

1. Re-check the visibility of all referenced repositories. Read upstream changes since the recorded snapshot date and versions (commit log, changelog, release notes, upstream skills).
2. Update the cached essentials and the escalation references; remove what is no longer true.
3. Update the snapshot date (block and frontmatter) and the synchronized versions, also in the `README.md` index.
4. Bump the date only after an actual comparison with upstream. A refresh without a real check is forbidden.

### Conflict Rule

Upstream is authoritative. If a snapshot contradicts its upstream source, the upstream source wins; fix the snapshot. If upstream itself is inconsistent, report it to the user and propose a fix in the upstream repository (see `Writing Knowledge Back Upstream`).

## Writing Knowledge Back Upstream

Work in this repository may produce new knowledge about a problem domain (e.g. from user input, troubleshooting, pasted fragments, or comparing several repositories). It is explicitly allowed to write such knowledge from here into the upstream problem-domain repositories and their full-version skills. Knowledge must not end up only in a snapshot.

1. **Find the owner.** Determine the owning repository via `smart-standards-library-ecosystem` and its full-version skill (e.g. `ujmw`, `fuse-fx`, `ushell-*` in `UShell.Docs`), or the repository's `most-important-details-for-*` skill for repository-specific implementation details.
2. **Use a local clone.** Write only into a local clone of the upstream repository. If none exists, ask the user to clone it (the user chooses the Git root).
3. **Follow the upstream repository's rules.** Load its `include-additional-mandatory-skills` copy and its `most-important-details-for-*` skill first. Respect its documentation layout (`/doc`, never `changelog.md`) and its skill format.
4. **Upstream first, snapshot second.** Write the knowledge into the full-version skill (or `/doc`) first, then refresh the affected snapshot skill from it. Never place knowledge only in a snapshot - also not temporarily, and also not when the snapshot existed before the upstream skill.
5. **Keep it public.** The admission and neutrality rules apply in full: upstream repositories are public. No organization names, internal URLs, personal notes or machine-specific paths.
6. **Show and confirm.** Present the planned upstream change briefly (target repository, skill, section) before writing. Commit or push only when the user asks; report which repositories were changed.

### Retire

Remove or merge a snapshot skill when its domain is discontinued or absorbed into another domain. Update the `README.md` index and the references in neighboring skills.

## Instruction Scope Cleanup (recurring, local)

Agent products persist preferences with poor scoping (GitHub Copilot offers only repository level, with user level as an emergency switch). General rules therefore accumulate in repository- and user-level instruction files of all harnesses. A recurring, local process collects them: scan the repositories in the Git roots and the known user-level locations, build an index of statements with a scope test ("does the scope make sense?" - e.g. detailed usage rules for a framework the repository only consumes via NuGet indicate a general rule), then ask the user clearly per group whether the knowledge is generally valid, into which skill or collection (this repository, upstream, an organization-owned collection) it moves, and whether the original is deleted, replaced by a pointer or marked as redundant. Nothing is moved or deleted before that decision.

User level keeps only personal preferences (language, tone, local tooling, routing reference); repository level keeps only a routing line and facts true for that repository alone.

Full procedure, scan locations, index format and redundancy marker: [references/instruction-scope-cleanup.md](references/instruction-scope-cleanup.md). The index of a run contains non-public content and is never committed here.

## Review Checklist (before delivering a change)

- Frontmatter: `name` matches directory, `smart-` prefix, `description` <= 1024 characters, `metadata.snapshot-date` present.
- Three layers present, in order; snapshot status block complete and consistent with the frontmatter.
- Admission rule met: every upstream repository is public (visibility checked) and owned by an admitted owner (`KornSW` only if listed); no personal developer notes.
- No consumer, company or customer names; no internal URLs; no machine-specific paths; no references to other skill collections.
- Escalation edge and upstream URLs present and plausible.
- `README.md` index and `smart-standards-library-ecosystem` updated.
- Report what was actually verified against upstream and what was not.
