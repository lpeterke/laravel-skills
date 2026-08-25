# Refresh Skills Catalog

Internal-only maintenance routine for developing this repo (`lpeterke/laravel-skills`) itself, not for use in a Laravel project.

This is deliberately **not** named `SKILL.md` and does **not** live under `skills/`. The `npx skills` CLI discovers any file literally named `SKILL.md` anywhere in the repo tree, full stop — a `metadata: { internal: true }` frontmatter flag only removes a skill from `--all` and the default listing, it does *not* stop an explicit `npx skills add lpeterke/laravel-skills --skill <name>` from installing it (confirmed by testing). Keeping this out of the `SKILL.md` naming convention entirely is what makes it truly uninstallable into a project, by any invocation. Follow this file directly — CLAUDE.md points here by path, not by skill name.

Run this after: `laravel-init` changes what it installs from Boost or mattpocock/skills, or periodically to catch upstream drift.

## Step 1 — Check for upstream drift

Re-verify the two things `laravel-init` currently assumes are still true:

- **Laravel Boost**: fetch Boost's current install/update docs and compare its documented `boost:install` / `boost:update` / MCP registration details against what `skills/laravel-init/SKILL.md` Step 2 and Step 6 actually run. For the skill *list* specifically, don't trust the docs website's "Available Skills" table — confirmed stale once already (it was still showing `pest-testing` after Boost had already replaced it with `testing-best-practices` in `v2.6.0`). Get the real list from the package source instead: `git clone --depth 1 --branch v<latest> https://github.com/laravel/boost.git`, then `find .ai -type d -regex '.*/skill/[^/]*$' | sort -u` — search at *any* depth, not a fixed one, since some packages version their skill by major (`.ai/livewire/4/skill/livewire-development`, `.ai/inertia-vue/2/skill/...`) while most don't. Dedupe by skill name; the top-level `.ai/<pkg>/` directory it's nested under is the triggering package.
- **mattpocock/skills**: confirm `setup-matt-pocock-skills`' completion marker path (`docs/agents/issue-tracker.md`, checked by `laravel-init` Step 7) is still what that skill writes, and that mattpocock/skills' current skill names match what's listed below.

Report drift here; don't silently edit `skills/laravel-init/SKILL.md` to fix it — that's a judgment call for whoever reviews the report.

## Step 2 — Regenerate the README's "Available skills" section

This section documents what a project actually ends up with after running `laravel-init` — not this repo's own `skills/` folder (that's just `laravel-init` itself, which isn't the point of the section). Two sources, both external and both worth re-checking for drift each run:

- **Laravel Boost's skills**: use the package-source method from Step 1, not the docs website. Each ships tied to a package and only installs into a project that has that package — except `laravel-best-practices` and `testing-best-practices`, which live under `.ai/laravel/skill/` (the framework itself) and so land in every project. List name + triggering package; don't invent a description Boost's own source doesn't give.
- **mattpocock/skills**: each skill's own frontmatter `description` is written for model-routing, not README prose — trim it to a short, plain first clause rather than reproducing it verbatim in full.

Replace the README's `## Available skills` section with the result: one table per source, plus a one-line caveat that Boost's list is conditional on installed packages and mattpocock's evolves independently of this repo. Keep entries to name + short gist — this is a pointer to the upstream source, not a copy of its docs.

## Step 3 — Report

Summarize what changed: which skills were added to or dropped from either table, and any upstream drift found in Step 1 that needs a human decision.
