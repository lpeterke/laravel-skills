# CLAUDE.md — developing this repo

This file is for an agent (or Lars) working *on* this repository itself — i.e. adding, editing, or restructuring skills — not for an agent using these skills inside some other Laravel project.

## The goal

This repo is Lars's personal, portable layer of AI-agent skills for Laravel/Statamic freelance work. It exists to solve one problem: getting consistent, high-quality AI coding assistance set up in a new or existing Laravel project should take one command, not a checklist of manual steps re-derived from memory each time.

It sits alongside two other pieces that are deliberately *not* duplicated here:

- **Laravel Boost** (`laravel/boost`) — official, per-project, tied to whatever packages that specific project has installed (Livewire, Filament, Pest, etc.). Installed/updated via Composer + `php artisan boost:install`/`boost:update`, not via this repo.
- **mattpocock/skills** — third-party general engineering skills (grilling for requirements, domain modeling, code review, TDD). Installed via `npx skills add mattpocock/skills`, not duplicated here.

This repo is for everything that's specific to *Lars* rather than to a project or to general engineering practice: his own Laravel/Statamic conventions, his own workflows, and — for now — the `laravel-init` skill that glues Boost + mattpocock/skills together so setting up a project is one instruction instead of three separate systems to remember.

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
- **A renamed skill is a delete plus an add, and `update` does neither.** The old name is flagged (`The following skills … appear to have been deleted upstream`) but kept, because `-y` means non-interactive and deletion is skipped; the new name is simply absent from the lock, so it's never fetched. Getting a rename into an existing project takes one `npx skills add lpeterke/laravel-skills --all -p`. `laravel-init` then removes the stale `init` itself on its next run, so nothing is left for Lars to do by hand — which is why the README documents no migration procedure. Prefer this pattern over a README step whenever a repo change would otherwise strand a project.

## Updating the `laravel-init` skill specifically

`laravel-init` is the orchestration point between this repo, Boost, and mattpocock/skills. If Boost's install/update commands change (Boost evolves independently of this repo), or if the mattpocock/skills repo restructures its own skill names, update `skills/laravel-init/SKILL.md` to match. Don't let it silently drift out of date with either upstream project.

Don't rely on remembering to check this by hand — `internal/refresh-skills-catalog.md` (see below) automates exactly this drift check.

## Keeping this repo's own docs in sync

`internal/refresh-skills-catalog.md` is this repo's own maintenance routine: it checks for drift against Boost and mattpocock/skills, and regenerates the README's "Available skills" section — the catalog of what `laravel-init` actually pulls into a project, sourced from Boost's and mattpocock/skills' current skill sets, not from this repo's own `skills/` folder. It's not a real skill by design (see *The goal* above), so follow it by reading the file directly rather than by name.

Run it after any bigger change to this repo — what `laravel-init` installs from Boost or mattpocock/skills changes, or anything else that could make the README or `laravel-init` drift from reality. When in doubt after a change, run it.

## Things to avoid

- Don't duplicate content that Boost or mattpocock/skills already own — link to / invoke them instead. The "Available skills" catalog is the one sanctioned exception: name + a short gist per skill, for discoverability, not their instructions. If a table entry grows past one line, it's become duplication.
- Don't bake in machine-specific paths, personal API keys, or anything not safe to commit publicly (this repo may end up public/shared).
- Don't add a `depends_on`-style mechanism expecting the skills CLI to auto-install sibling skills — that's not a shipped feature as of writing. `--all` already installs everything in this repo in one command; that's sufficient.
