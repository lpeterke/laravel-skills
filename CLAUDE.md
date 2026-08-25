# CLAUDE.md — developing this repo

This file is for an agent (or Lars) working *on* this repository itself — i.e. adding, editing, or restructuring skills — not for an agent using these skills inside some other Laravel project.

## The goal

This repo is Lars's personal, portable layer of AI-agent skills for Laravel/Statamic freelance work. It exists to solve one problem: getting consistent, high-quality AI coding assistance set up in a new or existing Laravel project should take one command, not a checklist of manual steps re-derived from memory each time.

It sits alongside two other pieces that are deliberately *not* duplicated here:

- **Laravel Boost** (`laravel/boost`) — official, per-project, tied to whatever packages that specific project has installed (Livewire, Filament, Pest, etc.). Installed/updated via Composer + `php artisan boost:install`/`boost:update`, not via this repo.
- **mattpocock/skills** — third-party general engineering skills (grilling for requirements, domain modeling, code review, TDD). Installed via `npx skills add mattpocock/skills`, not duplicated here.

This repo is for everything that's specific to *Lars* rather than to a project or to general engineering practice: his own Laravel/Statamic conventions, his own workflows, and — for now — the `init` skill that glues Boost + mattpocock/skills together so setting up a project is one instruction instead of three separate systems to remember.

Skills here are distributed via the [skills CLI](https://github.com/vercel-labs/skills) (`npx skills`), which discovers any `skills/<name>/SKILL.md` in this repo automatically — it does not read this file or the README to decide what exists. Keep that in mind: the README is for humans, this file is for agents editing the repo, and neither one gates what `npx skills add lpeterke/laravel-skills --all` actually installs.

## Repo structure

```
laravel-skills/
├── README.md              — human-facing install instructions, keep terse
├── CLAUDE.md               — this file
└── skills/
    └── <skill-name>/
        └── SKILL.md        — required: YAML frontmatter (name, description) + instructions
            (optional: scripts/, references/, assets/ subfolders per skill)
```

Every skill lives in its own folder under `skills/`. One `SKILL.md` per folder, folder name matches the skill's `name` frontmatter field, kebab-case.

## Adding a new skill

1. **Decide if it belongs here at all.** Ask: is this Laravel/Statamic-specific-to-Lars knowledge, or a personal workflow? If it's generic engineering practice, it probably belongs upstream in mattpocock/skills territory (or just isn't worth a skill). If it's project-specific rather than portable across all of Lars's projects, it belongs in that project's own `.ai/guidelines/`, not here.
2. **Write a pushy, specific description.** Skills under-trigger by default. The frontmatter `description` should name concrete trigger phrases and contexts, not just describe capability in the abstract. Compare: "Helps with Livewire components" (weak) vs. "Use whenever the user is building, reviewing, or debugging a Livewire component, mentions wire:model, wire:click, or asks about Livewire lifecycle hooks" (better).
3. **Keep `SKILL.md` under ~500 lines.** If it's growing past that, split detail into a `references/` file and point to it from the main body rather than inlining everything.
4. **Make it idempotent wherever it touches project state.** Every skill that runs commands (like `init` does) should check current state first and only change what's actually needed — never assume a clean slate, never assume it hasn't run before.
5. **Test it in a real Laravel project before committing.** Symlink or `npx skills add lpeterke/laravel-skills --skill <name> -p` into a scratch project and actually run it once.

## Updating the `init` skill specifically

`init` is the orchestration point between this repo, Boost, and mattpocock/skills. If Boost's install/update commands change (check `laravel/boost`'s own docs/CHANGELOG periodically — Boost evolves independently of this repo), or if the mattpocock/skills repo restructures its own skill names, update `skills/init/SKILL.md` to match. Don't let it silently drift out of date with either upstream project.

## Things to avoid

- Don't duplicate content that Boost or mattpocock/skills already own — link to / invoke them instead.
- Don't bake in machine-specific paths, personal API keys, or anything not safe to commit publicly (this repo may end up public/shared).
- Don't add a `depends_on`-style mechanism expecting the skills CLI to auto-install sibling skills — that's not a shipped feature as of writing. `--all` already installs everything in this repo in one command; that's sufficient.
