# Instruction Scope Cleanup (recurring, local)

Agent products offer poor scoping when they persist preferences. GitHub Copilot, for example, offers only "repository" with "user" as an emergency switch. As a result, general development rules pile up in repository-level or user-level instruction files where they do not belong, are duplicated across repositories, and drift apart. This process collects such misplaced instructions and moves them into the skill collections where they belong. It is knowledge management housekeeping, not a one-time migration.

## When to Run

- On explicit request (e.g. "instruction cleanup", "collect misplaced instructions", "tidy up the Copilot instructions").
- Periodically on every developer machine; recommended monthly, and after sessions in which an agent product has persisted new preferences.
- Proactively suggest a run (do not start it unasked) when, during other work, a harness instruction file is found that contains substantial general rules.

The run is local: it scans what is on the current machine. It never pushes or commits without being asked.

## Target Scopes

| Scope | Belongs there | Does not belong there |
| --- | --- | --- |
| User level | Genuinely personal preferences: language of communication, tone, verbosity, personal tooling and shell, local paths of this machine, the routing reference to `include-additional-mandatory-skills` (section 6 of that skill). | Development decisions, coding standards, architecture rules, framework usage rules. |
| Repository level (harness instruction files) | A short routing line to the include skill, and facts true only for this repository (build commands, solution layout, domain vocabulary). Prefer `/doc` or a `most-important-details-for-*` skill for longer repository knowledge. | Anything that would be equally true in another repository. Personal preferences. |
| Skills (open-source collection, this repository or upstream) | General, publishable rules: landscape libraries, general .NET / C# / ASP.NET Core practices, third-party framework usage that is not organization-specific. | Organization names, internal systems, personal notes (admission and neutrality rules). |
| Skills (organization-owned collections) | General rules specific to one organization or its customized frameworks. | Rules that are generally valid and publishable (move to the open-source layer and reference). |

## Scope Test

For each statement ask: **does the scope make sense?** A statement is misplaced when its natural validity is wider (or narrower) than the file it sits in. Indicators:

- It prescribes how to use a framework or library that this repository only **consumes** (NuGet / npm package reference, not a project reference or source in the repository). Verify in the project files. Detailed usage rules for a consumed package are a strong indicator of a general rule.
- The same or a similar statement appears in several repositories or in several harness files.
- It is a coding style, naming, architecture, testing or versioning rule without any reference to this repository's code.
- It contradicts or duplicates an existing skill.
- On user level: anything that is a technical decision rather than a personal preference.
- On repository level: personal preferences (e.g. answer language) - these belong on user level.

## Locations to Scan

Determine the Git roots and repositories as defined in section 4 of `include-additional-mandatory-skills` (all repository folders in `GIT` / `GIT-*` roots). Locations change between product versions; check the product documentation if a location seems missing, and report locations that could not be checked (e.g. instructions stored only in a cloud account or in an IDE settings UI).

Repository level (relative to each repository root, also in subfolders where the product supports nested files):

| Product | Files |
| --- | --- |
| GitHub Copilot | `.github/copilot-instructions.md`, `.github/instructions/*.instructions.md`, `.github/prompts/*.prompt.md`, instruction keys in `.vscode/settings.json` (`github.copilot.chat.*instructions*`) |
| Cross-product | `AGENTS.md` (any folder) |
| Claude Code | `CLAUDE.md`, `CLAUDE.local.md`, `.claude/CLAUDE.md`, `.claude/rules/*.md`, `.claude/commands/*.md` |
| Cursor | `.cursorrules`, `.cursor/rules/*.mdc` |
| Windsurf | `.windsurfrules`, `.windsurf/rules/*.md` |
| Others | `GEMINI.md`, `.clinerules` (file or folder), `.continue/rules/`, `.junie/guidelines.md`, `CONVENTIONS.md` referenced by aider configuration |
| Agent Skills | `.agents/skills/*/SKILL.md` other than copies of `include-additional-mandatory-skills` or organization twins, and other than `most-important-details-for-*` skills that describe only this repository |

User level (`~` = user profile; on Windows `%APPDATA%` for VS Code):

| Product | Locations |
| --- | --- |
| GitHub Copilot | VS Code user `settings.json` (`github.copilot.chat.*instructions*`), VS Code user prompts folder (`Code/User/prompts/*.instructions.md`, `*.prompt.md`), `~/.copilot/` (Copilot CLI instructions); personal instructions on github.com are not local - ask the user to paste them if they should be included |
| Claude Code | `~/.claude/CLAUDE.md`, `~/.claude/rules/`, `~/.claude/commands/`, auto memory `~/.claude/projects/*/memory/*.md` |
| Codex | `~/.codex/AGENTS.md` |
| Gemini | `~/.gemini/GEMINI.md` |
| Windsurf | `~/.codeium/windsurf/memories/global_rules.md` |
| Cursor | User rules live in the settings UI - ask the user to paste them if they should be included |

Skip the skill consolidation repositories themselves (`AI-SKILLS`, `AI-SKILLS-*`, `*-Docs`, `*.Docs`); their content is already in skills. Never scan or migrate content of a personal skill repository into another collection when its rules forbid that.

## Procedure

### 1. Build the Index (read-only)

Scan all locations, split every file into individual statements (instruction units) and record one row per statement:

| Column | Content |
| --- | --- |
| ID | Running number, stable within the run |
| Source | Level (user / repository), product, repository name + relative path (user level: location name), line |
| Statement | One-line summary, verbatim quote only if short |
| Indicator | Which scope-test indicator fired (e.g. "consumed NuGet package X", "also in repos A, B") |
| Proposed class | `keep-user`, `keep-repo`, `to-user` (personal preference in a repository), `general-open-source`, `general-organization`, `already-covered` (name the skill), `conflict` (name the skill), `obsolete` |
| Proposed target | Skill name and collection (this repository, an upstream repository, an organization collection), or `/doc` |
| Proposed disposition | `delete`, `replace-with-pointer`, `mark-redundant`, `keep` |

Group identical or near-identical statements from several sources into one row with several sources. Statements classified `keep-user` / `keep-repo` are listed compactly (count per file) so the user can still veto them.

Write the index to a local working file outside public repositories (e.g. the agent's scratch directory, or a location the user names). The index contains repository names and possibly organization-internal content; it must never be committed to this repository. Show the user a condensed view (per class and per target), not the raw file.

### 2. Ask the User - Clearly and per Group

Do not move or delete anything before the user has decided. Ask in numbered, answerable questions, grouped by proposed target, with a recommended default per group. For each group (or single row where needed) the user decides:

1. **Is this generally valid?** Yes / no (stays where it is) / wrong level (move between user and repository level).
2. **If yes, where does it go?** The proposed skill in this repository (general skill or snapshot, see the write-back rules: landscape knowledge goes upstream first), an organization-owned skill collection (name the collection and skill), another existing skill, or a new skill (follow the creation rules of the owning collection).
3. **What happens to the original?** `delete` (default when fully covered), `replace-with-pointer` (one line referencing the skill), or `mark-redundant` (keep the text, add the redundancy marker below).

For `conflict` rows, show both versions side by side and let the user decide; never resolve a conflict silently. For `already-covered` rows, propose `delete` or `replace-with-pointer` directly.

### 3. Execute the Decisions

1. **Move knowledge first.** Integrate each approved statement into exactly one owning skill, following that collection's maintaining guidelines (here: admission, neutrality, upstream first; organization collections: their own rules and, for organization special cases, their `<prefix>-<same-name>` counterpart). Neutralize statements written into public collections: no repository, organization or customer names from the source.
2. **Then clean the source.** Apply the chosen disposition. Redundancy marker (use the comment syntax of the file type):

   ```markdown
   <!-- redundant: maintained in skill <skill-name> (<collection repository>) since YYYY-MM-DD - edit there, not here -->
   ```

   A pointer replacement reads like: `Rules for <topic>: see skill <skill-name> in <collection repository>.` Where a repository's harness file becomes empty except for routing, reduce it to the routing reference to `include-additional-mandatory-skills`.
3. **Respect the owning repositories.** Each changed repository follows its own rules (its include-skill copy, `/doc`, never `changelog.md`). Present the planned change per repository before writing. Commit or push only when the user asks; report every changed repository and user-level file.
4. **User-level files** are changed only after the user approved the concrete change; back up the previous content in the working file of the index.

### 4. Report and Remember the Run

Report: number of statements scanned, moved (per target skill), deleted, marked redundant, kept, and the locations that could not be checked. Keep the index of the run in the local working location; on the next run, compare against it and ask only about new or changed statements (decisions `keep-*` remain valid until the statement changes).
