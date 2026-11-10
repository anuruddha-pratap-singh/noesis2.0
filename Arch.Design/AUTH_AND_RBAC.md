# Noesis 2.0 — Authentication and Authorization Specification

Version: 0.1 proposed contract · 10 November 2026  
Status: design only; Auth settings, RLS, grants, sessions and security tests are not yet implemented.

Related contracts: [SYSTEM_ARCHITECTURE.md](SYSTEM_ARCHITECTURE.md), [DATABASE_SCHEMA.md](DATABASE_SCHEMA.md), [API_SPECIFICATION.md](API_SPECIFICATION.md).

## 1. Security model

Use Supabase Auth for identity and PostgreSQL for live authorization. A verified login answers who the user is. Access additionally depends on current account status, event/festival assignment, explicit permission, accepted roster or review assignment, resource state, time window and, for staff, MFA.

Default deny. Every request must validate the relevant relationship and scope, including reads, exports, uploads, signed links, worker commands and direct RPC calls. A UUID, QR, route prefix, role in local storage, owner name in a PRD, or hidden frontend button is never sufficient authorization. This follows OWASP's per-request authorization guidance. [OWASP authorization](https://cheatsheetseries.owasp.org/cheatsheets/Authorization_Cheat_Sheet.html)

### 1.1 Data classifications

| Classification | Examples | Permitted handling |
|---|---|---|
| Public | Published event rules/schedule, approved names/placements | Explicit sanitized projections only |
| Participant-private | Profile, own entry, accepted roster, status, submission | Owner/accepted team and explicitly authorized scoped staff |
| Staff-private | Event roster contacts, evidence, judge notes, draft scores | Permission AND event/assignment scope; MFA |
| Restricted operational | Room credentials, answer keys, session state, provider secrets | Narrow purpose-specific functions; never general lists/exports |
| Auditable history | Consent, approvals, ledger, corrections, exceptions | Append-only application history; scoped redacted inspection |

Captains may see members' display names, acceptance, college/eligibility state and required game identity fields. They do not automatically receive every member's private email/phone. Judges receive submitted artifacts and rubric data, not contact lists. AV receives approved media/cue/roster labels, not unrelated competition records.

## 2. Authentication flows

### 2.1 Participant flow

PROPOSED initial provider: verified email OTP/magic link with a production SMTP provider. Google login may be added if the college approves it and account-linking tests pass. Support providers through Supabase Auth rather than a custom password database. Do not enable anonymous participant sign-in or public staff signup.

1. Request sign-in; return generic responses that do not reveal account existence.
2. Validate callback/state/PKCE where applicable; only allow exact approved redirect origins/paths.
3. Complete verification through Supabase Auth.
4. Bootstrap/recover exactly one `profiles` record for the Auth user; generated participant code remains stable.
5. Complete name, college and private contact details; accept required consent versions personally.
6. Open dashboard; eligibility and permissions are fetched fresh, not accepted from browser metadata.

Email verified does not mean phone verified or uniquely identified human. A phone/name match cannot automatically merge two Auth identities. Provide an audited administrative duplicate-resolution process preserving accepted rosters, consent, financial records and published results. Never let staff impersonate a participant to accept consent.

For onsite Stock Exchange entry, assist the actual person through verified account creation/login before wallet issuance. If organizers require offline/unverified identity admission, that is a new approved identity workflow; it must not create shared accounts or fake email identities.

### 2.2 Staff flow and MFA

Staff log in through the same Auth system, receive scoped assignments through administration, enroll MFA and complete a second-factor challenge. PROPOSED: require AAL2 for **all staff-private reads and staff mutations**, including roster exports, signed media links, judge scores, room credentials and desk operations. Participants may use AAL1 for their own flows; a staff member may still read their own participant profile at AAL1.

Enforce AAL2 in the database commands/read predicates and Storage policies, not only a redirect. Supabase provides an `aal` token claim for this purpose. Role alone does not prove MFA. [Supabase MFA](https://supabase.com/docs/guides/auth/auth-mfa)

Enroll volunteers and test devices before event day. Recovery for a lost factor requires an independently verified authorized recovery process and audit; there is no shared desk password or hidden MFA bypass. Organizer approval of recovery does not authorize impersonation or falsifying consent.

### 2.3 Session lifecycle

- Verify access-token signature, issuer/project, audience, expiry and supported algorithm at the gateway. Decoding unverified JWT JSON is insufficient.
- Supabase signs/refreshes sessions; use its SDK rather than copying token logic into forms. Keep refresh/access tokens out of URLs, logs, analytics and exported files.
- Every private DB command/read predicate checks current `user_access_state`; staff-sensitive access also checks `session_id` is an existing unrevoked session through a narrowly controlled helper. Use live assignments rather than long-lived permission claims.
- Signout/revocation may leave an otherwise valid access token until expiry. Strict staff paths therefore check the session record, not signature alone. Supabase documents the JWT `session_id` linkage to `auth.sessions`. [User sessions](https://supabase.com/docs/guides/auth/sessions)
- Suspension updates live app state first, then revokes the intended sessions. Access-state checks make suspension effective even if revocation retries. Removing an event assignment immediately removes that event scope for subsequent commands.
- Sensitive commands serialize against access-state/assignment revocation using database locks. If a command acquires authorization locks before revocation, it may finish first; after revocation commits, newly authorized commands cannot proceed under the old grant. Already delivered data cannot be retracted.
- Account deletion is not a session-revocation mechanism. Revoke/suspend first and follow retention/anonymization rules; do not cascade-delete operational history.

PROPOSED operational settings: staff inactivity lock in UI after 15 minutes, prompt reauthentication after extended inactivity, short signed-download lifetime, and short-lived access JWTs chosen after refresh/offline rehearsal. These values are not claims about default Supabase configuration. Avoid imposing unrealistic short JWT expiry without testing refresh behavior.

### 2.4 Browser/session storage and subdomains

A Vite SPA may use SDK-managed browser persistence. This makes XSS prevention critical: no raw HTML rendering of participant text, strict Content Security Policy, sanitized external content, reviewed dependencies, and no embedded untrusted scripts on authenticated pages. Do not claim HttpOnly-cookie protection when using local-storage SDK tokens.

On logout/account switch, stop subscriptions, clear query caches, sensitive forms and local offline data as policy requires. Shared desk devices need named sessions and visible active identity. Never cache another volunteer's roster automatically.

Start with one application origin. If trade/override subdomains are adopted, do not share broad domain cookies or pass tokens in query strings. Use a reviewed same-identity authentication flow per origin; isolate the intentionally buggy Override sandbox on a separate origin with no production auth token, network privilege or real data.

## 3. Roles, scopes and separation of duties

Roles are permission bundles. Effective access is a permission evaluated against a specific resource, not the role label alone. Participant/captain eligibility is relationship-derived; they are not universal staff grants.

| Bundle | Scope | Intended permissions |
|---|---|---|
| Participant | Own identity and accepted entries | Profile, consent, own submissions/notifications/results |
| Team captain | Current accepted captain of one entry | Invite/manage prelock roster, submit team work, respond finalist; Doomsday only when selected |
| Registration staff | Explicit event(s) | Verify entries, permitted contacts, confirm eligible entries, event roster export if granted |
| Check-in volunteer | Explicit event(s)/day if restricted | Minimal lookup and individual attendance; no general export by default |
| Reviewer | Assigned submission versions | Artifact access, own reviews, relevant rubric/feedback |
| Judge coordinator | Explicit event | Assign reviews, coverage checks; no automatic final-result publication |
| Referee | Explicit sports/coding event | Match/lobby scores, proof, approved advancement |
| Event operator/marshal | Explicit event | Approved round controls/effects; market operation only through separate permission |
| AV/stage operator | Explicit event/show/session | Approved tracks/cues/order, playback checks, package acknowledgement |
| Finance | Explicit festival or event allocation | Payment verification/refund/reconciliation; no judge notes |
| Event lead | Explicit event | Rule/selection/result approval and approved event operations; not global permissions |
| Festival administrator | Explicit festival | Staff assignments, festival config, approved cross-event reporting; no implicit right to unrelated secrets |
| Audit reader | Explicit festival/event | Redacted audit; no mutation |
| Backend worker | Dedicated service identity + allowlisted operation | Due jobs/outbox/import processing; never a human staff role |

PROPOSED separation of duties: the originator of a result/rule/selection proposal cannot approve their own proposal. A distinct authorized lead/delegate must approve it. Publication may be performed by that approver if separately permitted. This is an engineering proposal, not a supplied staffing commitment; confirm staffing before enabling live workflows. If organizers approve a single-person exception, record the policy and exceptional approval reason explicitly rather than silently disabling the check.

No automatic super-admin from email domain, profile name, first signup, or source code. Festival administrator does not inherently bypass approval, read answer keys, edit ledger, award points, or access every participant contact. Emergency access is time-limited, purpose-specific, independently granted and audited; it cannot bypass ledger invariants or erase history.

## 4. Authorization data model

These tables belong to the [database contract](DATABASE_SCHEMA.md). All authorization tables are non-exposed and unavailable to participant DML. Keys use UUIDs; common creation/revision conventions apply.

| Table | Schema | Fields / constraints |
|---|---|---|
| `permissions` | private | `code text PK`, description, sensitivity; deployment-managed fixed catalog |
| `roles` | private | `code text PK`, description, allowed_scope_kind; no arbitrary role names from clients |
| `role_permissions` | private | composite PK `(role_code,permission_code)`; deployment-reviewed bundles |
| `staff_assignments` | private | id, profile_id, festival_id, event_id nullable, role_code, starts_at, expires_at, revoked_at nullable, granted_by, reason, revision; event must belong to festival |
| `assignment_permissions` | private | assignment_id, permission_code; optional explicit narrowed subset; cannot add permissions beyond bundle/delegator ceiling |
| `delegation_limits` | private | grantor_assignment_id, grantable_role_code, permitted_scope, maximum_expiry; prevents privilege/scope escalation |
| `user_access_state` | private | profile_id PK, state active/suspended/anonymized, reason, changed_by, revision, changed_at; checked live |
| `judge_conflicts` | private | reviewer_id, event_id, conflicting_entry_id, declared_at, resolved_by nullable, reason; conflict blocks assignment/read/review as policy requires |
| `registered_devices` | private | id, festival_id, assigned_staff_id, permitted_event_id, status, enrolled_at, expires_at, revoked_at; offline-device authorization |
| `service_principals` | private | id, code UQ, allowed_operation_codes, status, credential_reference, rotated_at; never browser-visible secrets |
| `access_change_events` | private | id, actor_id, target_profile_id, assignment_id nullable, change_kind, reason, occurred_at, request_id; append-only |

`staff_assignments` has a partial uniqueness rule preventing duplicate equivalent live grants. Nullable event scope has explicit semantics: a festival-scoped grant authorizes only permissions designed to span that festival. An event-scoped permission cannot become global by passing `event_id = null`.

Authoritative query concept:

```text
allow(actor, permission, resource):
  verified non-anonymous identity AND active profile/access state
  AND live session when operation is staff-sensitive
  AND (eligible ownership/accepted-membership relation
       OR active assignment with permission AND matching resource scope)
  AND required MFA assurance
  AND state/deadline/approval constraints
  AND no explicit conflict/lock prohibition
```

Permission checks derive festival/event from the stored resource. They do not trust path/query/body scope as proof. Nested IDs must all refer to that same scope. JWT `user_metadata` is user-editable and MUST NOT hold trusted permissions. Even server-controlled `app_metadata` can be stale until token refresh; use it for UI hints at most, with database grants authoritative. [Supabase RLS guidance](https://supabase.com/docs/guides/database/postgres/row-level-security)

## 5. Permission catalog and resource matrix

### 5.1 Permission codes

| Area | Codes and purpose |
|---|---|
| Governance | `festival.configure`, `decision.propose`, `decision.approve`, `event.rules.propose`, `event.rules.approve`, `event.publish`, `event.capacity.manage` |
| Identity/access | `staff.assignments.grant`, `staff.assignments.revoke`, `accounts.suspend`, `audit.read` |
| Registration | `entries.read`, `entries.contacts.read`, `entries.confirm`, `entries.cancel`, `entries.waitlist.promote`, `rosters.change.approve`, `rosters.export` |
| Submissions/review | `submissions.configure`, `submissions.assigned.read`, `submissions.freeze`, `rubrics.propose`, `rubrics.approve`, `reviews.assign`, `reviews.write`, `selection.propose`, `selection.approve`, `selection.reserves.promote` |
| Competition | `rounds.configure`, `rounds.control`, `scores.import`, `scores.verify`, `matches.manage`, `matches.results.write`, `matches.advance`, `lobbies.manage`, `lobbies.credentials.manage`, `bugs.configure`, `bugs.review`, `features.configure` |
| Effects | `challenges.configure`, `challenges.release`, `effects.configure`, `effects.activate`, `effects.reverse` |
| Market | `market.entry.verify`, `market.wallet.issue`, `market.prices.publish`, `market.control`, `market.valuation.finalize`, `market.audit.read` |
| Attendance/admissions | `checkin.lookup`, `checkin.write`, `checkin.roster.verify`, `checkin.exception.approve`, `checkin.offline.export`, `checkin.offline.reconcile`, `passes.import`, `passes.use`, `attendance.report` |
| Performer/stage | `performances.review`, `stage.order.manage`, `stage.media.read`, `stage.packages.create`, `stage.playback.verify`, `stage.handoff.create`, `stage.handoff.acknowledge` |
| Results/Cup | `results.draft`, `results.approve`, `results.publish`, `disputes.resolve`, `cup.policy.propose`, `cup.policy.approve`, `cup.affiliations.lock`, `cup.calculate`, `cup.publish` |
| Finance/Arcade | `payments.verify`, `refunds.approve`, `arcade.coupons.issue`, `arcade.coupons.redeem`, `arcade.coupons.void`, `arcade.coupons.refund`, `arcade.stock.adjust`, `arcade.reconcile` |
| Operations | `announcements.publish`, `exports.request`, `jobs.inspect`, `jobs.retry` |

Owning a broad permission such as `exports.request` does not grant underlying data access. Export policy intersects requested kind, event scope and specific roster/contact/finance permissions. Similarly, `jobs.retry` cannot make an unapproved effect or result publishable.

### 5.2 Resource-specific access

| Resource / action | Participant/captain | Permitted staff | Public |
|---|---|---|---|
| Profile and contacts | Self; captain gets limited roster labels | Contacts only with explicit event relationship + permission | Approved result display name only |
| Entry/accepted roster | Own individual or accepted team membership | Assigned event registration/desk/lead, field-limited | None |
| Pending invitations | Inviter and verified intended recipient; token not public | Minimal support if explicitly granted | No recipient/account lookup |
| Consent acceptance | Self only | Inspect evidence for assigned entry; cannot accept for person | None |
| Submission draft | Own individual entrant or current captain; members may read accepted team draft if policy permits | Scoped support; do not let staff author contestant work | None |
| Submitted/frozen artifact | Accepted roster member on permitted entry | Assigned reviewer/judge or explicit event artifact permission | None by default |
| Review | Performer receives own released feedback; participant does not see private judge notes | Assigned reviewer writes own; lead sees authorized event review coverage | None |
| Selection and finalist response | Own eligible entry; captain responds | Scoped selection authority | Only explicitly approved announcement |
| Doomsday prompt/attempt | Selected locked captain, own assignment, release/window checks | Approved marshal; answer key needs separate restricted access | None |
| Group effects | Own team's allowed outcome | Authorized marshal acts on approved transition | Approved safe summaries only |
| Market wallet/order | Own verified player only | Explicit operator/audit scope; operator cannot trade as player | Safe approved ranks if published |
| BGMI credentials | Assigned accepted active squad in access window | Credential manager, not every referee | Never |
| Check-in | Own dashboard receipt | Assigned desk; per-person write; exceptions separate | Never |
| Media/cue package | Own performer entry/status | Authorized AV/stage with exact approved audio version | No raw media unless separately approved |
| Draft score/result | Own provisional status only if explicit policy permits | Scoped referee/lead; result publication separate | Never |
| Published result | Own/accepted historical team context | Same approved version plus allowed proof | Sanitized current projection |
| Cup | Own independent consenting squad | Explicit Cup authority | Enabled approved standings only |
| Coupon/refund/stock | Own receipt if participant-linked; no desk mutation | Distinct desk/finance permissions | Public Arcade info only |
| Audit/export | Own receipt, not global audit | Explicit scope/columns; expiry and download recheck | Never |

## 6. PostgreSQL RLS and function security

### 6.1 Exposure and grants

1. Expose only `api` for application PostgREST. Keep `app` and `private` out of exposed schemas. Managed Auth/Storage endpoints continue through their official APIs.
2. Enable RLS on every domain table, including non-exposed `app` tables, as defense in depth. No policy means denial. Use explicit `TO anon`/`TO authenticated` and ownership/scope predicates; `TO authenticated` alone does not authorize every row.
3. Revoke default function EXECUTE from PUBLIC and set matching default privileges for **each function-creating owner**. Grant named functions only. Never use blanket EXECUTE on all future functions.
4. `api.read_*` functions are SECURITY INVOKER, selecting explicit columns from RLS-protected app data. Grant only the underlying SELECT/column privileges necessary for permitted reads. Schema non-exposure prevents raw REST access but does not replace RLS.
5. `api.cmd_*` wrappers are SECURITY INVOKER and invoke named `private.cmd_*` implementations. Authenticated roles receive narrowly required schema USAGE and EXECUTE for these wrappers/implementations, not arbitrary private-table access. Public anon grants exist only for public-read operations.
6. Private command implementations that require protected multi-table writes use reviewed SECURITY DEFINER, a fixed empty search path, fully qualified references, no dynamic client SQL, and a dedicated NOLOGIN owner with minimum required object privileges. The owner must not be a general superuser/BYPASSRLS role. If it owns target tables and therefore bypasses their RLS, that bypass is intentional and every command must perform complete explicit authorization/invariant checks.
7. Authorization helpers are in `private`, narrowly executable only where needed. Helpers needing privileged `auth.sessions` reads use a dedicated owner granted only that capability; participant roles get no access to managed Auth tables. Public projection predicates are explicitly safe booleans and do not expose private data.
8. Internal worker functions revoke EXECUTE from anon/authenticated; backend credentials live only in protected workers. Service-role privileges bypass normal RLS, so audit and allowlist their use carefully. User-facing gateway requests never substitute service credentials.

Supabase documents the invoker/definer difference, function grant controls, and the special care needed for RLS and views. Use invoker-security views if any are introduced; do not ship an owner-security view exposing protected tables. [Functions](https://supabase.com/docs/guides/database/functions), [RLS](https://supabase.com/docs/guides/database/postgres/row-level-security)

### 6.2 Policy patterns (illustrative, not migrations)

```sql
-- Self-profile reads. Production helper also checks active access/session policy.
create policy profile_self_read
on app.profiles for select to authenticated
using (auth_user_id = (select auth.uid()));

-- If direct profile edits were introduced, BOTH predicates and column grants
-- are necessary. Current contract routes edits through commands instead.
-- USING restricts old rows; WITH CHECK restricts the resulting row.
```

Table policies must align with actual schema relationships. Examples of required predicates:

- Entries: self owner OR accepted current member OR scoped staff with `entries.read` and AAL2.
- Contacts: self OR scoped staff with `entries.contacts.read` and a valid entry/person relationship; pending invitation is not sufficient.
- Submission versions/assets: accepted eligible entry member OR active nonconflicted reviewer assignment to that exact version OR explicit scoped artifact role.
- Review notes: only own assigned reviewer/authorized event lead; released performer feedback is a separate projection.
- Public result rows: only when result version equals the result set's current approved published pointer.
- Staff lists and audit: live assignment plus permission, event/festival scope, AAL2 and active session.

Avoid recursive policies such as membership SELECT policy querying the same membership policy. Use bounded private helper predicates with reviewed privileges and indexes. A helper with a row-dependent argument must not be incorrectly treated as a single global cached decision. Ensure RLS query plans are measured using actual roles and representative data.

If multiple permissive policies are combined, OR semantics can accidentally widen access. Use restrictive predicates where appropriate for MFA/suspension, or incorporate the complete predicate in each relevant policy. Staff AAL2 requirements must not accidentally block an AAL1 participant from reading their own data, or be bypassed through a broad self/team policy.

## 7. Storage, signed capabilities, QR and exports

### 7.1 Storage buckets

PROPOSED private buckets: `submission-quarantine`, `submission-assets`, `performance-assets`, `result-evidence`, `private-exports`. Public branding assets may use a separate public bucket containing no participant data.

Upload policy validates intent ownership, entry/stage permission, path, expiry, size/type limits and state. Finalized immutable objects deny participant UPDATE/DELETE; no upsert into frozen paths. Download policies validate current relationship and asset readiness. A scan mark sent by the client is never trusted. Use Storage APIs for objects; do not manipulate `storage.objects` metadata to simulate moving/deleting bytes. [Storage access control](https://supabase.com/docs/guides/storage/security/access-control)

Signed URLs are temporary bearer capabilities. Short expiry bounds, but does not eliminate, post-revocation exposure. Prevent logging/referrer leakage and do not include signed URLs in persistent receipts or public caches. AV offline copies cannot be remotely revoked while disconnected; register devices, scope copies minimally, set operational expiry, and require witnessed deletion after the approved retention period.

### 7.2 QR and token handling

Participant QR contains only an opaque participant identifier or signed opaque reference; no name, email, phone or Auth token. It can locate a record only through an authenticated scoped desk lookup. A public participant code/QR is not a secret and does not by itself prove identity or authorize check-in.

Invitation/pass/coupon tokens use cryptographically secure randomness, hashes at rest, bounded lifetime and purpose-specific verification. Reject token reuse after terminal state; rate-limit guesses. Never put private tokens in analytics, logs or public pages. Passes/coupons additionally enforce quantity/state/expiry under lock.

### 7.3 Export and offline controls

Authorization is checked at request, generation, and download. Store requesting actor, scope, filters, columns, manifest and expiry. Staff may lose access while a job runs; recheck before publishing the asset. Downloads require current permission, not merely ownership of the job ID.

Offline roster includes participant code, display name, entry/event/day, eligibility snapshot and necessary roster labels; contact details only if explicitly approved and necessary. Encrypt local copies on registered devices; avoid service-worker caching of general private APIs. Offline observations retain staff/device/source IDs. Staff impersonation, global uniqueness and eligibility changes cannot be solved by an offline roster alone; reconciliation flags conflicts.

CSV exports neutralize spreadsheet formula prefixes (`=`, `+`, `-`, `@`, including whitespace/control-character tricks) while preserving legitimate original data in the database. Free-text filenames, HTML and user names require context-appropriate escaping.

## 8. Delegation, approvals and emergency procedures

Grant assignment requires AAL2, `staff.assignments.grant`, matching delegation limit, target verified profile, expiry and reason. Grantor cannot broaden festival/event scope, grant a stronger role than allowed, or approve their own escalation. Revocation uses live database state; do not wait for JWT refresh.

Performer approvals, result publication, payment verification, exceptions, effect activation/reversal and stock/refund changes are independently permissioned. A desk exception cannot create a paid payment record, convert an unselected hackathon team to selected, or waive member consent. If an approved policy permits an admission exception, record that specific exception without rewriting the underlying competition status.

Emergency steps: identify incident → suspend affected access → pause impacted operation → preserve logs → issue time-limited scoped access through authorized personnel → record every correction → independently review. There is no command to delete audit trails, reset winners invisibly, bypass ledger balance, read arbitrary database tables, or execute arbitrary participant code.

Bootstrap the first festival administrator using a controlled reviewed deployment/admin operation tied to a verified Auth user ID. Record the assignment and reason. Do not expose a public bootstrap endpoint or grant privilege to whichever account signs up first.

## 9. Application threats and mitigations

| Threat | Required mitigation |
|---|---|
| Cross-event ID guessing / BOLA | Server-derived resource scope, RLS, explicit command checks, negative tests |
| Role metadata tampering / mass assignment | Live grants, allowlisted fields, protected owner/status fields |
| Stale JWT after suspension/revocation | Live access state, assignments, sensitive session existence checks |
| SQL injection / privileged function misuse | Bound typed inputs, fixed functions, qualified identifiers, no arbitrary SQL |
| XSS in names/problem links/media | Escaped rendering, approved URL schemes, CSP, isolated media processing |
| SSRF through project/video URL | Approved providers, private-IP/DNS/redirect controls, bounded isolated workers |
| Malicious upload / archive bomb | Quarantine, byte/signature/scan checks, resource limits, no code execution |
| Credential/answer leakage | Restricted tables/functions, no-store, no realtime/export/log copies |
| Replayed scan/trade/coupon/webhook | Authenticated scope, idempotency, unique constraints, signature/event-ID validation |
| Concurrent capacity or balance manipulation | Locked DB transaction with post-lock time/state checks |
| Export leakage / shared devices | Explicit columns, expiry, MFA, download recheck, registered offline devices |
| Unauthorized result/score changes | Separate approvals, immutable versions, pinned roster/source, audit |
| Compromised frontend publishable key | Key is not privilege; RLS/function checks enforce access; never expose backend secret |
| Compromised backend credential | Separate worker identities/allowlists, rotation, minimal privileges and auditing; acknowledge service-role blast radius |

## 10. Authorization test catalog

All cases are unexecuted specifications. Run through HTTP, direct RPC, underlying SQL roles in local tests, Storage operations, signed-download generation and realtime where applicable. Do not test RLS only as the table owner/service role; those may bypass it.

| ID | Actor / stimulus | Expected outcome |
|---|---|---|
| AUTH-01 | Anonymous user reads catalog/current approved results | Only sanitized published rows |
| AUTH-02 | Anonymous user reads profile/entry/draft/asset/contact | Denied; no existence disclosure |
| AUTH-03 | Forged/expired/wrong-project JWT | Denied before data/command |
| AUTH-04 | Anonymous Auth identity tries participant actions | Denied despite authenticated database role |
| AUTH-05 | Participant changes user_metadata role to admin | No new permission |
| AUTH-06 | Old app_metadata claims survive revoked grant | Live assignment check denies access |
| AUTH-07 | Suspended/deleted-session staff uses still-valid JWT | Sensitive read/write denied |
| AUTH-08 | Staff has only event A grant; requests event B/other festival | Denied across nested IDs, lists, exports and RPC |
| AUTH-09 | Festival-scoped non-global permission with null event | No accidental unrestricted scope |
| AUTH-10 | Assignment before starts_at / at expires_at / revoked | Denied outside valid interval |
| AUTH-11 | AAL1 participant reads own dashboard | Allowed; staff MFA rule does not overblock |
| AUTH-12 | AAL1 staff reads roster/notes/export or mutates | MFA_REQUIRED through all access paths |
| AUTH-13 | Another member/nonmember reads private contact | Denied; captain gets only approved labels |
| AUTH-14 | Invitation sent to another verified email | Attacker cannot accept by guessing token/entry ID |
| AUTH-15 | Expired/revoked/declined invite reused | No membership added |
| AUTH-16 | Invite recipient has not accepted | No team draft/artifact privileges from invite alone |
| AUTH-17 | Former member/captain accesses current draft | Denied; historical result visibility follows snapshot policy |
| AUTH-18 | Noncaptain modifies team roster/submits team work | Denied unless explicit approved delegated action exists |
| AUTH-19 | Captain tries to accept consent for members | Denied; personal acceptance required |
| AUTH-20 | Unselected captain/selected noncaptain/other team reads challenge | Denied without question/answer leakage |
| AUTH-21 | Captain requests unreleased wave/answer rule | Denied; no keys in asset/API/source map |
| AUTH-22 | Reviewer accesses unassigned version/event | Denied |
| AUTH-23 | Conflicted or revoked reviewer opens artifact/reviews | Denied; reassignment required |
| AUTH-24 | Reviewer alters another judge's review | Denied; lead correction workflow preserves authorship |
| AUTH-25 | Referee tries final publication without result permission | Denied |
| AUTH-26 | Originator approves own rule/selection/result | Denied under adopted separation policy; explicit approved exception only |
| AUTH-27 | Judge/desk requests participant export | Denied unless export and underlying data permissions granted |
| AUTH-28 | BGMI squad accesses another lobby/outside window | No credentials; correct squad/window allowed |
| AUTH-29 | Referee without credential permission lists lobby metadata | Credentials omitted |
| AUTH-30 | Player places order against another wallet | Server ignores/rejects injected owner; no trade |
| AUTH-31 | Market operator edits ledger directly | Denied; audited narrow correction only |
| AUTH-32 | Desk volunteer verifies payment/selection using exception | Denied; underlying payment/selection unchanged |
| AUTH-33 | Captain scan attempts bulk teammate attendance | Denied; individual evidence required |
| AUTH-34 | AV downloads unrelated video/contact roster | Denied; exact approved cue/audio access only |
| AUTH-35 | Arcade desk issues refund/stock adjustment without grant | Denied |
| AUTH-36 | Grantor requests stronger role/broader event/self-escalation | Denied; delegation ceiling enforced |
| AUTH-37 | Participant calls worker/private/internal RPC | No EXECUTE or explicit denial before mutation |
| AUTH-38 | Public function/view accidentally exposes private table | Grant/exposure test fails CI; invoker/RLS projection checked |
| AUTH-39 | Insert/update changes owner/event/captain/status | Mass assignment denied; WITH CHECK/command invariants preserve scope |
| AUTH-40 | Upload/finalize/read/overwrite/delete another entry asset | Denied at Storage and command boundary |
| AUTH-41 | Download authorization after grant revocation | New URL denied; existing URL limited to documented lifetime |
| AUTH-42 | Offline export generated after requester loses grant | Job cannot publish new accessible package |
| AUTH-43 | Revoked device/staff submits offline attendance | Quarantined/rejected for authorized reconciliation, no silent acceptance |
| AUTH-44 | Realtime subscription before role revoke | No future private payloads emitted; fetch denied; do not assume instant disconnect |
| AUTH-45 | Logout/account switch on shared browser | Tokens/subscriptions/private caches cleared; no previous roster shown |
| AUTH-46 | QR copied by another participant | Lookup only; staff verification/eligibility still required |
| AUTH-47 | Coupon/pass/invite token brute force | Rate-limited, hashed tokens, nonenumerating errors |
| AUTH-48 | XSS/CSV formula/hostile filename/profile text | Safe rendering/export, no script/formula execution |
| AUTH-49 | Callback/open redirect to attacker origin | Rejected; tokens not placed in unapproved destination |
| AUTH-50 | Auth identity linking or email change | Verified supported flow, same stable profile, no account takeover/automatic unsafe merge |
| AUTH-51 | First arbitrary signup | Participant only; no bootstrap admin privilege |
| AUTH-52 | Audit reader attempts mutation/delete history | Denied; redacted scope only |
| AUTH-53 | Log inspection after credential/answer/token requests | No secret values; useful safe request/outcome IDs remain |
| AUTH-54 | Lost MFA factor / shared login workaround | No bypass; documented independently verified recovery |
| AUTH-55 | Concurrent grant revocation and sensitive command | Defined lock ordering, no post-revocation authorization with stale grant |
| AUTH-56 | Result correction / participant leaves team | Public version stays approved; historical roster identity accurate |
| AUTH-57 | SQL table owner/service key used as sole security test | Test harness rejects this as insufficient; run real caller roles |
| AUTH-58 | Multi-policy OR path bypasses MFA/suspension | Negative suite catches widening; effective predicate remains restrictive |
| AUTH-59 | Temporary emergency grant expires | Automatic live denial; audit preserved; no permanent broad role |
| AUTH-60 | New table/function/storage bucket introduced | Deployment check requires explicit exposure/grants/policies/test coverage |

### 10.1 Access-test fixture matrix

For each protected resource, exercise owner, accepted teammate, current captain, pending invitee, former member, unrelated participant, assigned staff, unassigned staff, expired staff, suspended staff, AAL1 staff, assigned reviewer, conflicted reviewer and service worker. Repeat with a valid resource ID from another event/festival and a nonexistent ID. Verify field omission as well as row denial. Check HTTP/RPC/Storage access parity.

## 11. Release requirements and sources

Before launch: inventory exposed schemas/functions, verify all table RLS/grants, scan frontend bundles for secrets, test MFA enrollment/recovery, verify redirects/CORS, inspect Storage bucket visibility, audit staff assignments/expiry, test suspension/revocation, and execute negative-scope suites. Before event day: rehearse desk devices, credential rotation, overnight operator access, backup restoration, offline export deletion and stage/AV permissions.

Research checked 10 November 2026; revalidate provider behavior during implementation:

- [Supabase RLS](https://supabase.com/docs/guides/database/postgres/row-level-security)
- [Supabase database function privileges](https://supabase.com/docs/guides/database/functions)
- [Supabase Auth sessions](https://supabase.com/docs/guides/auth/sessions)
- [Supabase MFA](https://supabase.com/docs/guides/auth/auth-mfa)
- [Supabase Storage access controls](https://supabase.com/docs/guides/storage/security/access-control)
- [Supabase Realtime authorization](https://supabase.com/docs/guides/realtime/authorization)
- [OWASP authorization](https://cheatsheetseries.owasp.org/cheatsheets/Authorization_Cheat_Sheet.html)
- [OWASP file uploads](https://cheatsheetseries.owasp.org/cheatsheets/File_Upload_Cheat_Sheet.html)

These specifications propose controls and tests; they do not constitute a completed security audit or a guarantee that an unimplemented system is secure.
