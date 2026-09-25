# MDK implementation work

Status: non-normative action plan, 2026-09-25. Read the [overview](README.md) and
[protocol contract](protocol-plan.md) first. Paths below are in `marmot-protocol/mdk`, not this specification repository.
Existing-source evidence is pinned to `31a3722577805b49925142069f98037c9e75d6a2`; refresh locations before implementation.
New filenames and API concepts are proposed implementation units, not claims that those APIs already exist.

## Source map and interface boundaries

| Existing source | Starting point / required change |
| --- | --- |
| `crates/cgka-engine/src/app_components.rs` | `staged_commit_requires_admin`, `commit_ordering_priority_for_staged`, `is_allowed_non_admin_commit` currently lack parent/committer context |
| `crates/cgka-engine/src/message_processor/send.rs` | Invite resulting-state checks around 315–326 depend on admin grants; removal around 465–524 is account-addressed |
| `crates/cgka-engine/src/message_processor/ingest.rs` | Validate against candidate parent; preserve/authenticate leaf provenance currently omitted from `MessageReceived` |
| `crates/cgka-engine/src/distributed_convergence.rs`, `openmls_projection.rs` | Apply the same authority/ack rules during retained-branch replay and selection changes |
| `crates/cgka-engine/src/group_lifecycle.rs` | Early Welcome dedup around 963; authorization around 1305; maintenance scheduling around 1457 |
| `crates/cgka-engine/src/key_package.rs` | `build_fresh_key_package` currently marks generated packages last-resort; add a separate enrollment-purpose builder |
| `crates/cgka-engine/src/capabilities.rs`, `feature_registry.rs`, `identity.rs` | Registry knowledge, advertised support, and identity-proof/signing boundaries |
| `crates/cgka-engine/src/own_commit_intent.rs` | Invite recovery currently concludes completion from account presence |
| `crates/marmot-app/src/client/invite_recovery.rs` | Reuse publication machinery, replace account-presence predicate for enrollment |
| `crates/marmot-app/src/runtime/mod.rs`, `runtime/account_worker.rs` | Device-scoped sign-out, worker lifecycle, serialized mutation and durable retries |
| `crates/marmot-app/src/directory/member_key_packages.rs`, `key_package_records.rs` | Multi-slot evidence without deletion; preserve compatible candidate ranking |
| `crates/marmot-app/src/external_signer.rs`, `runtime/onboarding/single_device.rs` | Exact signer-template verification and conditional onboarding notices |
| `crates/traits`, `crates/cgka-session`, `crates/storage-sqlite` | Typed operations, engine-to-runtime facts, transactional durable storage |
| `crates/marmot-uniffi`, `crates/marmot-c` | Public outcomes, progress/cancellation, exact device removal and documentation |
| `crates/cgka-conformance-simulator/src/family.rs` | Extend `adversarial-reliability/multi-device-account/v1`; existing account/device topology is reusable |

Define the following contracts in shared traits or crate-private types at the layer that owns them. Final Rust names
can follow repository conventions, but semantics must remain consistent across packages and bindings.

| Concept | Required inputs | Required output/invariant | Owner/consumer |
| --- | --- | --- | --- |
| Commit classification | Candidate parent reference/state; authenticated committer; all proposals with inline/reference provenance; resulting state; UpdatePath | Valid authority class plus priority, or typed rejection; never classify on shape alone | M2; all engine paths |
| Leaf target | MLS group ID, parent/branch reference, exact leaf identity/lineage, expected account | Fresh validated target or stale-target result; no index-only deletion after reuse | M3; M6/M8 |
| Enrollment attempt reference | Account-device scope, group ID, local attempt ID, pairing session, intent hash, package reference | Stable local operation identity and exact retry correlation; no account-presence shortcut | M4; M5–M8 |
| Package purpose | Package reference and persisted ordinary/enrollment-only purpose, group/attempt binding | Same-account validation or rejection even after intent retirement | M4; Welcome admission |
| Verified acknowledgement | MLS-authenticated branch and leaf lineage, exact enrollment fields and retained attempt | Verified ack fact, duplicate, stale/no-authority, or invalid; no acceptance from account pubkey alone | M3/M5; M6 |
| Enrollment snapshot | Durable facts and current selected membership/eligibility | Versioned typed status, pending obligations and allowed actions, separate from chat content | M6; M8 |
| Carrier session | Valid descriptor/Hello, explicit local permission, resource budget, monotonic deadlines | Authenticated bounded records or typed terminal reason; no device/group authority | M7; enrollment coordinator |

Use opaque MLS `GroupId` bytes; do not impose the 32-byte Nostr routing-ID width on them. Do not expose raw secrets,
full protocol bytes or internal database paths through UI status objects.

## M1: cleanup and invitation evidence independent of #417

**Files:** runtime sign-out paths, KeyPackage lifecycle traits/storage, directory resolution and outcome conversions.
**Trackers:** [#1697][i1697], [#1570][i1570], [#1696][i1696], [#1641][i1641], [#1706][i1706].
**Produces:** device-owned publication cleanup and truthful invitation outcomes for M6/M8.

- [ ] Persist the installation's stable publication slot, exact signed current event, and snapshotted target relays.
  Relay discovery of account-authored packages is not proof of local ownership. Migrate ambiguous old records without
  claiming unknown sibling slots; return skipped/unproven ownership when evidence is insufficient.
- [ ] On sign-out/wipe, delete only locally owned publications. Prepare and persist exact signed deletion events and
  per-target obligations in lifecycle storage that survives account-directory/key deletion. Retry those exact events
  after restart without invoking a removed/external signer. Report unsigned failures separately from queued publication.
- [ ] Preserve prior unconsumed private material only for its documented delayed-Welcome lifetime. Expiry/consumption
  retirement does not delete another device's publication. Never infer account-wide revocation from device cleanup.
- [ ] Add multi-slot evidence after collapsing addressable-slot replacements. Multiple current unexplained slots produce
  `multiple_unattributed_slots`/`delivery_at_risk`; resolution does not delete, prune or retarget packages.
- [ ] Keep one compatible chosen package per initially invited account and preserve #1910's current ranking policy.
  A client label is a hint, not authenticated device identity. Relay incompleteness cannot prove no sibling exists.
- [ ] Integrate durable per-recipient publication outcomes with recipient-side ingest/catch-up outcomes. Expose commit
  publication, Welcome transport acknowledgement and recipient join as separate facts. Do not block fast canonical
  operation responses on complete Welcome fanout; continue durable background delivery.
- [ ] Add integration cases I01–I05: two devices/sign-out, partial deletion, restart after wipe, unknown slots, same-slot
  supersession, orphan-like newest package, compatible-ranking fallback, and partial Welcome fanout.

**Verification:** targeted `marmot-app`/storage/binding tests plus `just fast-ci`. **Commit boundary:** split durable
cleanup and invitation evidence into independent reviewed PRs; neither advertises multi-device membership support.

## M2: one context-aware authority classifier

**Files:** `app_components.rs`, send/ingest, `openmls_projection.rs`, `distributed_convergence.rs`, group lifecycle,
capability handling and disband validation; proposed focused `same_account_membership.rs` and engine tests.
**Consumes:** P3 matrix. **Produces:** classifier/resulting-state validation used by M3–M5.

- [ ] Implement a single classifier taking the candidate parent, validated committer, proposal provenance and complete
  resulting state. Return authority and convergence priority together to prevent independent decisions drifting.
- [ ] Preserve baseline self-update/SelfRemove and ordinary admin operations. Match the exact enabled same-account
  shapes first with ordinary priority even for an admin; then evaluate independently authorized admin shapes. Invalid
  MLS/proofs/resulting state fail regardless of actor. Do not classify all mixed/admin operations as component failures.
- [ ] Run resulting-state integrity checks for every locally generated Commit, including invitation without admin grants,
  removal, self-update, capability updates and disband-related paths. Add the new invariant hook to all receive/replay paths.
- [ ] Register `0x800d` as known and data-less, reject data entries in every location and validate support/required lists
  through the existing application-component negotiation mechanism. Keep raw registry recognition distinct from advertising.
- [ ] Enforce creation/enablement support, enabled cap and disable guard. Preserve unknown optional component bytes.
- [ ] Build matched send/ingest/replay cases for the full P3 matrix, including referenced admin operations, missing path,
  sixth leaf, duplicate targets, wrong-account package, mixed proposals, invalid disable and malformed component data.
- [ ] Keep advertised support off until all receive and replay cases pass; then thread support through the separate
  `supported_app_components` set for new KeyPackages and own-leaf updates. Enabling a group remains an explicit operation.

**Verification:** proposed `same_account_membership` engine integration target, existing capability/component tests and
send/ingest/replay parity tests. **Commit boundary:** classifier and invariants before any production advertisement.

## M3: exact-leaf removal and acknowledgement provenance

**Files:** send/ingest/replay, engine/session/trait operations, retained-operation storage, proposed `enrollment_ack.rs`,
and removal/binding tests. **Consumes:** M2, P4. **Produces:** leaf-specific mutation and verified ack events.

- [ ] Introduce a sibling-removal intent separate from account-level `RemoveMembers`. Accept 1–4 exact same-account
  targets bound to expected parent/lineage; reject self/duplicates/wrong-account/stale targets. Revalidate at staging.
- [ ] Retain the target identity and expected parent in durable operations. Retry exact exposed bytes; before exposure a
  changed parent returns a stale/reconciliation result rather than removing the new occupant of a reused leaf index.
- [ ] Preserve ordinary admin account removal and provide explicit admin leaf removal where required by P3. No binding
  may convert “remove this device” into whole-account removal because the old API is more convenient.
- [ ] Prefer engine-side ack verification using authenticated MLS sender/branch data before `MessageReceived` loses leaf
  provenance. Alternatively add a protected event/storage contract carrying that provenance; do not trust a payload field
  asserting its own sender leaf. Apply the same verifier during replay and after reopen.
- [ ] Track uninterrupted self-update lineage from the enrolled leaf across relevant retained branches. Remove/Add breaks
  lineage even if account, index, or apparent device label matches. Bind every ack field to the exact staged attempt.
- [ ] Persist the verified ack fact before reporting completion; re-evaluate current selected membership independently
  after branch changes. Ack application messages do not render as chat and cannot inflate account witness quorum.
- [ ] Test A01–A07 and I06: another sibling sends an otherwise correct ack; index reuse; self-update descendant; replay
  under the wrong branch; stale removal intent; partial account removal; surviving sibling push registrations.

**Verification:** targeted engine/session tests with real MLS sender authentication; runtime tests consume only verified
outcomes. **Commit boundary:** trait/storage/API support and engine validation, then binding exposure in M8.

## M4: purpose-bound KeyPackages and durable enrollment ledger

**Files:** `key_package.rs`, engine/storage trait definitions, `storage-sqlite` migrations and transactions; proposed
`crates/traits/src/enrollment.rs`, `crates/cgka-engine/src/enrollment.rs`, and focused storage modules/tests.
**Consumes:** P4 lifecycle. **Produces:** atomic durable records and purpose lookup for M5/M6/M7.

- [ ] Define stable per-group attempt keys and durable independent facts from P4. Retain exact correlation bytes/digests,
  exposure ambiguity, authority/lineage, expiry, selected membership and pending ack/cleanup obligations. Use bounded
  indexed work enumeration; no full history scan on every worker turn.
- [ ] Create an enrollment-package builder that does not call `mark_as_last_resort`. Atomically store OpenMLS private
  bundle and enrollment-only purpose before returning public bytes. The normal relay generator remains separate.
- [ ] Keep consumed package purpose/rejection metadata after secret retirement. Missing approved intent, cancellation,
  expiry, supersession or invalidation never selects the ordinary admin Welcome path.
- [ ] Implement transactional first-join versus terminal-refusal decisions. A durable successful join cannot later be
  declared never joined; a durable refusal cannot later admit that attempt. Serialize competing user/network actions.
- [ ] Make storage migrations additive and fail closed. Existing ordinary packages retain their existing meaning;
  unknown/corrupt enrollment metadata is not converted to an ordinary package. Snapshot/rollback includes all new facts.
- [ ] Define secret erasure, private-file permissions, record/byte bounds and safe compaction. A storage-limit error
  prevents creating more exposed work; it does not delete live obligations or change remote membership claims.
- [ ] Preserve exclusive local account-device database ownership and refuse supported archive/import paths that activate
  a copied live MLS identity as a second installation. Create fresh keys for new devices. Test X01–X03, including the
  explicit limit that local locks and MLS authentication cannot detect all remote copies of the same leaf secrets.
- [ ] Fault-inject write failures, cancellation, crash and reopen at B1–B8. Assert no public exposure before purpose, no
  receipt before durable approval, and no visible join without a durable ack obligation.

**Verification:** transactional storage tests and engine tests on a persistent backend, not only an in-memory ledger.
**Commit boundary:** shared contracts/migration/persistence before callers expose the new package bytes.

## M5: atomic Welcome admission and replay-safe ack scheduling

**Files:** `group_lifecycle.rs`, publish/maintenance, enrollment-ack verifier, session integration and Welcome tests.
**Consumes:** M3/M4. **Produces:** durable join and acknowledgement obligations consumed by M6.

- [ ] Inspect purpose and prior-success correlation before generic `WelcomeAlreadyProcessed` exits. For a recognized
  success, compare the exact serialized Welcome digest before any MLS processing/expiry checks; reject altered bytes.
- [ ] Route new enrollment Welcomes through all P4 checks. Use snapshot/transactional validation to ensure rejection
  leaves group state and a consumable one-shot package unchanged. Preserve ordinary Welcome behavior separately.
- [ ] Persist accepted group, consumed-package lifecycle, Welcome/GroupInfo digests, stable enrollment identity and ack
  obligation atomically. A crash cannot leave joined state without work that produces the required first payload.
- [ ] Gate own-leaf maintenance/self-update and other outbound application traffic until the ack reaches the defined
  publication boundary. Do not block ingress, transport backfill or read-only UI; surface pending publication explicitly.
- [ ] Retry exact exposed ack message bytes while publication is ambiguous. After a later duplicate Welcome triggers
  recovery, generate a fresh MLS-protected ack from a valid current lineage while preserving correlation fields.
- [ ] Check lineage/record validity at each recovery attempt. Removed, discarded, invalidated or Remove/Add-replaced
  leaves send no ack. Rejoin does not resurrect old ack obligations.
- [ ] Test E01–E10, A04–A07 and B5–B7 with maintenance due immediately, expired first delivery, successful replay after
  expiry, lost init key after success, wrong sponsor key, invalid Welcome rollback and signer/transport refusal.

**Verification:** targeted engine and app/session integration tests; no broad relaxation of `WelcomeAlreadyProcessed`.
**Commit boundary:** routing/atomic join, followed by ack scheduling/replay where reviewers can verify each invariant.

## M6: publication and reconciliation orchestration

**Files:** `client/invite_recovery.rs`, `own_commit_intent.rs`, account worker/session bridge; proposed focused
`crates/marmot-app/src/client/enrollment.rs` and runtime enrollment coordinator/tests.
**Consumes:** M1/M3–M5. **Produces:** resumable typed per-group operations for M7/M8.

- [ ] Reuse exact-byte publication/fanout infrastructure without reusing “account already present” as an enrollment
  completion predicate. Require exact added lineage, local key usability and current selected membership evidence.
- [ ] Stage no Add until its receipt is durably observed. Check approved deadline/margin and fresh parent before first
  exposure; if the sponsor signature key changed since approval, obtain a new approved intent instead of rewriting it.
- [ ] Preserve publish-before-apply for Adds and Removes. Treat any possible external exposure as requiring exact-byte
  reconciliation; a network timeout is not proof rollback is safe. Background Welcome delivery remains durable.
- [ ] Limit one active attempt per group. Start with sequential Add publication; if parallelizing across groups, use a
  bounded local queue, initially at most four publishers. Partial success is per group, never a cross-group rollback.
- [ ] Expose independent statuses for approved, awaiting receipt, publication pending/uncertain, locally joined, ack
  pending/verified, currently selected/not-selected, published-unacknowledged, retry-blocked, terminal-admission and
  cleanup pending/selected. Terminality and selected membership are facts, not a single monotonic progress percentage.
- [ ] Implement fresh-session reconciliation after restart/expiry. Authenticate old-attempt references, atomically store
  refusal/key-loss evidence, preserve successful-join evidence, and never infer a negative result from missing response.
  Verify refusal with the original joining key and fresh-session challenge from D5; reject another same-account signer.
  If full state loss destroyed that key, retain uncertainty and offer separately authorized explicit removal, never
  automatic cleanup labelled as proven enrollment failure.
- [ ] Remove a stranded leaf only with positive terminal evidence and fresh exact-target authorization. If the sponsor
  disappeared or was removed, expose authorized-admin/remaining-sibling recovery or a no-authority result.
- [ ] Keep replacement Add blocked while the previous enrollment branch is eligible. Consume core permanent-ineligibility
  evidence, not a local timeout/retirement bit. If retired state returns, report unusable leaf and run explicit cleanup;
  never acknowledge or mark enrollment complete from account presence.
- [ ] Handle explicit retained-group discard/rejoin through the established boundary. Preserve accepted history under
  its provenance/retention policy without copying cryptographic state. Never manufacture a secret-free replay mechanism.
- [ ] Define cancel behavior before exposure, after ambiguous exposure and after join. Cancellation ends new user work,
  not existing remote facts; retain necessary obligations and allow bounded later reconciliation.
- [ ] Test concurrent enrollment near the cap, concurrent cleanup, sponsor removal, uncertain publication, branch loss/
  revival, group deletion, cancellation and repeated restart. Persist seed and trace for each failure.

**Verification:** R01–R06, E07–E10 and application/runtime integration cases. **Commit boundary:** coordinator and recovery
as separately testable changes; do not hide outcomes behind a new single `success` boolean.

## M7: pairing runtime and one carrier

**Files:** proposed `crates/marmot-app/src/pairing/{mod,codec,session,carrier}.rs`, runtime hooks and external signer;
carrier-specific module/crate chosen after D4; dial-safety docs and local artifact handling.
**Consumes:** P2/P5 codec rules, M4/M6 operations. **Produces:** platform-neutral authenticated pairing sessions.

- [ ] Reuse only the reviewed lifecycle ideas from closed #1282: bounded sessions, monotonic expiration, supersession,
  terminal key erasure and fail-closed restart. Port to the current runtime boundary and #417's sponsor-displayed QR;
  do not import its old DH payload or engine ownership assumptions wholesale.
- [ ] Implement canonical descriptor/Hello/proof/record/object codecs against fixtures. Verify exact kind-453 signer
  templates, account identity, event ID/fields/signature and refusal/late-response behavior through existing signer
  abstractions. Key schedule/AAD must match vectors byte-for-byte.
- [ ] Support one active channel per account-device runtime; a new local session supersedes the previous local channel.
  Do not claim globally one session per account without a shared directory. Destroy secret/sequence state on restart;
  fresh pairing reconciles durable enrollment without resuming old encryption nonces.
- [ ] Implement approval ordering and intent receipts from P2, including authenticated records arriving before approval.
  Maintain distinct replay-admission and deferred-application states so buffered valid records are eventually processed.
- [ ] Bound plaintext to 65,536 bytes/record and each complete object to 16 MiB. Proposed initial local budgets: at most
  four live reassembly objects, 32 MiB aggregate buffered object bytes, 64 KiB pre-approval buffer, and 65,536 accepted
  record sequence entries per session. Overflow closes/restarts the channel with a typed resource result and preserves
  enrollment obligations. Record these as local pilot limits; negotiate or standardize any interoperability minimum
  explicitly instead of silently changing wire validity. Load-test maximum objects sequentially and concurrently.
- [ ] Reject impossible lengths before allocation; validate offset arithmetic, identical/conflicting overlaps, completion
  hash, object type, batch membership, flags and direction. Retain enough replay evidence to detect conflicting ciphertext;
  if the replay budget is exhausted, terminate rather than evicting evidence and accepting sequence reuse.
- [ ] Use signed absolute expiry plus monotonic elapsed timers: two-minute default QR, five-minute maximum descriptor
  lifetime, five-minute channel idle timeout, thirty-minute channel absolute lifetime, and the approved intent bound.
  Supersession, cancellation, sleep/wake, failed signing and backward wall-clock changes cannot extend a session.
- [ ] Implement D4 carrier framing, connection lifecycle and scoped destination permissions. Keep carrier operations out
  of the generic engine/storage layers. Validate/pin allowed resolved addresses and tear down listeners/dial permission
  when pairing ends. Refused OS local-network permission is a typed recoverable outcome, not a silent fallback to an
  insecure or unversioned relay channel.
- [ ] Demonstrate codecs from two implementations and either interoperable network exchange or the declared single-family
  pilot. Test Android/Apple/desktop roles selected by D4, background interruption, split/coalesced reads and reconnect.

**Verification:** W/C cases, platform carrier tests and resource-budget tests, plus signer callback tests. **Commit
boundary:** codec, session lifecycle, coordinator integration, then carrier/platform integration with independent gates.

## M8: bindings, opt-in and client flows

**Files:** runtime commands/outcomes, `marmot-uniffi`, `marmot-c`, binding READMEs/method references, onboarding and native
client feature code in their own repositories. **Consumes:** M1–M7. **Produces:** a limited, accurately represented pilot.

- [ ] Expose start/scan/approve/select/status/cancel/reconcile operations with stable attempt references, bounded progress,
  allowed actions, and privacy-safe errors. Expose exact leaf targets separately from account removal. Document event
  ordering, replay after subscription/restart, and when cancellation leaves remote work pending.
- [ ] Provide group opt-in and capability status. An upgraded binary may support the feature while a group remains
  disabled; an unsupported member blocks enablement without being evicted. Disabling rejects multi-leaf resulting state.
- [ ] Replace the unconditional `MultiDeviceUnsupported` message only for supported workflows. Preserve notices that
  existing ineligible groups do not join, history is absent, future initial invitations still select one device, and a
  signer/account must already be available. Pairing is not full account migration or recovery from losing all devices.
- [ ] Render partial enrollment per group and distinguish transport delivery from authenticated join and selected
  membership. Display deadline/signer/network/resource limitations and a safe retry/reconcile action. Do not label a
  merely published Add as a completed device link.
- [ ] Render removal as “this device in this group” or “this account in this group.” Bulk per-group progress may be a
  later orchestration UI, but v1 never promises removed everywhere or rotated account-wide authority.
- [ ] Explain shared-leaf compromise accurately: deleting the shared leaf removes both physical copies on the selected
  branch; fresh enrollment is required for clean replacements. Never promise selective clone removal or universal clone
  detection, and distinguish compromised account-signing authority from compromised group/device state.
- [ ] Keep ack/control payloads out of chat, unread/notification counts and private-history sync. Reuse existing sibling
  push-token tests to prove one leaf's removal does not delete another's registration. Count witnesses by account.
- [ ] Exercise generated Swift/Kotlin/C interfaces, external-signer refusal/restart, cancel/back navigation, QR scanning,
  mixed-version peers and native background/suspend behavior. The runtime owns protocol and durable retry decisions;
  clients own rendering, permission prompts and explicit choices.
- [ ] Update binding documentation and release/integration guidance. Do not bump workspace/conformance versions as a
  side effect; follow manual release policy when an actual release is authorized.

**Verification:** F01–F06, `just binding-docs-gate`, generated binding smokes and native journeys. **Commit boundary:** MDK
API/docs and each consuming client change with explicit compatible version pins; then R1 pilot.

## Per-package implementation discipline

For each package, add the named behavioral test at the owning layer, run it to establish the unsupported/failing case,
implement the minimum behavior, and run it again. Test the policy at the boundary where it matters: real MLS sender
authentication, persistent storage reopen, exposed-byte retry or a runtime/client outcome. Do not substitute tests that
merely mirror internal functions. Run the applicable commands in [validation](validation-and-rollout.md#verification-commands),
review the diff, and make a signed commit before pushing an implementation branch. Never merge automatically.

[i1697]: https://github.com/marmot-protocol/mdk/issues/1697
[i1570]: https://github.com/marmot-protocol/mdk/issues/1570
[i1696]: https://github.com/marmot-protocol/mdk/issues/1696
[i1641]: https://github.com/marmot-protocol/mdk/issues/1641
[i1706]: https://github.com/marmot-protocol/mdk/issues/1706
