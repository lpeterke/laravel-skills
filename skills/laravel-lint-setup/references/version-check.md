# Checking installed lint packages for updates, including major bumps

A project that installed this setup a year ago and never touched it again is exactly the kind of drift this skill
should close, not just tolerate. For every package already required, compare the installed version against the
latest tag straight from the package's own repo — not Packagist, not a docs site — the same reasoning as
`laravel-init`'s Boost check: docs pages have been caught lagging actual releases before.

Don't reach for an associative array for the package→repo mapping — **macOS's default `/bin/bash` is 3.2, which
predates `declare -A` entirely** (`declare: -A: invalid option`, confirmed on this machine), and zsh's associative
array syntax is different again. A small function called once per package is the form that's actually portable across
both shells this repo has to work in:

```bash
check_lint_package_version() {
  pkg="$1"; repo="$2"
  grep -q "\"$pkg\"" composer.json || return 0

  latest=$(git ls-remote --tags --refs "https://github.com/$repo.git" 2>/dev/null \
    | awk -F/ '{print $NF}' | grep -E '^v?[0-9]+\.[0-9]+\.[0-9]+$' | sed 's/^v//' | sort -V | tail -1)
  installed=$(composer show "$pkg" 2>/dev/null | grep -m1 '^versions' \
    | grep -oE 'v?[0-9]+\.[0-9]+\.[0-9]+' | head -1 | sed 's/^v//')
  echo "$pkg: latest=$latest installed=$installed"
}

check_lint_package_version "laravel/pint" "laravel/pint"
check_lint_package_version "larastan/larastan" "larastan/larastan"
check_lint_package_version "rector/rector" "rectorphp/rector"
check_lint_package_version "driftingly/rector-laravel" "driftingly/rector-laravel"
```

If `git ls-remote` fails for a package (offline, rate-limited), don't block on it — skip that package's check, leave
its constraint alone, and say in the summary that the version check couldn't run for it.

For each package where a check succeeded, compare `latest` to `installed` by leading version number:

- **Same major, `latest` ≠ `installed`** → stays inside the existing constraint:
  ```bash
  composer update <pkg>
  ```
- **Already on `latest`** → nothing to do for that package.
- **Major behind** → this is the case worth slowing down for. `composer update` never crosses a major boundary on its
  own, so bumping means widening the constraint — which can break the project in ways a minor/patch update won't.
  Before running the bump, fetch that package's upgrade guide and read what changed, since the file that carries it
  differs per project (verified by checking all four): Larastan ships `UPGRADE.md`, Rector ships `UPGRADING.md`, Pint
  and rector-laravel ship neither — for those two, a major bump has historically meant new formatting rules /
  additional Rector rules, not breaking API changes, so their `CHANGELOG.md` is the next thing to check instead.
  ```bash
  for candidate in UPGRADE.md UPGRADING.md; do
    curl -fsSL "https://raw.githubusercontent.com/<repo>/HEAD/$candidate" 2>/dev/null && break
  done
  ```
  (`<repo>` is the second argument passed for that package above — e.g. `rectorphp/rector` for `rector/rector`, not
  the Packagist name.)
  Read what you get back for anything that touches what this skill manages directly:
  - **Larastan**: changes to `phpstan.neon`'s `includes:`/`parameters:` shape, or to what a given `level` now requires
    (3.0's guide, for example, stopped inferring Eloquent relation generics from method bodies — a real increase in
    reported errors at the same level, not a regression this skill introduced).
  - **Rector**: changes to the `RectorConfig::configure()` builder methods this skill's `rector.php` calls
    (`withPaths`, `withComposerBased`, `withPreparedSets`) — if one was renamed or dropped, Step 5's template needs
    updating, not just the version bump.
  - **Pint**: changes to `pint.json`'s shape or to the `Pint/laravel_blade` rule specifically.
  - Anything else in the guide is the project's own code to fix, not this skill's config — mention it, don't attempt
    to fix it here.
  Then bump the constraint and reinstall:
  ```bash
  composer require --dev <pkg>:^<latest-major>
  ```
  **Say plainly in the summary** that a major version changed, quote the before/after, and summarize whatever from
  the guide actually applies — this is the same "do it, disclose it" pattern `laravel-init` uses for its own
  major-version bumps, for the same reason: the constraint change is real and reviewable, not something to bury in a
  file count. Real breakage from a major bump (new PHPStan errors, a Rector rule now conflicting with Pint) will
  surface in Step 7 either way — that's what `composer test` is for.
