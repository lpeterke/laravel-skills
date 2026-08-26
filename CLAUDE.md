# CLAUDE.md — developing this repo

This file is for an agent (or Lars) working *on* this repository itself — i.e. adding, editing, or restructuring skills — not for an agent using these skills inside some other Laravel project.

## The goal

This repo is Lars's personal, portable layer of AI-agent skills for Laravel/Statamic freelance work. It exists to solve one problem: getting consistent, high-quality AI coding assistance set up in a new or existing Laravel project should take one command, not a checklist of manual steps re-derived from memory each time.

It sits alongside two other pieces that are deliberately *not* duplicated here:

- **Laravel Boost** (`laravel/boost`) — official, per-project, tied to whatever packages that specific project has installed (Livewire, Filament, Pest, etc.). Installed/updated via Composer + `php artisan boost:install`/`boost:update`, not via this repo.
- **mattpocock/skills** — third-party general engineering skills (grilling for requirements, domain modeling, code review, TDD). Installed via `npx skills add mattpocock/skills`, not duplicated here.

This repo is for everything that's specific to *Lars* rather than to a project or to general engineering practice: his own Laravel/Statamic conventions and his own workflows. Two skills so far, and they're deliberately independent of each other:

- **`laravel-init`** — glues Boost + mattpocock/skills together so setting up a project's *AI tooling* is one instruction instead of three separate systems to remember. It installs and refreshes `laravel-lint-setup` along with everything else, but never runs it.
- **`laravel-lint-setup`** — Pint + Blade formatting + Larastan + Rector, to `laravel/livewire-starter-kit` parity (Blade formatting and Rector are additions on top). Standalone and user-invoked.

**Keep them separate.** AI tooling and linting don't intersect: `laravel-lint-setup` depends on nothing `laravel-init` does (its only precondition is a `composer.json` requiring `laravel/framework`), and `laravel-init` doesn't invoke it. That's a deliberate boundary, not an oversight — `laravel-lint-setup` rewrites `composer.json`, reformats every Blade file, and lets Rector rewrite application logic, which is a categorically different blast radius from installing agent skills, and it's the user's call when to accept it. Don't "helpfully" chain them.

Skills here are distributed via the [skills CLI](https://github.com/vercel-labs/skills) (`npx skills`), which discovers any `SKILL.md` file anywhere in this repo automatically (confirmed by reading the CLI's own source — it isn't scoped to `skills/`) — it does not read this file or the README to decide what exists. Keep that in mind: the README is for humans, this file is for agents editing the repo, and neither one gates what `npx skills add lpeterke/laravel-skills --all` actually installs.

A skill can opt out of the default `--all`/listing behavior with `metadata: { internal: true }` in its frontmatter (confirmed from the CLI source). This is a partial exclusion, not a hard one: an explicit `npx skills add ... --skill <name>` still installs it regardless of the flag. For repo-development tooling that must never be installable into a project under any invocation, don't use `metadata: internal` — don't name the file `SKILL.md` at all, and keep it outside `skills/`. The CLI's discovery is a literal filename match against `SKILL.md` anywhere in the tree; anything else is invisible to it, full stop. See `internal/refresh-skills-catalog.md` for the pattern.

## The README

Keep it brutally short. It answers exactly three questions for someone who wants to *use* these skills in a Laravel project: how do I install them, how do I invoke them, how do I update them. Nothing else earns a place.

- **Commands, not prose.** A heading and a code block. At most one line of context under it, and only when the command is genuinely ambiguous without it.
- **Consumer-facing only.** Anything about developing *this* repo — testing a change, publishing one, how the skills CLI behaves internally, why a step exists — goes in this file instead. If Lars is the only person who'd ever run it, it isn't README material.
- **No migration or one-off cleanup steps.** When a change here would strand an already-installed project, fix it inside the skill so it self-heals on the next run. Don't push the fix onto the reader.
- **No machine-specific paths.** This repo may end up public; `/Users/lars/...` doesn't belong in it.
- **The "Available skills" section is generated, never hand-edited.** Run the routine in `internal/refresh-skills-catalog.md` to regenerate it — don't write catalog entries by hand, that's exactly the kind of drift a manually maintained list accumulates.

If you're about to add a fourth section, the answer is almost always that the content belongs here.

## Repo structure

```
laravel-skills/
├── README.md              — install, use, update, available skills. Commands + a generated catalog (see below)
├── CLAUDE.md               — this file
├── internal/               — repo-dev-only routines; never discoverable by the skills CLI (no SKILL.md filename, not under skills/)
│   └── refresh-skills-catalog.md
└── skills/
    ├── laravel-init/       — AI-tooling entry point; installs Boost, mattpocock's set, and this repo's own skills
    ├── laravel-lint-setup/ — Pint + Blade formatting + Larastan + Rector. Standalone; user-invoked
    └── <skill-name>/
        └── SKILL.md        — required: YAML frontmatter (name, description) + instructions
            (optional: scripts/, references/, assets/ subfolders per skill)
            (optional: metadata: { internal: true } — excludes it from --all/listing, but not from an explicit --skill install; see above)
```

Every published skill lives in its own folder under `skills/`. One `SKILL.md` per folder, folder name matches the skill's `name` frontmatter field, kebab-case. Repo-dev-only routines that must stay unpublishable go in `internal/` instead, as a plain `.md` file — see above for why.

## Adding a new skill

1. **Decide if it belongs here at all.** Ask: is this Laravel/Statamic-specific-to-Lars knowledge, or a personal workflow? If it's generic engineering practice, it probably belongs upstream in mattpocock/skills territory (or just isn't worth a skill). If it's project-specific rather than portable across all of Lars's projects, it belongs in that project's own `.ai/guidelines/`, not here.
2. **Write a pushy, specific description.** Skills under-trigger by default. The frontmatter `description` should name concrete trigger phrases and contexts, not just describe capability in the abstract. Compare: "Helps with Livewire components" (weak) vs. "Use whenever the user is building, reviewing, or debugging a Livewire component, mentions wire:model, wire:click, or asks about Livewire lifecycle hooks" (better).
3. **Keep `SKILL.md` under ~500 lines.** If it's growing past that, split detail into a `references/` file and point to it from the main body rather than inlining everything.
4. **Make it idempotent wherever it touches project state.** Every skill that runs commands (like `laravel-init` does) should check current state first and only change what's actually needed — never assume a clean slate, never assume it hasn't run before.
5. **Test it in a real Laravel project before committing.** `npx skills add /Users/lars/code/laravel-skills --skill <name> -p` into a scratch project and actually run it once. See *Distribution mechanics* below for why that local install needs re-running after every edit.

## Distribution mechanics

Shipping a change from this repo to a project:

```bash
# here
git commit -am "…" && git push

# in the project that should get it
npx skills update -p -y
```

To try a change before pushing, install from the path instead (`npx skills add <path-to-this-repo> --all -p`) and re-run that after every edit — see below for why `update` won't do it.

The details, worth knowing precisely because several of these behaviours are silent:

- **`npx skills update` refetches from GitHub, not from the working copy.** Committing isn't enough; unpushed edits are invisible to every consuming project. Push, then update there.
- **`skills-lock.json` is written automatically on every `add`.** It is not a user-chosen pin — it records each skill's `source`, `sourceType`, and a `computedHash`, and `update` rewrites it in place. Never treat its presence as a reason to stop and ask before updating. (`npx skills experimental_install` rebuilds a project from it.)
- **`update` silently skips local-source skills.** Only entries with `"sourceType": "github"` are refetched. A skill installed from a path (`npx skills add /Users/lars/code/laravel-skills --all -p`) is recorded as `"sourceType": "local"` and ignored — the CLI prints `No project skills to update.` and exits 0, not an error. To pick up further local edits, re-run the `add`. Once the change is pushed, re-`add` from `lpeterke/laravel-skills` so the project is back on the GitHub source and future updates work.
- **`add` copies, it doesn't symlink the source.** The copy lands in the project's `.agents/skills/<name>/`; the per-agent directories (`.claude/skills/` etc.) are symlinks into that. So editing this repo never live-updates an installed project.
- **`laravel-init` updates itself one run late.** The agent follows the copy already installed in the project. Its own update step pulls the new version, but the run in progress finishes on the old instructions. Any change to it therefore takes effect on the next invocation, in a fresh session.
- **`update` never adds a skill that isn't already in the lock.** It iterates `skills-lock.json` and refreshes those entries only. Verified: a project whose lock listed 2 of a repo's 37 skills still had exactly 2 after `npx skills update -p -y`. New skills added to this repo therefore reach a project only via `npx skills add`, never via `update`.
- **`add` overwrites an already-installed skill in place.** It is not a no-op on a skill that's already there — the installed `SKILL.md` is replaced with the current upstream copy. Verified by editing a source skill and re-running `add --all`: the change landed. This is why `laravel-init` Step 5's `npx skills add lpeterke/laravel-skills --all -p -y` is the update mechanism for this repo's own skills, and why they need no separate `update` handling.
- **A renamed skill is a delete plus an add, and neither `add` nor `update` does the delete.** `update` flags the old name (`The following skills … appear to have been deleted upstream`) but keeps it, because `-y` means non-interactive and deletion is skipped; `add --all` fetches the new name and leaves the old directory sitting there. Verified: after renaming a skill upstream, a re-run of `add --all` left *both* directories installed. `laravel-init` Step 5 therefore ends with an explicit prune — it lists the repo's current skills over the GitHub contents API and removes any `skills-lock.json` entry sourced from `lpeterke/laravel-skills` that's no longer among them. That generalises what used to be a hardcoded cleanup for the `init` → `laravel-init` rename, so a future rename needs no new code and no README migration step. Prefer this pattern over a README step whenever a repo change would otherwise strand a project.
  - The prune's one real hazard: if the API call fails, the "current skills" list is empty and *everything* from this repo matches as stale. Confirmed by testing — the filter marked the live skill for removal. The step guards with a hard `exit 0` on an empty list; don't remove that guard.

## Updating the `laravel-init` skill specifically

`laravel-init` is the orchestration point between this repo, Boost, and mattpocock/skills. If Boost's install/update commands change (Boost evolves independently of this repo), or if the mattpocock/skills repo restructures its own skill names, update `skills/laravel-init/SKILL.md` to match. Don't let it silently drift out of date with either upstream project.

Don't rely on remembering to check this by hand — `internal/refresh-skills-catalog.md` (see below) automates exactly this drift check.

One structural note: `laravel-init` Step 5 is what keeps this repo's own skills current in every project, and it takes
three parts — `npx skills add lpeterke/laravel-skills --all -p -y`, then `npx skills update -p -y`, then the stale-skill
prune. Each covers a gap the others don't:

| Change here | Reaches a project via |
| --- | --- |
| Edit to an existing skill | `add --all` (it overwrites in place) |
| Brand-new skill | `add --all` only — `update` can't see it, it's not in the lock |
| Renamed or deleted skill | the prune only — `add` leaves the old copy, `update` won't delete under `-y` |

Keep all three when editing Step 5. Dropping the `add` strands projects on the skill set they were first installed
with; dropping the prune leaves dead skills installed forever.

## Upstream sources

Everything in this repo that mirrors someone else's decisions has an upstream to re-derive it from. When Lars asks to
"bring a skill up to date", these are the places to look — go to the source, not to memory, and not to a docs page when
the source is code.

**`laravel-lint-setup`** — three upstreams, checked in this order:

| Source | What to read | What in the skill depends on it |
| --- | --- | --- |
| [laravel/livewire-starter-kit](https://github.com/laravel/livewire-starter-kit) | `pint.json`, `phpstan.neon`, `composer.json`'s `scripts`, `.editorconfig` | Steps 2, 4, 6, 8 — this is the parity target, everything else is secondary |
| [Pint docs](https://laravel.com/framework/docs/pint) | The *Custom Rules* section, esp. `Pint/laravel_blade`; the CLI options | Step 2's opt-in, Step 7's troubleshooting |
| [laravel/pint](https://github.com/laravel/pint) source | `app/Fixers/LaravelBlade/Fixer.php::prettierDependencies()`, `app/Actions/EnsurePrettierIsConfigured.php`, `app/Enums/NodePackageManager.php` | Step 3's exact npm constraints and package-manager detection |
| [larastan/larastan](https://github.com/larastan/larastan) | README install + config section | Step 1's package name, Step 4's `includes:` path and level |
| [rectorphp/rector](https://github.com/rectorphp/rector) | README, `templates/rector.php.dist`, `RectorConfigBuilder` | Step 5's config shape and prepared-set names |
| [driftingly/rector-laravel](https://github.com/driftingly/rector-laravel) | README's *Automate Laravel Upgrades* section | Step 5's `withComposerBased(laravel: true)` |

Three things learned from the source that no docs page states, and that the skill depends on:

- **The Blade rule's Prettier prompt is fatal to an agent.** When `prettier`/`prettier-plugin-blade`/
  `prettier-plugin-tailwindcss` are missing, Pint calls `confirm(default: false)`; non-interactively that resolves to
  *no* and Pint `abort(1)`s. The skill installs the packages itself rather than letting Pint offer to. Read
  `EnsurePrettierIsConfigured::installMissing()` if this ever seems to have changed.
- **The exact npm constraints live in PHP, not in `package.json` or the docs** — `Fixer::prettierDependencies()`.
  They were `prettier ^3.8.4`, `prettier-plugin-blade ^3.2.2`, `prettier-plugin-tailwindcss ^0.8.0` when the skill was
  written. Pint's abort message quotes the current ones precisely, so a mismatch is self-diagnosing at runtime.
- **The starter kit does *not* enable Blade formatting.** Its `pint.json` is bare `{"preset": "laravel"}`. The
  `Pint/laravel_blade` opt-in is Lars's addition on top of parity — don't "fix" it back to match the starter kit on a
  future drift check.
- **Pint emits JSON, not a table, when an agent runs it** (via `laravel/agent-detector`). One
  `{"tool":"pint","result":…,"files":[…]}` object per line. Verified live, and documented in the skill's Step 7 so a
  future agent doesn't read the missing summary line as a failure.

And four more from testing Rector, all of which the skill depends on:

- **`rector process --dry-run` exits 2 with pending changes, 0 when clean.** That non-zero exit is the only reason
  `refactor:check` works as a gate in `composer test`. Verified both directions.
- **Rector's output is not Pint-formatted.** After `rector process`, `pint --test` exits 1 — Rector doesn't add the
  blank line before `return` that the Laravel preset wants. Hence `refactor` before `lint`, and `refactor:check` before
  `lint:check` in the `test` script. Don't reorder those.
- **`codeQuality: true` adds `declare(strict_types=1)`** via `SafeDeclareStrictTypesRector`; `deadCode: true` doesn't.
  Isolated by testing each prepared set alone. The rule is guarded by a per-file static `isFileStrictTypeSafe()` check,
  which can't see a caller coercing a scalar at runtime — so it's a real, if bounded, behaviour change and the skill
  reports it explicitly rather than slipping it into a file count.
- **Rector needs `require.php` in `composer.json`.** Without it, `withPhpSets()` dies with *"We could not find local
  composer.json to determine your PHP version"*. Every real Laravel skeleton has one; the failure only shows up on
  hand-rolled `composer.json` files.

**Rector is not starter-kit parity.** `laravel/livewire-starter-kit` ships no `rector.php` and no Rector dependency —
this is Lars's addition, like the Blade rule. Don't remove it on a future drift check for not matching the starter kit.

The drift check for all of this is automated in `internal/refresh-skills-catalog.md` Step 1. Prefer running that over
checking these by hand.

## Keeping this repo's own docs in sync

`internal/refresh-skills-catalog.md` is this repo's own maintenance routine: it checks for drift against Boost and mattpocock/skills, and regenerates the README's "Available skills" section — the catalog of what `laravel-init` actually pulls into a project, sourced from Boost's and mattpocock/skills' current skill sets, not from this repo's own `skills/` folder. It's not a real skill by design (see *The goal* above), so follow it by reading the file directly rather than by name.

Run it after any bigger change to this repo — what `laravel-init` installs from Boost or mattpocock/skills changes, or anything else that could make the README or `laravel-init` drift from reality. When in doubt after a change, run it.

## Things to avoid

- Don't duplicate content that Boost or mattpocock/skills already own — link to / invoke them instead. The "Available skills" catalog is the one sanctioned exception: name + a short gist per skill, for discoverability, not their instructions. If a table entry grows past one line, it's become duplication.
- Don't bake in machine-specific paths, personal API keys, or anything not safe to commit publicly (this repo may end up public/shared).
- Don't add a `depends_on`-style mechanism expecting the skills CLI to auto-install sibling skills — that's not a shipped feature as of writing. `--all` already installs everything in this repo in one command; that's sufficient.
