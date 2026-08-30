# CLAUDE.md — developing this repo

This file is for an agent (or Lars) working *on* this repository itself — i.e. adding, editing, or restructuring skills — not for an agent using these skills inside some other Laravel project.

## The goal

This repo is Lars's personal, portable layer of AI-agent skills for Laravel/Statamic freelance work. It exists to solve one problem: getting consistent, high-quality AI coding assistance set up in a new or existing Laravel project should take one command, not a checklist of manual steps re-derived from memory each time.

It sits alongside four other pieces that are deliberately *not* duplicated here:

- **Laravel Boost** (`laravel/boost`) — official, per-project, tied to whatever packages that specific project has installed (Livewire, Filament, Pest, etc.). Installed/updated via Composer + `php artisan boost:install`/`boost:update`, not via this repo.
- **mattpocock/skills** — third-party general engineering skills (grilling for requirements, domain modeling, code review, TDD). Installed via `npx skills add mattpocock/skills`, not duplicated here.
- **EveryInc/compound-engineering-plugin** — exactly three of its 33 skills: `ce-compound`, `ce-compound-refresh` and `ce-doc-review`. Installed via `npx skills add`, not duplicated here. See *Upstream sources* for why the other 31 are excluded.
- **olgunozoktas/livewire-alpine-skills** — all five of its skills (`livewire-reference`, `livewire-security`, `livewire-performance`, `alpinejs-reference`, `alpinejs-security`), gated by whether the project has Livewire, Alpine, or neither. Installed via `npx skills add`, not duplicated here. See *Upstream sources* for the gate logic and why this repo previously took only one of its three skills.

This repo is for everything that's specific to *Lars* rather than to a project or to general engineering practice: his own Laravel/Statamic conventions and his own workflows. Four skills so far:

- **`laravel-init`** — glues Boost + mattpocock/skills together so setting up a project's *AI tooling* is one instruction instead of three separate systems to remember. It installs and refreshes `laravel-lint-setup` along with everything else, but never runs it.
- **`laravel-lint-setup`** — Pint + Blade formatting + Larastan + Rector, to `laravel/livewire-starter-kit` parity (Blade formatting and Rector are additions on top), plus Pest as this repo's strongly preferred testing framework — installed if missing, existing PHPUnit tests migrated to Pest syntax, kept current including major versions. Standalone and user-invoked.
- **`laravel-task`** — the one that actually does work. Grill the requirements, write a plan, grill the plan, implement it routed by the project's real stack, cover it with a Pest test, run `composer test`, capture the learning. It owns the *sequence* and delegates the *substance*: almost nothing in it is Laravel knowledge, it's knowledge about which installed skill to reach for and when. It depends on `laravel-init` having run (Step 0 checks and stops if not) but never runs it, and it never runs `laravel-lint-setup` either.

- **`laravel-audit`** — the read-only counterpart to `laravel-task`. Same orchestration shape, but it produces a judgement rather than a change: runs the olgunozoktas static scanners for hard evidence, judges idiom against `laravel-best-practices` and the project's own `.ai/rules/`, and reports verdicts in three fields (implementation, performance, security). **Its read-only contract is the whole point** — it excludes three skills specifically because they write (`infer-conventions` writes `.ai/rules/`, `ce-compound` writes `docs/solutions/`, `improve-codebase-architecture` grills the user into implementing). If an audit finding needs fixing, that's a fresh `laravel-task` invocation, never a continuation.

**Keep `laravel-init` and `laravel-lint-setup` separate.** AI tooling and linting don't intersect: `laravel-lint-setup` depends on nothing `laravel-init` does (its only precondition is a `composer.json` requiring `laravel/framework`), and `laravel-init` doesn't invoke it. `laravel-task` is the one skill that *reads* the others' output — it needs `laravel-init` to have run and it prefers `laravel-lint-setup` to have run — but it invokes neither, for the same reason: it reports what's missing and names the skill, and the user decides. That's a deliberate boundary, not an oversight — `laravel-lint-setup` rewrites `composer.json`, reformats every Blade file, lets Rector rewrite application logic, and migrates existing tests from PHPUnit to Pest, which is a categorically different blast radius from installing agent skills, and it's the user's call when to accept it. Don't "helpfully" chain them.

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
    ├── laravel-lint-setup/ — Pint + Blade formatting + Larastan + Rector + Pest (strongly preferred). Standalone; user-invoked
    ├── laravel-task/        — the work loop: grill → plan → grill → implement → Pest test → composer test → capture
    ├── laravel-audit/       — read-only assessment: scanners → skill-routed judgement → 3-field verdict
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
- **`add` overwrites an already-installed skill in place.** It is not a no-op on a skill that's already there — the installed `SKILL.md` is replaced with the current upstream copy. Verified by editing a source skill and re-running `add --all`: the change landed. This is why `laravel-init` Step 6's `npx skills add lpeterke/laravel-skills --all -p -y` is the update mechanism for this repo's own skills, and why they need no separate `update` handling.
- **A renamed skill is a delete plus an add, and neither `add` nor `update` does the delete.** `update` flags the old name (`The following skills … appear to have been deleted upstream`) but keeps it, because `-y` means non-interactive and deletion is skipped; `add --all` fetches the new name and leaves the old directory sitting there. Verified: after renaming a skill upstream, a re-run of `add --all` left *both* directories installed. `laravel-init` Step 6 therefore ends with an explicit prune — it lists the repo's current skills over the GitHub contents API and removes any `skills-lock.json` entry sourced from `lpeterke/laravel-skills` that's no longer among them. That generalises what used to be a hardcoded cleanup for the `init` → `laravel-init` rename, so a future rename needs no new code and no README migration step. Prefer this pattern over a README step whenever a repo change would otherwise strand a project.
  - The prune's one real hazard: if the API call fails, the "current skills" list is empty and *everything* from this repo matches as stale. Confirmed by testing — the filter marked the live skill for removal. The step guards with a hard `exit 0` on an empty list; don't remove that guard.

**Step numbering.** `laravel-init` is now ten steps: 1 Laravel check, 2 Boost, 3 gitignore, 4 mattpocock, 5
olgunozoktas, 6 EveryInc, 7 refresh+prune, 8 MCP check, 9 mattpocock setup, 10 summary. Inserting a step means
renumbering every later heading *and* every in-body cross-reference ("as Step 4 does…", "Step 7's `npx skills
update`") — those cross-references are load-bearing and have gone stale once already. Grep for `Step [0-9]` after any
insertion.

## Updating the `laravel-init` skill specifically

`laravel-init` is the orchestration point between this repo, Boost, and mattpocock/skills. If Boost's install/update commands change (Boost evolves independently of this repo), or if the mattpocock/skills repo restructures its own skill names, update `skills/laravel-init/SKILL.md` to match. Don't let it silently drift out of date with either upstream project.

Don't rely on remembering to check this by hand — `internal/refresh-skills-catalog.md` (see below) automates exactly this drift check.

One structural note: `laravel-init` Step 7 is what keeps this repo's own skills current in every project, and it takes
three parts — `npx skills add lpeterke/laravel-skills --all -p -y`, then `npx skills update -p -y`, then the stale-skill
prune. Each covers a gap the others don't:

| Change here | Reaches a project via |
| --- | --- |
| Edit to an existing skill | `add --all` (it overwrites in place) |
| Brand-new skill | `add --all` only — `update` can't see it, it's not in the lock |
| Renamed or deleted skill | the prune only — `add` leaves the old copy, `update` won't delete under `-y` |

Keep all three when editing Step 7. Dropping the `add` strands projects on the skill set they were first installed
with; dropping the prune leaves dead skills installed forever.

## Upstream sources

Everything in this repo that mirrors someone else's decisions has an upstream to re-derive it from. When Lars asks to
"bring a skill up to date", these are the places to look — go to the source, not to memory, and not to a docs page when
the source is code.

**`laravel-init` Step 5 — [olgunozoktas/livewire-alpine-skills](https://github.com/olgunozoktas/livewire-alpine-skills).**
That repo shipped three skills when first adopted (`laravel-init` took one, `livewire-security`) and now ships five:
`livewire-reference`, `livewire-security`, `livewire-performance`, `alpinejs-reference`, `alpinejs-security`. All five
are taken now — the two-of-three exclusion from the first integration is history, not current policy. Re-read this
whole section before assuming otherwise; it explains both why the exclusion existed and why it stopped applying.

**Why the original `livewire-development` was excluded, and why that's no longer the situation.** It collided by name
with Boost's own skill, and the collision was destructive, not cosmetic: Boost writes to
`base_path($agent->skillsPath().'/'.$skill->name)` → `.claude/skills/livewire-development`
(`SkillWriter.php:38`, `ClaudeCode.php:62`), through a `copyDirectory()` that calls `deleteDirectory($target)` first
(`SkillWriter.php:206`), and `deleteDirectory()` unlinks a symlink (`SkillWriter.php:156`) — exactly the symlink
`npx skills` creates. So `boost:install --skills` would silently unlink it and drop Boost's copy in place, while
`.agents/skills/` and `skills-lock.json` still claimed the other skill was installed. **The upstream repo's own
`CHANGELOG.md` (1.0.0) confirms this was noticed and fixed on their end**: `livewire-development` → `livewire-reference`
and `alpinejs-development` → `alpinejs-reference`, both renamed specifically because "Laravel Boost ships its own
skill named `livewire-development`, and an identical name read as a replacement for it." No stub was left at either
old name — the maintainer's own words: "a stub would restore the collision." With the rename, `npx skills add
olgunozoktas/livewire-alpine-skills --skill livewire-development` fails outright (the name doesn't exist upstream
anymore), so there's no residual risk of a future edit accidentally reintroducing the old name.

The `alpinejs-development` exclusion was never about a collision — Boost ships no Alpine skill at all — it was purely
that a fourth upstream skill wasn't worth tracking until it paid for itself. `alpinejs-security` (new in 1.3.0) is
what changed that calculus: Alpine's own docs mention security in six lines across 56 pages, and the skill's core
finding — HTML-escaping a value into `x-data`/`x-on`/`x-init` does not protect it, because Alpine compiles the raw
attribute text with `new AsyncFunction` (Seam B), not just the sink it warns about (`x-html`, Seam A) — is real,
verified against `alpinejs/alpine` source (`evaluator.js` around line 94, `mutation.js:49`), and something neither
Boost nor mattpocock/skills covers anywhere.

**The gate in Step 5 has two tiers, not one**, because `alpinejs-reference`/`alpinejs-security` are useful without
Livewire — both skills' own frontmatter names Rails, Django, Hotwire and plain HTML as valid contexts. Projects with
Livewire (checked via `composer.lock`, not `composer.json`'s `require` — see below) get all five; projects with
`alpinejs` in `package.json` but no Livewire get just the two Alpine skills; projects with neither get none, pruned
back down automatically if a previous run installed more.

**The Livewire check is deliberately not the same as Boost's own.** Boost's `DiscoverPackagePaths::$mustBeDirect`
withholds `livewire-development` unless `livewire/livewire` is a *direct* `composer.json` requirement — sound for
Boost's purpose (skip Livewire-authoring guidance for a team that never hand-writes a component because Filament
pulled it in for them) but wrong for security/performance relevance: every Filament resource, action, and widget *is*
a Livewire component under the hood, so the leak surface and the request cost are identical whether the dependency
arrived directly or transitively. `composer.lock` is checked instead — verified against a real project (keikaku) where
Filament is the only direct Livewire-adjacent requirement and `livewire/livewire` appears only in `composer.lock`; a
direct-only check reported "skipped" on a project shipping six Filament packages built on Livewire.

What was verified before integrating the five-skill set, spot-checked against `livewire/livewire@4.x` source,
`alpinejs/alpine` source, and `laravel/octane` source (not taken on the skill's word): the checksum failure
rate-limiter is keyed on client IP alone (`Checksum.php`'s `rateLimitKey()` — `'livewire-checksum-failures:' .
request()->ip()`, 10 failures, 600-second decay, matching the skill's claim exactly); `ModelSynth` restores a model
property through `newQueryForRestoration($key)->useWritePdo()->firstOrFail()`, so every restoration hits the write
connection; Octane registers `PrepareLivewireForNextOperation` **by default** (`ProvidesDefaultConfigurationOptions.php`),
which calls `LivewireManager::flushState()` — confirming the skill's own withdrawn-and-corrected 1.2.0→1.2.1 finding;
Alpine's expression compiler really does use `new AsyncFunction` on raw attribute text, and the document-wide
`MutationObserver` really does make an injected `x-*` attribute an execution sink. All five skills' self-test suites
were run and pass: `livewire-reference` 53/53, `livewire-security` 24/24, `livewire-performance` 13/13,
`alpinejs-reference` 30/30, `alpinejs-security` 26/26. `livewire-security/bin/verify-facts.php`, run against a real
Filament app (keikaku), reported 28/28 statements still holding. Each skill also ships `bin/check-update.sh`, which
compares a locally installed copy against the upstream repo and reports one line when stale — not wired into
`laravel-init` (that's what Step 6's `npx skills update` already does at the lock-file level), but useful for a human
checking currency by hand.

Treat this upstream as more proven than at first adoption but still young — its `CHANGELOG.md` shows an active pattern
of self-correction (a wrong Octane finding shipped in 1.2.0 and was withdrawn one release later with the full
reasoning kept for the record; a false-positive Alpine URL rule that misfired 14/14 times on a real audit was fixed
in 1.3.1) which is a *good* sign about how the maintainer works, not a red flag — but it means this repo's own content
is still moving fast. Re-run the self-tests and `verify-facts`/`verify-facts.php` after any version bump rather than
assuming last quarter's audit still holds.

**`laravel-init` Step 6 — [EveryInc/compound-engineering-plugin](https://github.com/EveryInc/compound-engineering-plugin).**
A 33-skill Claude Code plugin implementing a brainstorm → plan → build → review → capture loop. Exactly two of the 33
are taken: `ce-compound`, `ce-compound-refresh` and `ce-doc-review`.

**Why three and not more, and why this isn't likely to change.** The first four stages of that loop are already covered
by skills this repo installs, and installing both sets would give the agent two answers to every question. Concretely:
`ce-plan`/`ce-brainstorm` compete with mattpocock's `grilling`/`to-spec`; `ce-work` is not just an implementer but a
full orchestrator that claims branch placement, worker dispatch, canonical commits, a mandatory `ce-code-review`
completion gate and a PR tail — it would *replace* `laravel-task` rather than serve it, which is the specific reason
it was rejected; `ce-debug` overlaps `diagnosing-bugs`; `ce-code-review` overlaps `code-review` and Claude Code's own
`/code-review`; `ce-simplify-code` overlaps `/simplify`; `ce-handoff` overlaps `handoff`.

**`ce-doc-review` was excluded on that same rule at first pass, and that was wrong** — corrected in a later audit and
worth recording so it isn't re-excluded. It reviews a written plan through role-specific lenses and returns findings,
and mattpocock ships **no** plan-review skill at all. `grilling` is not one: it interrogates *the user* in rounds, so
pointing it at a document spends the user's attention on findings a reviewer produces unattended. `laravel-task`
Step 1c now runs `ce-doc-review` non-interactively first and reserves `grilling` for what it surfaces that is
genuinely the user's decision. Its handoff menu offers `ce-plan`/`ce-work`; Step 1c is told to ignore it, since
`laravel-task` Step 2 is that run's next stage. The rest
(`ce-commit-push-pr`, `ce-babysit-pr`, `ce-dogfood`, `ce-promote`, `ce-product-pulse`, `ce-sweep`, `ce-test-xcode`,
`ce-retune`, `ce-proof`…) is shipping/product workflow, iOS, or plugin-maintenance tooling with no place in a
freelance Laravel engagement. Don't reopen this on a drift check unless one of the *excluded* skills has changed
character — a version bump alone isn't a reason.

The compound pair is the other genuinely additive part: writing a solved problem up as a durable, grounded repo
learning under `docs/solutions/` that the next session reads, and auditing those learnings for drift. Boost has no
equivalent; mattpocock has none either (`handoff` compacts a conversation, which is a different artifact with a
different lifetime). `laravel-task` Step 5 is the caller.

Verified empirically before adopting, all four load-bearing:

- **`npx skills add` works against a Claude Code plugin repo.** The CLI matches `SKILL.md` by filename anywhere in the
  tree and doesn't care about `plugin.json`. Both skills installed with `references/`, `scripts/` and `assets/`
  intact — `ce-compound` ships 13 reference files, 7 subagent prompt files and 3 scripts; `ce-doc-review` ships 15
  references, 8 persona files and 2 scripts. Every bundled path either one names resolves from the installed
  location (checked exhaustively on both, zero misses).
- **Neither needs the plugin runtime.** `CLAUDE_PLUGIN_ROOT` appears in exactly four skills — `ce-optimize`,
  `ce-setup`, `ce-sweep`, `ce-test-browser` — and none is in the set. Several skills that *do* look self-contained
  aren't; grep before adding any more.
- **Neither hard-depends on an uninstalled sibling.** `ce-compound` names `ce-simplify-code` only to forbid invoking
  it, and `ce-oracle`/`ce-issues` only as optional manual follow-ups. Its one real sibling dependency is
  `ce-compound-refresh` — which is why the set is two, not one.
- **`mode:non-interactive` is a genuine contract.** Both parse the token and then ask no blocking question in any
  phase, terminating on a parseable `Documentation complete` / `Documentation skipped <reason>`. That's what makes
  `ce-compound` callable from inside `laravel-task`'s run without deadlocking it.

Note the write surface: `ce-compound` writes under `docs/solutions/` (relocatable via
`.compound-engineering/config.yaml`'s `docs_root`) and may create or edit a root `CONCEPTS.md`. Installing it touches
nothing; only invoking it does.

**`laravel-task`** — it has no single upstream to re-derive from, because it's pure orchestration over skills that do.
Its drift risk is entirely in the *names* it routes to. Four things it depends on, each verified from source rather
than docs:

- **Boost's skill set is per-project, not fixed — but all 14 skills live in Boost's own repo.** Corrected after an
  initial `find` that matched only `SKILL.md` and so missed the ones shipped as `SKILL.blade.php` (exactly the trap
  `internal/refresh-skills-catalog.md` already warns about — glob `SKILL.*`). The right command is
  `find .ai -type d -regex '.*/skill/[^/]*$'`, which returns all 14 under `.ai/<guideline-name>/[<major>/]skill/`.
  What varies per project is which of them get *installed*: `discoverPackagePaths()` keeps only those whose
  `.ai/<name>/` directory matches a package the project actually has, with `PackageRegistry::guidelineName()` doing
  the mapping (`laravel/folio` → `folio`, `livewire/flux` → `fluxui-free`, `@inertiajs/vue3` → `inertia-vue`).
  `resolveFirstPartyBoostPath()` is a *fallback* letting a first-party package ship its own
  `resources/boost/skill/` — not the primary mechanism. `laravel-task` Step 0 therefore reads the live list via
  `php artisan boost:list-skills` (signature confirmed in `src/Console/ListSkillCommand.php`) rather than hardcoding,
  and Step 0's `composer.lock` probes use Boost's own registry names so the probe list and the routing table agree.
- **The two skill directories don't overlap.** `npx skills` writes `.agents/skills/<name>` and symlinks into
  `.claude/skills/`; Boost writes straight into `.claude/skills/<name>` via `SkillWriter`. Reading only
  `.agents/skills/` misses every Boost skill. Step 0 reads both sources.
- **`grill-me` is not callable by the model.** Its entire body is "Call the Skill tool with `grilling`", and it
  carries `disable-model-invocation: true` — it's the user's trigger phrase, not an entry point. `laravel-task`
  invokes `grilling` directly, twice (once on the requirements, once on the written plan). Same for `implement`,
  which is also `disable-model-invocation: true` and therefore a deliberate reach, not an automatic one.
- **`git stash` does not stash untracked files, which made the first version of Step 4 draw the wrong conclusion.**
  The "was this failing before I touched it?" baseline ran `git stash && composer test`, and the brand-new Pest test
  written moments earlier in Step 3 is untracked — so it survived the stash and ran as part of the supposedly clean
  baseline. A failing new test then reads as a pre-existing failure and gets waved through, exactly backwards. Fixed
  with `git stash push -u`, verified in both directions. Any future baseline-comparison step in any skill here needs
  `-u` for the same reason.
- **`composer test` is `laravel-lint-setup`'s script, not Laravel's.** A stock project has no such script; a
  `laravel-lint-setup` project runs `config:clear` → `refactor:check` → `lint:check` → `types:check` → `artisan test`.
  Step 4 checks for it and degrades to the narrowest available test command rather than assuming.

**`laravel-audit`** — like `laravel-task` it has no single upstream, but unlike `laravel-task` it depends on the
olgunozoktas skills' `bin/` **tooling** rather than just their prose, and that tooling has sharp edges. Four things
established by running the scanners, all of which the skill routes around and none of which is documented upstream:

- **`scan-performance.php` searches only `<root>/app`, `<root>/resources/views` and `<root>/packages`.** Given a
  feature subdirectory it looks for `app/` *inside it*, finds nothing, prints `0 finding(s)` and exits 0 — which
  reads as a clean bill of health. Verified in both directions on the same code: 3 findings from the project root, 0
  from `app/Livewire`. The skill always passes `$ROOT` and filters afterward. **This is the single most dangerous
  behaviour in the toolchain**, because its failure mode is a false clean rather than an error.
- **`scan.php` (security) does *not* share that flaw.** It falls back to the given directory when no conventional
  path exists, with a source comment saying why ("rather than reporting a clean result for a tree nobody looked at").
  The two sibling scanners genuinely differ; don't assume symmetry between them on a future check.
- **`review-security.py` has a verified false negative on its own headline rule.** `looks_like_php()` treats `->`
  anywhere in the matched fragment as a sign the value is PHP rather than an Alpine expression, and it tests the
  whole region *including* the Blade interpolation. So `x-data="{ name: '{{ $user->name }}' }"` is silently
  suppressed while `{{ $displayName }}` in the identical position is flagged — i.e. the most common Blade
  interpolation shape in existence defeats the rule. Confirmed by calling `scan()` on both directly. The suppression
  is there for a real reason (Blade's `:prop="$var"` vs Alpine's `:attr`, the false-positive class fixed in 1.3.1);
  it is simply too broad. Its own self-test cannot catch this — all six fixtures use bare `{{ $var }}`.
  **`laravel-audit` compensates with an explicit grep and says so in its report boundary.** Worth reporting upstream;
  re-test it after any version bump before removing the workaround.
- **Exit code is the finding count, not a status.** Non-zero is a normal result. Don't let a future edit treat it as
  a tool failure.

All four scanners' self-tests pass from the installed `.agents/skills/<skill>/bin/` location (24/24, 13/13, 26/26,
30/30), and all three were run end-to-end against a scratch project from those installed paths.

Two more constraints the skill is built around:

- **`code-review` (mattpocock) is diff-scoped.** It reviews `git diff <fixed-point>...HEAD` and asks the user for a
  fixed point, so it does not apply to auditing existing code with no meaningful diff. `laravel-audit` invokes it
  directly when the scope *is* a branch or PR, and otherwise borrows the transferable part — its 12-smell Fowler
  baseline, its "repo standard overrides the baseline" rule, and its refusal to merge separate axes into one ranking
  (which is why the audit reports three fields and never averages them).
- **`infer-conventions` writes.** Boost's skill records what it learns as durable rules under `.ai/rules/` — its own
  words, "record what you learn as durable, path-scoped rules". That makes it unusable in a read-only skill, but its
  *output* is the best available answer to "is this idiomatic for this app", so `laravel-audit` reads `.ai/rules/`
  and never runs the skill. Same shape of exclusion for `ce-compound` (writes `docs/solutions/`) and
  `improve-codebase-architecture` (grills the user into implementing one of its findings).

**`laravel-lint-setup`** — checked in this order:

| Source | What to read | What in the skill depends on it |
| --- | --- | --- |
| [laravel/livewire-starter-kit](https://github.com/laravel/livewire-starter-kit) | `pint.json`, `phpstan.neon`, `composer.json`'s `scripts`, `.editorconfig` | Steps 3, 5, 7, 9 — this is the parity target, everything else is secondary |
| [Pint docs](https://laravel.com/framework/docs/pint) | The *Custom Rules* section, esp. `Pint/laravel_blade`; the CLI options | Step 3's opt-in, Step 8's troubleshooting |
| [laravel/pint](https://github.com/laravel/pint) source | `app/Fixers/LaravelBlade/Fixer.php::prettierDependencies()`, `app/Actions/EnsurePrettierIsConfigured.php`, `app/Enums/NodePackageManager.php` | Step 4's exact npm constraints and package-manager detection |
| [larastan/larastan](https://github.com/larastan/larastan) | README install + config section | Step 1's package name, Step 5's `includes:` path and level |
| [rectorphp/rector](https://github.com/rectorphp/rector) | README, `templates/rector.php.dist`, `RectorConfigBuilder` | Step 6's config shape and prepared-set names |
| [driftingly/rector-laravel](https://github.com/driftingly/rector-laravel) | README's *Automate Laravel Upgrades* section | Step 6's `withComposerBased(laravel: true)` |
| [pestphp/pest](https://github.com/pestphp/pest) | `composer.json`'s own `require.php` (the version floor for a major), GitHub release notes | Step 2's install/upgrade logic, `references/version-check.md`'s Pest branch |
| [laravel/installer](https://github.com/laravel/installer) source | `src/NewCommand.php::installPest()` | Step 2's entire migration sequence — traced from here, not invented |

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
  `{"tool":"pint","result":…,"files":[…]}` object per line. Verified live, and documented in the skill's Step 8 so a
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

**Pest is a stronger stance than Rector or the Blade rule: it's not an addition on top of the project's choice, it
*replaces* the project's choice.** `laravel-lint-setup` migrates an existing PHPUnit project to Pest as part of a
normal run — this is deliberate, not scope creep, but it is a real divergence from the starter kit, not parity with
it: `laravel/livewire-starter-kit`'s own `composer.json` requires `phpunit/phpunit`, confirmed directly — it is on
PHPUnit, not Pest (its `"pestphp/pest-plugin": true` entry is only an `allow-plugins` stub, the same harmless
leftover found in a bare `laravel new` skeleton, not an actual dependency). Don't cite the starter kit as
justification for this one; Pest here is purely Lars's own preference, stated explicitly by him, not inferred from
any upstream.

Four things learned by testing the actual migration end to end, none of them documented anywhere and all load-bearing:

- **`nunomaduro/collision` — a default Laravel dependency, not a Pest package — is what makes `php artisan test`
  route to Pest.** Read from source: its `TestCommand::usingPest()` is `function_exists('\Pest\version')`, true the
  moment `pestphp/pest` is required via Composer. `pestphp/pest-plugin-laravel` is not what does this — it's the
  Laravel-specific helper package `laravel/installer` pairs with core Pest by convention, confirmed from
  `NewCommand.php`, not required for `artisan test` detection. Corrected mid-session after initially attributing
  this to the wrong package; verify the mechanism, not just that a plausible-sounding package exists for it.
- **`pest --drift` fails under `laravel/pao`'s agent-output wrapper**, with an error (`InvalidOption: The [--drift]
  argument only accepts the directory to convert as argument`) that has nothing to do with the actual cause.
  `PAO_DISABLE=1` fixes it — verified: identical command, only the env var differs, one throws, one converts. This
  is the same package that produces Pint's/PHPStan's JSON output elsewhere in this skill, so its interference here
  isn't a coincidence — it's worth treating as a recurring suspect whenever a CLI tool this skill drives behaves
  inexplicably under non-interactive execution.
- **Pest ships no `UPGRADE.md`, `UPGRADING.md`, or `CHANGELOG.md`** (all confirmed 404 on `pestphp/pest`). A major
  bump's only guidance is its GitHub release notes, which for v5.0.0 is one line pointing at a `pestphp.com`
  announcement page rather than a structured list.
- **Pest 5 raised its PHP floor to `^8.4`** — a real gate on a major bump, not just an upgrade-guide curiosity. The
  skill lets `composer require`'s own resolution failure report this rather than trying to pre-parse it, and does
  not bump the project's PHP version to force it through.

The drift check for all of this is automated in `internal/refresh-skills-catalog.md` Step 1. Prefer running that over
checking these by hand.

## Keeping this repo's own docs in sync

`internal/refresh-skills-catalog.md` is this repo's own maintenance routine: it checks for drift against Boost and mattpocock/skills, and regenerates the README's "Available skills" section — the catalog of what `laravel-init` actually pulls into a project, sourced from Boost's and mattpocock/skills' current skill sets, not from this repo's own `skills/` folder. It's not a real skill by design (see *The goal* above), so follow it by reading the file directly rather than by name.

Run it after any bigger change to this repo — what `laravel-init` installs from Boost or mattpocock/skills changes, or anything else that could make the README or `laravel-init` drift from reality. When in doubt after a change, run it.

## Things to avoid

- Don't duplicate content that Boost or mattpocock/skills already own — link to / invoke them instead. The "Available skills" catalog is the one sanctioned exception: name + a short gist per skill, for discoverability, not their instructions. If a table entry grows past one line, it's become duplication.
- Don't bake in machine-specific paths, personal API keys, or anything not safe to commit publicly (this repo may end up public/shared).
- Don't add a `depends_on`-style mechanism expecting the skills CLI to auto-install sibling skills — that's not a shipped feature as of writing. `--all` already installs everything in this repo in one command; that's sufficient.
