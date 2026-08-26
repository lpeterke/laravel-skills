---
name: laravel-lint-setup
description: Set up or verify a Laravel project's linting, formatting, static analysis and testing framework so it matches the official laravel/livewire-starter-kit — Laravel Pint (with Blade formatting opted in via Pint/laravel_blade), Larastan/PHPStan, Rector with the driftingly/rector-laravel extension, and Pest PHP (the strongly preferred testing framework — installs it if missing, migrates existing PHPUnit-style tests to Pest syntax, upgrades an outdated Pest including major versions) — wires the composer lint/lint:check/refactor/refactor:check/types:check/test/ci:check scripts, then runs the whole suite green. Use whenever the user says "set up linting", "lint setup", "add pint", "add larastan", "add phpstan", "add rector", "set up rector", "add pest", "install pest", "migrate to pest", "convert phpunit to pest", "set up static analysis", "automated refactoring", "format my blade files", "make composer test work", asks why `composer lint` or `composer test` is missing or failing, or when opening a Laravel project whose pint.json / phpstan.neon looks missing, partial or out of date, or that's still on PHPUnit. Standalone — nothing else needs to have run first. Safe to run repeatedly — it detects what's already correct and only fixes the gaps.
---

# Laravel Lint Setup

Brings a Laravel (or Statamic) project's code-style, static-analysis and testing-framework setup to parity with the
official [laravel/livewire-starter-kit](https://github.com/laravel/livewire-starter-kit), plus two deliberate
additions the starter kit doesn't ship: Blade formatting, opted in through Pint's `Pint/laravel_blade` rule, and
**Pest as the strongly preferred testing framework** — this repo's standing preference, not a per-project toss-up.
A project still on PHPUnit isn't left alone; it gets migrated.

The end state, every time:

| Piece | Target |
| --- | --- |
| `laravel/pint` (dev) | installed, recent enough to know `--blade` |
| `larastan/larastan` (dev) | installed, 3.x |
| `rector/rector` + `driftingly/rector-laravel` (dev) | installed |
| `pestphp/pest` + `pestphp/pest-plugin-laravel` (dev) | installed, latest |
| `pint.json` | `{"preset": "laravel", "rules": {"Pint/laravel_blade": true}}` |
| prettier deps (dev, npm) | `prettier`, `prettier-plugin-blade`, `prettier-plugin-tailwindcss` |
| `phpstan.neon` | larastan + carbon extensions, starter-kit paths, level 7 |
| `rector.php` | same paths, `withComposerBased(laravel: true)`, deadCode + codeQuality |
| `tests/` | Pest-initialized (`tests/Pest.php` present); any PHPUnit-style test classes migrated to Pest syntax |
| composer scripts | `lint`, `lint:check`, `refactor`, `refactor:check`, `types:check`, `test`, `ci:check` |
| verification | `composer test` exits 0 |

**Every step below is a detect-then-act check.** Read the current state first, change only what's actually wrong, and
say so. A second run should report "already correct" for each piece and spend its time on the final `composer test`,
not on rewriting files that were already right.

## Step 0 — Confirm the project, and check the working tree

`composer.json` must exist in the project root and require `laravel/framework`. If not, stop — this is Laravel-specific.

Then look at the tree, because two later steps can rewrite a lot of files at once:

```bash
git status --porcelain | head -20
```

Two events in this skill produce a large, first-time diff: Step 8 enabling Blade formatting reformats **every**
`.blade.php` file in the project, and Step 2 migrating existing PHPUnit-style tests to Pest (via `pest --drift`)
rewrites every test file it touches, with no dry-run available. Both are fine and reviewable on a clean tree, and
genuinely awkward landing mixed into work in progress. So:

- **Clean tree** → proceed without asking for either. Each is one reviewable diff, and `git checkout .` undoes it.
- **Dirty tree, and the rewrite hasn't happened yet** (`Pint/laravel_blade` not yet in `pint.json`, or PHPUnit-style
  test classes still present) → say plainly what the first run will rewrite on top of their uncommitted changes, and
  ask whether to continue now or after they commit. This is the one place in this skill worth blocking on.
- **Dirty tree, but that rewrite already happened** (i.e. a re-run) → proceed. The rewrite is already applied; the
  next run only touches what drifted since.

## Step 1 — Composer packages

```bash
grep -q '"laravel/pint"' composer.json && echo pint-present || echo pint-missing
grep -q '"larastan/larastan"' composer.json && echo larastan-present || echo larastan-missing
grep -q '"rector/rector"' composer.json && echo rector-present || echo rector-missing
grep -q '"driftingly/rector-laravel"' composer.json && echo rector-laravel-present || echo rector-laravel-missing
grep -q '"nunomaduro/larastan"' composer.json && echo LEGACY-LARASTAN
```

- **Missing** → install only the ones missing:
  ```bash
  composer require --dev laravel/pint larastan/larastan rector/rector driftingly/rector-laravel
  ```
  Deliberately no explicit versions: composer resolves the current releases and writes the caret constraints itself,
  so this skill never carries a version number that goes stale. The starter kit's floors are `^1.27` (Pint) and `^3.9`
  (Larastan); whatever composer picks today is at or above both.
- **Present** → before touching any config file, check whether it's actually current — including major
  version bumps. Full procedure (a shell function portable across bash 3.2 and zsh, the upgrade-guide check
  for major bumps, per-package notes on what to read in that guide) is in
  `references/version-check.md` — follow it now, then come back here.
- **`driftingly/rector-laravel` without `rector/rector`** can't happen — it requires Rector — but the reverse is
  common: a project that adopted Rector without the Laravel rules. Add the extension in that case; it's what teaches
  Rector about facades, Eloquent, collections and the Laravel version sets.
- **Rector needs `require.php` in `composer.json`.** Verified: with no PHP constraint declared, a `rector.php` using
  `withPhpSets()` dies with *"We could not find local composer.json to determine your PHP version"*. Every real
  Laravel skeleton declares one, but check — if it's absent, add it to match the project's actual PHP rather than
  working around it in `rector.php`.
- **`nunomaduro/larastan` (the pre-3.0 name)** → migrate, it's the same project renamed:
  ```bash
  composer remove --dev nunomaduro/larastan
  composer require --dev larastan/larastan
  ```
  Then fix any `includes:` line in the phpstan config still pointing at `vendor/nunomaduro/larastan/...` (Step 5
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

## Step 2 — Pest: install, initialize, and migrate remaining PHPUnit tests

**This is a standing preference, not a per-project question.** Pest is strongly preferred over PHPUnit for every
project this skill touches — a project that's on PHPUnit isn't left as-is, it gets migrated as part of this run.

```bash
grep -q '"pestphp/pest"' composer.json && echo pest-present || echo pest-missing
grep -rl 'extends TestCase' tests --include='*Test.php' 2>/dev/null | grep -v '/TestCase\.php$'
```

The second command finds PHPUnit-style test classes still in the codebase (a heuristic, like the competing-tooling
check in Step 1 — it won't catch every possible PHPUnit idiom, but it catches the common case). Its output matters
whether or not Pest is already installed: a project can be "on Pest" for new work while still carrying old
PHPUnit-class tests nobody's converted.

- **`pest-missing`** → install and initialize, then migrate whatever the second command found. The full sequence —
  traced from `laravel/installer`'s own `NewCommand::installPest()`, not invented, because that's the exact recipe
  Laravel uses when a fresh project picks Pest — plus two non-obvious failure modes verified by running it end to
  end, is in `references/pest-migration.md`. Follow it now, then come back here.
- **`pest-present`, no PHPUnit-style classes found** → nothing to do.
- **`pest-present`, PHPUnit-style classes still found** → Pest is installed but the migration was never finished
  (or the project adopted Pest for new tests only). Jump straight to `references/pest-migration.md`'s Drift section
  — skip the package-install part, the packages are already there.
- **Already installed, but outdated** → covered by `references/version-check.md`, the same function-based check used
  for the other four packages. `pestphp/pest` and `pestphp/pest-plugin-laravel` are two more calls into it.

## Step 3 — `pint.json`

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

## Step 4 — Prettier dependencies for the Blade rule

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

If there's no Node, **do not enable the blade rule** — back out the `Pint/laravel_blade` line from Step 3, finish the
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

## Step 5 — `phpstan.neon`

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
  and say so explicitly in the summary — that's a real tightening, and Step 7 handles the fallout.
- **Existing `ignoreErrors`, `excludePaths`, `baseline` include, `parameters` of any other kind** → preserve all of it.
- **Existing `phpstan-baseline.neon`** → keep its `includes:` entry. Don't regenerate it here; Step 7 decides that.

## Step 6 — `rector.php`

Rector is the one tool here that rewrites **application logic**, not just whitespace or type annotations. Configure it
conservatively and let it be ratcheted up later; a lint setup is not the moment to refactor someone's codebase.

Target for a project with no `rector.php`:

```php
<?php

declare(strict_types=1);

use Rector\Config\RectorConfig;

return RectorConfig::configure()
    ->withPaths([
        __DIR__.'/app',
        __DIR__.'/bootstrap/app.php',
        __DIR__.'/config',
        __DIR__.'/database',
        __DIR__.'/routes',
    ])
    ->withComposerBased(laravel: true)
    ->withPreparedSets(
        deadCode: true,
        codeQuality: true,
    );
```

Why each line:

- **Same paths as `phpstan.neon`.** Keeping the two tools' scope identical means one mental model for "what's
  analysed". Drop any path this skeleton doesn't have, same as Step 5. `tests/` is deliberately excluded to match
  PHPStan; adding it later is a one-line change and a reasonable thing to want.
- **`withComposerBased(laravel: true)`** is the current recommended way to get the Laravel rules — it reads
  `composer.json` and applies the version sets for Laravel (and Faker, Livewire, Cashier when installed) up to the
  installed version. Don't use `LaravelSetList` / `LaravelLevelSetList` constants: rector-laravel's own README marks
  them **deprecated** in favour of this, and they need hand-editing on every Laravel upgrade.
- **`deadCode` + `codeQuality`, and not `withPhpSets()` or `typeDeclarations`.** Those two prepared sets are the
  well-behaved ones. `withPhpSets()` applies every PHP-version migration at once, and `typeDeclarations` adds inferred
  types to signatures across the codebase — both produce large diffs and belong in a deliberate upgrade, not here.

### `codeQuality` adds `declare(strict_types=1)` — say so

`codeQuality` includes `SafeDeclareStrictTypesRector`, which prepends `declare(strict_types=1)` to files. Confirmed by
isolating the sets: `deadCode` alone doesn't do it, `codeQuality` alone does.

It's guarded — the rule calls `isFileStrictTypeSafe()` and skips any file where it detects scalar coercion — but that
check is *static and per-file*, so it can't see a caller passing `"5"` to an `int` parameter at runtime. Under strict
types that becomes a `TypeError` instead of a silent cast. On a well-typed codebase this is a non-event; on an older
one it's a real behaviour change.

So: **enable it, and report it.** Name it in the Step 10 summary along with how many files got the declare. If the
project is legacy enough that this looks risky, skip that one rule rather than dropping `codeQuality` wholesale:

```php
use Rector\TypeDeclaration\Rector\StmtsAwareInterface\SafeDeclareStrictTypesRector;

    ->withSkip([
        SafeDeclareStrictTypesRector::class,
    ])
```

Merging into an existing `rector.php`:

- **Already has `withComposerBased(laravel: true)` or a `LaravelSetList` entry** → the Laravel rules are wired up.
  Leave the config alone; only mention the deprecation if it's using the old constants.
- **Rector configured but no Laravel rules** → add `->withComposerBased(laravel: true)` and nothing else. Don't
  restructure their sets, skips or paths.
- **Uses the old `static function (RectorConfig $rectorConfig)` callback style** (Rector 0.x) → it still works. Note
  it as outdated and leave it; converting a config to the fluent builder is a migration, not a setup step.
- **Any existing `withSkip`, `withRules`, `withConfiguredRule`, custom paths** → preserve all of it verbatim.

## Step 7 — Composer scripts

The starter kit's five, plus the two Rector ones:

```json
"lint": [
    "pint --parallel"
],
"lint:check": [
    "pint --parallel --test"
],
"refactor": [
    "rector process"
],
"refactor:check": [
    "rector process --dry-run"
],
"types:check": [
    "phpstan analyse"
],
"test": [
    "@php artisan config:clear --ansi",
    "@refactor:check",
    "@lint:check",
    "@types:check",
    "@php artisan test"
],
"ci:check": [
    "Composer\\Config::disableProcessTimeout",
    "@test"
]
```

`composer test` is the single command that runs the whole suite — refactoring check, style check, static analysis,
then the test suite — and `ci:check` is the same thing with composer's process timeout lifted, for CI.

Two things about this ordering, both load-bearing:

- **`refactor:check` runs before `lint:check`, not after.** Rector's output is not Pint-formatted — verified: after
  `rector process`, `pint --test` exits 1 because Rector doesn't add the blank line before `return` that the Laravel
  preset wants. So the fix order is always `composer refactor` **then** `composer lint`, and putting the check in the
  same order means the first failure you see is the first one to fix. Reversed, you'd format, then refactor, then have
  to format again — three cycles instead of two.
- **`refactor:check` is a dry run and never writes.** `rector process --dry-run` prints the diff it *would* apply and
  exits **2** when anything is pending, **0** when clean — verified both ways. That non-zero exit is what makes it a
  usable gate; `composer test` fails loudly rather than quietly refactoring the codebase mid-test-run.

Merging notes:

- **The default Laravel skeleton already defines `test`** as `["@php artisan config:clear --ansi", "@php artisan test"]`.
  Expand it in place to the four-entry version above rather than adding a differently-named script.
- **`@php artisan test` needs no branching on PHPUnit vs Pest.** Verified from source:
  `nunomaduro/collision` — a default dependency of every Laravel app, not a Pest package — provides the `test`
  artisan command, and its `usingPest()` check is `function_exists('\Pest\version')`. That's true the moment
  `pestphp/pest` is required via Composer, whether or not `pestphp/pest-plugin-laravel` is also installed. So
  `php artisan test` routes to Pest with zero config either way, same command, same script. Don't add a check for
  which one is installed; there's nothing to branch on.
- **A project that already has its own `test`** doing something else (e.g. `pest --parallel` invoked directly rather
  than through `artisan test`, or a coverage run) → keep the command it runs and insert `@lint:check` and
  `@types:check` ahead of it. Don't replace someone's test invocation.
- **An existing `rector` or `refactor` script** → normalise to `refactor` / `refactor:check`, keeping whatever flags
  it already passed. If it ran `rector process` with no dry-run counterpart, add one; a checker that mutates the
  working tree is useless in `test`.
- **Existing `lint`/`format`/`analyse` scripts** pointing at Pint or PHPStan → normalise to the names above so
  `test` can reference them, and keep an old name as an alias (`"format": ["@lint"]`) if it's likely in muscle memory
  or a Makefile. Say what you renamed.
- Edit `composer.json` by parsing the JSON, and keep the file's existing indentation (4 spaces in every Laravel
  skeleton).

## Step 8 — Run it green

Order matters on a first run: the `:check` scripts only *report*, so fix first, then verify. And Rector goes before
Pint, for the reason in Step 7 — Rector's output needs formatting afterwards, so doing it the other way wastes a pass.

```bash
composer refactor      # Rector rewrites code in place
composer lint          # Pint then formats PHP + Blade, including whatever Rector just wrote
composer test          # config:clear → refactor:check → lint:check → types:check → artisan test
```

**Review the Rector diff before moving on.** This is the one part of the setup that changes program logic rather than
layout — dead code removed, `declare(strict_types=1)` added, Laravel-version rules applied. `git diff` it, and if
anything looks wrong, narrow `rector.php` (skip the rule, or drop a path) rather than accepting a rewrite nobody
reviewed.

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
- `require Node.js to be installed` / `prettier dependencies … to be installed` → Step 4 didn't take. Re-check
  `node -v` and the three devDependencies.
- `do not satisfy the versions required` → install the exact versions from Pint's message.

**`refactor:check` fails after `composer refactor` already ran.** A second dry run should exit 0; if it exits 2 again,
Rector didn't converge.
- **Rector and Pint fighting** — Rector reformats something Pint then undoes, or the reverse, so each run dirties the
  other. Identify the rule from the diff and `withSkip([...])` it. Formatting is Pint's job; a Rector rule that argues
  with the Laravel preset should lose.
- **Genuinely non-idempotent rule** (rare) — Rector applies a change, then wants to change it again. Skip that rule
  and note it.
- `We could not find local "composer.json" to determine your PHP version` → the project has no `require.php`
  constraint. Add one (Step 1) rather than hardcoding a version in `rector.php`.
- **Enormous diff on the first run.** Not a failure, but stop and check it's wanted before committing. If it's too
  much to review, cut scope in `rector.php` — drop to `deadCode: true` only, or narrow `withPaths()` to `app/` — get
  the pipeline green, and widen later. A reviewed small diff beats an unreviewed large one.
- **Very slow.** Rector caches between runs; the first pass on a large codebase is the slow one. If it's a problem in
  CI, `rector process --dry-run --no-progress-bar` trims output noise, and `ci:check` already lifts composer's
  process timeout.

**`types:check` fails.**
- `Allowed memory size … exhausted` → change the script to `phpstan analyse --memory-limit=2G`.
- `Internal error` or a flood of unknown-class errors → `composer dump-autoload`, then retry. Stale autoload maps are
  the usual cause.
- `Path … does not exist` → a path in Step 5's list that this skeleton doesn't have. Remove it.
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
- **Failures this skill caused are in scope, from three sources.** Blade formatting rewrites whitespace/attribute
  layout, breaking assertions on exact HTML strings or Dusk selectors — fix the assertion, the new formatting is
  correct. Rector changed logic, so a newly failing test may be a genuine regression — find the rule from `git diff`
  and `withSkip([...])` it, don't paper over it. Pest's `--drift` conversion (Step 2) is the third: a converted test
  failing where the PHPUnit original passed is a Drift miss on that file — data providers, mocks, and custom
  assertion classes are the constructs worth checking by hand.
- Environment failures — no `.env`, no `APP_KEY`, missing `database/database.sqlite` — aren't lint problems either,
  but they're cheap to clear and block verification, so just do it:
  ```bash
  test -f .env || cp .env.example .env
  php artisan key:generate
  ```

Re-run `composer test` after each fix until it's green.

## Step 9 — `.editorconfig` (usually a no-op)

The starter kit ships the standard Laravel `.editorconfig` — LF, UTF-8, 4-space indent, final newline, trimmed
trailing whitespace, 2-space YAML. Every Laravel skeleton has the same file, so this check normally passes untouched.
If it's missing, write it; if it exists, leave it alone unless it contradicts Pint (e.g. `indent_size = 2` for PHP,
which would have the editor and Pint undoing each other on every save) — flag that conflict rather than silently
rewriting a file the user chose.

## Step 10 — Report

Say what the state is and what you changed, per piece. On a repeat run most lines are "already correct", and that's
the useful signal:

```
✅ Packages: laravel/pint ^1.30, rector/rector ^2.6, driftingly/rector-laravel ^2.5 already current
⚠️  larastan/larastan: v2.9 → v3.10 (major upgrade available) → composer.json constraint updated. Per
   larastan's UPGRADE.md, 3.0 stopped inferring Eloquent relation generics from method bodies — expect new
   errors at the same level; see types:check below for how they were handled.
⚠️  Pest: not installed (project was on PHPUnit only) → installed pestphp/pest ^5.1 + pest-plugin-laravel
   ^5.0, ran pest --init, migrated 3 PHPUnit-style test files to Pest syntax via --drift. Review
   git diff tests/ — Drift left one unused import that composer lint below already cleaned up.
✅ pint.json: already had preset laravel; added Pint/laravel_blade
✅ Prettier deps: prettier, prettier-plugin-blade, prettier-plugin-tailwindcss installed via npm
✅ phpstan.neon: created (larastan + carbon extensions, 5 paths, level 7)
✅ rector.php: created (same 5 paths, withComposerBased(laravel: true), deadCode + codeQuality)
✅ Composer scripts: lint, lint:check, refactor, refactor:check, types:check added; test expanded; ci:check added
⚠️  composer refactor: changed 46 files — dead code removed, declare(strict_types=1) added to 31 files.
   Review the diff; skip SafeDeclareStrictTypesRector in rector.php if strict types aren't wanted yet.
✅ composer lint: reformatted 214 files (203 Blade — first run with the rule on)
⚠️  types:check: 137 errors at level 7 → baselined to phpstan-baseline.neon. Burn it down over time.
✅ composer test: passing

Run the full suite any time with: composer test
```

Always close with the `composer test` line — that one command is the deliverable.

**Never report the Rector run as a bare file count.** It is the only step that changed program logic, so say what
kinds of change it made and flag `declare(strict_types=1)` explicitly whenever it was added. A user skimming the
summary should not first learn about strict types from a production `TypeError`.

## Edge cases

- **No Composer / no `vendor/`** → run `composer install` first; every check here reads `vendor/`.
- **No Node** → Blade formatting off, everything else on. Covered in Step 4.
- **Statamic** → nothing special. Antlers is untouched by both tools; Blade files (if any) format normally.
- **Pint pinned by another package** (a shared config package requiring an old Pint) → don't force the upgrade; report
  the conflict and skip the Blade rule if `--blade` isn't supported.
- **Rector already run by CI or a git hook** → check `.github/workflows/` and `.git/hooks/` before adding the scripts,
  so the project doesn't end up refactoring twice under two different configs.
- **Statamic and Rector**: only `withPaths()` entries are touched, so addons and content are untouched. Statamic apps
  often keep code in `app/` alone, which the default paths already cover.
- **CI doesn't run these checks.** If `.github/workflows/` exists, grep it for `ci:check`/`pint`/`phpstan`/`rector`. If nothing
  runs them, mention that CI won't catch style or type regressions and that the starter kit's workflow simply runs
  `composer ci:check`. Don't write a workflow file unasked — that's a CI change, not a lint config change.
- **Re-run on an already-correct project** → every check reports "already correct" and the run ends at Step 7 with a
  single `composer test`. That's the intended second-run behaviour, not a wasted pass.
