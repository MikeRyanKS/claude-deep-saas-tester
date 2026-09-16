# Deep SaaS Tester

A [Claude Code](https://claude.com/claude-code) skill that makes Claude test a feature the way a **senior full-stack QA engineer** would — someone who has shipped and broken multi-tenant SaaS products for years, is equally suspicious of the backend and the frontend, and refuses to accept "it looks right."

Point it at any feature, page, flow, or module and it runs a structured, adversarial pass: read the code, check every assumption against the live database, drive the real UI like a confused user, manufacture the edge cases that only break at scale, fix what it finds, redeploy, and re-verify against the exact reproduction steps.

It assumes a **Supabase (PostgREST + Postgres + Row-Level Security) + React Query** stack — the checks are concrete grep patterns and SQL queries for that stack — but the methodology carries over to any multi-tenant web app.

---

## Why this exists

A "looks correct" code-review pass misses a whole category of bugs that are invisible until you actually run the thing against real data. Bugs this methodology was built to catch:

- A list view silently **dropping today's records** because an unbounded query hit PostgREST's 1000-row cap and returned the wrong 1000, in the wrong order — no error, just quietly wrong.
- A "reschedule" flow that had **never once worked in production** because a `CHECK` constraint was missing a value the app's own code writes. The paired `INSERT` succeeded and only the follow-up `UPDATE` failed, so users saw a confusing generic error next to a half-applied change.
- Six dashboard metrics reading **zero forever** from a casing mismatch (`'completed'` vs `'Completed'`).
- Soft-deleted rows **leaking into six unrelated screens** because one `.is('deleted_at', null)` was missing and nobody grepped the rest.
- A public API endpoint that verified the caller's tenant but **never their role**, letting any user trigger an owner-only action inside their own tenant.
- A broad `FOR ALL` RLS policy silently making every narrower "only the Owner can…" policy on the same table **decorative**.

None were visible from reading the UI once. All were found by refusing to trust the first plausible-looking result.

---

## What it does

It also handles a bigger ask: **"audit the whole app, module by module, until every feature is covered."** A single-feature audit and a full-app sweep use the same six phases below, but a full-app sweep is too big for one context window and needs to survive a context reset or a brand-new session picking it up cold — so the skill sets up a persistent tracker file first (every module enumerated up front, a status per module, a session log, and a dedicated place for cross-cutting findings once the same bug shape turns up in two or three unrelated modules). It also knows the sharpest failure mode unique to full-app auditing: a fix made while auditing module N — especially a new permission/RLS restriction — can regress a module M that was already marked done, if the restriction was modeled only on module N's own dedicated page instead of on every place in the app that touches the same data.

Six phases, run in order:

| Phase | What happens |
|---|---|
| **1 — Static code audit** | Every query against the entity gets checked against a fixed list: row-cap/ordering safety, `deleted_at` filtering, React Query cache-key collisions, cache-invalidation completeness, enum/constraint integrity, legacy-column drift, list-key stability, write-failure handling, explicit-column select coverage, destructive-action confirmation, RLS/permission backing. |
| **2 — Live browser testing** | Drive the real deployed app (or dev server) with browser automation. Exercise *every* control, not just the happy path. Combine filters. Check flow-through to every other screen after a mutation. Reload and re-check. Confirm every "it worked" against the database directly. |
| **3 — Scale & edge cases** | Manufacture the conditions small test data can't reveal: `generate_series` past pagination and row-cap boundaries, soft-deleted / legacy / boundary-value records, fill-rate comparisons, active multi-tenant isolation probes. |
| **4 — Fix, verify, deploy, re-verify** | Root-cause before patching. Type-check and build. One focused commit per fix. Migrations require explicit human confirmation. Confirm the deployed chunk is actually live before re-testing. |
| **5 — Cleanup** | Remove every test row, revert every temporarily-mutated record, and verify the cleanup the same way you'd verify a fix. |
| **6 — Report** | Concrete failure scenario + root-cause mechanism + how it was verified, for every finding. Honest about what wasn't tested. Flags bug *classes*, not just instances. |

### The bug-pattern catalog

[`skills/deep-saas-tester/references/known-bug-patterns.md`](skills/deep-saas-tester/references/known-bug-patterns.md) is a catalog of **39 recurring bug classes**, each with:

- a general **mechanism** description,
- a concrete **"how to find more instances"** check (a grep, a SQL query, a comparison), and
- an **anonymized example**.

Claude checks every feature against every pattern in this file — because cache-key collisions, unbounded queries, and missing `deleted_at` filters each recur across unrelated modules. Each pattern has a per-project **instance log** you fill in as you go, so a bug found once isn't re-found the hard way. After every audit, the skill appends any newly discovered pattern to the catalog.

---

## Install

### Option A — as a plugin (recommended)

In an interactive `claude` session:

```
/plugin marketplace add MikeRyanKS/claude-deep-saas-tester
/plugin install deep-saas-tester@mikeryanks-skills
```

To auto-enable it for a whole team, commit this to the repo's `.claude/settings.json`:

```json
{
  "extraKnownMarketplaces": {
    "mikeryanks-skills": {
      "source": { "source": "github", "repo": "MikeRyanKS/claude-deep-saas-tester" }
    }
  },
  "enabledPlugins": ["deep-saas-tester@mikeryanks-skills"]
}
```

### Option B — as a plain skill

Copy the skill folder into your personal skills directory:

```bash
git clone https://github.com/MikeRyanKS/claude-deep-saas-tester
cp -R claude-deep-saas-tester/skills/deep-saas-tester ~/.claude/skills/
```

Or into a single project (checked into that repo, scoped to it):

```bash
cp -R claude-deep-saas-tester/skills/deep-saas-tester <your-project>/.claude/skills/
```

Restart the Claude Code session to pick it up.

---

## Use

Claude invokes it automatically when you ask it to **test / deep test / audit / QA / stress-test / find bugs in / carefully verify** a feature, or ask *"have you actually tested X?"*. You can also invoke it explicitly:

```
/deep-saas-tester the invoicing module
```

Good prompts:

- `Deep test the appointments booking flow — code, live UI, and the database.`
- `Audit the reports dashboard. I think some numbers are wrong.`
- `Before we ship the new permissions screen, stress-test it like a real QA engineer.`
- `/deep-saas-tester everywhere customer data surfaces, not just the profile page`
- `/deep-saas-tester the whole app — all features, one by one, till they're all done`

It works best when Claude has:

- **a SQL execution tool** connected (a Supabase MCP server, or equivalent) for ground-truth checks,
- **browser automation** available for the live pass, and
- a **running dev server or deployed URL** to test against.

It degrades gracefully without them — it'll do the static pass and tell you what it couldn't verify.

---

## Contributing

Found a bug class this catalog doesn't cover? Open a PR adding it to `known-bug-patterns.md` using the template at the bottom of that file: mechanism, a repeatable "how to find more instances" check, and an anonymized example. Keep project-specific instance logs out of the shared catalog.

---

## License

[MIT](LICENSE) © 2026 MikeRyanKS

Not affiliated with or endorsed by Anthropic. "Claude" and "Claude Code" are trademarks of Anthropic.
