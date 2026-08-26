# Refresh Skills Catalog

Internal-only maintenance routine for developing this repo (`lpeterke/laravel-skills`) itself, not for use in a Laravel project.

This is deliberately **not** named `SKILL.md` and does **not** live under `skills/`. The `npx skills` CLI discovers any file literally named `SKILL.md` anywhere in the repo tree, full stop — a `metadata: { internal: true }` frontmatter flag only removes a skill from `--all` and the default listing, it does *not* stop an explicit `npx skills add lpeterke/laravel-skills --skill <name>` from installing it (confirmed by testing). Keeping this out of the `SKILL.md` naming convention entirely is what makes it truly uninstallable into a project, by any invocation. Follow this file directly — CLAUDE.md points here by path, not by skill name.

Run this after: `laravel-init` changes what it installs from Boost or mattpocock/skills, or periodically to catch upstream drift.

## Step 1 — Check for upstream drift

Re-verify the two things `laravel-init` currently assumes are still true:

- **Laravel Boost**: fetch Boost's current install/update docs and compare its documented `boost:install` / `boost:update` / MCP registration details against what `skills/laravel-init/SKILL.md` Step 2 and Step 6 actually run. For the skill *list* specifically, don't trust the docs website's "Available Skills" table — confirmed stale once already (it was still showing `pest-testing` after Boost had already replaced it with `testing-best-practices` in `v2.6.0`). Get the real list from the package source instead: `git clone --depth 1 --branch v<latest> https://github.com/laravel/boost.git`, then `find .ai -type d -regex '.*/skill/[^/]*$' | sort -u` — search at *any* depth, not a fixed one, since some packages version their skill by major (`.ai/livewire/4/skill/livewire-development`, `.ai/inertia-vue/2/skill/...`) while most don't. Dedupe by skill name; the top-level `.ai/<pkg>/` directory it's nested under is the triggering package.
- **mattpocock/skills**: confirm `setup-matt-pocock-skills`' completion marker path (`docs/agents/issue-tracker.md`, checked by `laravel-init` Step 7) is still what that skill writes, and that mattpocock/skills' current skill names match what's listed below.
- **The lint toolchain** that `laravel-lint-setup` targets — all listed with what to compare in CLAUDE.md's *Upstream sources* section. Re-check each against `skills/laravel-lint-setup/SKILL.md`:
  - [laravel/livewire-starter-kit](https://github.com/laravel/livewire-starter-kit) — clone shallow and diff its `pint.json`, `phpstan.neon` (paths and `level`), and `composer.json`'s `scripts` block against Steps 3, 5 and 7 of the skill (its `scripts` will not have Rector's `refactor`/`refactor:check` — that pair is deliberately ours, not parity). This is the parity target for Pint and Larastan; if the starter kit changes its level or renames a script, the skill is wrong until it follows. It is **not** the parity target for Pest — the starter kit's `composer.json` requires `phpunit/phpunit`, confirmed directly, so re-checking this file will never show Pest and that's not a drift signal.
  - [Pint docs](https://laravel.com/framework/docs/pint) — check the `Pint/laravel_blade` section still describes the same opt-in, and whether new `Pint/`-prefixed custom rules are worth adopting (`Pint/phpdoc_type_annotations_only` exists and is deliberately *not* enabled — it strips comment prose, which is a bigger call than a formatting opt-in).
  - **Pint's own source, for the prettier constraints.** Step 4 of the skill pins `prettier`, `prettier-plugin-blade` and `prettier-plugin-tailwindcss` to the versions Pint requires. Those live in `app/Fixers/LaravelBlade/Fixer.php::prettierDependencies()` in [laravel/pint](https://github.com/laravel/pint), not in any docs page. If they've moved, the skill's `npm install -D` line installs versions Pint then rejects with "do not satisfy the versions required".
  - [larastan/larastan](https://github.com/larastan/larastan) — confirm the recommended `includes:` (`vendor/larastan/larastan/extension.neon`) and the current major are unchanged. A 3.x → 4.x bump would likely move the include path or the package name, exactly as 2.x → 3.x moved it from `nunomaduro/larastan`.
  - [rectorphp/rector](https://github.com/rectorphp/rector) — check `RectorConfig::configure()`'s builder methods still carry the names Step 6 uses (`withPaths`, `withComposerBased`, `withPreparedSets`, `withSkip`) and that `--dry-run` still exits non-zero on pending changes. If that exit code ever becomes 0, `refactor:check` silently stops gating anything in `composer test` — check it, don't assume it.
  - [driftingly/rector-laravel](https://github.com/driftingly/rector-laravel) — the README's *Automate Laravel Upgrades* section. Step 6 uses `withComposerBased(laravel: true)` specifically because that README marks `LaravelSetList`/`LaravelLevelSetList` deprecated. If that recommendation flips again, the skill needs to follow.
  - [pestphp/pest](https://github.com/pestphp/pest) and [pestphp/pest-plugin-laravel](https://github.com/pestphp/pest-plugin-laravel) — check for a newly-added `UPGRADE.md`/`UPGRADING.md`/`CHANGELOG.md` (none existed as of writing, confirmed by checking directly), and re-check the PHP floor Pest's own `composer.json` declares (`^8.4` as of v5) since `references/version-check.md`'s Pest branch quotes that number.
  - [laravel/installer](https://github.com/laravel/installer) — re-diff `src/NewCommand.php::installPest()` against Step 2 of `laravel-lint-setup`. This is the entire source of that step's migration sequence; if the installer's own recipe changes (a new plugin, a different removal order), the skill is stale until it follows.

Report drift here; don't silently edit `skills/laravel-init/SKILL.md` or `skills/laravel-lint-setup/SKILL.md` to fix it — that's a judgment call for whoever reviews the report.

## Step 2 — Regenerate the README's "Available skills" section

This section documents what a project actually ends up with after running `laravel-init`. Three sources — two external and drifting, one local:

- **Laravel Boost's skills**: use the package-source method from Step 1, not the docs website. Each ships tied to a package and only installs into a project that has that package — except `laravel-best-practices` and `testing-best-practices`, which live under `.ai/laravel/skill/` (the framework itself) and so land in every project. List name + triggering package; don't invent a description Boost's own source doesn't give.
- **mattpocock/skills**: each skill's own frontmatter `description` is written for model-routing, not README prose — trim it to a short, plain first clause rather than reproducing it verbatim in full.
- **This repo's own skills**: generate from `skills/*/SKILL.md` — one row per folder, name plus a short gist trimmed from its frontmatter `description` the same way. Unlike the other two tables this one is derived from the working copy, so it's authoritative rather than a drift check; it exists so the README shows that `laravel-init` isn't the only thing here.

Replace the README's `## Available skills` section with the result. Order the sections with this repo's own first (it's the reason someone installed this), then Boost, then mattpocock.

**Tables and one-line source labels only — no explanatory prose between them.** This section is a catalog, and every past attempt to annotate it has been cut again. Specifically, do not reintroduce:

- **Caveats about the other two sources drifting** — that Boost's skills are conditional on installed packages, that mattpocock's is a curated subset, why particular skills were excluded. It's all already in `skills/laravel-init/SKILL.md` Step 4 and in CLAUDE.md, which is where it belongs; a consumer reading the README doesn't need the reasoning behind the curation.
- **Usage instructions for any listed skill**, including how to run the lint suite. If a command is worth knowing, it goes in that skill's one-line table cell (`laravel-lint-setup`'s cell already carries `composer test`) — never in a sentence under the table. The README answers install / invoke / update and nothing else; see CLAUDE.md's *The README* section.

If a table row can't carry what needs saying in one line, that's a sign the content belongs in CLAUDE.md or the skill itself, not that the README needs a paragraph.

## Step 3 — Report

Summarize what changed: which skills were added to or dropped from either table, and any upstream drift found in Step 1 that needs a human decision.
