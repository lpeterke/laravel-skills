# laravel-skills

Lars's personal collection of agent skills for Laravel projects, installed via the [skills CLI](https://github.com/vercel-labs/skills).

## Install

```bash
npx skills add lpeterke/laravel-skills --all -p
```

## Use

Ask your agent: **"laravel init this project"**.

## Update a project

```bash
npx skills update laravel-init -p -y
```

Then, in a fresh agent session, ask it to run `laravel-init` — that pulls in everything else (skills, Boost, the MCP check).
