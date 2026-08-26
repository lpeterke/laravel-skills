# Installing Pest and migrating existing PHPUnit tests

The recipe below is `laravel/installer`'s own, read directly from `src/NewCommand.php::installPest()` — this is
exactly what happens when someone runs `laravel new` and picks Pest, not a sequence assembled from general knowledge
of the ecosystem. Two steps in it (`--drift`'s missing dry-run, and a specific env var it needs) were verified by
running the whole thing end to end in a scratch project, because neither is documented anywhere.

## 1 — Swap the packages

```bash
grep -q '"phpunit/phpunit"' composer.json && composer remove phpunit/phpunit --dev --no-update
composer require pestphp/pest pestphp/pest-plugin-laravel --no-update --dev
composer update pestphp/pest pestphp/pest-plugin-laravel
```

- **Guard the `remove` on the package actually being a *direct* dependency.** Most Laravel skeletons list
  `phpunit/phpunit` directly in `require-dev`; if it's already absent (a project that added Pest by hand before),
  `composer remove` on something not required errors — the `grep` above is what the installer's own flow doesn't
  need (it always starts from a fresh skeleton) but this skill does, since it runs on existing projects.
- **Removing `phpunit/phpunit` doesn't remove PHPUnit.** Pest itself depends on `phpunit/phpunit` internally (Pest
  is built on top of it), so it stays installed transitively — confirmed by checking Pest's own `composer.json`.
  This step only removes the now-redundant *direct* requirement, matching what the installer does.
- **No explicit version.** Same reasoning as the other four packages: composer resolves current and writes the
  constraint itself.
- **`phpunit.xml` needs no changes.** Verified: `pest --init` (next step) reports `phpunit.xml .. File already
  exists.` and leaves it untouched. Pest reads the existing config as-is.

## 2 — Initialize

```bash
./vendor/bin/pest --init --no-interaction < /dev/null
```

- **Prints a "star us on GitHub?" prompt, but doesn't block on it.** Verified with stdin explicitly closed
  (`< /dev/null`): it renders the prompt text, resolves to its default (no), and exits 0. Nothing to route around,
  unlike Pint's Prettier confirm — just don't be alarmed by prompt-shaped text in the output.
- **Idempotent.** Re-running it on an already-initialized project reports every file as "already exists" and still
  exits 0. Safe to run unconditionally rather than gating it on whether `tests/Pest.php` exists yet.
- **Existing PHPUnit-class tests keep running after this, untouched.** Verified: a project can have both
  `tests/Pest.php` and old `extends TestCase` class-based tests, and `php artisan test` runs all of them together
  with no config split. `pest --init` alone doesn't migrate anything — that's the next section, and it's the reason
  this skill doesn't stop here.

## 3 — Migrate whatever PHPUnit-style tests Step 2's grep found

Skip this section entirely if that grep found nothing — a fresh project with no tests, or one already fully on Pest,
has nothing to convert.

```bash
composer require pestphp/pest-plugin-drift --dev --no-interaction
PAO_DISABLE=1 ./vendor/bin/pest --drift tests
composer remove pestphp/pest-plugin-drift --dev --no-interaction
```

Three things about this, none of them documented anywhere and all confirmed by running it:

- **`PAO_DISABLE=1` is required, not optional.** Without it, `pest --drift tests` fails immediately with
  `Pest\Exceptions\InvalidOption: The [--drift] argument only accepts the directory to convert as argument` — even
  though exactly one directory argument was given, which is precisely what the plugin's own source says should
  work. The cause is `laravel/pao`, the same agent-output package behind the compact JSON this skill sees from Pint
  and PHPStan: when it detects an agent-driven, non-interactive shell (which this always is), something in its
  output-capturing hooks corrupts Drift's argument parsing before Drift ever sees a clean argv. Setting
  `PAO_DISABLE=1` for this one command sidesteps it entirely — verified: identical command, only the env var
  differs, one throws, one converts cleanly. Don't spend time debugging the "InvalidOption" message itself if you
  hit it; it has nothing to do with the actual argument.
- **No dry-run exists.** Confirmed from `pestphp/pest-plugin-drift`'s source — the plugin has exactly one mode, and
  it writes. This is why Step 0 gates on a clean tree before this section runs at all; there's no `--dry-run` to
  fall back on the way Pint and Rector have one.
- **`pest-plugin-drift` is a one-time tool, not a dependency.** `composer remove` it immediately after — the
  installer does the same. Leaving it installed does no active harm, but it has no purpose once the migration is
  done and would show up as a stray dev dependency in every future `composer show`.

### What the conversion actually produces

Verified on a small mixed project (a `#[Test]`-attributed method, a `test_snake_case` method, and a plain assertion
test): Drift correctly converts PHPUnit assertions to Pest's `expect()->toEqual()` / `->toBe()` style and PHPUnit
method names to `it('...', fn () => ...)` / `test('...', fn () => ...)`, dropping the `Tests\Unit`/`Tests\Feature`
namespace and the `extends TestCase` boilerplate entirely (a plain Pest file needs neither).

One rough edge to expect, not to worry about: a `use PHPUnit\Framework\Attributes\Test;` import can survive the
conversion unused, since Drift removes the attribute but not always the import backing it. **`composer lint`
(Step 8) cleans this up as a side effect** — confirmed: Pint's Laravel preset includes `no_unused_imports`, and it
strips exactly this leftover. Rector's `deadCode` set does **not** catch it (tested directly) — don't rely on Rector
for this, Pint is what's actually doing the work here.

### After converting

- **Review the diff.** Every test file Drift touched changed syntax, even though the assertions themselves are
  supposed to be equivalent. `git diff tests/` and skim it — this is the same discipline Step 8 already asks for
  after a Rector run, for the same reason: a tool rewrote logic-adjacent code unattended.
- **Run the suite before moving on**, not just at the very end in Step 8: `php artisan test`. A failure here is
  either a genuine Drift miss (rare, but data providers, mocks, and custom assertion classes are the kind of thing
  worth checking by hand if present) or a pre-existing failure Drift had nothing to do with — tell them apart the
  same way Step 8 already does for Rector-caused vs pre-existing failures.
