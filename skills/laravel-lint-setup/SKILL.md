---
name: laravel-lint-setup
description: Set up or verify a Laravel project's linting, formatting and static analysis so it matches the official laravel/livewire-starter-kit — Laravel Pint (with Blade formatting opted in via Pint/laravel_blade) plus Larastan/PHPStan — wires the composer lint/lint:check/types:check/test/ci:check scripts, then runs the whole suite green. Use whenever the user says "set up linting", "lint setup", "add pint", "add larastan", "add phpstan", "set up static analysis", "format my blade files", "make composer test work", asks why `composer lint` or `composer test` is missing or failing, or when opening a Laravel project whose pint.json / phpstan.neon looks missing, partial or out of date. Standalone — nothing else needs to have run first. Safe to run repeatedly — it detects what's already correct and only fixes the gaps.
---

# Laravel Lint Setup

Brings a Laravel (or Statamic) project's code-style and static-analysis setup to parity with the official
[laravel/livewire-starter-kit](https://github.com/laravel/livewire-starter-kit), plus one deliberate addition the
starter kit doesn't ship: Blade formatting, opted in through Pint's `Pint/laravel_blade` rule.

The end state, every time:

| Piece | Target |
| --- | --- |
| `laravel/pint` (dev) | installed, recent enough to know `--blade` |
| `larastan/larastan` (dev) | installed, 3.x |
| `pint.json` | `{"preset": "laravel", "rules": {"Pint/laravel_blade": true}}` |
| prettier deps (dev, npm) | `prettier`, `prettier-plugin-blade`, `prettier-plugin-tailwindcss` |
| `phpstan.neon` | larastan + carbon extensions, starter-kit paths, level 7 |
| composer scripts | `lint`, `lint:check`, `types:check`, `test`, `ci:check` |
| verification | `composer test` exits 0 |

**Every step below is a detect-then-act check.** Read the current state first, change only what's actually wrong, and
say so. A second run should report "already correct" for each piece and spend its time on the final `composer test`,
not on rewriting files that were already right.

## Step 0 — Confirm the project, and check the working tree

`composer.json` must exist in the project root and require `laravel/framework`. If not, stop — this is Laravel-specific.

Then look at the tree, because Step 4 can rewrite a lot of files:

```bash
git status --porcelain | head -20
```

Enabling Blade formatting for the first time reformats **every** `.blade.php` file in the project — routinely hundreds
of files in an existing codebase. That is fine and reviewable on a clean tree, and genuinely awkward if it lands mixed
into work in progress. So:

- **Clean tree** → proceed without asking. The reformat is one reviewable diff, and `git checkout .` undoes it.
- **Dirty tree, and `Pint/laravel_blade` is not yet in `pint.json`** → say plainly that the first Pint run will
  reformat every Blade file on top of their uncommitted changes, and ask whether to continue now or after they commit.
  This is the one place in this skill worth blocking on.
- **Dirty tree, blade rule already enabled** (i.e. a re-run) → proceed. The formatting is already applied; the next run
  only touches files that drifted.

## Step 1 — Composer packages

```bash
grep -q '"laravel/pint"' composer.json && echo pint-present || echo pint-missing
grep -q '"larastan/larastan"' composer.json && echo larastan-present || echo larastan-missing
grep -q '"nunomaduro/larastan"' composer.json && echo LEGACY-LARASTAN
```

- **Missing** → `composer require --dev laravel/pint larastan/larastan` (install only the ones missing). Deliberately
  no explicit version: composer resolves the current release and writes the caret constraint itself, so this skill
  never carries a version number that goes stale. The starter kit's floors are `^1.27` and `^3.9`; whatever composer
  picks today is at or above both.
- **`nunomaduro/larastan` (the pre-3.0 name)** → migrate, it's the same project renamed:
  ```bash
  composer remove --dev nunomaduro/larastan
  composer require --dev larastan/larastan
  ```
  Then fix any `includes:` line in the phpstan config still pointing at `vendor/nunomaduro/larastan/...` (Step 3
  rewrites it anyway) and any `NunoMaduro\Larastan\` references in custom rules or the config.
- **Pint present but too old to know `--blade`** — check the capability, not the version number:
  ```bash
  ./vendor/bin/pint --help 2>/dev/null | grep -q -- '--blade' && echo blade-capable || echo blade-too-old
  ```
  If `blade-too-old`, `composer require --dev laravel/pint` (no version) to move the constraint to current.

While here, note any competing style tooling and **report it — don't remove it unasked**:

```bash
ls .php-cs-fixer.php .php-cs-fixer.dist.php phpcs.xml phpcs.xml.dist duster.json 2>/dev/null
grep -Eq '"(friendsofphp/php-cs-fixer|squizlabs/php_codesniffer|tightenco/duster)"' composer.json && echo COMPETING-TOOL
```

Pint *is* PHP-CS-Fixer with a preset, so running both means two configs fighting over the same files. Duster wraps
Pint + PHPStan + PHPCS and will duplicate everything this skill sets up. Say which one you found, say it should be
retired in favour of Pint, and let the user decide — a project may have a `.php-cs-fixer.php` for a reason.

## Step 2 — `pint.json`

Target, matching the starter kit's `{"preset": "laravel"}` with the Blade rule added:

```json
{
    "preset": "laravel",
    "rules": {
        "Pint/laravel_blade": true
    }
}
```

Branch on what's there:

- **No `pint.json`** → write exactly the above.
- **Exists, `{"preset": "laravel"}` only** → add the `rules` block, leave everything else byte-identical.
- **Exists with other rules / `exclude` / `notPath` / a non-`laravel` preset** → **merge, never replace.** Add
  `Pint/laravel_blade: true` into the existing `rules` object and leave every other key alone. A deliberate `psr12`
  preset or a hard-won `exclude` list is a decision someone made; this skill's job is the Blade opt-in, not
  overwriting their style. Mention the non-standard preset in the summary as a deviation from starter-kit parity.
- **Already has `Pint/laravel_blade: true`** → nothing to do.

Do the merge by parsing the JSON, not with a regex — `php -r` or `node -e` is fine — and preserve 4-space indentation.

## Step 3 — Prettier dependencies for the Blade rule

This is the step that makes the difference between a working setup and a mystifying failure, so don't skip it.

`Pint/laravel_blade` is implemented by shelling out to Prettier with `prettier-plugin-blade` and
`prettier-plugin-tailwindcss`. Pint checks for them on every run, and when they're missing it prints a warning and
calls `confirm(default: false)` to offer installing them. Under an agent's non-interactive shell that prompt resolves
to its default — **no** — and Pint then `abort(1)`s with "…require the following prettier dependencies to be
installed". So `composer lint` fails with a prompt the user never saw. Install the dependencies up front instead.

First, Node has to exist at all:

```bash
node -v || echo NO-NODE
```

If there's no Node, **do not enable the blade rule** — back out the `Pint/laravel_blade` line from Step 2, finish the
rest of the setup (Pint on PHP files, Larastan, scripts all work fine), and report that Blade formatting is off
because Node isn't installed and can be turned on later by re-running this skill.

With Node present, install the three packages as dev dependencies, matching the constraints Pint itself requires:

```bash
npm install -D prettier@^3.8.4 prettier-plugin-blade@^3.2.2 prettier-plugin-tailwindcss@^0.8.0
```

Notes that matter:

- **Match the project's package manager.** Pint detects it from the lockfile and will complain in that manager's
  terms. `bun.lock`/`bun.lockb` → `bun add -d`, `pnpm-lock.yaml` → `pnpm add -D`, `yarn.lock` → `yarn add -D`,
  otherwise npm.
- **Already installed?** Check `package.json`'s `devDependencies` for all three at satisfying versions and skip if so.
  If a package is present but *below* Pint's constraint, Pint aborts with an explicit "do not satisfy the versions
  required" message — bump it with the same install command.
- **No `package.json`** (API-only or Statamic-without-a-frontend project) → `npm init -y` first, or let the install
  create one. Pint would otherwise write a minimal one itself.
- These are the constraints Pint's `Fixers/LaravelBlade/Fixer.php` declares today. If Pint ever raises them it says so
  precisely in its abort message — take the versions from that message rather than from this file.

Worth knowing so you don't chase phantom bugs: the rule deliberately skips Envoy files, Boost guideline templates
under `resources/boost/guidelines/`, and email views under `resources/views/emails/` and `resources/views/mail/`. Those
staying unformatted is correct behaviour, not a failure.

## Step 4 — `phpstan.neon`

The starter kit's config verbatim:

```neon
includes:
    - vendor/larastan/larastan/extension.neon
    - vendor/nesbot/carbon/extension.neon

parameters:
    paths:
        - app/
        - bootstrap/app.php
        - config/
        - database/
        - routes/

    level: 7
```

PHPStan reads `phpstan.neon`, `phpstan.neon.dist` or `phpstan.dist.neon`. Edit whichever already exists; create
`phpstan.neon` if none does.

Adjust the template before writing:

- **Only list paths that exist.** `bootstrap/app.php` is absent from some older skeletons; a missing path is a hard
  PHPStan error, not a warning.
- **Statamic projects**: keep the same paths. Antlers templates aren't PHP and Blade files aren't in `paths`, so
  nothing extra is needed.
- **Drop the Carbon include if it isn't there**: `test -f vendor/nesbot/carbon/extension.neon`.
- **Skip the larastan include entirely if `phpstan/extension-installer` is installed** — it registers extensions
  automatically and an explicit include on top of it double-registers services. Check with
  `grep -q '"phpstan/extension-installer"' composer.json`.

Merging into an existing config:

- **Existing `includes`** → add the larastan line if absent, keep every other include.
- **Existing `paths`** → union with the starter-kit list, keep any extra paths the project already analyses.
- **Existing `level`** → the target is **7**. Leave a level above 7 alone (never lower it). Raise a lower level to 7,
  and say so explicitly in the summary — that's a real tightening, and Step 6 handles the fallout.
- **Existing `ignoreErrors`, `excludePaths`, `baseline` include, `parameters` of any other kind** → preserve all of it.
- **Existing `phpstan-baseline.neon`** → keep its `includes:` entry. Don't regenerate it here; Step 6 decides that.

## Step 5 — Composer scripts

The starter kit's five, verbatim:

```json
"lint": [
    "pint --parallel"
],
"lint:check": [
    "pint --parallel --test"
],
"types:check": [
    "phpstan analyse"
],
"test": [
    "@php artisan config:clear --ansi",
    "@lint:check",
    "@types:check",
    "@php artisan test"
],
"ci:check": [
    "Composer\\Config::disableProcessTimeout",
    "@test"
]
```

`composer test` is the single command that runs the whole suite — style check, static analysis, then the test suite —
and `ci:check` is the same thing with composer's process timeout lifted, for CI.

Merging notes:

- **The default Laravel skeleton already defines `test`** as `["@php artisan config:clear --ansi", "@php artisan test"]`.
  Expand it in place to the four-entry version above rather than adding a differently-named script.
- **A project that already has its own `test`** doing something else (e.g. `pest --parallel`, or a coverage run) →
  keep the command it runs and insert `@lint:check` and `@types:check` ahead of it. Don't replace someone's test
  invocation.
- **Existing `lint`/`format`/`analyse` scripts** pointing at Pint or PHPStan → normalise to the names above so
  `test` can reference them, and keep an old name as an alias (`"format": ["@lint"]`) if it's likely in muscle memory
  or a Makefile. Say what you renamed.
- Edit `composer.json` by parsing the JSON, and keep the file's existing indentation (4 spaces in every Laravel
  skeleton).

## Step 6 — Run it green

Order matters on a first run: `lint:check` only *reports*, so fix first, then verify.

```bash
composer lint          # Pint fixes PHP + Blade in place
composer test          # config:clear → lint:check → types:check → artisan test
```

`composer test` exiting 0 is the definition of done for this skill. If it doesn't, work the failure by stage — the
output names which script failed.

One surprise to expect: **Pint detects that an agent is running it and switches to JSON output**, one object per line
like `{"tool":"pint","result":"fixed","files":[{"path":"…","fixers":["Pint/laravel_blade"]}]}` — no human-readable
table, no summary line. That's normal, and it's the better format to work from. Count fixed files from the `files`
array rather than looking for a total Pint won't print, and trust the exit code (`0` clean, `1` failed) over reading
the text.

**`lint:check` fails after `composer lint` already ran.** Pint fixed nothing on the second pass but still reports
errors, which means a file can't be formatted rather than merely being unformatted.
- `PrettierException` / a Blade file named in the error → that template has a syntax error Prettier can't parse, or a
  construct that trips the plugin. Fix the template if it's genuinely broken; if it's valid Blade the plugin chokes on,
  add that one file to `notPath` in `pint.json`, note it in the summary, and move on. Don't disable the rule wholesale.
- `require Node.js to be installed` / `prettier dependencies … to be installed` → Step 3 didn't take. Re-check
  `node -v` and the three devDependencies.
- `do not satisfy the versions required` → install the exact versions from Pint's message.

**`types:check` fails.**
- `Allowed memory size … exhausted` → change the script to `phpstan analyse --memory-limit=2G`.
- `Internal error` or a flood of unknown-class errors → `composer dump-autoload`, then retry. Stale autoload maps are
  the usual cause.
- `Path … does not exist` → a path in Step 4's list that this skeleton doesn't have. Remove it.
- **Real analysis errors.** Count them: `./vendor/bin/phpstan analyse --error-format=table | tail -5`.
  - **Roughly a dozen or fewer** → fix them. At level 7 they're almost always missing parameter/return types, a
    missing `@var` on an Eloquent relation, or a genuinely unhandled null. These are small, safe edits.
  - **More than that** → don't refactor the codebase under the banner of a lint setup. Baseline it:
    ```bash
    ./vendor/bin/phpstan analyse --generate-baseline
    ```
    Then add `- phpstan-baseline.neon` to the config's `includes:`. Level 7 now applies to all new and changed code
    while existing debt is parked in one reviewable file. **Report the baselined error count prominently** — that
    number is the debt, and it's the whole reason to mention it.
  - Never lower the level to make errors disappear. Baseline instead; a baseline shrinks over time, a lowered level
    doesn't.

**`artisan test` fails.**
- First establish whose fault it is: `git stash && composer test; git stash pop`, or just check whether the failures
  reference formatting at all. **Pre-existing application test failures are out of scope** — this skill sets up
  linting, it doesn't fix the app. Report them, confirm `lint:check` and `types:check` both pass, and say plainly that
  the suite is red for reasons that predate this run.
- **Failures the reformat caused are in scope.** Blade formatting rewrites whitespace and attribute layout, which can
  break assertions on exact HTML strings, snapshot tests, and Dusk/browser selectors keyed to markup. Fix the
  assertion (the new formatting is the intended output), not the formatting.
- Environment failures — no `.env`, no `APP_KEY`, missing `database/database.sqlite` — aren't lint problems either,
  but they're cheap to clear and block verification, so just do it:
  ```bash
  test -f .env || cp .env.example .env
  php artisan key:generate
  ```

Re-run `composer test` after each fix until it's green.

## Step 7 — `.editorconfig` (usually a no-op)

The starter kit ships the standard Laravel `.editorconfig` — LF, UTF-8, 4-space indent, final newline, trimmed
trailing whitespace, 2-space YAML. Every Laravel skeleton has the same file, so this check normally passes untouched.
If it's missing, write it; if it exists, leave it alone unless it contradicts Pint (e.g. `indent_size = 2` for PHP,
which would have the editor and Pint undoing each other on every save) — flag that conflict rather than silently
rewriting a file the user chose.

## Step 8 — Report

Say what the state is and what you changed, per piece. On a repeat run most lines are "already correct", and that's
the useful signal:

```
✅ Packages: laravel/pint ^1.30 already present, larastan/larastan ^3.10 installed (new)
✅ pint.json: already had preset laravel; added Pint/laravel_blade
✅ Prettier deps: prettier, prettier-plugin-blade, prettier-plugin-tailwindcss installed via npm
✅ phpstan.neon: created (larastan + carbon extensions, 5 paths, level 7)
✅ Composer scripts: lint, lint:check, types:check added; test expanded; ci:check added
✅ composer lint: reformatted 214 files (203 Blade — first run with the rule on)
⚠️  types:check: 137 errors at level 7 → baselined to phpstan-baseline.neon. Burn it down over time.
✅ composer test: passing

Run the full suite any time with: composer test
```

Always close with the `composer test` line — that one command is the deliverable.

## Edge cases

- **No Composer / no `vendor/`** → run `composer install` first; every check here reads `vendor/`.
- **No Node** → Blade formatting off, everything else on. Covered in Step 3.
- **Statamic** → nothing special. Antlers is untouched by both tools; Blade files (if any) format normally.
- **Pint pinned by another package** (a shared config package requiring an old Pint) → don't force the upgrade; report
  the conflict and skip the Blade rule if `--blade` isn't supported.
- **CI doesn't run these checks.** If `.github/workflows/` exists, grep it for `ci:check`/`pint`/`phpstan`. If nothing
  runs them, mention that CI won't catch style or type regressions and that the starter kit's workflow simply runs
  `composer ci:check`. Don't write a workflow file unasked — that's a CI change, not a lint config change.
- **Re-run on an already-correct project** → every check reports "already correct" and the run ends at Step 6 with a
  single `composer test`. That's the intended second-run behaviour, not a wasted pass.
