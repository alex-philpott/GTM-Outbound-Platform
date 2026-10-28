# Architecture proposal

Status: **proposal, awaiting review**. Source spec: [`SPEC.pdf`](./SPEC.pdf).

This document covers the whole platform so that Phase 1 lays foundations that later
phases build on without rework. Phase 1 itself is detailed in
[`PHASE-1-PLAN.md`](./PHASE-1-PLAN.md).

---

## 1. Stack

| Concern | Choice | Notes |
|---|---|---|
| Web app | **Next.js (App Router) + TypeScript (strict)** | Server Components and Server Actions keep most logic server-side. No separate API service needed for v1. |
| Database | **Postgres** (Supabase-hosted, UK/EU region) | Plain Postgres features only (no Supabase-specific APIs in app code), so we can move hosts. Local dev uses Docker Postgres. |
| ORM / migrations | **Drizzle ORM + SQL migration files** | *Alternative to Prisma.* Reason: we need hand-written SQL in migrations (append-only triggers, role grants, partial indexes) and Drizzle keeps migrations as plain SQL we own. Typed queries, no engine binary. |
| Auth | **Better Auth** (email/password, DB sessions) | Self-hosted in our DB, admin-created users only (no self sign-up). Has SSO/OIDC/SAML plugins, so SSO is a config change later rather than a rewrite. Alternative: Supabase Auth (see questions). |
| Validation | **zod** | Shared by forms, Server Actions, env config and AI structured output. |
| Background jobs | **pg-boss** (Postgres-backed queue + cron) | Salesforce sync, 06:00 Europe/London signal job, email digest. No Redis needed. Runs in a small worker process alongside the web app. |
| Salesforce | **jsforce**, OAuth via connected app (Phase 2) | Wrapped behind a `CrmSource` interface; Phase 1 uses a mock implementation. |
| AI | **Anthropic API** via `@anthropic-ai/sdk` (Phase 3) | One `AiService` is the only caller. Model is set by env var, not hard-coded. |
| File storage | Supabase Storage (S3-compatible API) | Content uploads (PDF/Word/Markdown). Text extracted on upload (`pdf-parse`, `mammoth`). |
| Email (digest only) | Transactional provider (e.g. Postmark/Resend) — Phase 3a | Only for internal digests to our own users. Outreach is never sent from this platform until Phase 5 integrates the existing sequencer. |
| Tests | **Vitest** (unit + integration against real Postgres), **Playwright** (a few e2e flows) | Integration tests run on a throwaway database per run. |
| CI | GitHub Actions: lint, typecheck, unit + integration tests with a Postgres service container | |

---

## 2. Code layout

```
/
├─ docs/                         spec, architecture, phase plans, ADRs
├─ prompts/                      versioned prompt templates (Phase 3+), e.g. prompts/account-brief/v1.md
├─ db/migrations/                SQL migrations (generated + hand-written triggers/grants)
├─ seed/                         deterministic seed data (UK retail, fictional)
├─ src/
│  ├─ app/                       Next.js routes — thin; call services only
│  │  ├─ (auth)/login
│  │  └─ (app)/ home, accounts, content, manager, admin, audit
│  ├─ server/
│  │  ├─ db/                     Drizzle schema + client
│  │  ├─ auth/                   session, current user
│  │  ├─ authz/                  policy functions: canViewAccount, canApproveStep, …
│  │  ├─ audit/                  the ONLY writer to audit_event
│  │  ├─ domain/                 services: users, accounts, assignments, content, suppression, …
│  │  ├─ integrations/
│  │  │  ├─ crm/                 CrmSource interface; mock (P1), salesforce (P2)
│  │  │  └─ enrichment/          ContactProvider interface; zoominfo (P3b)
│  │  ├─ ai/                     AiService — the ONLY model caller (P3)
│  │  └─ jobs/                   pg-boss workers: sync, daily signals, digest
│  ├─ components/                UI components
│  └─ lib/                       shared utils, zod schemas, env config
└─ tests/                        unit, integration, e2e
```

**Boundaries enforced by lint** (`no-restricted-imports`), so the quality bar is checked
by CI rather than by convention:

- `@anthropic-ai/sdk` may only be imported in `src/server/ai/**`.
- `jsforce` may only be imported in `src/server/integrations/crm/salesforce/**`.
- The `auditEvent` table object may only be imported in `src/server/audit/**`.
- `src/app/**` may not import `src/server/db/**` directly; it goes through services.

---

## 3. Cross-cutting design

### 3.1 Every write goes through a service, and every service write is audited

All mutations use one helper that writes the change and its `AuditEvent` **in the same
transaction**, so a change cannot exist without its audit record:

```ts
await mutate(ctx, { action: 'assignment.reassigned', entityType: 'key_account_assignment', entityId }, async (tx) => {
  const before = await assignments.get(tx, entityId);
  const after  = await assignments.update(tx, entityId, { repId: newRepId });
  return { before, after, metadata: { reason } };
});
```

`ctx` carries the actor (user id, or `system:<job>`), request id and timestamp.

### 3.2 Audit log is append-only at the database level

Not just "the app doesn't call UPDATE":

- A trigger on `audit_event` raises an exception on `UPDATE`, `DELETE` and `TRUNCATE`.
- The app connects as a role with `INSERT, SELECT` only on `audit_event`; migrations run as a separate owner role.
- Optional (recommended): each row stores `prev_hash` + `hash` (SHA-256 over the row and the previous hash), giving tamper-evidence that a nightly check can verify.

Tests assert that an UPDATE/DELETE from the app role fails.

### 3.3 Authorisation in one place

`src/server/authz` holds pure policy functions (`canViewAccount(user, account)`,
`canReassign(user)`, `canApproveStep(user, step)`, …). Services call them; pages never
re-implement rules. Rules from the spec:

| | Rep | Manager | Admin |
|---|---|---|---|
| Own key accounts | view/edit | view/edit | view |
| Other reps' accounts | — | view | view |
| Reassign accounts, set caps | — | yes | yes |
| Audit log | own accounts' entries | global | global |
| Content library | add, edit own | add, edit own | manage all, deactivate |
| Users, personas, channel profiles, field mapping, prompts | — | — | yes |
| Suppression list | add (opt-out) | add | add/remove |

Removal from the suppression list is admin-only and audited.

### 3.4 Hard-constraint guards (built as code, not just policy)

| Constraint | Mechanism |
|---|---|
| Salesforce sandbox only | `SalesforceSource` refuses to start unless the login/instance URL is `test.salesforce.com` or a `*.sandbox.my.salesforce.com` domain. Production needs `SF_ALLOW_PRODUCTION=true` **and** an admin toggle, both audited. |
| Read-only Salesforce until Phase 4 | The client wrapper only exposes query methods. Write methods live in a separate `SalesforceWriter` that is only constructed when `SF_WRITEBACK_ENABLED=true`. |
| Every message has an owner | `sequence_step.owner_id` is `NOT NULL` with a FK. The step state machine (§3.6) rejects approval by anyone other than the owner or an active delegate. |
| Nothing sent automatically | No sending code until Phase 5. Statuses `sent` / `marked_sent` can only be set by the owner's action or a sequencer callback. |
| No LinkedIn automation | LinkedIn steps are tasks: copy-to-clipboard + "open profile" link + "mark as done". There is no LinkedIn client in the codebase. |
| Secrets | Validated by `src/lib/env.ts` (zod) at boot. `.env*` is git-ignored. CI runs secret scanning. |
| Data minimisation | `contact` has only business columns (name, title, seniority, department, business email, LinkedIn URL, role history). CRM sync stores **whitelisted mapped fields only**, never the raw record. |
| Licensed sources only | Discovered contacts must carry a `source` from an enum of licensed providers (`zoominfo`, …). There is no free-form source. |
| First-touch compliance | Generation of the first email to a `discovered` contact must include the opt-out + privacy-notice block (wording from Legal, stored as an admin setting). The approval gate checks it is present. |
| Suppression | `suppression.assertNotSuppressed(contact)` is called by generation, sequence enrolment, approval and send. It matches on contact id **and** normalised email/domain. |

### 3.5 External systems behind interfaces

```ts
interface CrmSource {            // Phase 1: MockCrmSource (reads seed); Phase 2: SalesforceSource
  listAccounts(since?: Date): AsyncIterable<CrmAccount>;
  listContacts(since?: Date): AsyncIterable<CrmContact>;
  listOpportunities(since?: Date): AsyncIterable<CrmOpportunity>;
}
interface ContactProvider {      // Phase 3b: ZoomInfoProvider, or RepAssistedProvider (CSV/list export)
  searchMissingRoles(account, roles): Promise<DiscoveredContact[]>;
  verifyEmail(email): Promise<EmailVerification>;
}
interface OutreachSink {         // Phase 5: existing sequencer/email platform
  pushApprovedStep(step): Promise<ExternalRef>;
}
```

### 3.6 Approval as a state machine

`sequence_step.status`: `draft → awaiting_approval → approved → (sent | marked_sent)`, plus
`rejected`, `paused`, `skipped`, `stopped`. Transitions live in one module, are unit tested
exhaustively, and each transition writes an audit event. Approval requires:

1. actor is the owner or an active delegate of the owner;
2. contact not suppressed, email verified (for email steps);
3. unsupported-claim warnings acknowledged;
4. second (manager) approval present when a rule requires it (C-level, customer with open escalation, first use of a template);
5. first-touch compliance block present when needed.

If the owner is deactivated or marked away, their pending steps move to `paused` and an
alert is raised for their manager.

### 3.7 AI layer (Phase 3, designed now)

- `AiService.run(task, input)` is the only entry point. Each task is defined by a prompt
  template file (`prompts/<task>/v<n>.md` with front-matter: version, model params, output schema name).
- Input is assembled as a **context snapshot**: every fact handed to the model carries a
  `ref` (e.g. `sf:Contact/003…/PreviousEmployer@2026-10-14`, `content:<id>`, `note:<id>`).
  The snapshot is stored with the decision so evidence keeps its values after the CRM changes.
- Output is forced into a zod schema via tool use: `conclusion`, `confidence`,
  `rationale`, `claims[] {text, refs[]}`, `assumptions[]`, `missing_data[]`, plus task-specific fields.
- The service validates that each claim's refs exist in the snapshot; claims without valid
  refs are stored as `unsupported` and drive the UI warning.
- Every call writes `ai_decision` + `evidence_item` rows + an audit event (model, prompt
  version, content items referenced, output) in one transaction.
- Feedback, edit diffs and outcomes link to `ai_decision.id`.

### 3.8 Time, money, ids

- All timestamps `timestamptz` in UTC; displayed in Europe/London.
- Money as integer minor units + currency (`acv_minor`, `currency = 'GBP'`).
- App ids are UUIDv7 (time-ordered). Salesforce ids stored as 18-char text in `sf_id` (unique, nullable).

---

## 4. Data model

Spec entities, with the additions needed to make them work. **P** = phase in which the
table is created. Tables are only migrated in the phase that uses them, so early phases
don't carry dead schema, but the shape is agreed now.

### Identity and configuration
| Table | P | Key columns |
|---|---|---|
| `user` | 1 | id, name, email, role (`rep`/`manager`/`admin`), team_persona_id, manager_id, key_account_cap, status (`active`/`away`/`deactivated`), notification prefs |
| `team_persona` | 1 | key (`sdr`, `nb_ae`, `am`, `csm`, `leader`), name, default view config (jsonb), briefing style, recommended actions |
| `approval_delegation` | 1 | owner_id, delegate_id, starts_at, ends_at, created_by |
| auth tables (`session`, `account_credential`, …) | 1 | managed by Better Auth |
| `buyer_persona` | 1 | name, title keywords, seniority keywords, priorities, pains, meddpicc_elements[], tone guidance |
| `product` | 1 | name, family, description (needed for whitespace + cross-sell) |
| `channel_profile` | 4 | channel, char/word limits, structure rules, tone, CTA style, examples |
| `field_mapping` | 2 | sf_object, sf_field, target_attribute, transform, active |
| `app_setting` | 1 | key/value (e.g. renewal window, compliance footer text, signal job time) |

### Accounts, contacts, customers
| Table | P | Key columns |
|---|---|---|
| `account` | 1 | sf_id, name, industry, employee_band, revenue_band, tech_stack (jsonb), intent_signals (jsonb), crm_owner_id, customer_status (`prospect`/`customer`/`former_customer`), has_open_escalation, last_synced_at |
| `customer_profile` | 1 | account_id, contract_start, contract_end, renewal_date, acv_minor, currency, account_manager_id, csm_id, health, risk_notes |
| `customer_product` | 1 | account_id, product_id, brand/region/business unit, since (supports the whitespace grid) |
| `opportunity` | 1 | sf_id, account_id, name, type (new/expansion/cross-sell/renewal), stage, amount_minor, close_date, is_closed, is_won |
| `contact` | 1 | sf_id (null until pushed), account_id, name, title, seniority, department, buyer_persona_id, persona_source (`ai`/`rep`), email, email_verification, linkedin_url, previous_employer, job_change_flag, source (`crm`/`zoominfo`/…), status (`in_crm`/`suggested`/`approved`/`rejected`), last_synced_at |
| `contact_role_history` | 1 | contact_id, company, title, from, to |
| `key_account_assignment` | 1 | rep_id, account_id, motion, priority (1–5), source (`crm_owner`/`manual`), assigned_by, assigned_at, ended_at |
| `suppression` | 1 | contact_id?, email?, domain?, reason, added_by, added_at, removed_at, removed_by |

### Content
| Table | P | Key columns |
|---|---|---|
| `content_item` | 1 | title, type (case study, value prop, persona guide, battlecard, tone guide, objection handling, winning email), body_md, file_key, extracted_text, owner_id, active |
| `content_tag` | 1 | content_item_id, dimension (`industry`/`persona`/`meddpicc`/`product`/`motion`), value |

### Selling intelligence (Phase 3)
`meddpicc_record` (account_id, opportunity_id?, element, status `unknown`/`hypothesis`/`confirmed`, evidence, confidence, source `ai`/`rep`, ai_decision_id, confirmed_by),
`stakeholder_assessment`, `account_brief` (versioned, every version kept), `ai_decision`,
`evidence_item`, `feedback`, `prompt_version`, `context_snapshot`.

### Sequences (Phase 3 single message, Phase 4 full)
`sequence_template` (+ `sequence_template_step`, persona variants), `sequence`, `sequence_step`
(owner_id NOT NULL, approver_id, second_approver_id, generated_content, edited_content,
status, approved_at, sent_at, marked_sent_by), `outcome`.

### Signals and alerts (Phase 3a)
`signal`, `signal_type_config` (on/off + thresholds per motion), `alert`, `alert_signal`
(grouping), `alert_status_history`, `notification`.

### Audit
`audit_event` (P1): id, occurred_at, actor_type (`user`/`system`), actor_id, action,
entity_type, entity_id, account_id?, contact_id?, sequence_id? (denormalised for filtering),
before (jsonb), after (jsonb), metadata (jsonb), request_id, prev_hash, hash.
Indexes on (entity_type, entity_id), account_id, contact_id, actor_id, occurred_at.

---

## 5. Phase map

| Phase | Builds | Key new tables |
|---|---|---|
| 1 Foundations (mock data) | scaffold, auth, roles, audit, key accounts, content library, suppression, seed | identity, accounts, content, audit, suppression |
| 2 Salesforce sandbox read | connected app OAuth, sync job, field mapping UI, sync errors | field_mapping, sync_run |
| 3 AI layer | AiService, persona match, briefs, MEDDPICC, stakeholders, coverage map, content suggestions, single message + approval, evidence + feedback | ai_*, meddpicc, stakeholder, brief, feedback |
| 3a Signals & alerts | daily job, detectors, alert routing, statuses, dashboards, digest | signal, alert, notification |
| 3b Contact discovery | ZoomInfo/rep-assisted, verification, review queue, new-joiner alerts | discovery_run |
| 4 Sequences & write-back | templates, multi-step generation, stop conditions, SF write-back, outcomes, AI quality dashboard, prompt versioning UI | sequence_*, channel_profile, outcome |
| 5 Sending | integrate existing sequencer/email platform | external_ref |
