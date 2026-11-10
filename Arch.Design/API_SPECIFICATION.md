# Noesis 2.0 — API Specification

Version: v1 / design revision 0.1 · 10 November 2026  
Status: proposed contracts; endpoints, RPC functions, integrations and automated tests are not implemented.

References: [architecture and decision register](SYSTEM_ARCHITECTURE.md), [schema](DATABASE_SCHEMA.md), [authorization](AUTH_AND_RBAC.md). This is the application API, not a replacement implementation of Supabase Auth or Storage.

## 1. Transport and implementation contract

Base URL: `https://<project-ref>.supabase.co/functions/v1/noesis-api/v1`. A reverse proxy may expose `/api/v1` on the festival domain, preserving identical semantics. Production and staging URLs must never be interchangeable in client configuration.

Browser → Edge HTTP gateway → narrowly exposed `api` RPC → non-exposed domain data/commands. User requests forward the caller JWT to RPC, not a service-role JWT. Public endpoints call explicit public-read RPCs. Every RPC independently validates permission and domain rules because it can be called without the gateway. Gateway handles routing, strict request parsing, JWT verification, response serialization, CORS, HTTP status mapping and additional abuse limits.

For each registry operation `operation_name`, create a single corresponding `api.operation_name` function. Read functions accept `p_input jsonb`. Mutation wrappers accept `p_input jsonb`, `p_idempotency_key text`, and `p_expected_revision bigint DEFAULT NULL`. Private implementations are not independently exposed; no arbitrary table-name, SQL, function-name, or action dispatcher argument is allowed. Inputs are validated in the database as well as at Edge.

Example mapping: `POST /entries/{entry_id}/confirm` → `api.cmd_confirm_entry` with canonical `p_input = {"entry_id":"..."}`. The Edge route owns `entry_id`; it cannot be overridden by a body field. Do not overload PostgreSQL function names by signature. [Supabase database functions](https://supabase.com/docs/guides/database/functions)

RPC implementations return a structured success/domain-error envelope; Edge maps domain errors to the HTTP statuses below. Direct RPC transport status may differ from Edge's HTTP status, but its domain outcome and safeguards must match. For expected rejected attempts, the database can preserve an outer quota/audit update while rolling back business writes in a subtransaction. Unexpected SQL exceptions roll back the transaction; they never return a success envelope.

### 1.1 Request headers

| Header | Contract |
|---|---|
| `Authorization: Bearer <access_token>` | Required for private endpoints; verified issuer/audience/signature/expiry; derive actor from verified claims |
| `Content-Type: application/json` | Required for JSON bodies; multipart/file bytes use Storage instead |
| `Idempotency-Key: <uuid>` | Required for every domain mutation, including retryable updates and async job requests; stored with actor/operation/request hash |
| `If-Match: "<revision>"` | Required for operations marked **R** below; stale revision → 412; missing → 428 |
| `X-Request-ID` | Optional bounded UUID; server generates if invalid/missing; no sensitive free text |

Auth SDK operations and provider webhooks use provider-specific replay/security mechanisms rather than user idempotency headers. CORS allows only explicit approved origins; reject wildcard credential policies. Bearer-auth business APIs do not use ambient cookies. If cookie-backed auth is added, revise CSRF/session contracts first. CORS is not authorization.

### 1.2 Common shapes

```ts
type UUID = string;
type Instant = string;        // RFC 3339 with explicit offset; output normalized UTC
type Decimal = string;        // exact numeric e.g. "1250.00", not exponent/NaN
type Quantity = string;       // bounded base-10 integer for bigint values
type Revision = string;       // bigint serialized as decimal string

interface Success<T> {
  data: T;
  meta: { request_id: UUID; server_time: Instant; replayed?: boolean };
}
interface Page<T> {
  items: T[];
  next_cursor: string | null;
  snapshot_revision?: Revision;
}
interface Failure {
  error: {
    code: string;
    message: string;
    field_errors?: Record<string, string[]>;
    retryable: boolean;
    decision_codes?: string[];
  };
  meta: { request_id: UUID; server_time: Instant };
}
interface CommandReceipt {
  command_id: UUID;
  resource_id: UUID;
  state: string;
  revision: Revision;
  next_actions: string[];
}
```

Lists use `limit` default 25, maximum 100 and opaque keyset cursor. Cursor binds sort, filters, caller visibility and snapshot where needed; never trust cursor content without validation. Default stable sort `(created_at,id)`; results use placement plus stable tie order. No unbounded list, global contact search, arbitrary SQL filter, or client-selected export columns. Search is event-scoped, bounded, and rate-limited.

All DTOs use explicit allowlists. Public results contain display names, team name, event, approved score/placement, prize category, source version and update time. They never contain email, phone, room credentials, submitted answers, judge notes, raw tokens or provider secrets.

### 1.3 Errors and retry rules

| HTTP | Codes / meaning |
|---|---|
| 400 | `MALFORMED_REQUEST`, invalid JSON/cursor/unknown field |
| 401 | `AUTH_REQUIRED`, `TOKEN_INVALID`, `SESSION_REVOKED` |
| 403 | `FORBIDDEN`, `MFA_REQUIRED`, `ACCOUNT_SUSPENDED`; no protected data disclosed |
| 404 | `NOT_FOUND`; also use for unowned/unassigned resource IDs to avoid existence disclosure |
| 409 | `INVALID_STATE`, `CAPACITY_FULL`, `MEMBERSHIP_CONFLICT`, `IDEMPOTENCY_CONFLICT`, `QUOTE_STALE`, `POLICY_PENDING`, `ROSTER_LOCKED`, `SOURCE_VERSION_STALE` |
| 410 | Expired public invitation token or deleted export; private resources still require authorization before distinguishing expiry |
| 412 / 428 | `REVISION_MISMATCH` / `REVISION_REQUIRED` |
| 413 / 415 | Request/upload too large / unsupported media type |
| 422 | `VALIDATION_FAILED`, `DEADLINE_PASSED`, `INELIGIBLE`, `CONSENT_REQUIRED`, `ASSET_NOT_READY`, invalid scoring/quantity |
| 429 | `RATE_LIMITED`, `COMMAND_BUDGET_EXCEEDED`; include `Retry-After` |
| 503 | `DEPENDENCY_UNAVAILABLE`, `RETRYABLE_TRANSACTION`; same key may be retried |
| 500 | Unexpected failure; safe generic message with request ID; no SQL/stack/secret exposure |

An opaque scan duplicate is **200 with the existing receipt**, not an error. Duplicate entry creation may return the existing own draft if it is semantically the same request; a new conflicting request returns 409. HTTP arrival time alone never grants eligibility past a server deadline.

### 1.4 Idempotency and optimistic concurrency

Canonical request hash includes operation, path IDs, validated payload and expected revision, scoped to the verified actor. Same key/hash replays the original safe business receipt; different hash conflicts. Authorization is checked on replay, so a revoked staff user cannot retrieve prior private data. Receipts do not contain ephemeral signed URLs; those are minted after a fresh access check.

The business mutation and completed receipt are atomic. A competing identical command waits for the transaction or receives a documented retryable response. A timeout is an unknown outcome: query own receipt or retry the **same** key. Retry transient failures with bounded backoff/jitter; do not generate a new key automatically. Never retry a validation/permission error blindly.

**R** indicates compare-and-swap using the resource's current revision. Atomic actions such as scan, trade, redeem, answer and pass-use use locked invariant checks rather than stale UI revisions. An actual replay is recognized before rejecting a stale revision, after current access validation.

## 2. Identity, catalog and administration registry

Notation: `F` festival ID; `E` event ID; route variables are UUIDs unless explicitly slugs. Listed input fields are exhaustive for the operation; unknown fields are rejected. `reason` is bounded nonempty text where present. All staff actions require current scoped permission and AAL2 per RBAC.

| Method / route | RPC operation | Input | Successful response / guard |
|---|---|---|---|
| GET `/festivals/{slug}` | `read_festival` | Published slug | 200 public festival/date/approval labels |
| GET `/festivals/{F}/events` | `read_events` | category/day/cursor/limit | 200 Page<EventSummary>, published only |
| GET `/events/{E}` | `read_event` | E | 200 public rules, required fields, approved cap status, sessions |
| GET `/festivals/{F}/schedule` | `read_schedule` | day/category | 200 published session/venue list; explicit reporting vs contest |
| GET `/festivals/{F}/announcements` | `read_announcements` | cursor/limit | 200 authorized audience projection |
| GET `/me` | `read_me` | none | 200 own participant code/profile/completion/capability hints |
| PUT `/me/profile` **R** | `cmd_update_profile` | display_name, college_id, phone | 200 own profile; email changed only through Auth verification |
| POST `/me/consents` | `cmd_accept_consent` | consent_version_id, optional entry_id | 201 acceptance for current user; never another member |
| GET `/me/dashboard` | `read_dashboard` | F | 200 own entries, states, deadlines, next actions, notices |
| GET `/me/notifications` | `read_notifications` | cursor/limit | 200 own notifications and delivery-safe status |
| POST `/me/notifications/{id}/read` | `cmd_read_notification` | none | 200 own read receipt |
| GET `/me/commands/{key}` | `read_command_receipt` | operation query | 200 own authorized safe receipt or 404 |
| GET `/staff/events` | `read_staff_events` | F | 200 currently assigned scopes only |
| GET `/staff/decisions` | `read_decisions` | F/E/state | 200 permitted decision register |
| POST `/staff/decisions/{id}/versions` **R** | `cmd_propose_decision` | proposed_value, reason | 201 pending decision version |
| POST `/staff/decisions/{id}/approve` **R** | `cmd_approve_decision` | version_id, reason | 200 approval; dependency impacts recorded |
| POST `/staff/events/{E}/rules` **R** | `cmd_propose_rules` | rules, form_schema, decision_version_ids | 201 draft immutable candidate |
| POST `/staff/events/{E}/rules/{id}/approve` | `cmd_approve_rules` | reason, effective_at | 200 approved version; unresolved dependencies block |
| POST `/staff/events/{E}/submission-stages` | `cmd_configure_submission_stages` | approved_rule_version_id, stage definitions/windows/requirements | 200 version-checked stages; active/frozen stages require impact review |
| POST `/staff/events/{E}/rubrics` | `cmd_propose_rubric` | code, criteria, ranges, weights, decision_version_ids | 201 draft version with validated totals |
| POST `/staff/rubrics/{id}/approve` **R** | `cmd_approve_rubric` | reason | 200 immutable approved rubric; independent approver policy applies |
| PUT `/staff/events/{E}/sessions/{id}` **R** | `cmd_update_session` | kind, festival_day_id, starts_at, ends_at, venue_ids | 200 draft session, conflicts listed |
| POST `/staff/events/{E}/publish` **R** | `cmd_publish_event` | rule_version_id, session_ids | 200 publication; approvals/conflicts checked |
| POST `/staff/events/{E}/capacity` **R** | `cmd_set_capacity` | purpose, approved_limit, unit, rule_version_id | 200 pool revision; cannot erase live allocations |
| POST `/staff/announcements` | `cmd_publish_announcement` | F, optional E, audience, title, body, expires_at | 201 sanitized authorized announcement |
| GET `/staff/operations` | `read_operations` | F/E | 200 redacted job/release/delivery/health summary |
| POST `/staff/jobs/{id}/retry` | `cmd_retry_job` | reason | 202 bounded retry; cannot bypass underlying guard |

Supabase Auth SDK owns sign-in/signup/email verification/recovery/session refresh/signout/MFA. The application must not proxy passwords into its database or create a custom credential table. Auth event → idempotent profile bootstrap; dashboard may offer a retry if bootstrap is delayed. JWT refresh alone does not create another participant.

## 3. Entries, invitations, rosters and payments

| Method / route | RPC operation | Input | Successful response / guard |
|---|---|---|---|
| POST `/events/{E}/entries` | `cmd_create_entry` | kind, optional team_name, approved_source_code? | 201 own draft; no seat reserved |
| GET `/entries/{id}` | `read_entry` | id | 200 own/accepted-team/assigned-staff projection |
| PUT `/entries/{id}/form` **R** | `cmd_save_entry_form` | rule_version_id, answers, player_details | 200 draft; strict event-specific validation |
| POST `/entries/{id}/invitations` | `cmd_invite_member` | recipient_email, member_role | 202 generic delivery receipt; no account enumeration |
| POST `/invitations/accept` | `cmd_accept_invitation` | token | 200 accepted membership; verified recipient, expiry and event uniqueness |
| POST `/invitations/decline` | `cmd_decline_invitation` | token | 200 decline receipt for intended recipient |
| POST `/entries/{id}/invitations/{invite}/revoke` | `cmd_revoke_invitation` | reason | 200 captain's live invitation revoked |
| POST `/entries/{id}/members/{member}/release` **R** | `cmd_release_member` | reason | 200 authorized prelock removal/leave; captain integrity preserved |
| POST `/entries/{id}/captain-transfers` **R** | `cmd_request_captain_transfer` | to_profile_id, reason | 201 current captain's expiring transfer request |
| POST `/captain-transfers/{id}/accept` | `cmd_accept_captain_transfer` | none | 200 intended active successor's personal acceptance receipt |
| POST `/entries/{id}/captain` **R** | `cmd_transfer_captain` | new_captain_profile_id, accepted_transfer_request_id, reason | 200 approved accepted successor; locked policy checked |
| POST `/entries/{id}/submit` **R** | `cmd_submit_entry` | consent_version_ids, rule_version_id | 200 submitted or auto-confirmed only if approved deterministic policy permits |
| POST `/entries/{id}/confirm` **R** | `cmd_confirm_entry` | none | 200 staff/approved workflow confirmation; atomic cap/payment/roster checks |
| POST `/entries/{id}/withdraw` **R** | `cmd_withdraw_entry` | reason | 200 capacity released once; refund separately governed |
| POST `/entries/{id}/reopen` **R** | `cmd_reopen_entry` | reason | 200 allowed draft reopening before cutoff; fresh checks on resubmit |
| POST `/entries/{id}/roster-changes` **R** | `cmd_request_roster_change` | base_roster_version_id, proposed_change, reason | 201 request; consent required before acceptance |
| POST `/staff/roster-changes/{id}/resolve` **R** | `cmd_resolve_roster_change` | decision, reason | 200 approved new roster version or rejection |
| GET `/staff/events/{E}/entries` | `read_event_entries` | state/search/cursor/limit | 200 scoped Page<EntryDeskSummary> |
| POST `/staff/events/{E}/entries/{id}/cancel` **R** | `cmd_cancel_entry` | reason | 200 scoped cancellation, capacity release and notice |
| POST `/staff/events/{E}/waitlist/promote` | `cmd_promote_waitlist` | entry_id, reason | 200 fresh checks/allocation; approved waitlist order |
| POST `/entries/{id}/payments` | `cmd_create_payment` | fee_policy_version_id | 202 payment intent/job; disabled without approved fee policy |
| POST `/staff/payments/{id}/verify` **R** | `cmd_verify_payment` | evidence_asset_id, reason | 200 finance verification; amount/currency/ownership checks |
| POST `/staff/payments/{id}/refunds` | `cmd_request_refund` | amount, reason | 202 authorized refund request/reservation; provider completion asynchronous |
| GET `/entries/{id}/payments` | `read_entry_payments` | id | 200 own/finance safe status; never provider secrets |

Payment provider selection is PENDING. A provider adapter must create its own signed callback contract before enabling payment. Browser return URLs and uploaded screenshots alone cannot mark payment verified. Required staff verification remains a separate action even after provider payment confirmation.

## 4. Upload, submission, review and finalist registry

| Method / route | RPC operation | Input | Successful response / guard |
|---|---|---|---|
| POST `/entries/{id}/upload-intents` | `cmd_create_upload_intent` | stage_id, purpose, filename, declared_mime, declared_bytes | 201 intent ID, server path, expiry, permitted upload method; approved file limits |
| POST `/staff/events/{E}/upload-intents` | `cmd_create_staff_upload_intent` | purpose score_import/evidence/pass_list/reference, filename, declared_mime, declared_bytes | 201 scoped intent after corresponding import/referee/pass/mentor permission check |
| POST `/upload-intents/{id}/finalize` | `cmd_finalize_upload` | none | 202 inspect/scan job; client hash/type never trusted as verification |
| GET `/upload-intents/{id}` | `read_upload_intent` | id | 200 own scoped ready/quarantine/rejected status |
| POST `/assets/{id}/download` | `cmd_authorize_asset_download` | purpose | 200 short-lived authorized URL; not an idempotency-cached capability |
| GET `/entries/{id}/stages` | `read_entry_stages` | id | 200 authorized stage requirements, windows, states |
| PUT `/entries/{id}/submissions/{stage}` **R** | `cmd_save_submission` | draft_payload, asset_ids | 200 saved draft; separate from registration |
| POST `/entries/{id}/artifact-manifests` | `cmd_prepare_manifest` | stage_id, repository_url?, commit_ref?, video_asset_id?, project_url? | 202 safe capture/access-check job |
| GET `/artifact-manifests/{id}` | `read_manifest` | id | 200 authorized ready/pending/broken status and pinned commit |
| POST `/entries/{id}/submissions/{stage}/submit` **R** | `cmd_submit_submission` | rule_version_id, manifest_id?, asset_ids | 201 immutable version receipt; all stage requirements ready before cutoff |
| GET `/submissions/{id}/versions` | `read_submission_versions` | cursor/limit | 200 authorized immutable history |
| POST `/staff/submissions/{id}/freeze` **R** | `cmd_freeze_submission` | version_id, reason | 200 approved version freeze; no arbitrary late replacement |
| POST `/staff/review-assignments` | `cmd_assign_review` | submission_version_id, reviewer_id, rubric_version_id | 201 scoped conflict-checked assignment |
| GET `/staff/reviews` | `read_assigned_reviews` | E/state/cursor/limit | 200 own assignments or authorized lead overview |
| PUT `/staff/review-assignments/{id}/review` **R** | `cmd_save_review` | criterion_scores, private_notes?, performer_feedback? | 200 version/rubric validated draft |
| POST `/staff/review-assignments/{id}/submit` **R** | `cmd_submit_review` | none | 200 locked review revision with calculated total |
| POST `/staff/events/{E}/selection/preview` | `cmd_preview_selection` | rubric_version_id, rule_version_id | 202 immutable preview batch with stale/missing review warnings |
| POST `/staff/selection-batches/{id}/approve` **R** | `cmd_approve_selection` | source_digest, selected_entry_ids, reserve_entry_ids, reason | 200 atomic approved batch/quota and notices |
| POST `/entries/{id}/finalist-confirmation` | `cmd_respond_finalist` | selection_decision_id, response confirm/decline | 200 response before approved deadline |
| POST `/staff/events/{E}/reserves/promote` | `cmd_promote_reserve` | entry_id, vacancy_source_id, reason | 200 next approved reserve with eligibility recheck |

### 4.1 Upload and external-link rules

Use private buckets and resumable direct-to-Storage uploads for larger media. Upload-intent creation returns no public permanent URL. File extension, declared type, detected signature, size and scanning policy must agree. Keep files quarantined until validated; generated object names prevent path traversal/collision. Archives need expansion/file-count limits; submitted code is never executed by the portal. A dedicated isolated worker may be required for scanning and archival; do not assume these capabilities exist in Supabase automatically. [OWASP uploads](https://cheatsheetseries.owasp.org/cheatsheets/File_Upload_Cheat_Sheet.html), [Supabase Storage](https://supabase.com/docs/guides/storage/security/access-control)

No arbitrary server-side URL fetch. Allow approved HTTPS providers; reject private/link-local/loopback/cloud-metadata IPs, credential-bearing URLs and disallowed ports; revalidate DNS and every redirect, bound time/bytes, and redact query secrets. Access checks use isolated workers without production credentials. Failure remains visible as broken/pending; staff cannot silently mark an inaccessible project as frozen and judge-ready.

PROPOSED download URL lifetime: 60 seconds, adjustable after AV/upload rehearsal. It is a bearer capability until expiry; role revocation cannot be assumed to invalidate an already issued link instantly. Sensitive credential/answer endpoints never use signed object URLs. For stronger immediate download revocation, use a streaming authorization proxy and revise capacity sizing.

### 4.2 Event payload contracts

These fields are inside the approved event/stage schema, not free-form extra columns accepted by every command. Server rejects unknown fields, validates approved IDs, and distinguishes an empty draft from a complete submission. Bounds/file sizes must be configured through DEC-18 before uploads open.

| Context | Required fields from supplied PRDs | Notes |
|---|---|---|
| Common registration | name/profile, verified email, phone, college, personal required consents | Profile/contact records are canonical; entry stores necessary versioned evidence |
| Sprintathon registration | 3–5 accepted members plus approved form extras | Extra field definitions were not supplied; do not invent them |
| Sprintathon online solution | problem_id, idea_or_approach, solution_link OR ready solution_asset_id | Approved problem version; separate saved state/deadline from registration |
| Sprintathon final project | project/code/deployment link, exact captured commit/version, demo_video reference | README optional; immutable ready manifest required for judged assets |
| Algo Rumble entry | codeforces_handle, leetcode_handle | Both required; provider verification/mapping before contest |
| FIFA entry | name/contact/college through profile, platform, gamer_tag | Approved platform/edition values only |
| BGMI entry | team_name, captain, captain contact/college; four active accepted players with game_player_id and ign | Substitutes do not count as approved active players automatically |
| Override bug report | issue_id, reproduction, evidence_asset_id, fix_manifest_id | Reviewer applies approved duplicate/weight rules |
| Override feature submission | ready manifest for starter-repo build; required-feature evidence | Stage-specific approved rubric, 90-minute window |
| Rhythm Riot entry | profile contacts, approved solo/group details | Advance video and final audio are distinct required submissions |
| Rhythm Riot video/audio | ready video_asset_id / ready audio_asset_id with performance/cue linkage | Audio version pinned to stage slot and package |

Staff verdict, score approval, paid status, ownership and role fields never live inside participant-controlled form answers.

## 5. Doomsday, coding, sports and Override registry

| Method / route | RPC operation | Input | Successful response / guard |
|---|---|---|---|
| GET `/entries/{id}/doomsday` | `read_challenges` | id | 200 selected locked captain's released prompts/attempt allowance/effects only |
| POST `/staff/events/{E}/challenge-config` | `cmd_configure_challenges` | approved_rule_version_id, types, versioned variants/answer rules, four waves/windows/limits | 200 validated private configuration; secret fields never echoed/logged |
| POST `/staff/events/{E}/effect-config` | `cmd_configure_effects` | approved_rule_version_id, allowlisted effect definitions/transitions | 200 validated configuration; protected capabilities cannot be removed |
| POST `/challenge-assignments/{id}/answers` | `cmd_answer_challenge` | answer | 200 attempt/outcome/effect receipt; per-assignment attempt/window lock |
| POST `/staff/challenge-waves/{id}/release` | `cmd_release_wave` | reason | 200 approved due wave fallback; same source key as dispatcher |
| POST `/staff/effects/transitions` | `cmd_activate_transition_effect` | E, transition_code, approved_source_version_id, reason | 200 approved frozen bulk recipients; no individual challenges |
| POST `/staff/effect-recipients/{id}/reverse` | `cmd_reverse_effect` | reason | 200 one reasoned reversal |
| GET `/events/{E}/contest-access` | `read_contest_access` | E | 200 eligible participant's approved contest URL/own linked handle/instructions |
| POST `/staff/events/{E}/contest-config` **R** | `cmd_configure_contest` | provider, external_contest_id, settings_version, approved_capability_evidence | 200 draft/verified integration; unsupported Blackout blocks live start |
| POST `/staff/events/{E}/competition-stages` | `cmd_configure_competition_stages` | approved_rule_version_id, stages/rounds/durations/questions/qualification | 200 explicit approved definitions; unresolved values cannot activate |
| POST `/staff/rounds/{id}/start` **R** | `cmd_start_round` | approved_rule_version_id | 200 server started_at/closes_at; pending timing rules block |
| POST `/staff/rounds/{id}/close` **R** | `cmd_close_round` | reason | 200 preserved closed-round state |
| POST `/staff/stages/{id}/score-imports` | `cmd_import_scores` | source_asset_id, provider, mapping_version | 202 parse/validate; preview only, no final winners |
| GET `/staff/score-imports/{id}` | `read_score_import` | id | 200 mapped/unmapped/rejected rows and proof |
| POST `/staff/score-imports/{id}/approve` **R** | `cmd_approve_score_import` | source_digest, reason | 200 verified stage scores for correct entries |
| POST `/staff/events/{E}/brackets` | `cmd_create_bracket` | approved_seed_order, rule_version_id | 201 approved structure/byes; planning cap not assumed |
| GET `/events/{E}/bracket` | `read_bracket` | E | 200 permitted published bracket/provisional label; private details omitted |
| POST `/staff/matches/{id}/results` **R** | `cmd_record_match_result` | outcome, winner_entry_id?, scores, evidence_asset_id, reason? | 201 referee result version; entrants/settings validated |
| POST `/staff/matches/{id}/advance` **R** | `cmd_advance_match` | result_version_id | 200 approved one-time advancement; downstream conflict blocks correction |
| POST `/staff/events/{E}/stations` | `cmd_configure_stations` | stations, approved_settings_version | 200 scoped station inventory/settings |
| POST `/staff/matches/{id}/slot` **R** | `cmd_schedule_match` | station_id, starts_at, ends_at | 200 no station occupancy conflict |
| POST `/staff/stages/{id}/lobbies` | `cmd_create_lobbies` | maps, groups, entry_slots, approved_rule_version_id | 201 capacity-checked qualifier/final lobbies |
| POST `/staff/lobbies/{id}/credentials` **R** | `cmd_rotate_lobby_credentials` | credentials, access_opens_at, access_closes_at | 200 metadata receipt only; request body excluded from logs |
| GET `/lobbies/{id}/access` | `read_lobby_access` | id | 200 currently assigned eligible squad's credentials; no-store |
| POST `/staff/lobbies/{id}/scores` **R** | `cmd_record_bgmi_scores` | rows of entry_id/placement/eliminations/evidence_asset_id | 201 provisional match/stage/map score version |
| POST `/staff/lobbies/{id}/scores/verify` **R** | `cmd_verify_bgmi_scores` | score_version, reason | 200 verified scores; qualifier/final separate |
| GET `/entries/{id}/override` | `read_override` | id | 200 stage instructions, synthetic sandbox link, own frozen roster and scores |
| POST `/entries/{id}/bug-reports` | `cmd_submit_bug_report` | stage_id, issue_id, reproduction, evidence_asset_id, fix_manifest_id | 201 version-linked report before cutoff |
| POST `/staff/events/{E}/bug-issues` | `cmd_configure_bug_issues` | approved_rule_version_id, issue codes/points/reference evidence | 200 mentor-approved private issue definitions |
| POST `/staff/bug-reports/{id}/verdict` **R** | `cmd_review_bug_report` | verdict, duplicate_of_id?, criterion_scores, reason | 200 approved policy-based verdict/proof |
| POST `/staff/stages/{id}/feature-checks` **R** | `cmd_set_feature_checks` | rubric_version_id, checks | 200 approved candidate; live/frozen rubric cannot silently change |

Round start/close and submissions are server-authoritative. External judge timing must be verified through the adapter; browser timers alone cannot enforce code hiding or contest cutoffs. Private room credentials never appear in standings/export/realtime responses. Sandbox access must not reveal production secrets or reuse privileged staff sessions on its origin.

## 6. Virtual trading registry

| Method / route | RPC operation | Input | Successful response / guard |
|---|---|---|---|
| POST `/staff/markets/{id}/players` | `cmd_verify_market_player` | entry_id, verification_evidence, reason? | 201 verified onsite entry/player; no online pre-quiz |
| POST `/staff/markets/{id}/players/{player}/wallet` | `cmd_issue_wallet` | none | 201/200 one equal opening wallet/ledger receipt |
| GET `/markets/{id}` | `read_market` | id | 200 participant's current market state, version, prices/news, server time |
| GET `/markets/{id}/portfolio` | `read_portfolio` | id | 200 own cash/holdings/valuation; exact decimals and quantities |
| GET `/markets/{id}/orders` | `read_orders` | cursor/limit | 200 own trade history; scoped operator audit separately |
| POST `/markets/{id}/orders` | `cmd_place_order` | instrument_id, side buy/sell, quantity, expected_price_version_id | 201 executed trade/portfolio revision or atomic rejection |
| POST `/staff/markets/{id}/prices` **R** | `cmd_publish_market_prices` | complete_quotes, news, approved_source | 201 immutable complete price version visible consistently |
| POST `/staff/markets/{id}/open` **R** | `cmd_open_market` | reason | 200 approved session within window |
| POST `/staff/markets/{id}/pause` **R** | `cmd_pause_market` | reason | 200 paused; new orders rejected |
| POST `/staff/markets/{id}/resume` **R** | `cmd_resume_market` | reason | 200 open only before close deadline |
| POST `/staff/markets/{id}/close` **R** | `cmd_close_market` | reason | 200 frozen close price version/ledger sequence |
| POST `/staff/markets/{id}/valuation` | `cmd_finalize_market` | rule_version_id | 202 reproducible valuation; missing quote/ledger mismatch blocks |
| GET `/markets/{id}/standings` | `read_market_standings` | cursor/limit | 200 approved/provisional label and safe ranks; final awards through Results |

Order example:

```http
POST /markets/11111111-1111-4111-8111-111111111111/orders
Authorization: Bearer <participant-token>
Idempotency-Key: 22222222-2222-4222-8222-222222222222
Content-Type: application/json

{
  "instrument_id": "33333333-3333-4333-8333-333333333333",
  "side": "buy",
  "quantity": "2",
  "expected_price_version_id": "44444444-4444-4444-8444-444444444444"
}
```

The server chooses wallet, actor, quote price, gross value and execution time. Reject client attempts to set wallet balance, execution price or another player's ID. At/after `closes_at`, reject even if a worker has not yet changed market state. Reconnection re-fetches current market/portfolio; it never replays a new order automatically.

## 7. Check-in, admissions, performers and Arcade registry

| Method / route | RPC operation | Input | Successful response / guard |
|---|---|---|---|
| POST `/staff/events/{E}/check-in/lookup` | `read_check_in_lookup` | festival_day_id, qr_token OR participant_code OR registration_code | 200 minimal eligibility/person roster; identifiers not placed in URL analytics |
| POST `/staff/events/{E}/check-ins` | `cmd_check_in` | profile_id, entry_id, festival_day_id, optional approved_exception_id | 201/200 actual individual attendance receipt |
| POST `/staff/events/{E}/roster-verifications` | `cmd_verify_roster` | entry_id, roster_version_id, kind | 201 verification only; no bulk person attendance |
| POST `/staff/events/{E}/exceptions` | `cmd_create_desk_exception` | profile_id, entry_id?, kind, evidence_asset_id?, reason | 201 only explicitly permitted exception type |
| POST `/staff/events/{E}/offline-exports` | `cmd_request_offline_roster` | festival_day_id, device_id | 202 minimal expiring private roster export |
| POST `/staff/offline-imports` | `cmd_import_offline_check_ins` | export_id, device_id, observations | 202 bounded merge job; accepted/duplicate/conflict per observation |
| GET `/staff/offline-imports/{id}` | `read_offline_import` | id | 200 own authorized reconciliation outcome |
| POST `/staff/events/{E}/passes/import` | `cmd_import_passes` | approved_list_asset_id, decision_version_id | 202 validated minimal guest/creator list |
| POST `/staff/events/{E}/passes/use` | `cmd_use_pass` | token, festival_day_id, quantity | 201/200 locked entitlement use; competition registration untouched |
| GET `/entries/{id}/performance` | `read_performance` | id | 200 own performance/video/audio/review status |
| PUT `/entries/{id}/performance` **R** | `cmd_save_performance` | format, title, approved_group_details | 200 validated draft; video/audio via submission stages |
| POST `/staff/performances/{id}/review` **R** | `cmd_review_performance` | submission_version_id, status, feedback | 200 Pending/Changes needed/Approved; private comments |
| POST `/staff/events/{E}/stage-order` **R** | `cmd_set_stage_order` | ordered_performance_ids, cue_codes, audio_version_ids, roster_version_ids | 200 approved/pending-ready list; exact version linkage |
| POST `/staff/events/{E}/performance-packages` | `cmd_build_performance_package` | stage_order_revision | 202 versioned offline bundle/cue manifest |
| POST `/staff/performance-packages/{id}/playback-checks` | `cmd_record_playback_check` | stage_slot_id, device_role main/backup, outcome, reason? | 201 actual AV verification; new audio needs new checks |
| POST `/staff/events/{E}/coupons` | `cmd_issue_coupon` | fee_policy_version_id, payment_record_id? | 201 unique coupon at approved price; feature gated |
| POST `/staff/events/{E}/coupons/redeem` | `cmd_redeem_coupon` | token, prize_items? | 201/200 redemption/stock receipt; both atomic when prize included |
| POST `/staff/coupons/{id}/void` | `cmd_void_coupon` | reason | 200 approved void; redeemed state governed by policy |
| POST `/staff/coupons/{id}/refund` | `cmd_refund_coupon` | reason, approved_payment_refund_id | 200 refund linkage; no future redemption |
| POST `/staff/events/{E}/prize-stock` | `cmd_adjust_prize_stock` | sku, quantity_delta, kind, reason | 201 opening/issue/return/adjustment record; explicit authority |
| GET `/staff/events/{E}/arcade-reconciliation` | `read_arcade_reconciliation` | business_day | 200 coupons/payment/stock totals and discrepancies |

Offline import body is bounded (PROPOSED max 250 observations/request), each with `operation_id`, `profile_id`, `entry_id`, `festival_day_id`, `observed_at`, `observed_outcome`. Export manifest identifies the staff/device/event/day and rules snapshot. Client times are evidence, not trusted eligibility timestamps. Online reconciliation uses current authority and policy; conflicts remain provisional until an authorized resolution.

## 8. Results, Cup, exports and staff-access registry

| Method / route | RPC operation | Input | Successful response / guard |
|---|---|---|---|
| POST `/staff/events/{E}/results` | `cmd_create_results` | category_code, rule_version_id, rows with entry/roster/score/placement/proof/source | 201 draft version; valid entrants only |
| POST `/staff/result-versions/{id}/submit` **R** | `cmd_submit_results` | source_digest | 200 pending approval; content locked |
| POST `/staff/result-versions/{id}/approve` **R** | `cmd_approve_results` | reason | 200 approved metadata; event lead scope required |
| POST `/staff/result-versions/{id}/publish` **R** | `cmd_publish_results` | expected_current_version_id? | 200 current pointer/projection updated atomically |
| POST `/staff/result-versions/{id}/corrections` | `cmd_correct_results` | corrected_rows, reason, source_proof | 201 new draft; same submit/approve/publish lifecycle |
| GET `/events/{E}/results` | `read_public_results` | category/cursor/limit | 200 only current published safe rows with source version |
| GET `/me/results` | `read_my_results` | F | 200 published placements for own historical accepted roster |
| POST `/result-versions/{id}/disputes` | `cmd_submit_dispute` | entry_id, reason, evidence_asset_id? | 201 authorized pre-cutoff dispute |
| POST `/staff/disputes/{id}/resolve` **R** | `cmd_resolve_dispute` | outcome, reason, correction_version_id? | 200 reviewer outcome; correction not silently published |
| POST `/staff/stage-handoffs` | `cmd_create_stage_handoff` | result_version_id? OR performance_package_id?, recipient_role | 202 approved package export |
| POST `/staff/stage-handoffs/{id}/acknowledge` | `cmd_acknowledge_handoff` | manifest_hash | 200 recipient acknowledgement for exact version |
| POST `/festivals/{F}/cup/squads` | `cmd_create_cup_squad` | name | 201 optional squad; not an event team |
| POST `/cup/squads/{id}/invitations` | `cmd_invite_cup_member` | recipient_email | 202 independent consent invitation |
| POST `/cup/invitations/accept` | `cmd_accept_cup_invitation` | token, consent_version_id | 200 accepted before affiliation lock |
| POST `/cup/squads/{id}/leave` **R** | `cmd_leave_cup_squad` | reason | 200 prelock withdrawal; captain succession checked |
| POST `/staff/festivals/{F}/cup/policy` | `cmd_propose_cup_policy` | weights, eligibility, tie_rules, affiliation_lock_at, decision_version_ids | 201 draft policy |
| POST `/staff/cup/policies/{id}/approve` **R** | `cmd_approve_cup_policy` | reason | 200 approved immutable policy with explicit consent/lock rules |
| POST `/staff/festivals/{F}/cup/lock` | `cmd_lock_cup_affiliations` | policy_version_id | 200 frozen consenting squad affiliations after approved deadline |
| POST `/staff/festivals/{F}/cup/calculate` | `cmd_calculate_cup` | policy_version_id, source_results_digest | 202 versioned batch; best valid squad finish once/event |
| POST `/staff/cup/batches/{id}/publish` **R** | `cmd_publish_cup` | reason | 200 approved standings batch |
| GET `/festivals/{F}/leaderboard` | `read_cup_standings` | cursor/limit | 200 approved enabled Cup breakdown; POLICY_PENDING otherwise |
| POST `/staff/exports` | `cmd_request_export` | F, optional E, kind, allowed_filters | 202 export job; permission-specific columns only |
| GET `/staff/exports/{id}` | `read_export_job` | id | 200 status/expiry/manifest; no cross-scope existence disclosure |
| POST `/staff/exports/{id}/download` | `cmd_download_export` | none | 200 freshly authorized expiring URL; audit download issuance |
| GET `/staff/attendance-summary` | `read_attendance_summary` | F/E/day | 200 distinct festival/daily/event totals and pass admissions separately |
| POST `/staff/assignments` | `cmd_grant_assignment` | profile_id, role_code, festival_id, event_id?, expires_at, reason | 201 allowed delegated scope only; MFA and grant policy |
| POST `/staff/assignments/{id}/revoke` **R** | `cmd_revoke_assignment` | reason | 200 live revocation; audited |
| POST `/staff/accounts/{id}/suspend` | `cmd_suspend_account` | reason | 200 app access suspended; session handling per RBAC |
| GET `/staff/audit` | `read_audit` | F/E/action/cursor/limit | 200 scoped redacted audit; no arbitrary SQL |

Noesis Cup and payments remain disabled until approval. An unpublished result version remains invisible even if its UUID is known. A participant's historical placement uses the accepted roster snapshot at the time of judging, not today's mutable team membership.

## 9. Webhooks and internal workers

These are separate routes/credentials, not ordinary participant commands. Expose only after an adapter is selected and tested.

| Route / worker | Authentication | Behavior |
|---|---|---|
| POST `/webhooks/payments/{provider}` | Provider signature over raw bytes + replay/event ID validation | Persist verified event, enqueue reconcile; duplicate no-op; never trust browser redirect |
| POST `/webhooks/email/{provider}` | Provider signature and event ID | Update delivery attempt status only; cannot change entry/selection |
| POST `/webhooks/judge/{provider}` | Approved signature or authenticated pull integration | Import provisional source; unknown entries quarantined; no direct final publication |
| Internal due-job dispatcher | Dedicated protected backend identity; not user-supplied actor header | Claim bounded due jobs; invoke backend-only allowlisted operations |
| Internal asset/manifest worker | Isolated credential scoped to required objects/jobs | Verify bytes/archive; no production code execution; update source-version-checked job |
| Internal outbox/export worker | Separate protected worker credential | Recheck payload scope; bounded retries/leases; dead-letter visibility |

Service credentials are never in Vite environment variables. Internal worker RPCs revoke EXECUTE from anon/authenticated and require backend identity; participant JWTs cannot select worker actions. If a gateway disables default JWT verification for a signed provider webhook, the handler must verify the provider signature itself before any trusted mutation.

Supabase workers have execution/resource limits. Break long workflows into persisted jobs; no overnight loop inside an Edge request. [Edge limits](https://supabase.com/docs/guides/functions/limits), [Cron](https://supabase.com/docs/guides/cron)

## 10. Cache, realtime, observability and compatibility

- Private responses: `Cache-Control: no-store`; never cache credentials/signed links/contact lists at a CDN. Public catalog/results may use short cache TTL plus ETag and revision-aware invalidation. Stage displays show version and last refresh.
- Realtime sends safe invalidation hints `(resource_id,revision)` only. Reconnect fetches authoritative HTTP state; duplicate/out-of-order messages cannot roll state backward. No hidden answer/room/PII values on an already open channel.
- Log request ID, actor pseudonymous ID, operation, scoped resource, outcome code and duration. Redact tokens, invitations, QR bodies, credentials, contact bodies, answers and raw provider secrets.
- Use OpenAPI/JSON Schema generation from reviewed contracts when implementing. Shared TypeScript types do not validate untrusted requests; run runtime validation server-side.
- Compatible v1 changes are additive optional fields. New required fields or changed status semantics require versioning/migration and client rollout. Deploy database/API compatibility before frontend. Avoid two clients applying conflicting rule versions.

## 11. API test catalog

Run the cross-cutting families against **every applicable registry operation**, including direct RPC variants where marked. Business scenario IDs remain in the architecture; database and authorization catalogs are additional required suites.

| ID | Stimulus | Expected behavior |
|---|---|---|
| API-01 | Missing/expired/forged/wrong-project token | 401, zero mutation, safe error |
| API-02 | Correct user but wrong event/team/resource | 403/404 with no protected data; same denial through direct RPC |
| API-03 | Omit MFA on staff route | 403 MFA_REQUIRED, including signed export/credential routes |
| API-04 | Malformed JSON, content type, unknown fields, oversized body | 400/413/415; no partial parse/mutation |
| API-05 | Inject actor_id/state/paid/wallet_id/approval in payload | Unknown/forbidden field rejected; server-derived values unchanged |
| API-06 | Missing/invalid idempotency key | 400; no mutation |
| API-07 | Same key same payload after response loss | Same receipt and replay flag; one business action |
| API-08 | Same key different body/path/revision | 409 IDEMPOTENCY_CONFLICT |
| API-09 | Two identical in-flight requests | One commit; other waits/retries/replays without duplicate |
| API-10 | Missing/stale If-Match on R operation | 428/412; no lost update |
| API-11 | Replay successful R operation after revision advanced | Existing receipt after current access check, no second update |
| API-12 | Boundary timestamp and lock wait across deadline | Consistent 422; no client-clock/HTTP-arrival override |
| API-13 | Time without offset, local midnight, leap date, wrong day | Reject ambiguous invalid input; correct explicit IST/UTC output |
| API-14 | Decimal exponent/NaN/infinity/overflow/unsafe integer | 422; exact strings preserved for valid values |
| API-15 | Tampered cursor, huge limit, changed filters, deleted row | Safe bounded pagination; no duplicate/leaked scope |
| API-16 | Retry 503/timeout versus 422/403 | Same-key bounded transient retries only |
| API-17 | Auth refreshed midform; account switched in another tab | Preserve allowed draft, clear old private cache, never submit as wrong user |
| API-18 | Bypass gateway and call RPC directly | Same domain/permission/idempotency/quota denial |
| API-19 | Upload to unauthorized path/type/size; overwrite frozen file | Storage and finalize reject; no accepted submission |
| API-20 | Submit while scan/archive pending or worker failed | ASSET_NOT_READY; dashboard shows recoverable status |
| API-21 | URL redirects/DNS rebinding to metadata/private host | Fetch blocked; bounded worker and redacted error |
| API-22 | Revoked user requests export/download or replays receipt | Recheck denies; previously issued link only valid until documented expiry |
| API-23 | Fake/duplicate/out-of-order webhook | Invalid signature rejected; duplicates no-op; reconcile authoritative state |
| API-24 | Provider accepted then worker crashed | Recover/reconcile without duplicate business effect; uncertain email visible |
| API-25 | Stale quote order / order after pause/close | QUOTE_STALE/INVALID_STATE/DEADLINE_PASSED; no trade |
| API-26 | Scan/coupon/pass duplicate requests from two devices | Stable receipt or valid conflict; no duplicate attendance/stock/quantity use |
| API-27 | Offline batch mixed valid/duplicate/conflict observations | Per-item outcomes and resumable job; no false all-success summary |
| API-28 | Draft/unapproved result through API, RPC, export, realtime | No public leakage; staff response explicitly provisional |
| API-29 | Rule/selection source version changes before approval | 409 SOURCE_VERSION_STALE; regenerate preview |
| API-30 | Unicode names, HTML/script text, CSV formula values | Safe display and exported escaping; legitimate names preserved |
| API-31 | Search probes for account/invite/contact existence | Generic nonenumerating response and rate limits |
| API-32 | API/RPC burst after Edge rate-limit bypass | Database command budget/attempt limits preserve correctness |
| API-33 | Dependency or SQL error | Safe 5xx with request ID; logs contain diagnostic code without secrets |
| API-34 | Public/private caching, logout, shared desk device | No private response reused across users; only safe public content cached |
| API-35 | Out-of-order realtime updates / disconnect | Refetch current revision; no local authority or hidden payload |
| API-36 | Internal worker/webhook called using participant token | Denied before job claim or trusted mutation |
| API-37 | Endpoint/RPC/schema names drift during implementation | Contract validation fails CI; no undocumented business mutation path |
| API-38 | Zero-result export, expired asset, partial async job | Explicit truthful state; bounded retry; no broken-success download |
| API-39 | Frontend/API versions differ in supported fields | Documented compatible behavior or explicit version rejection |
| API-40 | Audit publication/role/exception/refund action | Actor/scope/source/reason/request IDs persisted without sensitive payload |
| API-41 | Configure stages/rubrics/effects after live use | Impact review/version checks; no silent rewrite of frozen rules or protected capability |
| API-42 | Staff source-file upload without corresponding import permission | Denied at intent, Storage and finalization boundaries |

Coverage is a cross-product, not forty standalone happy-path requests: exercise applicable auth, scope, validation, retries and concurrency for every mutation. Write provider-specific tests only after an adapter and its official signature contract are chosen.
