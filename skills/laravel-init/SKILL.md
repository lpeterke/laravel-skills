---
name: laravel-init
description: Bootstrap or refresh the AI tooling for a Laravel project in one pass — installs Laravel Boost if it's missing or updates it if it's already there, and installs or updates mattpocock/skills locally in the project. Use this whenever starting a new Laravel project, opening an existing Laravel project where Boost or mattpocock/skills look missing or outdated, or when the user says things like "laravel init", "init this project", "set up AI tooling", "bootstrap boost", or "update my skills". Always run this instead of manually piecing together composer/boost/npx-skills commands one at a time.
---

# Laravel Init

One skill that gets a Laravel project's AI tooling into a known-good state — whether the project is brand new or years old, and whether this is the first run or the fiftieth.

Everything below is idempotent: detect current state, do only what's needed, report what changed. Never assume — check first.

## Step 1 — Confirm this is a Laravel project

Check for `composer.json` in the project root and that it requires `laravel/framework`. If either is missing, stop and tell the user this doesn't look like a Laravel project rather than proceeding.

## Step 2 — Laravel Boost: install or update

```bash
grep -q '"laravel/boost"' composer.json 2>/dev/null && BOOST_PRESENT=1 || BOOST_PRESENT=0
```

Every branch below ends by running `php artisan boost:install --guidelines --skills --mcp`, always with all three flags explicit, never bare `boost:install` or `boost:update`. This was verified the hard way: in a real project, bare `boost:install` (original setup) and a later `boost:update --discover` (claiming "Boost guidelines and skills updated successfully") both silently left `.claude/skills/` with *zero* Boost-sourced skills, even though `php artisan boost:list-skills` correctly showed 6 available including `laravel-best-practices`. `boost:install --skills` fixed it immediately — 5 skills synced. Root cause: the interactive feature-selection prompt (guidelines/skills/mcp) that `boost:install` shows a human doesn't reliably get answered when an agent runs the command through a non-interactive shell, so skills silently stayed off; `boost:update`'s `--discover` only refreshes what's already enabled and won't retroactively turn a feature on. Passing the flags explicitly removes the prompt (and the failure mode) entirely — verified idempotent and safe to re-run, and it doesn't force re-selecting agents that are already configured in `boost.json`.

- **`BOOST_PRESENT=0` (fresh install):**
  ```bash
  composer require laravel/boost --dev
  php artisan boost:install --guidelines --skills --mcp
  ```
  Agent/IDE selection (Claude Code, Cursor, etc.) is a separate, genuinely interactive question the command still asks — let the user answer that live. The three flags only remove the feature-toggle prompt, not the agent one.

- **`BOOST_PRESENT=1` (already installed):** find the latest published version straight from the repo — not Packagist, not the docs site (already caught lagging once, see `internal/refresh-skills-catalog.md`) — and compare it against what's actually installed:

  ```bash
  LATEST=$(git ls-remote --tags --refs https://github.com/laravel/boost.git 2>/dev/null | awk -F/ '{print $NF}' | grep -E '^v[0-9]+\.[0-9]+\.[0-9]+$' | sort -V | tail -1)
  INSTALLED=$(composer show laravel/boost 2>/dev/null | grep -m1 '^versions' | grep -oE 'v?[0-9]+\.[0-9]+\.[0-9]+' | head -1)
  echo "latest=$LATEST installed=$INSTALLED"
  ```

  Before comparing `$LATEST` against `$INSTALLED`, also check that `composer.json`'s own constraint hasn't drifted from what's actually sitting in `vendor/`. `composer.json` is version-controlled and `vendor/` isn't, so a `git checkout`/merge/rebase that reverts `composer.json` alone leaves the file requiring an old major while a newer one stays physically installed until the next `composer install` re-resolves it — caught in a real project where `composer.json` still said `^1.8` but `vendor/` had `v2.6.0`:

  ```bash
  REQUIRED=$(grep -oE '"laravel/boost"[[:space:]]*:[[:space:]]*"[^"]+"' composer.json | grep -oE '"[^"]+"$' | tr -d '"')
  REQUIRED_MAJOR=$(echo "$REQUIRED" | grep -oE '[0-9]+' | head -1)
  INSTALLED_MAJOR=$(echo "$INSTALLED" | grep -oE '[0-9]+' | head -1)
  ```

  If `$REQUIRED_MAJOR` is behind `$INSTALLED_MAJOR`, fix the constraint to match what's actually installed right now, the same way the major-behind branch below does — don't just flag it and wait to be asked, this follows the same "do it, disclose it" pattern as any other major-constraint change in this step:

  ```bash
  composer require laravel/boost:^$INSTALLED_MAJOR --dev
  ```

  It isn't breaking anything the moment you find it, but the next unrelated `composer update` would try to satisfy the stale constraint and could silently downgrade Boost back to the old major — say so plainly in the summary, exactly as the major-behind branch below already does for its own constraint bumps.

  Then, depending on how `$LATEST` compares to `$INSTALLED` (now that the constraint agrees with what's installed):

  - **Major behind** (`LATEST`'s leading number is greater than `INSTALLED`'s): `composer update` respects the existing `^1.8`-style constraint in `composer.json` and will never cross a major boundary on its own, so add the new constraint explicitly —
    ```bash
    composer require laravel/boost:^<latest-major> --dev
    php artisan boost:install --guidelines --skills --mcp
    ```
    Past majors have restructured how skills/agents are laid out (v1→v2 did), so re-running the full setup is safer than assuming an update can reconcile the old shape. This changes `composer.json`'s constraint, which is a step up in blast radius from the usual same-major bump — say plainly in the summary that a major version changed and what it was before, so the user can review the diff.

  - **Same major, newer minor/patch** (`LATEST` ≠ `INSTALLED`, same leading number): stay inside the existing constraint —
    ```bash
    composer update laravel/boost
    php artisan boost:install --guidelines --skills --mcp
    ```

  - **Already on `LATEST`**: no composer command needed, but still run
    ```bash
    php artisan boost:install --guidelines --skills --mcp
    ```
    unconditionally — it re-derives the skill/guideline list from `composer.json` fresh each run, so a package added since the last pass (Livewire, Filament, Pest…) gets picked up even when Boost's own version hasn't moved. This is also what actually catches the "skills silently never got turned on" failure mode above; it isn't tied to a version bump.

  If `git ls-remote` fails (offline, GitHub unreachable) don't block — skip straight to running `boost:install --guidelines --skills --mcp` against whatever's currently required, and say in the summary that the version check couldn't run.

**Run this check on every single `laravel-init` pass, no exceptions** — fresh install, major bump, minor bump, or already-on-latest all land here the same way:

```bash
grep -q 'boost:update' composer.json 2>/dev/null && echo present || echo missing
```

If `missing`, offer to add `"@php artisan boost:update --ansi"` to `composer.json`'s `post-update-cmd` scripts — it keeps Boost's guidelines/skills/MCP config fresh on every future `composer update` without needing another `laravel-init` run. If the user says yes, actually make the edit (add a `post-update-cmd` array if none exists yet, append the entry if one does, don't duplicate if it's already there); don't just describe it and leave it for them to do by hand. If they decline, don't ask again on this run, but don't skip asking on the *next* run either — it's a live check against `composer.json`, not a one-time prompt, so a project that adds Boost later or has the line removed gets asked again.

Whichever branch ran: never hand-edit the generated `CLAUDE.md` / `AGENTS.md` / `.mcp.json` afterward — they're regenerated by these commands and edits will be lost.

## Step 3 — Gitignore the generated skill directories, keep the lockfile

The skills CLI writes fetched, regenerable output into a few directories: the canonical store (`.agents/skills/`), a duplicate "universal"-format copy (`agent/skills/`), and a symlink tree per wired-up agent (`.claude/skills/`, `.codex/skills/`, etc. — whichever agents were selected). None of that is source; `npx skills update`/`add` regenerates all of it, and committing it just means diff noise on every run. `skills-lock.json` is the opposite: small, and what makes the fetched copies reproducible on another machine — keep it committed.

```bash
git status -s | grep -E '^\?\? (\.agents/skills|agent/skills|\.[a-z]+/skills)/' | awk '{print $2}'
```

Add whichever of those paths aren't already in `.gitignore`:

```bash
grep -qxF '.agents/skills/' .gitignore 2>/dev/null || echo '.agents/skills/' >> .gitignore
grep -qxF 'agent/skills/' .gitignore 2>/dev/null || echo 'agent/skills/' >> .gitignore
```

`.agents/skills/` and `agent/skills/` are always created and always worth ignoring. For the per-agent symlink directories the `git status` check surfaced (e.g. `.claude/skills/`), add each one the same way — only the ones actually present, since that depends on which agents this project wired up.

If any of these directories are *already tracked* from before this step existed (`git ls-files .agents/skills .claude/skills agent/skills` returns matches), adding them to `.gitignore` alone won't stop tracking them. Flag this to the user and offer `git rm -r --cached <dir>` for each one rather than running it unasked — it rewrites the index.

## Step 4 — mattpocock/skills: install the curated set, prune the rest

Don't install the whole mattpocock/skills repo. A chunk of it isn't for this: the `in-progress/` bucket is explicitly beta ("public on purpose, feedback wanted, not shipped in the plugin"), `misc/` is "kept around but rarely used, not promoted" and includes skills that are hard-TypeScript (`migrate-to-shoehorn`, `setup-ts-deep-modules`) or npm-tooling-specific (`setup-pre-commit`'s Husky/lint-staged/Prettier stack) with no Laravel equivalent. Of the remaining promoted `engineering/`+`productivity/` skills, a few more are cut because they don't carry their weight for Lars's freelance Laravel work specifically: `triage` and `to-tickets` assume a formal issue tracker with triage labels, which these projects don't run; `wayfinder` is built for work bigger than a typical freelance engagement's scope; `ask-matt`, `teach`, and `wait-what` are generic productivity/meta tools that don't add Laravel-specific leverage.

That leaves this curated set — always ensure exactly these are installed, every run, regardless of what was there before:

```bash
MP_SKILLS=(grill-me grill-with-docs improve-codebase-architecture tdd setup-matt-pocock-skills handoff prototype grilling domain-modeling codebase-design diagnosing-bugs implement code-review research to-spec resolving-merge-conflicts wizard to-questionnaire writing-for-agents)

npx skills add mattpocock/skills --skill "${MP_SKILLS[@]}" --agent '*' -p -y
```

Use the array form (`MP_SKILLS=(...)`, expanded as `"${MP_SKILLS[@]}"`), not a plain space-separated string expanded unquoted — confirmed the hard way: under zsh (this machine's default shell), unquoted scalar expansion doesn't word-split the way it does in bash, so `--skill $MP_SKILLS` silently collapses the whole list into one bad skill name and the install fails with "No matching skills found." The array form expands correctly in both shells.

Then prune anything else this project has installed from mattpocock/skills — whether from an older, pre-curation run of this skill (which used `--all`) or from a manual `npx skills add`. Detect it from `skills-lock.json` rather than assuming: every mattpocock-sourced entry records `"source": "mattpocock/skills"` regardless of which skill it is, so anything with that source *not* in `MP_SKILLS` is stale:

```bash
STALE=$(node -e '
  const fs = require("fs");
  const curated = new Set(process.argv[1].split(" "));
  if (!fs.existsSync("skills-lock.json")) process.exit(0);
  const lock = JSON.parse(fs.readFileSync("skills-lock.json", "utf8"));
  const stale = Object.entries(lock.skills || {})
    .filter(([name, entry]) => entry.source === "mattpocock/skills" && !curated.has(name))
    .map(([name]) => name);
  console.log(stale.join(" "));
' "${MP_SKILLS[*]}")

[ -n "$STALE" ] && echo "$STALE" | xargs npx skills remove -p -y
```

Pipe through `xargs` rather than `npx skills remove $STALE -p -y` for the same reason as above — `xargs` does its own word-splitting on whitespace regardless of which shell is running, so it isn't subject to the bash/zsh discrepancy a raw unquoted expansion would be.

This is what makes a curation-list change here self-healing in every existing project on the next `laravel-init` run, the same pattern used elsewhere in this repo for renamed/dropped skills — no README migration step needed.

Keep both commands project-scoped (`-p`), not global — each project gets its own copy so one project's skill versions never silently affect another.

## Step 5 — Refresh every project-scoped skill

Always run this, on every pass, regardless of what Step 4 did:

```bash
npx skills update -p -y
```

This refetches every project-scoped skill from its source — the curated mattpocock set from Step 4, this repo's own skills, anything else installed. `-p` keeps it to project scope, `-y` skips the scope prompt.

Two things to know, and to tell the user when relevant:

- **This skill updates itself here, one run late.** The `laravel-init` instructions being followed right now are the copy already installed in the project. If `lpeterke/laravel-skills` changed upstream, this command pulls the new version, but *this* run still finishes on the old one. Say so in the summary, and suggest re-running `laravel-init` in a fresh session if the update actually moved it.
- **A `deleted upstream` warning for `init` is expected.** This skill used to be called `init`. Projects installed before the rename still carry that stale copy, and `update` reports it as deleted upstream but leaves it in place under `-y`. Clean it up:
  ```bash
  npx skills remove init -p -y
  ```
  Only do this when the stale skill is this one — check `skills-lock.json` and confirm the `init` entry's `source` is `lpeterke/laravel-skills` before removing anything. If there's no `init` entry, skip silently; most projects won't have one.
- **Locally-installed skills are skipped.** `npx skills update` only refetches skills whose `skills-lock.json` entry has `"sourceType": "github"`. Anything installed from a local path (`npx skills add /path/to/repo`) is silently ignored — it prints `No project skills to update.` rather than an error. To refresh those, re-run `npx skills add <path> --all -p`.

## Step 6 — Verify the Boost MCP server actually responds

Boost being installed doesn't mean its MCP server works — a PHP error, a broken `.env`, or a failed migration all leave the package in place and the server dead. Speak MCP to it directly:

```bash
printf '%s\n' '{"jsonrpc":"2.0","id":1,"method":"initialize","params":{"protocolVersion":"2024-11-05","capabilities":{},"clientInfo":{"name":"laravel-init","version":"1"}}}' | php artisan boost:mcp 2>&1 | head -c 400
```

A healthy server answers with a JSON-RPC result naming itself, e.g. `"serverInfo":{"name":"Laravel Boost"...}`, then exits when stdin closes. Anything else — a PHP stack trace, an exception, empty output — is the failure to report, with the output quoted.

Don't wrap this in `timeout`; macOS has no such command by default and the server already exits at EOF.

Then confirm the agent is actually wired to it. For Claude Code:

```bash
claude mcp list
```

Look for a `laravel-boost` entry marked connected. It may appear as `laravel-boost` or `plugin:laravel-boost:laravel-boost` depending on whether it came from `.mcp.json` or a plugin — both are fine, and a project with no `.mcp.json` at all is not a failure. If no Boost entry exists anywhere:

```bash
claude mcp add -s local -t stdio laravel-boost php artisan boost:mcp
```

For any other agent, say which one is in use and point at Boost's own registration details (`command: php`, `args: artisan boost:mcp`) rather than guessing at its config format.

## Step 7 — Configure mattpocock's skills non-interactively

`setup-matt-pocock-skills` configures the per-repo state the rest of mattpocock's engineering skills assume exists: issue tracker, triage label vocabulary, domain doc layout. It has `disable-model-invocation: true` and is normally a prompt-driven skill meant to be run by hand via `/setup-matt-pocock-skills`. Run it here instead, with fixed answers, so this whole pass stays non-interactive — these are freelance/solo Laravel projects, and the answers below are always the right ones for that shape of project.

Check whether it's already configured, using the same signal file the skill itself writes as its completion marker:

```bash
test -f docs/agents/issue-tracker.md && echo configured || echo not-configured
```

If `configured`, leave it alone — re-running it is only for switching issue trackers later, which is the user's call, not this skill's.

If `not-configured`, follow `setup-matt-pocock-skills`' own Explore → Write process (it's installed at `.agents/skills/setup-matt-pocock-skills/` from Step 4, seed templates included), but fix its three per-repo questions instead of asking them:

- **Section A (issue tracker): always "Local markdown."** Skip the git-remote check the skill would otherwise use to propose GitHub/GitLab — these projects don't run a shared tracker. Write `docs/agents/issue-tracker.md` from the seed template at `.agents/skills/setup-matt-pocock-skills/issue-tracker-local.md`.
- **Section B (triage labels): skip entirely.** `triage` isn't part of the curated set from Step 4, and the setup skill's own instructions already skip this section whenever `triage` isn't installed — nothing extra to do here.
- **Section C (domain docs): always single-context**, which is the skill's own default for everything except a pnpm/npm-workspace monorepo — never the case for a Laravel/Statamic project. Write `docs/agents/domain.md` from `.agents/skills/setup-matt-pocock-skills/domain.md`.

Skip the skill's own "confirm and edit" draft-review step too (its step 3) — showing a draft only matters when an answer was actually in question, and here none of them are.

Then write the `## Agent skills` block into whichever of `CLAUDE.md`/`AGENTS.md` already exists (create one only if neither exists, and ask the user which rather than guessing), following the setup skill's own file-selection rules — but omit the `### Triage labels` sub-block, since Section B never ran:

```markdown
## Agent skills

### Issue tracker

Local markdown, under `.scratch/`. See `docs/agents/issue-tracker.md`.

### Domain docs

Single-context. See `docs/agents/domain.md`.
```

## Step 8 — Report a summary

End with a short, concrete list of what actually happened, e.g.:

```
✅ Boost: v1.8.13 → v2.6.0 (major upgrade available) → composer.json updated, re-ran boost:install
✅ .gitignore: added .agents/skills/, agent/skills/, .claude/skills/
✅ mattpocock/skills: 19 curated skills installed/refreshed; pruned 2 stale (triage, ask-matt) from an earlier --all install
✅ npx skills update: 20 project skills refreshed (incl. laravel-init itself — re-run in a new session to use the updated version)
✅ Boost MCP: responds over stdio, connected in Claude Code
✅ mattpocock/skills setup: configured (local markdown tracker, single-context domain docs)
```

Don't just say "done" — call out anything that needs the user's attention, like a Boost interactive prompt that needs an answer, or a skill repo that failed to resolve.

## Edge cases

- **No `npx`/Node available:** tell the user directly — this step can't be silently skipped.
- **`boost:install` re-run by accident:** harmless — with the explicit flags it just re-syncs, it doesn't re-ask the agent/IDE question unless `boost.json` has no agents recorded yet.
- **`skills-lock.json` in the project:** not a pin, and not a reason to stop. The skills CLI writes this file automatically on every `add`; it records each skill's source and a content hash, and `update` rewrites it in place. Run the update without asking. (`npx skills experimental_install` restores from it if a project ever needs to be rebuilt from the lock.)
