---
name: laravel-audit
description: Audit an existing piece of a Laravel application — a file, a class, a Livewire component, a directory, or a whole feature — and deliver a verdict in three fields: implementation, performance, and security. Runs the Livewire/Alpine static scanners for hard evidence, then judges idiomatic quality against `laravel-best-practices` and the project's own recorded conventions. **Strictly read-only: it never edits a file, never fixes what it finds, and never starts an implementation.** Use whenever the user asks to audit, review, assess, critique, sanity-check or get a second opinion on existing code — "audit the checkout flow", "is this component any good", "review app/Livewire/Cart.php", "how's the security on this", "is this the Laravel way", "any performance problems here" — and especially before a refactor, a handover, or a release. Not for reviewing a diff you just wrote (that's `code-review`), not for fixing anything (that's `laravel-task`).
---

# Laravel Audit

Assesses code that already exists and reports on it. Same shape as `laravel-task` — it owns a sequence and delegates the substance to installed skills — but it produces a **judgement, not a change**.

## The read-only contract

**This skill does not modify the project. At all.** No edits, no new files, no `composer` install, no migration, no "quick fix while I'm here", no branch, no commit. If the audit turns up something urgent, it goes in the report as a recommendation and the user decides.

That constraint binds the skills this one delegates to as well. Three that are otherwise natural reaches here are **excluded because they write**:

| Don't invoke | Why | Do this instead |
| --- | --- | --- |
| `infer-conventions` (Boost) | Records what it learns as durable rules under `.ai/rules/` | *Read* `.ai/rules/` if it exists — see Step 3 |
| `ce-compound` | Writes a learning doc under `docs/solutions/` | Nothing. An audit isn't a solved problem |
| `improve-codebase-architecture` (mattpocock) | Reports, then grills the user through *implementing* one | Report the opportunity; stop there |

If the user asks mid-run for something found to be fixed, that's a new job: hand it to `laravel-task`, which owns planning and implementation. Don't quietly become it.

## Step 0 — Preconditions, inventory, stack

Same as `laravel-task` Step 0, and for the same reasons — read it there if the detail matters. In short:

```bash
ls .agents/skills 2>/dev/null | sort      # skills-CLI side
php artisan boost:list-skills 2>/dev/null # Boost side — a different directory
```

Both sources, because they don't overlap. Then the stack probes (`composer.lock` for PHP, `package.json` for JS) exactly as `laravel-task` Step 0 lists them — Livewire, Volt, Flux, Filament, Folio, Wayfinder, Pennant, MCP, Inertia + adapter, Alpine, Tailwind, Pest, Statamic.

A missing skill degrades the audit, it doesn't stop it — but **say so in the report**, because "no security findings" from a run where no security skill was installed is a fundamentally different statement from a clean scan, and the reader can't tell them apart unless you say which one happened.

## Step 1 — Resolve the scope, and confirm it

**An audit with a fuzzy scope produces a fuzzy verdict.** Turn whatever the user said into a concrete list of files before analysing anything.

- **A path** (`app/Livewire/Cart.php`, `app/Services/`) — done, that's the scope.
- **A feature or domain** ("the checkout flow", "invoicing") — search for it. Routes, controllers, Livewire components, models, jobs, views, and the tests that cover them. Follow the call graph outward one hop from what you find; a checkout audit that misses the payment job is not an audit of checkout.
- **"The recent work" / a branch / a PR** — resolve to the diff's file set. This is the one case where `code-review` applies directly (see Step 3).
- **Nothing specific** — ask what to audit. Don't audit the whole application on a vague prompt; the report would be too shallow to act on.

Then **show the file list and get confirmation before going further.** Cheap to correct now, expensive after a full analysis of the wrong twelve files. Include a count and flag anything surprising you pulled in.

Note the two boundaries that shape the rest of the run:

- **`SCOPE`** — the files being judged.
- **`ROOT`** — the project root (`git rev-parse --show-toplevel`). Step 2 needs this separately, and conflating the two silently breaks a scanner.

## Step 2 — Run the static scanners

Do this **before** the judgement work in Step 3. It's cheap, it's objective, and its findings are verifiable facts with line numbers rather than opinions — which is exactly what makes a verdict defensible.

The olgunozoktas skills ship real static analysers in their `bin/` directories, and they survive installation (`.agents/skills/<skill>/bin/`). Self-tests all pass from the installed location: `scan.php` 24/24, `scan-performance.php` 13/13, `review-security.py` 26/26, Alpine `review.py` 30/30.

```bash
S=.agents/skills
ROOT=$(git rev-parse --show-toplevel 2>/dev/null || pwd)

# Livewire security — takes a DIRECTORY. Safe on a subdirectory (see below).
php "$S/livewire-security/bin/scan.php" <scope-dir-or-$ROOT>

# Livewire performance — takes a DIRECTORY, and it MUST be $ROOT. See the trap below.
php "$S/livewire-performance/bin/scan-performance.php" "$ROOT"

# Alpine — take FILES, not directories. Blade templates in scope.
python3 "$S/alpinejs-security/bin/review-security.py" <files…>
python3 "$S/alpinejs-reference/bin/review.py" <files…>
```

Four things about these tools that are not obvious and that will produce a wrong verdict if ignored. All verified by running them, not by reading their docs:

**1. `scan-performance.php` only looks in `<root>/app`, `<root>/resources/views` and `<root>/packages`.** Hand it a feature subdirectory and it searches for `app/` *inside that subdirectory*, finds nothing, prints `0 finding(s)` and exits 0. That reads exactly like a clean bill of health. Verified in both directions: the same code reported 3 findings from the project root and 0 from `app/Livewire`. **Always pass `$ROOT`**, then filter the output down to `SCOPE` yourself.

**2. `scan.php` (security) does not share that flaw** — it falls back to scanning the given directory when none of the conventional paths exist, with a comment in its source saying why ("rather than reporting a clean result for a tree nobody looked at"). So it's safe on a subdirectory. The two sibling scanners genuinely differ here; don't assume symmetry.

**3. Exit code is the finding count, not success/failure.** A non-zero exit is a normal result with findings, not a crash. Don't report it as a tool error.

**4. `review-security.py` has a verified false negative on its own headline rule.** Its `looks_like_php()` suppression matches `->` anywhere in the fragment, and it tests the *whole* matched region including the Blade interpolation. So `x-data="{ name: '{{ $user->name }}' }"` — server data compiled as JavaScript, the single most important thing that scanner exists to catch — is silently suppressed, while `{{ $displayName }}` in the identical position is correctly flagged. Confirmed by running both through `scan()` directly. Its own self-test can't catch this: all six fixtures use simple `{{ $var }}`.

The suppression is there for a real reason (Blade's `:prop="$var"` component binding collides with Alpine's `:attr` shorthand, a false-positive class the maintainer fixed in 1.3.1) — it's just too broad. **So a clean Alpine security scan is not evidence of anything.** Grep the scope yourself for server interpolation inside an Alpine expression attribute and read the hits:

```bash
grep -rnE '(x-data|x-init|x-effect|x-show|x-if|x-text|x-model|x-on:[a-z]+|@[a-z]+|:[a-z-]+)\s*=\s*"[^"]*(\{\{|\{!!)' <scope>
```

Every hit is a candidate. The rule to judge it by — from `alpinejs-security`, verified against Alpine's source — is that Alpine compiles the raw attribute text with `new AsyncFunction`, so HTML-escaping does not protect it; `@js()` or `Js::from()` does.

## Step 3 — Judge it against the skills

Scanners find *shapes*. They can't tell you whether this is a good solution. That's this step.

**`laravel-best-practices` (Boost) applies to every audit, always** — it's the framework-wide standard for what idiomatic Laravel looks like, and "is this the Laravel way" is the core question. Invoke it and judge the scope against it.

**Read the project's own conventions before judging.** A repo standard beats a general one, every time:

```bash
cat .ai/rules/index.md 2>/dev/null; ls .ai/rules/ 2>/dev/null   # Boost's infer-conventions output — READ, never run
cat CLAUDE.md AGENTS.md CONTRIBUTING.md CODING_STANDARDS.md 2>/dev/null
cat docs/agents/domain.md 2>/dev/null                            # mattpocock domain doc, if laravel-init configured it
```

If `.ai/rules/` records a deliberate against-the-grain choice, code following it is *correct*, not a finding. Flagging a documented convention as a defect is the fastest way to make an audit worthless.

Then route by what Step 0 detected — same table as `laravel-task` Step 2, with the security and performance skills carrying much more weight here:

| Detected | Invoke |
| --- | --- |
| `livewire` | `livewire-security`, `livewire-performance` (the two verdict fields), plus `livewire-reference` and `livewire-development` (Boost) for idiomatic judgement |
| `alpine`, or Livewire front-end code | `alpinejs-security`, `alpinejs-reference` |
| `flux` / `volt` | `fluxui-development` / `volt-development` (Boost) |
| `inertia-{vue,react,svelte}` | the matching `inertia-*-development` (Boost) |
| `tailwind` | `tailwindcss-development` (Boost) |
| `folio` / `wayfinder` / `pennant` / `mcp` | the matching Boost skill, when the scope touches it |
| any scope with tests | `testing-best-practices` (Boost) and `tdd` (mattpocock) — for coverage quality, see Step 4 |
| architectural scope | `codebase-design` (mattpocock) — vocabulary for judging module depth. **Not** `improve-codebase-architecture`, which implements |

**On `code-review` (mattpocock): it is diff-scoped, and that's a real constraint, not a detail.** It reviews `git diff <fixed-point>...HEAD` and needs a fixed point from the user. So:

- **Scope is a branch, PR or "the recent work"** → invoke it directly. It's the right tool, and its two parallel axes (Standards / Spec) are better than anything this skill would improvise.
- **Scope is existing code with no meaningful diff** → don't force it. Borrow the part that transfers: its **12-smell baseline** (mysterious name, duplicated code, feature envy, data clumps, primitive obsession, repeated switches, shotgun surgery, divergent change, speculative generality, message chains, middle man, refused bequest) and apply it to the file set. Two of its rules carry over exactly and matter here: **the repo's documented standard overrides the baseline**, and **every smell is a labelled judgement call, never a hard violation**.

Also borrow its discipline of not merging separate axes into one ranking. This skill's three fields exist for the same reason: a component can be idiomatic and slow, or fast and leaky, and averaging those hides both.

## Step 4 — The verdict: three fields

Report exactly three, always, in this order. Never collapse them into one score, and never skip one — "not applicable" is itself a finding worth stating.

Each field gets **one rating**, on this shared scale:

| | Meaning |
| --- | --- |
| ✅ **Solid** | Idiomatic and safe as far as this audit could see. Nothing worth changing |
| 🟡 **Minor** | Works and is safe; some polish available. Nothing that would block a release |
| 🟠 **Needs work** | Real problems that will cost later — a latent bug, a design that fights the framework, a cost that grows with traffic |
| 🔴 **Serious** | Fix before shipping. Exploitable, or broken under conditions the app will actually meet |

**Every finding cites `file:line` and is labelled by how you know it:**

- **Verified** — a scanner hit, or something read directly from framework source. State the tool or file.
- **Judgement** — an opinion about design or idiom. Perfectly legitimate; just labelled, so the reader can weigh it.

That label is the most useful thing in the report. It's the difference between "this is exploitable" and "I'd have written it differently", and conflating them is how audits lose credibility.

**Implementation** — is this a good, idiomatic Laravel solution? Framework features used where they fit (or reinvented), the right abstraction level, correct layering, naming, error handling, the smell baseline, and adherence to this project's own recorded conventions. **Test coverage belongs here** — untested code is an implementation weakness. Judge whether the tests test behaviour at a seam or restate the implementation, and say so.

**Performance** — what does this cost, and how does that scale? N+1 queries, missing indexes on what's actually queried, work in a loop that belongs outside it, unbounded collections, missing eager loads, sync work that should be queued, caching that's absent or wrong. For Livewire: everything `livewire-performance` covers — snapshot size, models on public properties (re-queried through the *write* connection on every request), `wire:model.live` on text inputs, `wire:poll` without an interval. Be explicit that static analysis finds *shapes*, not measurements: a costly shape on a page rendered twice a day is not a problem, and saying so is part of an honest verdict.

**Security** — authorization on every public entry point (not just authentication), mass assignment, injection, what's exposed to the client, secrets, validation. For Livewire: every public property ships to the browser in `wire:snapshot` and comes back mutable unless `#[Locked]`, and every public method is callable from the browser. For Alpine: the `new AsyncFunction` seam from Step 2. If the scope has no security surface at all, say "no security surface in scope" rather than defaulting to ✅ — they're different claims.

## Step 5 — Report

Lead with the three verdicts, then the evidence. Keep it readable — this is for a person deciding what to do next.

```
## Audit: checkout flow (9 files, app/Livewire/Checkout/*, app/Actions/PlaceOrder.php, 2 views)

Implementation  🟡 Minor
Performance     🟠 Needs work
Security        🔴 Serious

### 🔴 Security
1. [Verified — scan.php] app/Livewire/Checkout/Payment.php:24 — `public $customerEmail` is published in
   wire:snapshot and is browser-writable. Add #[Locked], or move it out of component state.
2. [Verified — scan.php] app/Livewire/Checkout/Payment.php:61 — `deleteOrder()` is public, so it's callable
   from the browser, and has no authorize/policy/gate call.
3. [Verified — manual grep; the scanner's `->` suppression hides this] checkout.blade.php:12 —
   `x-data="{ email: '{{ $user->email }}' }"` compiles server data as JavaScript. Use @js($user->email).

### 🟠 Performance
1. [Verified — scan-performance.php] Payment.php:19 — `public Order $order` is re-queried on every request,
   through the write connection. ~40 extra writes/min at current traffic.
2. [Judgement] PlaceOrder.php:88 — line-item loop queries per iteration. Eager-load, or batch.

### 🟡 Implementation
1. [Judgement] PlaceOrder.php — 140-line handle() doing validation, payment and notification. Three
   responsibilities; the notification half is what's untested.
2. [Judgement] No test covers a declined payment — the branch most likely to be wrong.

### Crucial improvements
1. #[Locked] on the identity properties, and an authorize() in deleteOrder(). — Security 🔴
2. @js() for the Alpine interpolation. — Security 🔴
3. Drop the model property to an ID and resolve it in a computed. — Performance 🟠

### Audit boundary
- Static analysis only; nothing was executed and no traffic was measured.
- alpinejs-security's scanner has a known false negative on interpolations containing `->`; the Blade
  files in scope were grepped by hand to compensate. Other Alpine files outside scope were not.
- livewire-performance was run from the project root and filtered to scope, as it requires.
```

Two rules on **crucial improvements**: only what's genuinely worth acting on, ranked by severity, each naming the field it fixes. Three to five items, not fifteen — a list nobody reads has the same value as no list. And they stay **recommendations**: no diffs, no patches, no "shall I fix it?" mid-report.

Always close with the **audit boundary** — what was and wasn't checked. An audit that doesn't state its limits invites the reader to assume it covered everything. Name the scanner false negative whenever an Alpine scan ran; name any skill that was missing from Step 0.

If everything is genuinely fine, say so plainly and briefly. ✅/✅/✅ with a two-line justification is a perfectly good result and doesn't need padding out with manufactured nitpicks.

## Edge cases

- **The scope is too big.** More than ~30 files and the report goes shallow. Say so, propose a split, and audit the highest-risk slice first rather than skimming everything.
- **The user wants it fixed.** That's `laravel-task`, in a fresh invocation. Offer the handoff; don't start.
- **Nothing installed but Boost.** Run with `laravel-best-practices` alone and say clearly in the boundary that the Livewire/Alpine scanners weren't available, so those verdicts rest on reading alone.
- **The scope is a package or vendor code.** Audit it if asked, but say plainly it isn't this project's to change, and skip the "crucial improvements" section in favour of an upgrade/replace recommendation.
- **A scanner crashes** (missing `php`/`python3`, syntax error in scanned source). Report the tool failure as a gap in the boundary. Never silently drop a field to ✅ because its evidence didn't run.
- **The audit finds something actively exploitable.** Lead with it, state it plainly, and say it needs fixing before release. Still don't fix it — but make sure it can't be missed.
