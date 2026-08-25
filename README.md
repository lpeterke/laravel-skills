# laravel-skills

Lars's personal collection of agent skills for Laravel projects. Installable in any project with one command via the [skills CLI](https://github.com/vercel-labs/skills).

## Install

In any Laravel project, run:

```bash
npx skills add lpeterke/laravel-skills --all -p
```

That's it — this pulls in every skill in this repo (currently just `init`, more may be added over time).

## Use

Ask your agent (Claude Code, Cursor, etc.) to **"init this project"**. It will install Laravel Boost (or update it if already present) and install or update [mattpocock/skills](https://github.com/mattpocock/skills), all in one pass. Safe to run again anytime — it only changes what's actually out of date.

## Updating later

```bash
npx skills update
```

Pulls the latest version of every skill installed from this repo (and any others) into the current project.
