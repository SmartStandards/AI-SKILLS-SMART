# KVU for WordPress Plugins (GitHub Actions Variant)

WordPress plugins are not built with Azure DevOps and `kvu.exe`. For them, a minimal Python re-implementation of the KVU flow runs directly in GitHub Actions: it versions the repository from `doc/changelog.md`, retrofits a self-update mechanism into the plugin (GitHub as the update server) and publishes a GitHub release with a custom plugin ZIP. An agent can apply it completely with the assets in [../assets/wordpress/](../assets/wordpress/).

Assets synchronized 2026-10-21 from the **online** state of the leading template `WP-AutoUpdate-G2` @ `f5228a1` (2026-09-09); all 19 public `KornSW/WP-*` repositories carried identical copies at that date. Local clones may be older - compare with the online template before applying (section 7); if it is newer, use it and refresh these copies.

> **Known defect (2026-10-21):** `classify()` in `release.py` uses `re.search(r"breaking\\s*change", ...)` and `r"new\\s*feature"` - a doubled backslash inside a raw string, so the regex requires a literal backslash between the words. As a result **`New Feature` and `Breaking Change` are never recognized** (neither plain nor bold); every release is a patch, only `**MVP**` works (during 0.x). Evidence: `WP-KnowledgeRepo` v1.0.4 contains "new Feature: ..." and was a patch release. Fix: `r"breaking\s*change"` / `r"new\s*feature"`, upstream in `WP-AutoUpdate-G2` first. Until fixed, tell the user when a feature or breaking change is expected to bump the version.

## 1. What a Release Run Does

Trigger: push to `master` (or manual `workflow_dispatch`). The job is skipped when the head commit message contains `VERSIONING`.

1. Detect the plugin entries (section 3).
2. Create `doc/changelog.md` if missing; read `## Upcoming Changes`, else take the commit subjects since the latest merged version tag (max. 20, `VERSIONING` commits ignored, skip markers stripped), else `New revision without significant changes`.
3. Compute the version **once** for the whole repository (one versioning unit, even with several plugins) and materialize `doc/changelog.md` and `doc/versioninfo.json` (same schema as KVU, `releaseType` always empty).
4. Per plugin:
   - get or create the persistent updater prefix in `doc/<slug>.updater-prefix.txt` (`ksw<token><4 hex>`; an existing `<prefix>_bootstrap( __FILE__ );` in the entry file is reused);
   - render `.kvu/templates/self-update.php` with that prefix into `<plugin dir>/self-update.php`;
   - set the entry headers `Version`, `Plugin URI` (`https://github.com/<owner>/<repo>`) and `Update URI` (`https://raw.githubusercontent.com/<owner>/<repo>/master/doc/<slug>.update.json`);
   - insert or replace the managed block between `/*************** SELF-UPDATE ***************/` and `/*******************************************/` (after the `ABSPATH` guard, else after the header docblock);
   - write `doc/<slug>.update.json` (name, version, author, homepage, `download_url` of the release asset, requires, requires_php, description, changelog HTML of this release);
   - build `dist/<slug>.zip` with exactly one top-level folder `<slug>/`.
5. Validate the generated state (headers, metadata version, bootstrap, ZIP layout).
6. Commit everything as `<triggering subject> -> VERSIONING [skip ci]`, create the annotated tag `v<version>`.
7. Push the tag, create the GitHub release `v<version>` with the ZIPs and the release notes, and only then push `master` - so a new `update.json` never points to an asset that does not exist yet.

## 2. Differences from KVU (`kvu.exe`)

| Topic | WordPress variant |
| --- | --- |
| Triggers | Intended: `new feature` / `breaking change` case-insensitive with any whitespace (also none), in changelog entries and commit subjects - like KVU; `**MVP**` must be bold. Currently broken, see the defect note above. Older script versions (e.g. stale local clones) only recognized the bold forms `**New Feature:**` / `**Breaking Change:**`. Canonical writing that works with all versions: `**New Feature:** ...`, `**Breaking Change:** ...`. |
| Branches / pre-releases | `master` only; no pre-release suffixes. |
| First version | `0.1.0` (`changeGrade` `initial`). |
| Targets | WordPress plugin headers, `update.json`, ZIP; no project or `.nuspec` files. |
| Tags | Creates `v<version>` tags and GitHub releases (they also bound the git fallback). |
| Change order | Entries grouped major -> minor -> fix in the release block. |

Everything else (0.x rule, `**MVP**` -> `1.0.0` only during 0.x, highest impact wins, git fallback only when `Upcoming Changes` is empty, commit back with skip-CI) follows the KVU rules of the main skill.

## 3. Repository Layout Requirements

- Public GitHub repository, main branch `master`, Actions allowed to write (Settings -> Actions -> General -> Workflow permissions: read and write). Private repositories are not supported (the plugin fetches `update.json` without a token).
- Plugin entry = a `.php` file with a `Plugin Name:` header, searched only in `./*.php` and `./*/*.php` (folders starting with `.` and `doc`, `dist`, `.kvu`, `.github` are ignored).
- Exactly one entry per top-level plugin folder; at most one entry in the root; a root plugin cannot be combined with plugin folders. Multi-plugin repositories: one top-level folder per plugin. The slug is the folder name (root plugin: file name without extension).
- `doc/` belongs to versioning and is never packed into the ZIP.

## 4. Applying It to a Plugin Repository

1. **Check** the requirements of section 3. Report deviations (wrong branch name, private repository, entry files in the wrong depth, an existing foreign update mechanism such as another updater library or a `Update URI` pointing elsewhere) and ask before changing anything.
2. **Copy** from [../assets/wordpress/](../assets/wordpress/) into the repository root, unchanged: `.kvu/release.py`, `.kvu/templates/self-update.php`, `.github/workflows/release.yml`. Add the lines of `gitignore.txt` (`dist/`, `__pycache__/`, `*.pyc`) to `.gitignore`.
3. **Do not pre-create** what the first run generates: `doc/`, `<plugin dir>/self-update.php`, the SELF-UPDATE block, `Plugin URI`, `Update URI`, `Version` changes, `update.json`, prefix files, tags or releases.
4. **Existing `doc/changelog.md`**: keep it; it must contain `## Upcoming Changes` and release headers in the form `## v x.y.z`. Otherwise propose a conformant conversion first.
5. **Changes**: describe the introduction via commit subject or an entry below `## Upcoming Changes`, using the canonical bold prefixes (section 2).
6. **Commit and push** only when the user asks. The push starts the first run.
7. **Verify** after the run: GitHub release `v<version>` with `<slug>.zip`, a `-> VERSIONING [skip ci]` commit on `master` containing `doc/changelog.md`, `doc/versioninfo.json`, `doc/<slug>.update.json`, `doc/<slug>.updater-prefix.txt`, `<plugin dir>/self-update.php` and the patched entry file. In WordPress the plugin row shows a `GitHub Repository` link; updates appear in the normal update UI.

## 5. Rules After Introduction

- Never edit generated artifacts by hand: `<plugin dir>/self-update.php` (overwritten on every run), the SELF-UPDATE block, the headers `Version` / `Plugin URI` / `Update URI`, `doc/<slug>.update.json`, `doc/versioninfo.json`.
- Never change or delete `doc/<slug>.updater-prefix.txt`: the prefix names the bootstrap function and updater class in installed plugins and must stay stable.
- Diagnostics: set the constant `<PREFIX>_SELF_UPDATE_DIAGNOSTICS` in the managed block to `true` only temporarily (adds an admin diagnostics page); the next release run resets it to `false`.
- Legacy single-plugin files `doc/update.json` and `doc/updater-prefix.txt` are migrated and removed automatically.
- Changes to the release logic go into the template `WP-AutoUpdate-G2` first, then to the plugin repositories and these assets.

## 6. Troubleshooting

| Symptom | Likely cause |
| --- | --- |
| Push fails in the workflow | Workflow permissions are read-only. |
| "No WordPress plugin entry file found" / "Multiple ... entry files" | Layout violates section 3. |
| Expected minor / major became a patch | Known regex defect (see top); with older script versions: prefix not bold. |
| WordPress shows no update | Repository private, `master` not yet pushed, or cached transient (use the refresh link in the plugin row or the diagnostics page). |
| Workflow does not run after a commit | Commit message contains `VERSIONING` or a skip marker. |

## 7. Upstream

| Repository | URL | Role |
| --- | --- | --- |
| `WP-AutoUpdate-G2` | https://github.com/KornSW/WP-AutoUpdate-G2 | Leading template (`.kvu/`, `.github/workflows/release.yml`, demo plugin); full technical documentation in `doc/Update-Logic-Dokumentation.md`. Its `README.md` still describes the older single-file names (`doc/update.json`). |
| `WP-AuthTokenHandling` | https://github.com/KornSW/WP-AuthTokenHandling | Production example using the template unchanged. |
| further `KornSW/WP-*` plugins | https://github.com/KornSW?tab=repositories&q=WP- | All carry the same copy of the release files; compare quickly via `raw.githubusercontent.com/KornSW/<repo>/master/.kvu/release.py`. |
