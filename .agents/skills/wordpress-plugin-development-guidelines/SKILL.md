---
name: wordpress-plugin-development-guidelines
description: Uniform conventions for building WordPress plugins in the landscape - starting a new plugin (project language, minimum versions), repository layout of a plugin repository (WP-<Name>, one top-level plugin folder = slug, doc/, .kvu/, GitHub Actions release), the entry file <slug>.php (header fields, ABSPATH guard, pipeline-managed SELF-UPDATE block, autoloader, activation and boot hooks), PSR-4 namespaced classes with C#-like readable formatting, naming and prefixing of options, tables, hooks and REST routes, admin pages (capabilities, nonces, settings), data storage and uninstall (data is kept), security basics, testing without a WordPress installation, and what an agent must never edit by hand. Load when creating a new WordPress plugin, adding features to an existing plugin, reviewing plugin code, or bringing a plugin repository in line with the other plugins.
metadata:
  version: "2026-10-22"
---

# WordPress Plugin Development Guidelines

## Purpose

All WordPress plugins of the landscape are built the same way, so that any plugin repository can be understood, released and maintained without re-learning its structure. This skill defines that common shape. It complements:

- `smart-solution-and-project-structure` - general repository and `/doc` layout. It applies to plugin repositories, with the language exception of section 1.
- `nuget-semantic-versioning-kvu`, reference `references/wordpress-plugins.md` - versioning, changelog, self-update and GitHub release of plugins. This skill does not repeat it.
- `csharp-coding-standards` - the readability model for the PHP formatting of section 5.

**Scope of the rules:** the rules apply in full to **new** plugins. Existing plugins are not refactored only to comply; deviations are named and aligned only when the user asks (see Agent Guidance). Rules marked **(open)** are not decided yet; follow the dominant variant of the repository at hand and ask the user.

## 1. Starting a New Plugin

Ask the user before writing anything:

1. **Project language** (e.g. English or German). The plugin description, all UI texts and the human-readable documentation (`README.md`, `/doc`) use this one language, consistently. This deliberately overrides the "documentation in English" rule of `smart-solution-and-project-structure` for plugin repositories. Record the choice in `doc/requirements.md`.
2. Plugin name and slug (section 3), and a one-sentence description.

Independent of the project language:

- **AI skills are always English** (`.agents/skills/**`).
- Source code identifiers and code comments are English.
- Commit messages and changelog prefixes follow `nuget-semantic-versioning-kvu`.

Minimum versions of a new plugin:

- `Requires PHP`: the newest PHP minor release that is generally available (stable) at project start. Check https://www.php.net/supported-versions.php instead of relying on memory.
- `Requires at least` (WordPress): the current WordPress major release at project start.
- Raise them later only on purpose (it is a compatibility change for installations).

## 2. Repository Layout

```text
WP-<PluginName>/
├── .agents/skills/                    # include-additional-mandatory-skills copy + plugin skill (English)
├── .github/workflows/release.yml      # copied unchanged from the release template
├── .kvu/release.py                    # copied unchanged from the release template
├── .kvu/templates/self-update.php     # copied unchanged from the release template
├── .editorconfig                      # mirror from csharp-coding-standards (2 spaces, CRLF)
├── .gitignore                         # at least: dist/  __pycache__/  *.pyc
├── LICENSE                            # GPL v3 text
├── README.md
├── doc/                               # changelog.md + generated release files + documentation
├── <slug>/                            # the plugin itself = exactly what lands in the ZIP
│   ├── <slug>.php                     # entry file
│   ├── self-update.php                # generated, never edit
│   ├── uninstall.php                  # optional, see section 7
│   ├── src/                           # PSR-4 classes, see section 4
│   ├── assets/                        # css/, js/, images
│   └── vendor/                        # bundled third-party code, namespaced
└── tests/                             # standalone PHP tests, never shipped
```

- Repository name `WP-<PluginName>` (PascalCase), public on GitHub, default branch `master`.
- Exactly **one** top-level plugin folder per repository. The folder name is the slug.
- Everything outside the plugin folder (`doc/`, `tests/`, `.kvu/`, ...) is never packed into the plugin ZIP.
- A new repository starts with the release files, `LICENSE`, `.gitignore`, `doc/changelog.md` (with `## Upcoming Changes`) and the plugin folder. The first push to `master` produces version `0.1.0`.

## 3. Identity and Naming

| Item | Convention | Example |
| --- | --- | --- |
| Slug / plugin folder / text domain | `<vendor>-<name>`, lower case, no separators inside the name | `kornsw-remotearticles` |
| Entry file | `<slug>/<slug>.php` | `kornsw-remotearticles/kornsw-remotearticles.php` |
| Plugin Name header | `<Vendor> <Name>` | `KornSW RemoteArticles` |
| PHP namespace | `<Vendor>\<PluginName>` | `KornSW\RemoteArticles` |
| Short prefix | `<vendor>_<abbr>_` (lower case) for options, tables, transients, hooks, global functions | `kornsw_ra_` |
| Constants | upper-case short prefix + `FILE`, `DIR`, `URL`, `VERSION` | `KORNSW_RA_DIR` |
| DB tables | `$wpdb->prefix . '<short prefix><name>'` | `wp_kornsw_ra_sources` |
| REST namespace | `<slug>/v1` | `kornsw-remotearticles/v1` |
| License | GPL-3.0: `LICENSE` file with the GPL v3 text, header `License: GPL-3.0` (never choose or change a license on your own) | |

- Everything global (options, transients, tables, hook names, global functions, constants, CSS class roots, JS globals) carries the plugin prefix; classes live in the plugin namespace.
- Forks of third-party plugins keep the original slug with a `-<vendor>-fork` suffix and keep the original author, adding "(featured by <vendor>)".

## 4. Entry File and Code Organization

Classes follow the PHP community standard **PSR-4**: namespace `<Vendor>\<PluginName>`, one class (or interface / enum) per file, file name = class name, sub-namespaces = sub-folders of `src/` (e.g. `src/Admin/SettingsPage.php` -> `<Vendor>\<PluginName>\Admin\SettingsPage`). The shipped plugin does not need Composer at runtime: the entry file registers a small PSR-4 autoloader.

Order inside `<slug>/<slug>.php`:

1. Header docblock.
2. `ABSPATH` guard.
3. The SELF-UPDATE block (inserted and maintained by the release pipeline - section 9).
4. Constants (`FILE`, `DIR`, `URL`, `VERSION`).
5. PSR-4 autoloader for the plugin namespace.
6. `register_activation_hook()` / `register_deactivation_hook()` pointing to static methods of the main class.
7. Boot via `add_action('plugins_loaded', ...)`.

```php
<?php
/**
 * Plugin Name:       KornSW <Name>
 * Description:       <one sentence in the project language>
 * Version:           0.1.0
 * Requires at least: <current WordPress major>
 * Requires PHP:      <newest stable PHP minor>
 * Author:            KornSW
 * License:           GPL-3.0
 * Text Domain:       <slug>
 */

declare(strict_types=1);

if (!defined('ABSPATH')) {
  exit;
}

define('KORNSW_XX_FILE', __FILE__);
define('KORNSW_XX_DIR', plugin_dir_path(__FILE__));
define('KORNSW_XX_URL', plugin_dir_url(__FILE__));
define('KORNSW_XX_VERSION', get_file_data(__FILE__, ['Version' => 'Version'])['Version']);

spl_autoload_register(function (string $className): void {
  $namespacePrefix = 'KornSW\\Xx\\';
  if (strncmp($className, $namespacePrefix, strlen($namespacePrefix)) !== 0) {
    return;
  }
  $relativeClassName = substr($className, strlen($namespacePrefix));
  $fileName = KORNSW_XX_DIR . 'src/' . str_replace('\\', '/', $relativeClassName) . '.php';
  if (is_readable($fileName)) {
    require_once $fileName;
  }
});

register_activation_hook(__FILE__, [\KornSW\Xx\Plugin::class, 'activate']);
add_action('plugins_loaded', [\KornSW\Xx\Plugin::class, 'boot']);
```

- `Version`, `Plugin URI` and `Update URI` are written by the release pipeline. Do not add or "fix" them by hand; a missing or older value in a working copy is expected.
- Never hard-code the version a second time (a separate version constant drifts from the header); read it from the header as shown.
- The entry file contains no logic. The main class `Plugin` is `final` and owns hook registration; features live in their own classes.
- Bundled third-party code goes to `<slug>/vendor/`, is moved into a sub-namespace of the plugin (`<Vendor>\<PluginName>\Vendor`) to avoid collisions with other plugins loading the same library, and keeps its license notice.

## 5. Formatting and Readability

Layout and readability follow `csharp-coding-standards` (sections 4, 8, 9), so that a C# developer reads the code fluently. PHP idioms are not bent: naming follows the PHP community standard (PSR-1 / PSR-12: classes `PascalCase`, methods and variables `camelCase`, constants `UPPER_CASE`), and WordPress API names stay as they are.

- Indentation: **two spaces**, no tabs.
- Opening braces at the end of the line for classes, methods, functions and control flow (`function boot(): void {`, `} else {`, `} catch (Exception $exception) {`).
- Braces for every control-flow block, also single statements. No alternative syntax (`if: ... endif;`) outside of template files.
- No ternary operator for normal control flow; prefer explicit `if` / `else`. No `@` error suppression.
- Explicit typing: `declare(strict_types=1);` in every PHP file, typed parameters, return types and properties; `?Type` for nullable values.
- Explicit instance access via `$this->`; descriptive names instead of abbreviations.
- Every class and method has a PHPDoc summary; comments explain intent, constraints and non-obvious decisions.
- Short array syntax `[]`. Strings in single quotes unless interpolation is needed.
- Files are UTF-8 **without BOM** (a BOM in a PHP file is sent as output and breaks headers). Omit the closing `?>` tag in pure PHP files.

## 6. Admin UI, Settings and Requests

- Admin pages require a capability check; settings pages use `manage_options`.
- Every state-changing request (form POST, admin-post, AJAX) checks a nonce (`check_admin_referer` / `wp_verify_nonce`) **and** a capability. GET requests never change state.
- An empty secret field in a settings form keeps the stored secret; removing a secret is an explicit action.
- Large tabular input travels as one JSON field (avoids `max_input_vars` limits).
- Cron / REST / AJAX code must not depend on interactive redirects or a logged-in browser.
- **(open)** Settings storage: one array option per plugin (`<short prefix>settings`) vs. many single options (both exist).

## 7. Data Storage and Lifecycle

- Own tables via `dbDelta()` in the activation hook, with a schema-version option (`<short prefix>db_version`) and an upgrade check on boot, because activation hooks do not run on plugin updates.
- Times are stored in UTC.
- Secrets (API keys, client secrets, tokens) are stored encrypted, never logged, and never shown in error messages.
- Disposable data (caches) and non-disposable data (sync state, user data) are kept apart; only caches may be wiped automatically.
- **Uninstall keeps the data.** Deactivating or deleting the plugin never drops tables or deletes options by default. A cleanup is offered only as an explicit action (e.g. an admin button or an `uninstall.php` guarded by an explicit setting), and it must ask whether the tables shall be kept; keeping them is the default answer.

## 8. Security Basics

- Escape on output (`esc_html`, `esc_attr`, `esc_url`, `wp_kses_post`), sanitize on input, `$wpdb->prepare()` for every query with variables.
- REST routes always define a `permission_callback`; a public route says so explicitly with `__return_true`.
- An external login must go through `wp_signon()` / the `authenticate` filter instead of setting auth cookies directly; otherwise 2FA and lockout plugins are bypassed.
- Plugins that need external access tokens consume the token infrastructure (`WP-AuthTokenHandling`) instead of implementing OAuth themselves.
- Outgoing HTTP via `wp_remote_*`, HTTPS only, credentials never inside URLs.

## 9. Never Edit by Hand

These are owned by the release pipeline (details: `nuget-semantic-versioning-kvu`, `references/wordpress-plugins.md`):

- `<slug>/self-update.php`, the block between `/*************** SELF-UPDATE ***************/` and `/*******************************************/`, the headers `Version` / `Plugin URI` / `Update URI`;
- `doc/<slug>.update.json`, `doc/versioninfo.json`, `doc/<slug>.updater-prefix.txt` (never change or delete);
- `doc/changelog.md` except for conformant entries below `## Upcoming Changes`;
- `.kvu/` and `.github/workflows/release.yml`: changes go into the template repository `WP-AutoUpdate-G2` first and are then rolled out to all plugin repositories identically.

## 10. Testing

- There is often no WordPress installation (and sometimes no native PHP) on the development machine. Keep domain logic free of WordPress calls where possible, so it can be tested with plain PHP scripts in `tests/` (`php tests/<name>.php`), using small stubs for the few WordPress functions needed.
- Always run at least a syntax check (`php -l`) of every changed file.
- Report honestly what was tested standalone and what still needs a live check in WordPress.

## 11. Documentation

- Repository docs follow `smart-solution-and-project-structure` (`README.md`, `doc/requirements.md`, `doc/architecture.md`, `doc/quickstart.md`, ...), written in the project language (section 1).
- A plugin that offers an API to other plugins documents it in `doc/api.md` (public functions, filters, actions) and keeps that API stable.
- Repository-specific AI knowledge goes to `.agents/skills/most-important-details-for-<slug-name>/SKILL.md` (English), not into loose `[AI-Skill] *.md` files in `doc/`.
- **(open)** Wrapping UI strings in `__()` / `esc_html__()` with the text domain (translation readiness) - rarely done so far.

## Agent Guidance

- Before creating or changing a plugin, read this skill, the plugin's own skill (if any) and the release reference.
- Starting a new plugin: ask the questions of section 1 first, copy the release files unchanged from the template, create the layout of section 2 and the entry file of section 4; do not pre-create generated files.
- When touching an older plugin that deviates from this skill (e.g. WordPress-style `class-*.php` files, hard-coded version constants, older minimum versions), do not refactor it silently. Name the deviation and propose aligning it separately from the functional change.
- For every **(open)** point you run into, follow the dominant variant of the repository at hand and ask the user to decide; record decisions here.

## Reference Repositories

| Repository | Role |
| --- | --- |
| [WP-AutoUpdate-G2](https://github.com/KornSW/WP-AutoUpdate-G2) | Leading template of the release files (`.kvu/`, `.github/workflows/release.yml`) with a minimal demo plugin. |
| [WP-AuthTokenHandling](https://github.com/KornSW/WP-AuthTokenHandling) | Reference plugin for multi-file layout, own tables, admin UI and a public API for other plugins (`doc/WordPress-Integration-API.md`); predates the class and formatting rules of this skill. |
