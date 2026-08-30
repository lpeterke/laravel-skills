---
name: laravel-task
description: Work a single task, feature, refactor or bugfix through a Laravel project end to end — grill the requirements, write a plan, grill the plan, implement it using whichever Boost/Livewire/Alpine/Inertia skills the project's actual stack calls for, cover it with a Pest test, run `composer test`, and capture the learning. Assumes `laravel-init` has already run so those skills are installed. Use whenever the user hands over a piece of work in a Laravel project — "add a…", "build the… feature", "fix the bug where…", "refactor…", "implement…", "laravel task", "work on this ticket", "start on this" — instead of jumping straight into editing files. Do not use for pure setup/tooling work (that's `laravel-init` / `laravel-lint-setup`) or for questions that need no code change.
---

# Laravel Task

The orchestration skill for actually doing work in a Laravel project, once `laravel-init` has put the tooling in place.

It owns the *sequence* — grill, plan, grill, implement, test, verify, summarize — and delegates the *substance* to skills that already exist. Almost nothing here is knowledge about Laravel; it's knowledge about which installed skill to reach for and in what order.

**Three rules that govern every step below:**

1. **Never invoke a skill you haven't confirmed is installed.** Step 0 builds the inventory. A skill named here that isn't in the inventory is skipped silently — say so in the summary, don't invent a substitute and don't stop.
2. **Delegate, don't paraphrase.** When a step says to use a skill, invoke it with the Skill tool and follow it. Don't summarize what you think it says from memory — several of these skills carry corrections to things that are plausible but wrong.
3. **This skill never sets tooling up.** If Boost or the mattpocock set is missing, say so and point at `laravel-init`; don't install anything from here. If `composer test` doesn't exist, point at `laravel-lint-setup`. Same boundary `laravel-init` keeps for the same reason — see its Step 9.

## Step 0 — Preconditions and inventory

Confirm this is a Laravel project (`composer.json` requiring `laravel/framework`); if not, stop and say so.

Then find out what's actually available, because every routing decision below depends on it. Two separate sources, and they do not overlap:

```bash
# Skills installed by the skills CLI (this repo, mattpocock, olgunozoktas, EveryInc)
ls .agents/skills 2>/dev/null | sort

# Skills installed by Boost — a different mechanism, a different directory
php artisan boost:list-skills 2>/dev/null
```

`npx skills` writes `.agents/skills/<name>` and symlinks it into `.claude/skills/`; Boost writes directly into `.claude/skills/<name>`. Reading only one of the two sources misses half the set. `boost:list-skills` is the authoritative list of Boost's side — it reflects the packages this project actually has, which is the whole point (a project without Inertia has no `inertia-*` skill and never will).

If `.agents/skills` is empty or absent, or `boost:list-skills` errors out, say plainly that `laravel-init` looks like it hasn't run and offer to run it — then stop. Working the task with none of the delegated skills present is a materially different (worse) job, not a degraded-but-fine one.

Then detect the stack, which decides Step 2's routing:

```bash
uses()      { grep -Eq "\"name\":\s*\"$1\"" composer.lock 2>/dev/null; }
uses_npm()  { grep -q "\"$1\"" package.json 2>/dev/null; }

uses livewire/livewire        && echo livewire
uses livewire/flux            && echo flux
uses filament/filament        && echo filament
uses inertiajs/inertia-laravel && echo inertia
uses_npm '@inertiajs/vue3'    && echo inertia-vue
uses_npm '@inertiajs/react'   && echo inertia-react
uses_npm '@inertiajs/svelte'  && echo inertia-svelte
uses_npm alpinejs             && echo alpine
uses_npm tailwindcss          && echo tailwind
uses pestphp/pest             && echo pest
uses statamic/cms             && echo statamic
grep -q '"test"' composer.json 2>/dev/null && echo composer-test
```

Check `composer.lock`, not `composer.json`'s `require` — a Filament project has Livewire transitively and every Filament resource *is* a Livewire component, so the Livewire skills are exactly as relevant there. This is the same reasoning `laravel-init` Step 5 documents for its own gate; keep the two consistent.

## Step 1 — Grill, plan, grill

The user handed over a task. It is almost never as specified as it looks.

**1a. Grill the requirements, before writing anything.** Invoke `grilling`. It interviews the user in rounds — the whole answerable frontier per round, each question with your recommended answer — until nothing is silently assumed.

> Invoke `grilling`, not `grill-me`. `grill-me` is a one-line stub whose entire body is "Call the Skill tool with `grilling`", and it carries `disable-model-invocation: true`, so it is the *user's* trigger phrase, not an entry point you can call. Verified by reading both files. If the user typed "grill me", that still lands here.

Two things `grilling` is explicit about that are easy to get wrong: **facts are your job, decisions are the user's.** Anything discoverable from the codebase, the schema, `composer.lock` or the docs — go find it, in parallel, rather than asking. Only put actual choices to the user. And don't act on the outcome until the user confirms shared understanding.

Skip 1a only when the task is genuinely unambiguous *and* small — a typo, a one-line copy change, a named test that fails for a stated reason. When in doubt, grill; the cost of one round of questions is far below the cost of building the wrong thing.

**1b. Write the plan.** Only after 1a settles. Keep it in chat for small work; write a file for anything multi-step. A useful plan names the files it will touch, the seam it will be tested at, and what "done" means — not just a list of verbs.

If the task is a *bug* rather than a feature, use `diagnosing-bugs` here instead of planning from a blank page: reproduce first, find the actual cause, and only then plan the fix. A plan written before the bug is understood is a guess.

If the work is big or structural enough that the domain vocabulary is in question, `domain-modeling` and `codebase-design` are the right reaches. Don't pull them in by default.

**1c. Grill the plan.** Invoke `grilling` again, this time against the written plan. Different job from 1a: 1a asks "what are we building"; 1c asks "will this plan survive contact with the codebase" — wrong seam, missed migration, a case the plan quietly doesn't handle, a decision made in a phase whose prerequisite is still open.

Revise the plan on what comes back. Then get the user's go-ahead before Step 2. **Steps 1 and 2 are separated on purpose — do not start editing files while the plan is still being grilled.**

## Step 2 — Implement, routed by stack

**`laravel-best-practices` (Boost) applies to every task, always.** It's framework-wide conventions for writing and refactoring Laravel PHP and it lands in every project regardless of stack. Invoke it before writing code, not after. This is the one non-negotiable entry in this step.

Then add whichever of these the Step 0 stack detection turned up. These are *and*, not *or* — a Livewire + Flux + Tailwind screen pulls all of them:

| Detected | Invoke |
| --- | --- |
| `livewire` | `livewire-development` (Boost), plus `livewire-reference`, `livewire-security`, `livewire-performance` (olgunozoktas) |
| `flux` | `fluxui-development` (Boost) |
| `alpine`, or any Livewire work touching the front end | `alpinejs-reference`, `alpinejs-security` (olgunozoktas) |
| `inertia-vue` | `inertia-vue-development` (Boost) |
| `inertia-react` | `inertia-react-development` (Boost) |
| `inertia-svelte` | `inertia-svelte-development` (Boost) |
| `tailwind` | `tailwindcss-development` (Boost) |

Boost's Livewire, Inertia and Tailwind skills ship in *version-specific* variants (Livewire v2/v3/v4, Inertia v1/v2, Tailwind v3/v4) and Boost installs the one matching the project. That's another reason to read the installed skill rather than answer from memory — the variant on disk is the one that's true here.

Pick from what Step 0's inventory actually listed. A stack signal with no matching installed skill means Boost hasn't re-run since that package was added: note it and suggest `laravel-init`, then carry on without it.

`livewire-security` and `alpinejs-security` earn their place on any task that renders user-controllable data or exposes a public component property — not just on work that sounds security-shaped. Their two central findings (a Livewire snapshot is client-visible and client-mutable unless `#[Locked]`; HTML-escaping a value into an `x-data`/`x-on`/`x-init` attribute does not protect it, because Alpine compiles the raw attribute text) both contradict what a reasonable person would assume.

**Implementation discipline**, whichever skills are in play:

- Follow the plan from Step 1. A decision the plan settled is settled — if implementation reveals it was wrong, stop and say so rather than quietly re-deciding.
- Match the surrounding code. Read a neighbouring controller/component/migration before writing a new one; the project's conventions beat any general guidance, including this skill's.
- Prefer `implement` (mattpocock) for multi-file work driven from a written spec. It's `disable-model-invocation: true`, so it's a deliberate reach, not an automatic one — but its loop (TDD at pre-agreed seams, typecheck often, full suite once) is the right shape for exactly this step.
- If the project is Statamic, remember that content lives in flat files under `content/` and views in Antlers or Blade — don't reach for Eloquent by reflex.

## Step 3 — Write a Pest test

**Write a test. This is the default and it does not need to be asked about.** There are exactly two ways out:

1. **The user said not to** when invoking this skill — "no test", "skip the tests", "just the fix". Explicit only; not inferred from brevity or urgency.
2. **A test genuinely adds nothing** — a copy change, a comment, a config value with no behaviour attached, a pure rename. Say which of the two applied in the summary; never let the step vanish silently.

Use `tdd` (mattpocock) for what makes a test worth keeping, and `testing-best-practices` (Boost) for Laravel-specific test design. If the project is on Pest — which `laravel-lint-setup` makes it, and which Step 0 detected — write Pest syntax (`it('…', function () { … })`), not PHPUnit classes, regardless of what any older test in the suite looks like.

If Step 0 found **no** `pestphp/pest`, don't install it from here and don't quietly write PHPUnit either. Write the test in the project's current framework, and note in the summary that `laravel-lint-setup` would migrate the suite to Pest.

Three rules about *what* to write, in priority order:

**Extend before you create.** Search for an existing test covering the same seam first. A new case added to a file that already sets up the right fixtures is worth more than a new file that rebuilds the same world — and it's how the suite stays navigable.

```bash
ls tests/Feature tests/Unit 2>/dev/null
grep -rln "<the class, route, or component you touched>" tests/ 2>/dev/null
```

**No sky-is-blue tests.** A test that would pass against a stub, that asserts a getter returns what the setter set, that re-states the implementation line by line, or that can't fail for any reason a user would care about — is negative value. It costs a maintenance slot and buys nothing. If the honest answer is "the only test I can write here is trivial", that's case 2 above: say so and skip it.

**Test at the seam, not through the internals.** For Laravel that usually means the HTTP route, the Livewire component's public surface (`Livewire::test(...)`), the job, or the command — not a private method reached through reflection. Behaviour survives refactors; internals don't.

Concretely: assert the outcome a user or caller would notice. The response status *and* what it says. The row that landed in the database with the values it should have. The mail/event/job that was dispatched. The authorization case that gets a 403. Bugfix work gets a regression test that fails against the old code — if it passes before your fix, it isn't testing the bug.

## Step 4 — Run the suite

If Step 0 found a `test` script in `composer.json`:

```bash
composer test
```

`laravel-lint-setup` wires that script to run `config:clear` → `refactor:check` → `lint:check` → `types:check` → `artisan test`, in that order, so a green run means Rector, Pint, Larastan and the tests are all satisfied — not just the tests. Read the failure by stage; `laravel-lint-setup`'s Step 8 troubleshoots each one, and it's the right thing to consult rather than guessing at a fix.

Two failure modes to keep separate, because they call for opposite responses:

- **Your change broke it** → fix it, re-run, repeat until green. Non-negotiable.
- **It was already broken** → confirm with `git stash && composer test; git stash pop`, then report it as pre-existing and leave it alone. Fixing unrelated failures inside a task is scope creep, and it makes the diff unreviewable. Say plainly that it was already failing and offer to look at it separately.

If there's no `test` script, run the narrowest thing that exists (`./vendor/bin/pest <path>`, or `php artisan test --filter=…`) and note in the summary that `laravel-lint-setup` would give the project a real `composer test`.

Don't commit unless the user asks. Finishing the work and shipping it are separate decisions, and this skill only owns the first.

## Step 5 — Capture the learning

If `ce-compound` is in Step 0's inventory and the task turned up something a future session would otherwise have to rediscover — a non-obvious cause, a framework behaviour that contradicted the obvious reading, a project-specific constraint that isn't written down anywhere — invoke it to write that up as a durable repo learning under `docs/solutions/`.

Its own preconditions are the gate, and they're deliberately strict: **solved, verified working, and non-trivial**, one learning per run. Judge them from the session rather than asking. Most tasks produce nothing that qualifies, and that's the normal outcome — writing up a routine CRUD addition just buries the learnings that matter. When nothing qualifies, skip the step and don't mention it.

## Step 6 — Summary

Short, concrete, and honest about what didn't happen:

```
✅ Plan: grilled (2 rounds), written to .scratch/plans/invoice-export.md, re-grilled — seam moved from the
   controller to the InvoiceExporter action
✅ Implemented: 4 files — app/Actions/ExportInvoices.php (new), InvoiceController, routes/web.php,
   resources/views/invoices/index.blade.php. Skills: laravel-best-practices, livewire-development,
   livewire-security, tailwindcss-development
✅ Test: extended tests/Feature/InvoiceExportTest.php with 3 cases (auth'd export, 403 for another
   tenant's invoice, empty-range response). No new file — the fixtures were already there.
✅ composer test: green (refactor:check, lint:check, types:check, artisan test — 148 passed)
ℹ️  Learning captured: docs/solutions/security/livewire-locked-tenant-id.md
```

Call out anything that needs the user's attention rather than burying it:

- a decision made under an assumption the grilling didn't fully settle
- a pre-existing failure left alone, quoted
- a stack signal with no matching Boost skill installed → suggest `laravel-init`
- no `composer test` / no Pest → suggest `laravel-lint-setup`
- a test deliberately skipped, and which of Step 3's two cases applied

Never end on a bare "done".

## Edge cases

- **The task turns out to be two tasks.** Say so at the end of Step 1 rather than building both. Grilling surfaces this often — it's a success of the step, not a delay.
- **The plan doesn't survive Step 2.** Stop, report what the codebase actually said, and re-grill the affected branch. Don't silently re-decide something the user settled.
- **`laravel-init` never ran.** Step 0 catches it. Offer to run it; don't emulate it.
- **The user asks to skip planning** ("just do it"). Honour it — but still write the test, still run the suite, and say in the summary that planning was skipped at their request.
- **Nothing to implement** — the answer is an explanation, not a change. Answer the question and don't manufacture a diff to justify the skill having run.
