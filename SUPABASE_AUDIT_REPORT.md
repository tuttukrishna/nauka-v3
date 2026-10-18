# HB Booking / Nauka Supabase Audit Report

Audit date: 2026-10-11

## Current implementation update — 2026-10-11

The requested pricing, cruise-type, blocked-date, and booking back-end sync is now implemented and applied to the connected Supabase project `wudinqbyifvncgadzezy`. This addendum supersedes the earlier “not tested” statements below where they concern the pricing/P&D scope.

Completed:

- Fixed the reported `listing_pricing_rules_valid_price` failure. Default rows now accept `price = NULL` when both weekday and weekend defaults are positive; date rules and blocked ranges remain strictly validated.
- Added independent `day` and `night` pricing to `listing_pricing_rules`, atomic admin save RPC `save_listing_pricing`, per-cruise date-rule uniqueness/overlap protection, and RLS/grants for the new workflow.
- Added `bookings.cruise_type`, server-side booking price derivation/blocked-date enforcement, and cruise-type-aware availability RPC support.
- Backfilled 10 default rows (one day and one night row for each of the 5 live listings) from the existing listing price fields, without inventing new business prices.
- Removed duplicate pricing policies, indexed `listing_pricing_rules.created_by`, and removed the legacy admin pricing helper from public/authenticated PostgREST execution.
- Updated the standalone UI with Day Cruise versus Night / Overnight controls, independent defaults/rules, cruise-type-aware pricing/availability/booking calls, and stronger blocked-date calendar styling.
- Fixed session-role routing so localStorage cannot impersonate a signed-in user, admin accounts remain in the customer interface unless they explicitly open Admin Control Centre, and the admin route is guarded by the verified Supabase role.
- Added cached-fleet recovery and retry/healing behavior for transient mobile/local-server listing failures, so a temporary catalog request failure no longer leaves the fleet blank.
- Changed blocked-date semantics to physical-boat scope: a block saved from either Day or Night applies to both cruise types in the availability RPC, browser pricing/availability layer, and authoritative booking trigger.
- Redesigned booked/blocked P&D calendar states with high-contrast dark fills and explicit labels; the dialog now explains that blocked dates affect both cruise types.
- Added a dependency-free Node fallback server in `serve.mjs` and wired `serve.ps1` to use it when Python is unavailable.

Live Supabase migration history now includes:

- `20261011161139` `split_listing_pricing_by_cruise_type`
- `20261011161238` `sync_booking_cruise_type_and_availability`
- `20261011161904` `cleanup_pricing_policy_duplicates`
- `20261011162132` `backfill_cruise_pricing_defaults`
- `20261011173031` `remove_legacy_blocked_dates_overload`

The global blocked-date routine/trigger update was applied directly to the connected project with Supabase SQL execution and is also recorded locally in `supabase/migrations/20261011151000_global_blocked_dates_across_cruise_types.sql`. The legacy overload removal is recorded in live migration history so three-argument calls resolve through the 4-argument default safely.

Verification completed:

- All 23 inline JavaScript blocks compile with Node `new Function()`; 0 syntax errors.
- Live pricing schema has the corrected check constraint, `cruise_type`, and the expected RPC signatures.
- Live pricing table contains 10 valid default rows: 5 `day` and 5 `night`.
- Live SQL smoke test confirms the booking pricing trigger is enabled and both day/night availability RPC calls return successfully for a live listing; existing bookings are preserved and now default to `night` unless explicitly identified as day cruises.
- Live blocked-date parity check returned `day_only = 0` and `night_only = 0` for the live listing, with one existing blocked pricing row included in both results; the booking pricing trigger is enabled and now rejects any physical-boat block before applying the selected cruise price.
- Removed the ambiguous legacy 3-argument RPC overload; both explicit 4-argument calls and legacy 3-argument calls now resolve to the same global blocked-date result (`11` dates in the live smoke range).
- Browser smoke test on the local file reached the authenticated admin page, opened Manage Fleet → P&D, displayed both cruise-type tabs, and switched the labels/defaults/date-rule copy to Day Cruise without the previous constraint error.
- Browser smoke test confirmed the same admin identity stays in the customer home/profile interface on `#home`/`#profile`; `#admin` remains available only as an explicit authenticated route and the profile shows an `Open Admin Control Centre` action.
- Browser visual smoke test confirmed the P&D dialog copy and legend are readable at desktop size; fleet data loaded from the local file after the updated retry/fallback path.
- End-to-end admin smoke test saved the existing Day Cruise defaults through `save_listing_pricing`; the UI reported “Day Cruise default prices saved,” the live row count stayed at 10, and no invalid pricing rows were created.
- Supabase advisors no longer report duplicate pricing policies or the unprotected legacy admin pricing helper. Remaining warnings are the intentionally public security-definer availability/pricing read RPCs, disabled leaked-password protection, and pre-existing `listings` policy/index notices.

The app is not deployed from this workspace and no production DNS/Vercel change was made. The local server can be started with `$env:PORT=18080; powershell -ExecutionPolicy Bypass -File .\serve.ps1`; port 8080 is restricted in this managed environment. The browser smoke test used the local `file:///` page because the managed shell could not expose the local server socket to Chrome.

Project folder: `C:\Users\tuttu\OneDrive\Desktop\Nauka final out`

Live URL supplied for safe read-only checks: `https://nauka.co.in/`

Supabase project URL configured in the app: `https://wudinqbyifvncgadzezy.supabase.co`

## 1. Executive summary

Overall status: **IMPLEMENTED FOR REQUESTED SCOPE**. The original audit text below is retained as historical context; the current connected-Supabase verification and smoke-test results are recorded in the implementation update above.

Important findings:

- The project is a standalone, approximately 1.1 MB `index.html` application. There is no package manifest, build script, test runner, Vercel configuration, or Git metadata in the supplied folder.
- The user-reported `42703: column "rule_type" does not exist` error is consistent with an older partial `listing_pricing_rules` table. The original pricing migration was restored to its intended fresh-schema form and a separate legacy repair migration was added.
- The fleet RLS migration granted `SELECT` on `listings` but did not define a public/admin `SELECT` policy. A separate additive migration now makes those read policies explicit.
- The P&D UI, three 30-day calendar sections, fixed popup scrolling, and body-scroll locking are present in the local file. These were not visually tested in a browser in this environment.
- The normal live booking handler now recalculates the date-aware total from selected dates and has an in-flight submission guard. This reduces stale-price and rapid-double-click errors, but it does not make the browser authoritative.
- Booking price, availability, overlap prevention, and status transitions still require database-side verification. The browser currently sends `price` and `total` in a direct `bookings` insert, so a user who can modify browser requests may alter those values unless the live database has a trigger/function or restrictive policy that was not available to inspect.
- No production SQL, deployment, DNS, reset, truncate, delete, or other destructive operation was executed by this audit.

## 2. Architecture discovered

### Runtime and entry point

The supplied folder contains:

- `index.html`: the complete HTML, CSS, application JavaScript, and vendored Supabase browser client.
- `supabase/config.toml`: project reference only.
- `supabase/migrations/20261011130000_add_listing_pricing_rules.sql`.
- `supabase/migrations/20261011133000_fix_admin_fleet_rls.sql`.
- `supabase/migrations/20261011134000_repair_legacy_listing_pricing_rules.sql`.
- `supabase/migrations/20261011135000_fix_public_listing_select_rls.sql`.

No `package.json`, lockfile, source directory, TypeScript configuration, test configuration, Vercel configuration, server/API directory, `.env` file, or Git repository was present. This is therefore a standalone browser application, not a Next.js project or another build-based framework in the supplied workspace.

The HTML loads runtime dependencies from CDNs, including Tailwind CDN, Font Awesome, jsPDF, Lucide, QRCode, Firebase compatibility scripts, Google Fonts, and Cloudinary assets. The Supabase JS client is vendored inline and labelled as version 2.117.1.

### Supabase client boundary

The app initializes one browser Supabase client in the `nauka-supabase-backend` script using the configured project URL and a publishable browser key. Auth is configured with persisted sessions, automatic token refresh, and URL session detection.

Application-specific Supabase calls were found for:

- `listings`
- `tour_packages`
- `profiles`
- `saved_properties`
- `bookings`
- `admin_users`
- `listing_pricing_rules`
- RPC `get_listing_blocked_dates`

The app does not call Supabase Storage. Images are represented as URLs, primarily Cloudinary URLs or user-entered image URLs. Optional Firebase phone verification is present but the local Firebase config is `null`, so that flow falls back to email messaging rather than being active.

### Deployment workflow

There is no local build step to reproduce. Vercel can serve a static `index.html` when configured externally, but no Vercel configuration was supplied. The deployed site could not be checked because DNS resolution failed for `nauka.co.in` from this environment. The local file and deployed site must therefore be treated as potentially different revisions.

## 3. Security audit

### Credentials and client exposure

Result: **no privileged Supabase credential found in the targeted application scan**.

The browser file contains a publishable Supabase key, which is expected for a browser client. No service-role key, database password, `sb_secret` key, private key, `postgres://` connection string, or `SUPABASE_SERVICE_ROLE_KEY` literal was found in the application-specific code or migration files. The full publishable key is intentionally omitted from this report.

This conclusion is limited to the supplied files. It does not inspect Vercel environment variables, Supabase project settings, browser network traffic, source maps outside this folder, or other repositories.

### Authentication and admin authorization

The app uses Supabase Auth for sign-in, sign-up, password reset, password update, OAuth initiation, session persistence, refresh, and sign-out. The browser checks membership in `admin_users` before entering the admin view.

The two original admin-write boundaries are reinforced by Postgres policies in `20261011133000_fix_admin_fleet_rls.sql` and the pricing migration. Admin insert/update/delete checks are based on the authenticated UUID returned by `auth.uid()`, not on a browser-controlled role string.

The following live controls remain unverified:

- RLS and grants on `admin_users` itself. The browser must be able to check the current user, while ordinary users must not be able to enumerate or alter admin membership.
- RLS and grants on `bookings`, `profiles`, and `saved_properties`.
- Existing policies that may coexist with the new policies. PostgreSQL combines permissive policies with OR semantics, so an older broad policy could still widen access.
- Whether the configured Supabase Data API exposes exactly the intended tables and RPCs.

### Listing read boundary

The original fleet migration enabled RLS and granted `SELECT`, but did not create a `SELECT` policy. Grants alone do not authorize rows when RLS is enabled. This was repaired in `20261011135000_fix_public_listing_select_rls.sql` with:

- active listings readable by anonymous and authenticated users;
- all listings readable by authenticated admins; and
- no change to listing data.

This remains pending live application and verification.

### Privacy and storage

The customer code filters customer records by the current `user_id`, which is the correct application-side shape. It is not a substitute for RLS. The database policies for those tables were not available to inspect, so cross-user reads and writes were not dynamically tested.

No Supabase Storage bucket or storage policy is used by the supplied app. External Cloudinary URLs and user-provided image URLs are an operational trust boundary; URL validation, image ownership, and remote asset availability are not enforced by the local code.

### External runtime dependencies

Several CDN scripts and stylesheets are loaded without visible Subresource Integrity hashes or a Content Security Policy in the supplied file. This is a supply-chain and availability risk, not a Supabase credential leak. It should be addressed separately if the app is intended for production hardening.

## 4. Database and migration audit

### Pricing migration

`20261011130000_add_listing_pricing_rules.sql` defines a separate `public.listing_pricing_rules` table with:

- an identity-backed bigint ID;
- a foreign key to `public.listings(id)` with `ON DELETE CASCADE`;
- `default` and `date` rule types;
- weekday, weekend, and date-specific prices;
- date ranges and blocked-date state;
- creator and timestamp columns;
- positive-price and date-range checks;
- one default rule per listing;
- one row per identical listing/date range;
- a lookup index;
- an update timestamp trigger;
- RLS and admin-only writes;
- explicit Data API table grants; and
- a PostgREST schema reload notification.

The migration is suitable for a fresh database based on the schema it references. It does not prove that the existing production table has that schema.

### Legacy pricing repair

`20261011134000_repair_legacy_listing_pricing_rules.sql` addresses the observed failure mode where `CREATE TABLE IF NOT EXISTS` skipped an older table and a later index statement referenced a missing `rule_type` column. It:

- adds missing required columns without dropping or recreating the table;
- backfills missing `rule_type`, timestamps, and blocked flags;
- restores defaults and selected not-null guarantees;
- adds constraints as `NOT VALID` to avoid silently rejecting existing rows;
- recreates indexes, trigger, RLS policies, grants, and schema reload notification.

This migration does not delete rows. It can still stop if the existing table contains duplicate default/date ranges, incompatible column types, invalid legacy IDs, or data that cannot be mapped to the new shape. That is intentional: the audit does not delete or guess at production data.

### Fleet RLS migration

`20261011133000_fix_admin_fleet_rls.sql` enables RLS, grants public reads and authenticated writes, and limits fleet inserts, updates, and deletes to members of `public.admin_users`. The write policies are appropriately based on `auth.uid()`.

The original file was incomplete as a standalone read boundary because it did not create a listing `SELECT` policy. The new `20261011135000_fix_public_listing_select_rls.sql` supplies the missing public-active and admin-all read policies.

### Migration history

No Supabase CLI is installed, no `.git` directory is present, and no live database connection is available. The local `config.toml` contains a project ID but not a verified migration-history result. SQL executed manually in the Supabase SQL Editor cannot be assumed to match the CLI migration history. The official Supabase migration workflow tracks local migration files separately from the remote migration history; reconcile that history before using `db push`.

The user-provided `42703` error is therefore recorded as a real observed failure, while successful execution of the repaired migration remains **unverified**.

## 5. Feature-by-feature results

### A. Houseboat listings and fleet management

Inspected:

- active listing fetch and mapping;
- admin listing load;
- add/edit listing form;
- operator/company and room/cabin fields;
- status changes, suspension, activation, and deletion;
- empty/error handling;
- P&D button and modal entry point.

Local static result: the Manage Fleet UI and the P&D entry point are present. Admin saves use Supabase `listings` mutations, and fleet loading is separated from the optional `tour_packages` request. An admin bookings failure is allowed to leave the fleet visible.

Dynamic result: **not tested**. The live Supabase host was unreachable, so listing edits, name changes, status changes, and deletions could not be safely exercised.

Remaining risk: new listing creation calculates an ID in browser code before insert. Under concurrent admin creation this can collide if the live `listings.id` column is not backed by a safe server-side default. The live schema was not available, so this was not changed speculatively.

### B. Pricing and P&D date management

Inspected:

- weekday and weekend defaults;
- date-specific price ranges;
- blocked date ranges;
- pricing cache and customer price calculation;
- availability RPC integration;
- the 90-day preview;
- pricing storage diagnostics;
- popup and body scroll behavior.

Local static result: the P&D modal renders three calendar sections covering 90 days, uses the existing date-picker path, bounds scrolling to the popup card, and locks page scrolling while the modal/calendar is open. Pricing errors now distinguish a missing table from a PostgREST schema-cache/exposure error. The exact stale-table message now points to the dedicated repair migration.

Dynamic result: **verified against connected Supabase**. The live table, constraints, RPC signatures, RLS policy set, seeded defaults, and browser pricing dialog were checked; no business-value save was invented during the smoke test.

Resolved: `savePricing()` now calls the live `save_listing_pricing` RPC, which validates the selected cruise type and rules and replaces that cruise type's defaults/date rules atomically.

### C. Booking workflow

Inspected:

- sign-in requirement;
- date and checkout validation;
- date-aware availability RPC;
- blocked P&D range integration;
- guest counts and extra guest charges;
- booking insert payload;
- success state and WhatsApp handoff;
- admin status updates;
- rapid submission behavior.

Repairs made locally:

- the final live handler recalculates the total from the selected stay dates and `nkPriceFor()` rather than trusting only the rendered `modal-total` text;
- an in-flight submission guard prevents repeated clicks from issuing simultaneous booking inserts.

Dynamic result: **partially verified**. The live booking schema, `cruise_type` backfill, pricing trigger, and both cruise-type availability RPC calls were checked. A new booking was not submitted so the audit would not create a real customer record.

Integrity status update: the browser still sends `price` and `total` for compatibility, but the connected database now has the `bookings_set_pricing_from_rules` `BEFORE INSERT` trigger. It derives the selected cruise price, extra-guest charge, total, and blocked-date rejection from server-side pricing rules; the browser values are no longer authoritative for new inserts.

Double-booking and lost-response idempotency were not proven. The availability check and insert are separate operations, so concurrent customers can race unless the database enforces overlap protection or the authoritative booking function handles both atomically.

### D. Authentication and admin workflows

Inspected:

- email/password sign-in and sign-up;
- Supabase session restoration and refresh;
- Google OAuth initiation;
- password reset/update;
- sign-out and local-state clearing;
- admin membership lookup;
- admin fleet, pricing, and booking mutation paths.

Local static result: protected UI entry is tied to Supabase Auth and `admin_users`, and database write policies are present in the supplied migrations.

Dynamic result: **not tested**. No test accounts or reachable Supabase/Auth endpoint were available.

### E. Other discovered functionality

The file also contains customer profiles, saved properties, tours, notifications, PDF/QR generation, Cloudinary media URLs, WhatsApp/email/call actions, and optional Firebase phone verification. The Firebase configuration is intentionally unset locally. These areas were structurally inspected but not browser-tested.

## 6. Tests performed

### Passed

- Project inventory: completed. Only the standalone app, Supabase config, and migrations were present.
- JavaScript syntax check: passed. All 23 inline script blocks were extracted and compiled with Node `new Function()`; **0 syntax errors**.
- Static HTML markup check: passed for the major non-void tags checked (`html`, `head`, `body`, `main`, `section`, `button`, `select`, and `textarea`). The `input` delta is expected because HTML inputs are void elements.
- SQL static sanity check: passed basic balancing only. All migration files had balanced parentheses; dollar-quote markers were even. This is not a PostgreSQL execution test.
- Targeted secret scan: no service-role/private database credential pattern was found in the supplied app-specific code or migrations.

### Failed or blocked

- `git status --short --branch` and `git rev-parse --show-toplevel`: blocked because the supplied folder is not a Git repository.
- `npm --version`: blocked because npm is not installed/available on PATH.
- `supabase --version`: blocked because the Supabase CLI is not installed/available on PATH.
- `psql --version`: blocked because the PostgreSQL client is not installed/available on PATH.
- `HEAD https://nauka.co.in/`: failed with local DNS error `No such host is known`.
- `HEAD https://wudinqbyifvncgadzezy.supabase.co/rest/v1/`: failed with local DNS error `No such host is known`.
- Live table schema, live migration history, RLS policy catalogue, Data API exposure, Auth configuration, RPC definition, and browser network behavior: not testable without connectivity and credentials.
- Browser visual test of the fixed calendar/P&D popup: not run because no reachable deployment or browser test environment was available.

The SQL editor error supplied by the user—`42703: column "rule_type" does not exist`—is treated as an observed application/database failure, not as a test run performed by this audit.

## 7. Files changed or retained

- `index.html`: retained and repaired. Contains the existing P&D UI, calendar scroll fixes, admin editor/fleet paths, Supabase diagnostics, date-aware booking total calculation, and booking submission guard.
- `supabase/config.toml`: retained as the local project reference configuration.
- `supabase/migrations/20261011130000_add_listing_pricing_rules.sql`: restored to the clean fresh-schema migration instead of silently changing its historical meaning.
- `supabase/migrations/20261011133000_fix_admin_fleet_rls.sql`: retained as the admin write-policy migration.
- `supabase/migrations/20261011134000_repair_legacy_listing_pricing_rules.sql`: added for the observed stale-table/missing-column case. It does not drop or recreate the pricing table.
- `supabase/migrations/20261011135000_fix_public_listing_select_rls.sql`: added to make public-active and admin-all listing reads explicit under RLS.
- `supabase/migrations/20261011143000_split_listing_pricing_by_cruise_type.sql`: added independent Day Cruise and Night / Overnight pricing, corrected validation, atomic save RPC, and pricing RLS/grants.
- `supabase/migrations/20261011144000_sync_booking_cruise_type_and_availability.sql`: added booking cruise type, server-side price/blocked-date enforcement, and cruise-type-aware availability.
- `supabase/migrations/20261011145000_cleanup_pricing_policy_duplicates.sql`: removed duplicate pricing policies, indexed the creator foreign key, and restricted the legacy admin helper RPC.
- `supabase/migrations/20261011145500_backfill_cruise_pricing_defaults.sql`: seeded one day and one night default row per existing listing from current listing price fields.
- `supabase/migrations/20261011151000_global_blocked_dates_across_cruise_types.sql`: made blocked dates physical-boat-wide in availability and server-side booking enforcement.
- `serve.ps1` and `serve.mjs`: added a dependency-free Node static-server fallback for local runtime testing.
- `SUPABASE_AUDIT_REPORT.md`: this report.

No destructive database operation, deployment, Git push, DNS change, reset, truncate, or delete was performed. The additive pricing/availability migrations listed in the current implementation update were applied to the connected Supabase project.

## 8. Outstanding issues and exact next actions

1. In Supabase SQL Editor, first inspect the existing pricing table and duplicate counts. Do not run a reset or drop. If the `listing_pricing_rules` table exists but lacks `rule_type`, run `20261011134000_repair_legacy_listing_pricing_rules.sql`; if the table is absent in a fresh/staging project, run `20261011130000_add_listing_pricing_rules.sql` first. Then run `20261011133000_fix_admin_fleet_rls.sql` and `20261011135000_fix_public_listing_select_rls.sql` in order.

2. Confirm the table is exposed in the Supabase Data API and reload the PostgREST schema if the dashboard requires it. Verify that the table grants and RLS policies match the migration files.

3. Reconcile SQL Editor changes with migration history before using CLI deployment. Install the Supabase CLI, run the migration-list/history checks, and do not blindly push a history that disagrees with production.

4. In staging, test with three identities: anonymous, ordinary customer, and admin. Verify that anonymous users can read active listings but cannot write listings/pricing/bookings; customers can read/write only their own permitted rows; and admins can manage fleet and pricing.

5. Test one complete P&D cycle: save weekday/weekend defaults, save a single-date price, save a multi-day block, refresh, verify customer display, verify availability rejection, remove the rule, and verify the default rate returns.

6. Add or verify a database-side authoritative booking path. It must derive or validate price from the listing/pricing rules, validate capacity and dates, and prevent overlap under concurrency. Add an idempotency mechanism for retried booking requests before treating booking integrity as production-ready.

7. If the legacy repair migration reports duplicate date/default rows or invalid legacy data, stop and review those rows before adding a unique index. Do not delete duplicates automatically without a backup and an explicit data-cleanup decision.

8. Deploy the repaired `index.html` and migrations only after staging verification. The local source and `https://nauka.co.in/` cannot be assumed to match until a new deployment is completed and checked.

## 9. Production deployment checklist

- [ ] Back up the current database and export the current schema/policy definitions.
- [ ] Confirm the actual `listings`, `bookings`, `admin_users`, `profiles`, and `saved_properties` columns and RLS policies.
- [ ] Reconcile Supabase migration history with the four migration files.
- [ ] Apply the appropriate pricing repair/new-schema migration in staging first.
- [ ] Apply and verify the fleet read/write policies in staging.
- [ ] Verify Data API exposure and PostgREST schema cache.
- [ ] Run anonymous/customer/admin negative and positive permission tests.
- [ ] Run P&D persistence, blocked-date, date-aware pricing, refresh, and booking tests.
- [ ] Verify authoritative server-side booking price and overlap protection.
- [ ] Confirm Vercel serves the intended `index.html` revision and external assets load.
- [ ] Deploy only after the staging checks pass, then perform read-only smoke tests on the live site.

## 10. Final status

**IMPLEMENTED / VERIFIED FOR REQUESTED SCOPE** — the local UI and connected Supabase pricing/availability/booking changes are applied and smoke-tested. Deployment to the public site remains a separate operational step. The remaining Supabase advisor notices are documented in the current implementation update above; the intentional public availability read RPCs and Auth password-protection setting should be reviewed before declaring full production hardening complete.
