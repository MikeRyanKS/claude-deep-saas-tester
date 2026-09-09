# Known Bug Patterns — SaaS on Supabase + React Query

A catalog of bug *classes* found while deep-auditing multi-tenant SaaS apps built on Supabase (PostgREST + Postgres + RLS) and React Query, organized by pattern rather than by feature, because these patterns recur across unrelated modules.

**How to use this file:**
- Before auditing any feature, read every pattern here and check the feature against each one, even the ones that seem unrelated.
- Each pattern has a **mechanism** (general, reusable), a **how to find more instances** check (a concrete grep/SQL/comparison), and an **anonymized example** so the shape is recognizable.
- Keep a per-project **instance log** under each pattern as you find real occurrences — use the template at the bottom. Instance logs are project-specific; the mechanism and the "how to find" check are not. When copying this skill into a new project, the instance logs start empty.
- Terminology: `<tenant_key>` = whatever column carries tenant identity in this codebase (`tenant_id`, `org_id`, `account_id`, `workspace_id`, `company_id`, …). "Member" = a row in the users/staff table. "Terminal status" = a status value after which a record is meant to be locked.

---

## 1. Unbounded queries silently truncated by PostgREST's default row cap

**Mechanism.** A query with no `.order()` and no `.limit()` / date-bound fetches *every* row for its filter and gets capped at PostgREST's default 1000 rows, in whatever order Postgres happens to return them — usually not chronological, and not the newest-first order a human expects. The failure is silent: no error, just a plausible-looking but wrong dataset. When the cutoff falls before "today," an active feature can show stale/historical data indefinitely while presenting as fully functional.

**How to find more instances.** For every `useQuery` against a table, ask "if this tenant had 5,000 rows in this table, would this query still return the ones that matter?" Then check the actual row count via SQL (`select count(*) from <table> where <tenant_key> = ...`) and compare to what the query would return. Fix by scoping to the actually-visible window (a date range, the ids actually referenced by already-loaded parent rows) rather than fetching all history.

**Anonymized example.** A calendar's appointment query had no order/limit; the tenant had 3,256 active rows; the returned 1000 started ~10 months in the past and never reached "today," so the calendar showed year-old data while silently omitting the current day.

**Instances in this project:**
- _(none logged yet)_

---

## 2. Missing `deleted_at` filtering lets soft-deleted rows leak back in

**Mechanism.** A soft-delete (`deleted_at = now()`) is meaningless if even one downstream query forgets `.is('deleted_at', null)` — the row reappears exactly as if never deleted, often in a completely different part of the app than where it was deleted from, which makes the bug look unrelated to deletion at all. Applies equally to **embedded/joined** selects (`table:related(...)` where the related table has its own `deleted_at` and RLS doesn't exclude soft-deleted rows) and to **service-role edge functions** (which bypass RLS entirely).

**How to find more instances.** Once found in one query against a table, grep *every* query against that same table in one pass: `grep -rl "from('<table>')" src` plus `grep -rn "<table>:" src` for embedded selects plus `grep -rn "<table>" supabase/functions`. This pattern has never appeared as a single isolated instance.

**Anonimized example.** One sweep found the same missing filter in a record's own detail tab, the main dashboard's "today" widget, a list-view "next X" column, two report queries, a CSV export column, a notification-scheduling lookup, and a public booking edge function's match-by-email step (which would silently re-attach new data to a deliberately-deleted record).

**Instances in this project:**
- _(none logged yet)_

---

## 3. React Query cache-key collisions

**Mechanism.** Two components fetch the same entity under the identical `queryKey` array but with different `.select()` shapes or filters. Whichever populates the cache first "wins" its shape for every consumer until `staleTime` elapses — so a partial preview fetch can silently serve its narrow shape to a component expecting the full row (an unguarded access on the missing columns then crashes it), or a capped fetch serves its arbitrary subset to a component that needed all rows.

**How to find more instances.** Grep the entity name across the whole `src/` tree, not just files you already know touch it, and diff every query's exact `.select()` list and filter set against every other query sharing (or nearly sharing) its key. Fix by giving preview/partial-shape queries their own context-named key (`entity-preview`, `entity-list`, `entity-sidebar`), never a second differently-shaped `['entity', id]`.

**Anonymized example.** `['records-select', tenantId]` was shared by three components: one capped with a narrow select, one uncapped narrow, one uncapped *with* extra search columns. Whichever loaded first either broke search in a modal for up to 5 minutes or showed "Unknown" on a board view.

**Instances in this project:**
- _(none logged yet)_

---

## 4. Missing or incomplete cache invalidation after a mutation

**Mechanism.** A mutation succeeds in the database but doesn't invalidate every query key that displays the affected data, so the UI shows stale state until a full reload — often on a *different* page/tab than the one the mutation happened on. **Sub-pattern:** a query key built as one hyphenated string (`'financial-reports-aging'`) instead of a proper array breaks `invalidateQueries({ queryKey: ['financial-reports'] })` prefix matching even when the entity name looks right at a glance.

**How to find more instances.** For every `insert` / `update` / `delete`, list every screen that could plausibly show this entity, then actually visit each one immediately after the mutation *without reloading* and confirm it updated. Also check sibling handlers in the same file — often one is correct and the one right next to it invalidates a subset.

**Anonymized example.** A drawer's delete handler invalidated only the dashboard key, while the status-change handler directly above it correctly invalidated the detail, list, and parent-tab keys — so deleting left the row visible on the list and the parent's tab until reload.

**Instances in this project:**
- _(none logged yet)_

---

## 5. DB constraint doesn't actually allow a value the app's code writes

**Mechanism.** The app's TypeScript, UI copy, and status-badge styling can all treat a value as fully first-class while the database's own `CHECK` constraint has never included it — so every write of that value fails at the DB layer. The failure often surfaces as a *different*, more confusing symptom: if the value is written by the second of two chained writes, the first write succeeds and the user sees a generic "could not save" error next to a partially-applied change.

**How to find more instances.** For every enum/status column, pull the actual constraint (`select pg_get_constraintdef(oid) from pg_constraint where conname = '<table>_<col>_check'`) and diff it against every literal value referenced in code and UI. Also run `select count(*) from <table> where <col> = '<suspect value>'` — if a value the code clearly writes has a real-world count of zero, that's the tell. This pattern is frequently *not* caught by code review even when it's in this catalog — the live click that triggers the write and reads the real `23514` error is what proves it.

**Anonymized example.** A `status` column never allowed `'Rescheduled'` despite the edit flow writing it, the dropdown offering it, and its badge color existing. The reschedule flow's `INSERT` of the replacement record succeeded, then the `UPDATE` marking the original failed with `23514` — user saw "could not save" with a new duplicate already created. Fixed via a migration (applied with explicit user confirmation).

**Instances in this project:**
- _(none logged yet)_

---

## 6. Casing mismatches between hardcoded string literals and actual DB values

**Mechanism.** A comparison like `status === 'completed'` silently never matches when the database stores `'Completed'` — no error, the branch just never fires, and the resulting metric quietly reads zero forever.

**How to find more instances.** Grep a file for `=== '[a-z]`, `!== '[a-z]`, and `['<lowercase>']` bracket lookups, then verify each suspect's real casing via `select distinct <col> from <table>` before fixing — don't guess the correct casing from convention.

**Anonymized example.** One reports file had six instances in one pass — `'completed'` / `'no_show'` / `'cancelled'` / `'finalized'` / `'procedure'` / `'waiting'` where the DB stored title-case or hyphenated variants — silently zeroing six real dashboard metrics since inception.

**Instances in this project:**
- _(none logged yet)_

---

## 7. Parallel/legacy columns for the same concept

**Mechanism.** Two columns exist for the same real-world fact because the write path changed at some point (an older seeded column vs. the one the app currently writes). Code that reads only one silently breaks for every row written via the other path, and which one is "currently correct" isn't obvious from the column name. A close variant: a related-but-distinct column left `NULL` because one flow never sets it (a quick action that skips a step the full flow does).

**How to find more instances.** When a field renders "Unassigned" / blank for rows that clearly have real data elsewhere in the app, check whether a second, similarly-named column exists on the same table before assuming the data is missing. Grep for `<concept>_id` variants. Confirm the read uses `current_col ?? legacy_col` with a documented comment.

**Anonymized example.** A reports "staff performance" query read only the legacy `owner_id` column; every record booked through the current app writes `assignee_id`, so live-app activity never counted toward the stats. A sibling component already had the `assignee_id ?? owner_id` fallback with a comment; this one hadn't adopted it.

**Instances in this project:**
- _(none logged yet)_

---

## 8. React list-key collisions causing ghost/duplicate rows

**Mechanism.** Keying a rendered list on a *derived display value* (a resolved name, a formatted string) instead of a stable id is fine until that value can legitimately collide — most commonly when a lookup map used to resolve the display value starts empty and populates a moment later, so the first render of a tab/table has every row transiently sharing the same fallback label (`"Unknown"`). React's reconciliation across that transition can leave stale first-render rows in the DOM, producing doubled rows that look like a data bug but are pure rendering.

**How to find more instances.** For every list `.map()`, check whether the `key` prop is a stable id field or something derived from a lookup map — especially one gated behind a `visitedTabs` / lazy-load pattern that starts empty. Fix by keying on the stable id.

**Anonymized example.** A staff-performance table keyed on `s.name` — every real member appeared twice, once correctly and once as an identical-numbers `"Unknown"` ghost, because the name-lookup query starts empty on the first render of that tab. Confirmed a data bug was *not* the cause by recomputing the aggregation via a raw `fetch()` in the console.

**Instances in this project:**
- _(none logged yet)_

---

## 9. Unsafe "quick action" that bypasses a multi-step flow's real invariant

**Mechanism.** A dropdown or one-click action offers a state transition as if it's equivalent to a proper guided flow, when the guided flow actually does more work (creates a linked record, updates a reference, stamps a reason) that the quick action skips entirely — leaving the record claiming to be the end result of a flow that never ran.

**How to find more instances.** For every status dropdown / quick-action, compare what it writes against what the "proper" flow for that same transition writes. If the proper flow creates or updates *other* records, the quick action that only flips one column is suspect.

**Anonymized example.** A "quick status update" dropdown offered `'Rescheduled'` as directly selectable, just flipping the column — unlike the real reschedule flow which creates the linked replacement and sets `rescheduled_from_id` / `reason`. Fixed by removing it from the selectable options while still rendering it as a *display* value for records already legitimately in that state: `OPTIONS.includes(current) ? OPTIONS : [...OPTIONS, current]`.

**Instances in this project:**
- _(none logged yet)_

---

## 10. Loading-state and genuine-empty-state are visually identical

**Mechanism.** A component that renders "—", "0 results", or an empty table the instant a filter/view changes — before its dependent lookup query resolves — is indistinguishable on screen from the same component correctly reporting nothing exists. Screenshotting once, immediately after the change, and concluding either "broken" or "empty" is a coin flip.

**How to verify which one it is.** Wait a beat (or perform another action) and check again; if it's still empty/wrong after settling, it's a real bug. Don't wave away a *real* instance as "just loading" without actually confirming, and don't call a transient flash a bug from one screenshot.

**Instances in this project:**
- _(none logged yet)_

---

## 11. Redundant/duplicate queries across sibling components for the same view

**Mechanism.** Two components that are never both visible at once (a List view and a Calendar view toggled by the same page) each independently query the same underlying data, so switching between them — or even sitting on one — does double the DB work for no benefit.

**How to find more instances.** For a page with view-mode toggles, check whether each view's data query has an `enabled: viewMode === 'x'` guard or runs unconditionally.

**Anonymized example.** A page's own list query ran regardless of `viewMode`, duplicating the Calendar child's separate query whenever the user was in Calendar mode. Fixed with `enabled: !!tenantId && viewMode === 'list'`.

**Instances in this project:**
- _(none logged yet)_

---

## 12. A UI control silently does nothing for a subset of records sharing one list/card

**Mechanism.** The same card/row template is reused for records that don't all have the same underlying shape — some have a linked parent record, some don't — and a control on that template (a dropdown, a button) only persists for the subset whose shape it was written against. For the rest, the control still renders as fully interactive, optimistically shows the pick as if it worked, with zero error, and only reverts on the next reload/refetch.

**How to find more instances.** For any card/row component reused across records with even slightly different shapes (nullable foreign keys, optional linked records), check every write handler: does it have a real `else` / fallback branch for records where its assumed shape doesn't hold, or does the write just silently no-op inside an `if (thisFieldExists) { ... }` with nothing after it?

**Anonymized example.** A room-assignment dropdown only ever wrote to `bookings.room_id`, gated behind `if (booking) {}` with no `else`. Records started via a different flow have `booking_id = null`, so for those the dropdown always showed the pick as selected, then reverted to "No Room" on reload — no error at any point.

**Instances in this project:**
- _(none logged yet)_

---

## 13. A resource-scoping variable is read from the wrong source of truth entirely

**Mechanism.** A view that intentionally spans multiple scopes at once (a global board showing every branch/team's records together) still has some *specific control* on it — usually a resource picker (rooms, seats, assignees) — that needs to know which scope a given record belongs to. There are (at least) three candidates, and only one is correct:

1. **Global UI selector state** (`useAuthStore().activeScopeId`, a top-nav dropdown) — wrong whenever the view spans more scopes than whatever's currently selected.
2. **A plausible-looking field already on the record** (e.g. `customer.branch_id`) — looks like exactly the right thing, but is often a frozen *historical* fact (which branch first registered this customer) with zero bearing on which scope a given *event* belongs to. **This is the trap**: easy to "fix" a wrong-global-selector bug by swapping in the record's own field and believe you're done — especially when it now agrees with another view. But if that other view derives from the *same* upstream historical field, agreement proves nothing.
3. **The scope the acting user actually had selected when this specific event/record was created** — usually not captured anywhere yet; has to be added as a new column, stamped at write-time from the session's current-scope selector. This is the one that's actually correct for "which scope did this event happen in."

The trap also applies to a field on the *acting user's own* record: persisting "current working scope" by writing it into a column that also serves a durable HR/admin/billing purpose corrupts that durable record for anyone who ever switches scope.

**How to find more instances.** For any resource-picker or scoping filter on a "spans everything" view, ask explicitly: is the scope concept here *durable per-entity* (set once, rarely changed) or *per-event* (which scope was this specific visit/order/record actually at)? If per-event, check whether the record has its own column for it — if the only candidate is a per-entity field borrowed from a related record, that's almost certainly the trap. Before persisting session-current state onto *any* existing column, ask "does this column already have another owner/purpose that session data would corrupt?"

**Instances in this project:**
- _(none logged yet)_

---

## 14. A shared "does the write, caller must invalidate" utility, called from many sites, where most callers forget

**Mechanism.** A plain (non-hook) utility function performs a real mutation but has no `queryClient` available to invalidate caches itself — so invalidation is left to each call site. With enough call sites, most get it wrong in slightly different ways, and the bug shows up as "screen X didn't update after doing Y elsewhere" — reading as one-off flakiness rather than the systemic gap it is.

**How to find more instances.** `grep -rn "utilityName(" src`, and for each call site list every query key that plausibly displays the data it changed, then check the call site invalidates all of them. The durable fix is centralizing invalidation inside the utility (thread `queryClient` in as a parameter, or via an exported singleton) rather than trusting N independent call sites.

**Anonymized example.** A `transitionState()` util was called from ~9 sites; only the one handler built around it invalidated the `['active-state', id]` key. Three others (a "new record" flow racing the drawer it opened, a finalize auto-transition, an assign auto-transition) didn't, so an open drawer could show a stale status from earlier in the session.

**Instances in this project:**
- _(none logged yet)_

---

## 15. Testing-process pitfalls (not app bugs — gotchas in how you verify)

These are ways this exact audit process has previously produced a *wrong conclusion*. Keep sharpening this section too.

- **`form_input` over `type`/`key` for native date/datetime-local inputs.** Synthetic keystrokes repeatedly mis-set segmented date inputs (typing `08/25/2026` produced `08/02/52026`). Get the `ref` via `read_page` and `form_input` a plain ISO string.
- **`form_input` isn't universally reliable either.** On a composite/masked input (a dial-code select + free-text pair feeding a computed value), `form_input` produced a saved record with the computed field `null` — looked exactly like a real bug. Redone with `left_click` + `type`, it saved correctly. Before reporting any live-test finding as a bug, if the value was set via `form_input` on anything other than a plain text/date/select field, reproduce once with `left_click` + `type` first.
- **`read_network_requests` capture gaps.** This tool has repeatedly reported zero or far fewer requests than demonstrably fired. Treat sparse/empty as inconclusive; patch `window.fetch` when you need certainty, especially for an error response body.
- **A single browser-session cache can mask a bug a fresh load reveals, or vice versa.** When a live-test result seems to contradict the code you just read, try both a hard reload and a same-session re-test before concluding either way.
- **Bulk-generate scale-dependent test data via SQL** (`generate_series` with a tagged field), not manual UI repetition.
- **Deliberately poison a cache key before testing whether a mutation invalidates it.** Visit the screen that populates the key with a stale value (state A), close it, perform the mutation elsewhere, reopen — if it's still on state A, the invalidation list is missing that key. A plain fresh-reload test never reveals this.
- **When a fix looks locally correct but a cross-view comparison still disagrees, keep pulling the thread.** Each fix's live-verification step is itself a chance to find the next bug — treat an unexpected mismatch during verification as a new lead, not noise.
- **Two views agreeing is not proof either is correct — check whether they share an upstream source first.** Cross-view consistency only counts as verification when the two views are independently derived. When the correct source of truth for a business concept is unclear, ask the domain owner rather than infer it from whichever field has a matching name.

---

## 16. A UI panel reads exclusively from a new structured table while the actual data lives in an older inline column nobody migrated

**Mechanism.** A feature gets rebuilt onto a cleaner data model (a dedicated child/detail table replacing a flat column on the parent), but nothing backfills existing rows and nothing in the new panel falls back to the old column. The UI isn't wrong on any single record you'd check by hand — it's wrong for the *majority* of records, silently, because the old write path (CSV import, a seed script, an older form) never stopped writing the column the new UI no longer reads.

**How to find more instances.** Whenever a feature reads from a dedicated child/detail table for something the parent table *also* has an inline column for, run a count comparison: `select count(*) from parent where inline_column is not null` vs `select count(distinct parent_id) from child_table`. A large gap is the bug, and it's invisible from reading either query in isolation.

**Anonymized example.** An Insurance card queried only the `customer_insurance` child table; the inline `customers.insurance_provider` column (populated by CSV import) was never read. 39% of customers had real data on file and showed "None on file." Fixed with a one-time backfill migration *plus* updating the importer to write the child row going forward.

**Instances in this project:**
- _(none logged yet)_

---

## 17. A column is only ever populated by one specific form's client-side logic

**Mechanism.** A derived/convenience column (an auto-computed classification, a denormalized label) is set correctly by the one form that has the JS to compute it, but the column has no DB default, no trigger, and nothing else derives it on the fly for display. Every row created any other way — import, seed, bulk op, API — has the column permanently `NULL`, and every screen reading the raw column shows blank/dash for those rows. Easy to miss because the one form that populates it is also the one you'd hand-test with. Worst variant: *no* code path populates it, yet a report panel is keyed on it and renders every row as "Unassigned" forever.

**How to find more instances.** For any column described as "auto-filled" / "derived" in the UI, check its real fill rate against a column it's derived from (`select count(derived_col), count(source_col) from <table>`). A big gap, especially with a high `count(source_col)`, means the derivation only happens in one client path. A shared "resolve it live from the source column when null" helper fixes every read site at once without a data migration (when the source is present).

**Anonymized example.** An `age_group` column was set only by one form's client-side date math. 99.6% of rows had it `NULL` despite having a `dob`. Fixed with a shared `resolveAgeGroup(dob, stored)` helper at every read site — no migration needed.

**Instances in this project:**
- _(none logged yet)_

---

## 18. A query selects a column that doesn't actually exist on that table

**Mechanism.** A `.select()` list names a column that was never real. PostgREST rejects the whole request with `42703`, `if (error) throw error` fires, and `useQuery`'s `data` falls back to `[]` / `undefined` — so it presents as "this feature has no data" rather than "this feature is broken." Pre-existing code doing this is easy to miss because it looks like every other query in the file; only the column name is wrong. Once you fix the first bad reference in a query, check every remaining column name in that same `.select()` — the same false assumption ("the parent table probably has this field because the child does") tends to produce more than one.

**How to find more instances.** Don't trust a table's shape from a TypeScript type or a sibling query. Pull the real column list (`select column_name from information_schema.columns where table_name = '<table>'`) and diff it against every `.select()` string touching that table. Per-event attribution fields often live one join away, not duplicated onto every table that logically relates to them.

**Anonymized example.** A billing report query selected `billing.branch_id` and `billing.assignee_id` — neither exists on `billing` (both live on `plans`, one join away). The query 400'd on every request, making Total Outstanding and both Aging tabs show zero for every tenant, permanently — live-confirmed against a tenant with 554 unpaid invoices that were completely invisible. A later same-session fix for a *different* bug reintroduced the identical mistake.

**Instances in this project:**
- _(none logged yet)_

---

## 19. A `GENERATED ALWAYS` column is written explicitly by app code

**Mechanism.** A column computed from other columns on the same row (`GENERATED ALWAYS AS (...) STORED`) can never be written directly — Postgres rejects the entire insert/update with `cannot insert a non-DEFAULT value into column`. If the app computes that value client-side and includes it in the payload "to be safe," every write through that path fails outright.

**How to find more instances.** `select column_name, generation_expression from information_schema.columns where is_generated = 'ALWAYS'` lists every such column. For each, grep every `.insert(...)` / `.update(...)` payload for that column name — any hit is a guaranteed-failing write. Bonus check: does any client-side *preview* of that value use a different formula than the real `generation_expression`?

**Anonymized example.** A `variance` column was `GENERATED ALWAYS AS ((actual - opening) - expected) STORED`. The create mutation computed `variance` client-side and included it in the `.insert()` — every submission failed. The client-side preview also used `actual - expected`, missing the `opening` term.

**Instances in this project:**
- _(none logged yet)_

---

## 20. A broad PERMISSIVE RLS policy silently overrides narrower role-gated ones on the same table

**Mechanism.** Postgres ORs together every PERMISSIVE policy that applies to a command — so a single broad policy covering `ALL` with only a tenant-isolation check (`<tenant_key> = fn_jwt_tenant_id()`) is by itself sufficient to authorize the command for *any* role in the tenant, even when narrower correctly-scoped policies also exist ("only Owner can change role"). The narrower policies aren't wrong — they're decorative, because the broad one already grants everything they'd restrict.

**How to find more instances.** For every table with more than one policy for the same command, pull them all (`select policyname, cmd, permissive, qual, with_check from pg_policies where tablename = '<table>'`) and check whether any PERMISSIVE policy's condition is a strict subset of another PERMISSIVE policy's condition on the same command. Special attention to a table-wide `<table>_all` / `<table>_tenant_all` style policy sitting alongside anything named `..._owner_only_...` or `..._admin_...`.

**Anonymized example.** A `members_tenant_all` (PERMISSIVE, `ALL`, tenant-check only) sat alongside `members_update_owner_only_role` — the broad policy alone authorized any member to `UPDATE` (including `role`) or `INSERT` any member row, bypassing the intended Owner-only rule. Fixed by dropping the broad policy and replacing with per-command policies (tenant-wide SELECT, Owner/Admin-only INSERT/DELETE, a narrow self-update policy with sensitive columns locked to their existing value via a `ROW(...) IS NOT DISTINCT FROM ROW(...)` check).

**Instances in this project:**
- _(none logged yet)_

---

## 21. A record type meant to be permanently locked after a status transition has no lock enforced below the UI

**Mechanism.** A detail view hides its edit/status controls once `status` reaches a terminal value (`'Finalized'`) — but if the DB has no RLS policy or trigger gating writes on that same condition, the lock is UI-only. Anyone who can reach the table's write policies (any role with ordinary write access, a direct API call, a future code path that forgets the check, a bug in the hiding logic) can still mutate a record meant to be permanently closed.

**How to find more instances.** Grep for `isFinalized` / `isLocked` / `status === '<terminal>'`-style conditionals that hide a button or disable a control, then check whether a RESTRICTIVE RLS policy (or DB trigger) enforces the same condition on `UPDATE` / `DELETE` for that table.

**Anonymized example.** A payroll-run drawer hid Mark-Paid/Finalize once `status === 'Finalized'`, but no RLS policy referenced `status` — any role with payroll write access could still `UPDATE`/`DELETE` a finalized run via direct API. Fixed with RESTRICTIVE policies (AND'd against every other applicable policy) requiring the run to still be `'Draft'`.

**Instances in this project:**
- _(none logged yet)_

---

## 22. A CSV import field's alias list shares an entry with another field's alias list, so the auto-mapper silently steals a column

**Mechanism.** An `autoMap()` that walks the field list in declaration order and returns the *first* field whose `key` / `label` / any `alias` normalizes to match a CSV header. If two fields' `aliases` both contain the same string, every CSV with a header matching that string always auto-maps to whichever field is declared *first* — silently, with a mapping that looks plausible on the Map Columns screen. Downstream, if two source columns map to the same target key, values often get *concatenated*, so the losing field's text pollutes the winning field's value — reading as corrupted data rather than an obviously missing field. Especially dangerous when the colliding alias is also the literal header in the app's *own* downloadable template.

**How to find more instances.** For every field-definition file exporting a `FieldDef[]`, flatten `key + label + aliases` per field and look for any string appearing in more than one field. Pay extra attention to generic header words (`name`, `description`, `location`, `type`) sitting on a field they don't obviously belong to. Live-test by uploading a CSV built from the app's own template and checking the Map Columns dropdowns, not just whether the import completes.

**Anonymized example.** An inventory `name` field aliased `'description'`, colliding with the actual `description` field. Since `name` is declared first, every CSV with a "Description" column (including the app's own template) mapped it onto Name and concatenated: `"Root Canal Therapy Endodontic treatment"` in Name, blank Description.

**Instances in this project:**
- _(none logged yet)_

---

## 23. A `public`-role RLS policy has no tenant scoping — anyone can read every tenant's data for that table

**Mechanism.** A table needing *some* unauthenticated (anon) read access — for a public widget, a booking page, an embed — gets a PERMISSIVE `SELECT` policy for role `public` that filters on a business condition (`is_active = true`) but has no `<tenant_key>` condition. Anon requests carry no JWT, so `fn_jwt_tenant_id()` is null and there's no way to scope such a policy per-request — so it grants read of *every* tenant's rows to *anyone*, unauthenticated. Easy to miss because the `qual` looks like a normal filter unless you specifically check for a tenant term and don't find one.

**How to find more instances.** `select tablename, policyname, roles, qual from pg_policies where roles::text ilike '%public%' or roles::text ilike '%anon%'`. For each, check whether `qual` references the tenant key. If not, treat it as a leak until proven otherwise — then check whether anything legitimate needs it (often the real public consumer goes through a service-role edge function and queries the table directly never happens, so the safest fix is to drop the policy). Verify by hitting the table's REST endpoint with only the anon key and confirming it returns nothing.

**Anonymized example.** A `*_public_read` policy on an availability table (`SELECT`, role `public`, qual `is_active = true AND deleted_at IS NULL` — no tenant key). An unauthenticated REST request returned rows for multiple tenants in one response. Nothing legitimate depended on it (the public page used a service-role edge function). Fixed by dropping it.

**Instances in this project:**
- _(none logged yet)_

---

## 24. A service-role edge function is exempt from RESTRICTIVE / billing-gate RLS and needs its own explicit check

**Mechanism.** RESTRICTIVE billing-lock policies (gating writes on `fn_tenant_is_writable()`) are scoped to role `authenticated` by design. A Supabase Edge Function running on the **service-role key** bypasses RLS entirely, RESTRICTIVE policies included. Any public/service-role path that creates real data needs its own explicit read of the tenant's subscription/billing status — "the RLS gate will catch it" silently fails for exactly the code most likely to run unattended (public APIs, webhooks, cron).

**How to find more instances.** `grep -l SUPABASE_SERVICE_ROLE_KEY supabase/functions/*/index.ts`, then for each check whether it performs a write reachable by an external non-staff caller and whether it checks the subscription/billing status against the same set `fn_tenant_is_writable()` uses (so the two never drift).

**Anonymized example.** A public-booking function checked only a `portal_enabled` flag, never `subscription_status` — a `read_only` / `suspended` / `cancelled` tenant, blocked from every other write, could still receive unlimited real public bookings.

**Instances in this project:**
- _(none logged yet)_

---

## 25. Two IDs submitted together in the same request are never checked for actually belonging to each other

**Mechanism.** An endpoint accepts two related foreign-key IDs in the same payload — a parent (`<tenant_key>`) and a child that should belong to it (`branch_id`, a family member id, a line item under a plan) — and uses both directly without confirming the child belongs to the claimed parent. Every individual `UUID_RE.test()` passes, so validation *looks* complete, but nothing stops a caller pairing a valid parent with a child that belongs to a different parent. Failure mode is subtle: lookups filtering by *both* IDs together silently never match, and the written row ends up with an internally-inconsistent parent/child pair — invisible until something later joins through the child and renders data from the wrong tenant.

**How to find more instances.** For every public or lightly-authenticated endpoint accepting more than one FK ID in the same request, check for an explicit query confirming the second ID's own parent-key column matches the first before either is used in a write or scoped lookup.

**Anonymized example.** A public-booking handler took `tenant_id` and `branch_id` from the body and never verified `branches.tenant_id === tenant_id`. A mismatched pair sailed past every check; the resulting rows carried a `branch_id` belonging to a different tenant. Fixed with a shared `validateBranch()` called before any other logic.

**Instances in this project:**
- _(none logged yet)_

---

## 26. A second write chained after a successful first write isn't wrapped in try/catch

**Mechanism.** A handler does write A, checks A's error correctly, then `await`s a second write B (often a helper that can itself throw) with no try/catch around it. If B fails, the exception propagates out as an unhandled rejection — and since the handler is fired from a plain `onClick` with no caller-side `.catch()`, nothing surfaces to the user. Worse, A already happened and isn't rolled back, so the record is left half-transitioned with no explanation, and any cache invalidation that was supposed to happen after B never runs.

**How to find more instances.** Grep for handlers doing `const { error } = await supabase...; if (error) {...; return}` followed by one or more further `await`s with no enclosing `try/catch`. Check whether the later step can actually throw (a helper doing its own `if (error) throw error` internally is exactly the shape) — if it can and the earlier write already committed something a user would notice, this applies.

**Anonymized example.** A `handleCheckIn` error-checked the status `UPDATE` correctly, but the following `await transitionState(...)` (which throws internally) had no try/catch — a failure there left the record showing "Checked-In" while the follow-on record was never created, no error shown, neither query re-invalidated.

**Instances in this project:**
- _(none logged yet)_

---

## 27. A route/nav-item permission key is coarser than the specific key the role matrix actually grants

**Mechanism.** A permission model has both a broad key (`/settings`) and a narrower key for one nested page (`/settings/catalog`); the role matrix explicitly grants the narrow key to a role that does *not* have the broad one. But the `<Route>` / `RoleGuard` wrapper and/or the sidebar nav config checks the broad key — because the page is structurally nested, it's assumed to inherit the broader gate. Since `hasPermission()` does exact string matching with no prefix logic, the role's explicit grant does nothing — they're silently blocked despite the matrix saying they have access. This is the opposite of a leak: a documented capability that simply doesn't work.

**How to find more instances.** For every route path with nested structure (`/parent/child`), check whether the router's `requiredPermission` and the nav-item's `permission` use the child's own key or the parent's. Cross-reference against every role's permission array — any role granted the child key but not the parent key is the exact signature.

**Anonymized example.** A `/settings/catalog` page was gated by the blanket `/settings` guard even though the matrix granted `/settings/catalog` to two roles that lack bare `/settings`. Both roles were fully unable to reach it — nav link hidden, URL redirected to `/unauthorized`. Fixed by giving the page its own dedicated guard and nav key.

**Instances in this project:**
- _(none logged yet)_

---

## 28. A role's entire documented write capability has no RLS INSERT/UPDATE policy backing it

**Mechanism.** A role's whole reason for existing (per docs, per the UI it's allowed into) is to create/edit records in some table — but no INSERT or UPDATE policy on that table actually includes that role. The form renders fine, submits fine, and the write is rejected by Postgres with a generic RLS violation that the app's *correct* error handling reports as a generic "could not save" — so the bug hides behind good error handling, not missing error handling. Often introduced when a broad PERMISSIVE policy is split apart (pattern #20) and the replacement per-command policies cover Owner/Admin/self but nobody asks "does every role the docs mention still have a path in here?"

**How to find more instances.** For every role, and every table the docs/UI say that role can create or edit, pull `select policyname, cmd, qual, with_check from pg_policies where tablename = '<table>'` and check whether ANY policy for `cmd IN ('INSERT','UPDATE')` includes that role (by name or via an override function). If none does, the role's write fails 100% of the time — Postgres RLS requires at least one applicable PERMISSIVE policy to pass.

**Anonymized example.** An HR role had route access and a fully-wired employee drawer but zero INSERT and zero UPDATE policy on `members` included role `HR`. Every HR-initiated create/edit was rejected, always. Fixed with a scoped INSERT policy (`role = 'Support Staff'` required) and a no-role-change UPDATE policy.

**Instances in this project:**
- _(none logged yet)_

---

## 29. A stale/typo'd role name in an RLS policy's role array silently excludes the role that was meant

**Mechanism.** A policy's role-check array (`fn_jwt_role() = ANY (ARRAY[...])`) contains a string that isn't — and never was — a real role value. It doesn't error; Postgres compares against a literal that can never match. Since the array also contains correct role names, the policy *looks* complete on a skim. The blast radius is a privilege *inversion*: whichever real role was meant ends up with LESS access than a nominally-lower-privilege role also in the array.

**How to find more instances.** Collect every distinct role-like string appearing in any policy's `qual` / `with_check`, then diff against the canonical role list (`staff_role_check` CHECK constraint, or the `AppRole` TS type). Any string in a policy but not in that list is a placeholder (rare, should be commented) or a bug.

**Anonymized example.** A `billing_all` `WITH CHECK` allowed writes to finalized billing rows for `Owner, Administrator, Receptionist, Accountant` — `'Accountant'` has never been a valid role (it was renamed to `Finance`). Effect: Finance, the role billing exists for, could never edit a finalized row while Receptionist could.

**Instances in this project:**
- _(none logged yet)_

---

## 30. A privileged service-role edge function checks WHO the caller's tenant is but not WHAT their role is

**Mechanism.** An edge function performs a genuinely privileged action (sending an invite that consumes a paid seat, finalizing a payout) using the service-role key, and correctly verifies the caller is an authenticated member of the claimed tenant — but stops there, never checking the caller's own role. Tenant isolation is intact (a low-privilege user can't do this to a *different* tenant), but *within* their own tenant, any role can trigger an owner-only action. Easy to miss because the function's comments focus entirely on the tenant-isolation half of the story and never mention role.

**How to find more instances.** For every service-role edge function, check whether it performs a write most base roles shouldn't trigger (granting/revoking access, financial finalization, deleting another member's data) and whether it queries the caller's OWN role and checks it against an explicit allow-list. Pay special attention to functions with an inverse counterpart (grant/revoke, create/delete) — if the counterpart has a role check, treat the absence here as a bug until proven a deliberate exception.

**Anonymized example.** An `invite-member` function (grants a login, consumes a seat) had no role check — any authenticated member of any role could invite an arbitrary email. Its sibling `revoke-member-access` correctly checks `role !== 'Owner' && role !== 'Administrator'` and 403s. Fixed by adding the identical block.

**Instances in this project:**
- _(none logged yet)_

---

## 31. A permission key survives a UI redesign that removes the element it gated

**Mechanism.** A `hasPermission()`-checked key was written to gate a specific UI element (a table column, a section, a button). Later that element is removed or restructured in an unrelated redesign — the developer doing the redesign has no reason to know the key existed, since removing a `<td>` doesn't touch the permissions file. The key survives untouched in the role matrix and stays toggleable in the Staff Permissions UI, silently doing nothing, while the data it protected either disappears or resurfaces elsewhere later with *no* permission check (whoever built the new location had no reason to think one was needed).

**How to find more instances.** For every key in the custom-permission options list, `grep -rln "'<the key>'" src` and confirm at least one hit is an actual `hasPermission(...)` call site, not just the definition. Zero call sites means the key is dead — then `git log -S "<the key>"` to find where it was originally wired, read that diff to see what it gated, and check where that data lives *today*.

**Anonymized example.** A `view_salaries` key gated two salary columns in a roster table. A later redesign replaced those columns and moved salary editing into a drawer tab — nothing re-applied the gate, so the tab was visible to anyone who could open the drawer, and the key kept being offered as a checkbox that did nothing.

**Instances in this project:**
- _(none logged yet)_

---

## 32. A design-system rule lives in docs + tokens, but every screen re-implements it locally and drifts

**Mechanism.** A convention is documented and partly encoded as tokens, but there's **no shared component** enforcing it — so each feature re-declares its own `Record<Status, string>` colour map / local badge function / inline class string. Individually each looks fine; collectively they diverge: the same domain value renders a different colour on different screens, a "neutral" default gets a bright hue somewhere, an undefined token (`bg-success-light`) gets copy-pasted into three files where it silently renders nothing. Cross-screen agreement is *not* evidence of correctness — two screens can share a wrong local map by copy-paste as easily as a right one.

**How to find more instances.** `grep -rn "Record<[A-Za-z]*Status" src` and `grep -rn "_STYLES\|_BADGE\s*[:=]\|function StatusBadge" src` — every hit is a local map that should be a call to one shared component + one central map. For contrast, `grep -rnE "bg-(success|warning|danger|info)-bg .*text-(success|warning|danger|info)\b"` (bright token as text on its own tint ≈ 2:1). For undefined tokens, extract every `--color-*` name from the CSS and grep for `(bg|text|border)-<name>` classes whose `<name>` isn't in that set.

**Anonymized example.** ~50 files: ~6 hand-rolled local badge fns, ~25 status-style maps, ~90 raw `bg-blue-100 text-blue-700` pairs, ~70 vivid-on-tint badges (~2.1:1), 3 undefined tokens rendering nothing. Same status ≠ same colour across screens. Fixed by building the missing enforcement layer (a tone vocabulary + shared components + one central map + an ESLint rule + a WCAG-contrast test).

**Instances in this project:**
- _(none logged yet)_

---

## 33. A single bad row in a multi-row `.insert()` fails the entire batch, and the failure handler blames every row in it

**Mechanism.** Postgres inserts are all-or-nothing per statement: `.insert([row1, ..., row500])` where any one row violates a constraint rolls back the *whole* statement — none of the 500 land. A chunked-insert helper that treats "the chunk returned an error" as "every row in this chunk failed" is accurate about *what happened* but produces a wildly misleading *reason* for the 499 valid rows — the reported message names the one row's actual problem (a duplicate key, a constraint name) attached to hundreds of rows that don't have it. Easy to miss in review because the error-handling *looks* correct.

**How to find more instances.** Grep for `.insert(<array>)` inside a loop that chunks a larger dataset, then check the `if (error)` branch: does it mark every item in the chunk as failed, or retry the failed chunk row-by-row to isolate the actual offender? Live-test by constructing a batch with N valid rows and exactly 1 that violates a real constraint, then checking whether N rows actually landed — not just what the summary UI reported.

**Anonymized example.** A CSV import's `insertInChunks` (shared across many entity types) blanket-marked a whole 500-row chunk as failed on any error. Importing 5 records where 1 had a genuine duplicate reported all 5 as `duplicate key violation`, including 4 brand-new rows. Fixed by retrying a failed chunk's rows one at a time.

**Instances in this project:**
- _(none logged yet)_

---

## 34. A reference field silently resolves to null with no message, while an identically-shaped sibling field warns

**Mechanism.** Two form/import fields resolve a human-readable reference (a name, a phone) to a foreign-key id via the same kind of lookup — one has an explicit `if (name && !id) { warn }` when the lookup fails; the other does the identical lookup with no such check, so an unresolved reference just becomes `null` with nothing said. Easy to miss because the sibling field's correct handling makes the code *around* the bug look complete.

**How to find more instances.** For any validation/transform function resolving more than one name-to-id reference, list every `xMap[xName] ?? null`-shaped resolution and confirm each has an accompanying `if (rawValue && !resolvedId) { message }` block — not just the first or most obviously important one.

**Anonymized example.** A CSV Appointments validator resolved both `assignee_name → assignee_id` and `branch_name → branch_id` the same way, but only `assignee_name` had a warning on failure — `branch_name` silently went to `null` with zero indication. Fixed by adding the identical warning treatment.

**Instances in this project:**
- _(none logged yet)_

---

## Template for a new entry

If a pattern already exists, add to its **Instances in this project** log:

```markdown
- `<file path>` — <one or two sentences: the concrete symptom a user would see, then the actual mechanism/root cause>. Fixed by <the fix, in a sentence>. *(<Feature> audit, <YYYY-MM-DD>)*
```

If the pattern itself is genuinely new, add a `## N. <Pattern Name>` section with:
- **Mechanism** — one paragraph, in general terms, not tied to the specific instance.
- **How to find more instances** — a concrete, repeatable check (a grep pattern, a SQL query, a specific comparison).
- **Anonymized example** — one or two sentences, no project-specific identifiers.
- **Instances in this project** — the real occurrence(s), using the line format above.
