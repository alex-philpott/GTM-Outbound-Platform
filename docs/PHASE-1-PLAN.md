# Phase 1 plan: Foundations with mock data

Status: **proposal, awaiting review**. Read with [`ARCHITECTURE.md`](./ARCHITECTURE.md).

**Goal:** a working, logged-in app with realistic UK retail data in which reps see their key
accounts, managers see and reassign across reps, admins manage users and content, and
every change is in an append-only audit log. **No external connections**: no Salesforce, ZoomInfo, Anthropic or email.

---

## Scope

**In:** scaffold and CI, data model for Phase 1 tables, auth and roles, team personas
(default views), audit log (write path, DB enforcement, viewer, CSV export), key account
assignment, account and contact views (read-only), content library, suppression list,
buyer persona admin, seed data, README/CHANGELOG.

**Out (later phases):** Salesforce sync, any AI, MEDDPICC, stakeholder maps, signals and
alerts, contact discovery, sequences, sending.

Phase 1 puts in two things that later phases need early:
- **suppression list + `assertNotSuppressed`**, because the quality bar requires tests for it and every later generation and send path depends on it;
- **`approval_delegation`** table and admin UI, because owner/delegate is part of the user model.

---

## Work breakdown

Ordered. Each step ends green in CI.

### 1. Scaffold
- Next.js App Router, TypeScript strict, pnpm, ESLint (with the import-boundary rules from ARCHITECTURE §2), Prettier.
- `src/lib/env.ts`: zod-validated env; app refuses to boot on missing/invalid config.
- `docker-compose.yml` with Postgres 16 for local dev; `.env.example` with no real values; `.gitignore` covers `.env*`.
- Vitest (unit + integration on a throwaway DB), Playwright (smoke).
- GitHub Actions: install → lint → typecheck → migrate test DB → tests.

### 2. Database and migrations
- Drizzle schema for the Phase 1 tables in ARCHITECTURE §4: `user`, `team_persona`, `approval_delegation`, auth tables, `buyer_persona`, `product`, `app_setting`, `account`, `customer_profile`, `customer_product`, `opportunity`, `contact`, `contact_role_history`, `key_account_assignment`, `suppression`, `content_item`, `content_tag`, `audit_event`.
- Hand-written migration: `audit_event` append-only trigger, `app_rw` role with INSERT/SELECT only on `audit_event`, hash-chain columns.
- Constraints worth having in the DB, not just the app:
  - unique active assignment per (account, motion): one rep owns a given motion on an account at a time;
  - `suppression` must have at least one of contact_id/email/domain;
  - `contact.source` and `status` enums; `customer_profile` only for `customer`/`former_customer` accounts (enforced in service + test).

### 3. Audit module
- `mutate(ctx, meta, fn)` helper (ARCHITECTURE §3.1); `recordEvent` for non-entity events (login, export, failed authz).
- Diffing: store `before`/`after` as whole JSON rows, with secrets/password hashes stripped by a per-entity serializer.
- Viewer: `/audit` (manager/admin global), plus an "Activity" tab on account and contact pages. Filters: rep, account, contact, action, date range. CSV export (streamed; the export itself is audited).
- Nightly hash-chain verification is deferred until we have a job runner (Phase 2); a CLI script `pnpm audit:verify` ships now.

### 4. Auth, roles and team personas
- Better Auth with email/password; DB sessions; secure cookies; password policy; rate-limited login.
- No self sign-up. Admins create users (name, email, role, team persona, manager, key account cap) and send a set-password link. **Phase 1 has no email**, so the link is shown to the admin to pass on (flagged in the UI as temporary).
- Deactivating a user ends their sessions, and their active assignments are listed for the manager to reassign.
- `authz` policy module + middleware guarding `(app)` routes.
- Team personas seeded (SDR/BDR, New business AE, Account manager, CSM, Sales leader). Each persona's home page shows widgets for what exists in Phase 1 (e.g. AM: customer accounts with renewal timeline and whitespace grid; CSM: customer accounts with health; leader: coverage by rep and motion). Placeholders mark widgets that arrive with later phases. Admins can edit which widgets each persona sees.

### 5. Key accounts
- **Default assignment:** a service `deriveDefaultAssignments()` turns the CRM owner into an assignment (`source = crm_owner`). Motion defaults: prospect → new business; customer → expansion (or renewal if renewal is inside the configured window); former customer → re-engagement. It runs against the `MockCrmSource` now and the Salesforce source in Phase 2.
- **Manual override:** a manager/admin can assign, reassign, change motion or priority. Each change needs a reason and is audited. Manual assignments are not overwritten by later default derivation.
- **Cap:** per-rep `key_account_cap`. Assigning beyond the cap is blocked with a clear message; a manager can raise the cap (audited).
- **Rep home:** key accounts ranked by priority, then nearest renewal / most recent activity (no triggers until 3a; the ranking function is a single place to add signals later).
- **Manager view:** all reps × accounts table with motion, priority, customer status, renewal date, filters by rep, persona, motion and status; bulk reassign.
- **Account page (read-only data):** firmographics, tech stack, contacts with buyer persona, opportunities; for customers: products owned, contract/renewal dates, ACV, AM/CSM, health, plus a **whitespace grid** (products × brands/regions/business units, owned vs not). Activity tab = audit.
- **Collision indicator (basic):** if an account has more than one active assignment (different motions), the page shows who else is working it. The full contact-level collision check comes with sequences.

### 6. Content library
- List with filters by type, tag dimensions, active flag, and full-text search (Postgres `tsvector` over title + body + extracted text).
- Create by paste (Markdown) or upload (`.txt`, `.md`, `.pdf`, `.docx`, size-limited). Text is extracted on upload so Phase 3 can pull it into context. Files go to object storage; in local dev, a local folder behind the same interface.
- Tags: industry, buyer persona, MEDDPICC element, product, motion (spec lists new business, expansion, cross-sell, renewal; we include re-engagement too).
- Reps can add and edit their own items; admins can edit or deactivate any. Deactivate is soft. All changes audited.
- Usage count and outcome scoring columns are not added until Phase 4.

### 7. Suppression list
- Admin page + "Suppress contact" action on a contact (reason required). Bulk CSV import for an existing opt-out list.
- `assertNotSuppressed()` matches contact id, normalised email and (optionally) whole domain. Contacts that are suppressed show a clear badge everywhere.
- Removal: admin only, reason required, audited.

### 8. Admin screens
Users, team personas (widget config), buyer personas (keywords, priorities, pains, MEDDPICC elements, tone, linked content), products, app settings (renewal window, compliance footer placeholder), approval delegations.

### 9. Seed data
Deterministic (fixed random seed) so tests and demos are repeatable. `pnpm db:seed` resets and loads.
- **Companies:** ~30 **fictional** UK retail accounts across grocery, fashion, home & DIY, health & beauty, sports, pure-play e-commerce and department stores. Roughly 55% prospects, 35% customers, 10% former customers. Realistic employee/revenue bands and tech stacks (e.g. Salesforce Commerce Cloud, Shopify Plus, Adobe, SAP, Klaviyo, Braze).
- **Customers:** products owned from the product catalogue, multiple brands/regions for whitespace, contract dates, renewal dates spread over the next 3–18 months (some inside the 9–12 month window), ACV, health, a couple with `has_open_escalation`.
- **Contacts:** 6–12 per account across CMO, Head of Ecommerce, Head of CRM, CTO, CDO, CFO, Procurement and adjacent titles; some with role history at a customer account (for the "moved from a customer" case later). All emails use reserved `*.example` domains, so nothing can reach a real person.
- **Opportunities:** open and closed-won/lost, mixed types.
- **Users:** 1 admin, 1 sales leader (manager), 1 manager, 2 new business AEs, 1 SDR, 1 account manager, 1 CSM. Passwords for dev only, printed by the seed script, never committed.
- **Content:** ~20 items (case studies, value props, persona guides, a battlecard, tone guide, objection handling, winning emails) with tags. Written as clear placeholders for our real material.
- A handful of suppression entries.

### 10. Docs
`README.md` (prerequisites, setup, env vars, run, test, seed, demo logins), `CHANGELOG.md`, an ADR for each stack choice that differs from the spec (Drizzle, Better Auth, pg-boss).

---

## Tests (quality bar for Phase 1)

| Area | Tests |
|---|---|
| Audit | every service mutation writes exactly one event with actor/entity/before/after; event and change roll back together; **UPDATE/DELETE on `audit_event` fails at the DB**; hash chain verifies and detects tampering; CSV export matches filters |
| Authz | rep cannot read/modify another rep's account (page and Server Action); manager can view all and reassign; only admin can manage users and remove suppressions |
| Assignments | default derivation from CRM owner + motion rules; manual override survives re-derivation; cap enforced; reassignment audited with reason |
| Suppression | match by contact id, email (case/whitespace/plus-address normalised), domain; removed entries no longer match; removal audited |
| Content | upload extraction for each file type; tag filters; deactivated items excluded from default queries |
| Env/guards | app fails to boot on missing secrets; no `.env` committed (CI check) |
| E2E (Playwright) | log in as rep → see own accounts only; manager reassigns an account → it shows in audit log |

Salesforce mapping tests begin in Phase 2. Approval-gate tests begin in Phase 3, when the first approvable message exists.

---

## Phase 1 demo / acceptance

1. `pnpm i && docker compose up -d && pnpm db:migrate && pnpm db:seed && pnpm dev` works from a clean clone using the README.
2. Log in as each team persona and see the matching default home.
3. Rep sees only their key accounts, ranked; opens a customer account and sees products owned, renewal date and whitespace.
4. Manager reassigns an account from SDR to AE with a reason; the cap blocks an over-assignment; both show in the audit log with before/after.
5. Rep uploads a PDF case study and tags it; it's searchable by its text.
6. Admin suppresses a contact; the badge shows on the account page.
7. Audit log filtered by account and date, exported to CSV.
8. CI green.

---

## Questions

These affect Phase 1 and are best answered before building starts:

1. **What do we sell?** Product names and families (and how customers buy them: per brand, region, business unit?). Needed for the product catalogue, whitespace grid, cross-sell and seed data. Without them I'll use placeholder products.
2. **Hosting and data residency.** Is Supabase acceptable, and in a UK/EU region? Any company standard (Vercel, AWS, Azure)?
3. **Auth.** OK with Better Auth (self-hosted, SSO-ready), or would you prefer Supabase Auth? Which IdP for SSO later (Entra ID, Okta, Google)?
4. **Manager scope.** Does a manager see *all* reps (as the spec says) or only their own team? Can a manager also own key accounts?
5. **Multiple reps per account.** Can an account have several active assignments with different motions (e.g. AM on expansion, new business AE on a cross-sell into a new brand)? I've assumed yes, with one rep per (account, motion).
6. **Key account cap default**, and how many reps and accounts to expect, so I can check scale assumptions.
7. **Delegated approvers.** Who sets delegations: the owner, their manager, or admin? Time-boxed only?
8. **Content contributions.** Do reps' content items go live straight away, or need admin review first?
9. **Existing sequencer / email platform** (Outreach, Salesloft, HubSpot, other?). This doesn't change Phase 1 but does shape the outcome and step models.
10. **Salesforce shape.** Where do products owned, contract dates and "open escalation" live (Assets, Contracts, OpportunityLineItems, Cases, custom fields)? Answer needed by Phase 2. Seed data mirrors whatever you tell me.
11. **Retention.** How long do we keep rejected suggested contacts and suppressed contacts' data? (UK GDPR.) Answer needed by Phase 3b.

## Assumptions (I'll go ahead on these unless told otherwise)

- Seed companies, people and emails are **fictional** (`*.example` domains), not real UK retailers, so demo data can't be mistaken for CRM data or reach anyone.
- One role per user (`rep` / `manager` / `admin`), plus one team persona. A sales leader is a `manager` with the `leader` persona.
- Motion lives on the assignment, not the account, as in the spec's `KeyAccountAssignment`.
- "Re-engagement" is a valid content tag motion, as well as a sequence motion.
- GBP only.
- English (UK) spelling in UI copy.
- The spec's Phase 1 says "no external connections", so file storage and the job runner use local implementations behind the same interfaces as production.
