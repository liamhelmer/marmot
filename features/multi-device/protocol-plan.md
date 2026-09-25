# Protocol amendments and decision gates

Status: non-normative action plan, 2026-09-25. Read the [overview](README.md) first. Requirements below describe changes
to propose and verify, not newly adopted Marmot rules. Proposed type/record names remain provisional until P5.

## Source and ownership map

Use [#417 at the reviewed revision][feature] for the starting proposal. Some named files exist only on that branch;
do not mistake their absence on master for permission to recreate the withdrawn draft.

| Surface | Existing/proposed owning files | Change responsibility |
| --- | --- | --- |
| User-visible enrollment | `features/multi-device.md` | Pairing flow, approval, failure outcomes, trust limitations; reference exact encoding owner |
| Optional account sync | `features/account-sync.md` on #417 | Resolve draft-message inclusion contradiction; prohibit use of enrollment channel for content |
| Same-account behavior | `app-components/same-account-membership-v1.md` on #417 | Data-less capability, exact shapes, enable/disable, resulting-state cap |
| Admin authority | `app-components/admin-policy-v1.md` | Preserve ordinary admin removal and state candidate-parent authorization precedence |
| Identity and packages | `foundation/identity.md`, `foundation/key-packages.md` | Capability support, purpose-bound admission, single-use package lifecycle references |
| Canonical bytes/proofs | `foundation/canonical-encoding.md`, `foundation/authorization-proofs.md` | Reuse existing rules; no second vector/proof encoding dialect |
| Registry/app payloads | `foundation/registries.md`, `foundation/application-messages.md` | Consistent component/kind reservations and acknowledgement processing references |
| Core lifecycle | `protocol-core/joining.md`, `group-setup.md`, `group-messaging.md`, `member-departure.md` | Welcome routing, duplicate recovery, authorization, retained-group rejoin, leaf departure |
| Convergence/durability | `protocol-core/convergence.md`, `publish-lifecycle.md`, `retained-history.md`, `durability.md` | Reference existing authority; specify feature facts without changing selection/finality |
| Carrier | Proposed `transports/pairing-local-network-v1.md` if D4 selects LAN | Exact rendezvous/framing and delivery behavior; no group-state authority |
| Local guidance | `implementation-model.md` | Scheduling, margins, retry budgets, carrier platform choices and implementation mapping |
| Discovery | `layout.md`, each affected surface `README.md`, `mip-coverage.md` | Current indexes, maturity labels, and withdrawn-draft mapping |

For pairing-specific exact bytes, P2/P5 should nominate one owning feature-specific protocol document consistent with
the repository's surface rules, rather than duplicating schemas across the flow, carrier, and component documents.
Do not move network endpoints into protocol-core or invent component-data for a data-less behavior marker.

## Decision gates

### D1: forward secrecy and the channel payload boundary

Owner: protocol/security with Android, Apple, and desktop maintainers. Evidence: the
[Aug. 25 #417 review][review-objections] and [subsequent review][review-followup].

- [ ] Compare PSK-only and authenticated ephemeral DH using the actual supported platform stacks. Record available
  primitives/libraries, added dependencies/binary footprint, mobile lifecycle constraints, implementation/review effort,
  and license/support considerations. A generic claim that Noise is large is insufficient evidence.
- [ ] Describe QR capture during pairing, later QR-secret recovery, active carrier alteration, account-key compromise,
  external-signer authorization, and memory capture separately. State what each permits and what remains MLS-protected.
- [ ] Recommend PSK-only v1 only with explicit acceptance that recovering the QR secret exposes recorded enrollment
  metadata. Restrict plaintext types to the defined enrollment controls: catalog IDs, selections, public KeyPackages,
  intents, approvals/receipts, and bounded reconciliation controls. No messages, drafts, attachments, archive sections,
  account secrets, group-event keys, or MLS state.
- [ ] If DH is selected before initial adoption, revise the entire transcript and fixtures: both ephemeral keys, roles,
  session/account binding, nonces, descriptor and negotiated version are authenticated; validate peer keys/shared secret
  and erase ephemeral private material. The previous one-sided DH mix-in does not bind the joiner key sufficiently.
- [ ] Record the decision and reviewer rationale. Any later incompatible channel construction uses a new version;
  future history transfer needs a separately reviewed forward-secret channel, not this v1 PSK payload extension.

**Exit:** explicit security acceptance and platform comparison linked from the owning proposal. A brief approval marker
does not substitute for a recorded response to the earlier concerns.

### D2: bound the fleet with evidence

Owner: protocol/conformance/performance. Start with five leaves per account, but evaluate the request for 16 or a
negotiated policy before freezing behavior.

- [ ] Measure total MLS state, retained-candidate storage, Add/Remove/Welcome bytes, validation time, convergence replay,
  and application fanout for 1, 2, 5, and 16 leaves of one account, including mixed-account groups. Record hardware,
  build profile, group sizes, sample counts, median/tail timings, and memory/storage peaks.
- [ ] Exercise sequential legitimate enrollment and a compromised leaf repeatedly adding/removing siblings. Separate
  per-Commit work, resulting-state limits, retained-branch amplification, and long-term churn; a leaf cap alone does not
  bound all resource use.
- [ ] Record whether five is an accepted product restriction, an evidence-based resource requirement, or should change.
  A negotiated cap would require its own authenticated policy design; do not add a local preference that changes validity.
- [ ] Correct the existing rationale: #417 allows one Add per Commit, not four. Keep remove-batch and account-leaf bounds
  distinct. If changing the cap, review both and all fixtures together.

**Exit:** one interoperable rule, measurements, explicit overflow behavior, and compatible-client consequences.

### D3: initial invitation is still a reachability limitation

Owner: protocol/product/directory. [MDK #1696][i1696] Phase 2 and the #417 review request a broader outcome than this v1.

- [ ] Keep one valid compatible selected KeyPackage for an account with no group leaf. Do not fan out across opaque
  public `d` slots or infer which slots are abandoned by recency.
- [ ] Specify UI and recovery when the chosen installation is unavailable: report risk, retry the exact exposed Welcome,
  and require explicit admin removal/reinvite when replacement is needed. Do not automatically admit another device.
- [ ] Record that #1696 Phase 1 improves evidence, not availability on every device. Preserve contemporary compatible
  package ranking, including the temporary policy from [MDK #1910][pr1910], rather than restoring an obsolete global
  newest-only resolver.
- [ ] Separate authenticated device-directory work from #417. Track the alternative account-wide invitation notice
  and deterministic fallback as a proposal with authentication/privacy/replay semantics, not an unversioned pilot hack.
- [ ] Correct #1696/#1697 dependency descriptions when their owners adopt this scope: #417 supplies group enrollment,
  not a public device registry or account-wide revocation record.

**Exit:** a documented limitation accepted for the pilot and an explicit future owner for reachability improvement.

### D4: carrier choice and destination authority

Owner: transport/platform/security.

- [ ] Choose one deployable profile, provisionally a local-network socket, and enumerate the exact supported platforms
  and both QR roles. Compare practical discovery, firewall, local-network permission, suspend/resume, and fallback behavior.
- [ ] Specify a stable carrier name/version, hint byte encoding, host/address/port rules, framing, length limits,
  connection/idle deadlines, duplicate/reconnect semantics, and terminal closure. A transport label with opaque data alone
  is not an interoperable carrier profile.
- [ ] For LAN, define a pairing-scoped destination permission: explicit user-initiated pairing, bounded session lifetime,
  allowed address/interface classes, DNS resolution validation and pinning, maximum dials, and teardown. Reject arbitrary
  redirects/proxies and metadata/service probing. Do not relax the shared relay/media host-safety policy or reuse an
  unrestricted development flag. Amend MDK's dial contract with the exact exception and tests before implementation.
- [ ] Produce either two independent carrier implementations with a demonstrated exchange, or label the first pilot
  single-implementation-family. Two codecs alone do not prove network interoperability.

**Exit:** carrier profile plus platform matrix and policy tests, or a bounded single-family pilot declaration.

### D5: terminality, cancellation and re-enrollment

Owner: protocol/recovery/security. Use the conservative v1 policy in P4: do not infer failure from time or local
retirement; fresh authenticated reconciliation establishes what a joiner retained. Do not invent a standalone negative
ack wire message without defining its authorization, persistence and race behavior.

- [ ] Decide and specify how fresh pairing correlates prior attempts without keeping old channel keys alive. Bind
  reconciliation to account, group, old attempt/intent, exact package reference and recorded Welcome/branch identity.
- [ ] Authenticate terminal refusal as the original joiner, not merely another client of the same account. Require a
  proof by the original joining leaf signing key bound in its KeyPackage, covering the fresh session/transcript challenge,
  exact old attempt and refusal. Define canonical signed bytes and replay rules in the reconciliation encoding owner.
  Loss of the init key while retaining this signer can be reported; loss of the original signer cannot prove refusal.
  In that case retain uncertainty. A separate explicit sibling/admin removal may still be authorized, but is not
  automatic cleanup based on a proven failed enrollment. A clone of the original signer remains indistinguishable.
  The proof is an authenticated assertion under the joiner's honest durable-state behavior, not cryptographic proof
  that a malicious original client erased every copy or cannot later equivocate. Explain that limit with the clone case.
- [ ] Record a terminal refusal durably on the joiner before declaring it to the sponsor; it prevents subsequent joining
  through that attempt. Refusal after successful join becomes a removal request, not a retroactive claim that no join occurred.
- [ ] Separate terminal *admission* from group *cleanup*: a refusal does not undo an exposed Add. Cleanup is a new
  authorized Remove with its own publication and branch-relative outcome.
- [ ] Leave unreachable peers in a truthful uncertain state. Stopping retries is finite; determining remote failure is
  not guaranteed. No destructive timeout is introduced to make the UI appear settled.

**Exit:** the P4 transition table covers positive, negative, unknown, crash, expiry, and branch-revival traces.

### D6: admin precedence

Owner: protocol/MLS. Adopt the P3 matrix after checking every owning document, particularly the ordinary admin-policy
rules. A narrow component denial is not automatically denial of an independently authorized admin operation; conversely
admin status does not elevate a valid narrow same-account operation's priority or bypass resulting-state invariants.

## P1: reconcile the design record

**Files:** owning documents in the map above; MDK `docs/explorations-of-multi-device.md`; linked trackers as separately
authorized maintainer work. **Consumes:** [discussion register](discussion-register.md). **Produces:** D1–D6 decisions,
an amended proposal revision, and a migration/terminology table.

- [ ] Pin current upstream heads, #417 head, supported MDK/client versions and all referenced issue states. Preserve
  the reviewed baselines so changed conclusions can be explained without silently replacing evidence.
- [ ] Reconcile MDK #1518's one-to-four Adds, separate capabilities, joiner-displayed QR, DH handshake, admin fallback,
  and incorrect losing-Welcome leaf-count explanation with the amended proposal. Clearly label abandoned alternatives.
- [ ] Record #1275/#1276 as historical design inputs where they require External Commits or old payloads. Record #1282
  and #251 as closed, unmerged sources of ideas, not ready-to-merge implementation dependencies.
- [ ] Resolve `account-sync.md`'s conflicting treatment of drafts in its own future content policy. Do not silently add
  drafts/history to enrollment to resolve the documentation contradiction.
- [ ] Record the accepted branch-relative revocation limitation and new reciprocal sibling-removal authority. The core
  rollback exposure is inherited; the ability of a compromised non-admin sibling to remove healthy siblings is still a
  feature-specific threat that the pilot must explain and test.

**Validation:** each disagreement in the discussion register maps to a decision/task; no stale document claims the
withdrawn design is current. **Commit boundary:** non-normative reconciliation/decision record before protocol adoption.

## P2: encoding and pairing control ordering

**Files:** pairing encoding/flow owner, `foundation/registries.md`, `implementation-model.md`, indexes.
**Consumes:** D1/D4. **Produces:** canonical control-object/record contract and byte fixtures consumed by M7.

- [ ] Replace the four batch vectors' mistaken byte bounds with variable byte-length vectors and explicit element
  bounds. Catalog and selection decode to 0–32 entries; KeyPackage and intent batches to 1–32. Keep the 16 MiB object
  maximum as a byte cap. An empty catalog is exactly the final batch with zero groups. More than 32 groups use batches.
- [ ] Specify batch numbering/completion, duplicate batch/object behavior, catalog group uniqueness, selection membership,
  group/package association, integer overflow, trailing bytes and shortest-length-prefix rejection. No duplicate group
  is silently deduplicated into a different approved object.
- [ ] Add `enrollment_intent_receipt` as candidate record type 4: joiner-to-sponsor only, canonical list of 1–32 distinct
  approved intent hashes bound to the current authenticated session. Define ordering and duplicate rejection in the
  encoding owner; use lexicographic hash order for this proposed set representation. Emit only after durable approval.
  The sponsor records the receipt idempotently and publishes no Add lacking its matching durable receipt.
- [ ] Resolve the earlier approval ordering separately. Preferred minimal change: the sponsor may send catalog objects
  after sending approval; the joiner does not accept/act on any object until approval authenticates and verifies. Define
  bounded buffering of early records within M7's budget, or an explicit retransmission behavior if buffering is refused.
  Remove the impossible requirement that the sender already know verification occurred. An intent receipt cannot serve
  as this earlier signal. If reviewers require explicit key confirmation instead, define a separate joiner-ready record
  before catalog exchange, update the registry/fixtures, and use one approach across implementations.
- [ ] Specify deterministic handling of duplicates versus deferred processing: an authenticated record buffered pending
  approval is processed once after approval; the replay filter must not discard its only usable copy. Conflicting same
  direction/sequence ciphertext terminates the session. Identical duplicate receipt never stages a second Add.
- [ ] Define retransmission of control objects over lossy carriers without re-encrypting different plaintext under a used
  nonce. Specify what may be resent byte-identically and how completion/missing chunks are recognized; no indefinite
  unbounded retry or invisible reliance on TCP ordering in the carrier-independent rules.
- [ ] Keep runtime elapsed deadlines monotonic. Validate signed Unix expiry separately with explicit clock-skew policy;
  clock correction cannot extend in-process TTL. Restart destroys channel secrets and requires fresh authentication.

**Validation:** W01–W07 and C01–C06, including 32/33-entry boundaries, zero catalog, approval reorder, lost receipt and
restart. **Commit boundary:** byte rules and fixtures together; no allocated identifiers with incomplete state semantics.

## P3: component lifecycle and complete authorization

**Files:** component, admin policy, group setup/messaging/departure, identity/capability references.
**Consumes:** D2/D6. **Produces:** one authority matrix plus resulting-state invariants, consumed by M2/M3.

- [ ] State all valid advertisement/requirement locations and reject component-data entries for `0x800d` everywhere,
  including GroupContext, LeafNode, KeyPackage, GroupInfo, AppEphemeral and SafeAAD.
- [ ] Enforce current-parent support on enablement, all founding leaves' support at creation, required support on later
  admission, and the resulting-state cap for every Commit while enabled. Disabling requires at most one leaf/account.
- [ ] Require UpdatePath for the narrow Add/Remove shapes. A narrow Add contains exactly one inline Add; a narrow Remove
  contains 1–4 distinct current same-account siblings, excluding the committer. No referenced/mixed proposals qualify.
- [ ] Validate identity proofs, exact account equality, package/signature-key uniqueness and current leaf targets against
  the candidate parent. Do not use whichever branch happens to be selected when validation runs.
- [ ] Replace both orphan-only admin-removal sentences. Keep ordinary admin removal of one leaf or an entire account,
  subject to existing admin/last-admin rules. Do not invent recovery when neither an authorized sibling nor admin survives.
- [ ] Align the matrix below across all owning documents. Scope negative fixtures to the authority being exercised.

| Candidate | Authority/result under proposed clarified rules | Priority |
| --- | --- | --- |
| Enabled parent; exact valid same-account one-Add shape | Current sponsor leaf; admin status immaterial | Ordinary |
| Enabled parent; exact valid same-account sibling-Remove shape | Current same-account leaf; admin status immaterial | Ordinary |
| Non-admin mixed, referenced, wrong-account or otherwise non-qualifying operation | No authority from this component; reject unless a separate existing narrow rule applies | None |
| Admin removes sibling plus another account's leaf, using ordinary valid admin operation | Evaluate ordinary admin authority independently | Privileged |
| Admin mixed/referenced operation otherwise permitted by baseline admin rules | Not authorized by the same-account exception; may pass ordinary admin validation | Privileged |
| Feature absent; non-admin attempts same-account Add/Remove | Reject this authorization path | None |
| Feature absent; independently valid admin operation | Preserve baseline admin behavior; no invented global multi-device ban | Baseline admin priority |
| Any operation violates MLS, identity proof, enabled-group cap or required capability invariants | Reject regardless of actor | None |
| Existing self-update / SelfRemove-only operation | Preserve its existing dedicated rules and priority | Baseline ordinary priority |

**Validation:** U01–U10 and A01–A03, including admin mixed removals, missing path, all data locations and capability
transitions. **Commit boundary:** matrix, component/core/admin text and matching vectors.

## P4: enrollment, Welcome and recovery contract

**Files:** `features/multi-device.md`, `protocol-core/joining.md`, durability/publish references, `implementation-model.md`.
**Consumes:** P2/P3 and D5. **Produces:** purpose routing, durable facts, obligations and recovery rules consumed by M4–M6.

### Independent facts, not a single success enum

Retain or reconstruct: account/group and session/attempt identity, approved intent/hash/deadline, exact package reference,
sponsor leaf binding, receipt, exact staged Commit and publication exposure, exact Welcome/GroupInfo digests, join fact,
acknowledgement obligation/evidence, added leaf lineage, local key availability, selected membership and branch eligibility.
Keep cleanup publication separate from terminal admission. A received ack remains historical evidence if membership later
loses selection; do not overwrite it with a misleading never-joined state.

| Trigger | Durable action and resulting status | Permitted next operation |
| --- | --- | --- |
| Package created | Persist private bundle plus enrollment-only purpose before exposure | Send its public bytes for the named group/session |
| Valid intent approved | Persist exact approval/deadline | Emit receipt; sponsor records it before Add |
| Intent expires/cancels before any Commit exposure | Retire admission, delete disposable secrets when safe, retain rejection metadata | Fresh session/package; no old Welcome admin fallback |
| Add staged but definitely not exposed | Apply existing safe staged-operation cancellation/rollback | Fresh work after rollback; no claim that ambiguous publication was cancelled |
| Add may have reached any peer | Preserve exact bytes and durable correlation; publication pending/uncertain | Reconcile/retry exact operation; do not create another Add |
| Published Add, first matching Welcome before deadline | Atomically validate/join, retain digests, record ack obligation | Ack before other outbound application traffic or own update |
| Matching Welcome after deadline with no previous successful join | Reject; persist refusal for later authenticated reconciliation | No admin fallback, no implicit cleanup inferred at sponsor |
| Successful join, byte-identical Welcome replay after deadline | Recognize durable success before expiry/generic dedup exits | Fresh ack from exact authorized lineage; do not consume package again |
| Same package reference, changed Welcome bytes | Reject without mutation | Retain legitimate receipt/record |
| Ack absent after retry budget | Published-but-unacknowledged | Stop automatic retries; preserve bounded recoverable evidence |
| Fresh reconciliation with original-joiner proof establishes durable refusal, including lost init material | Record durable admission terminality, separate cleanup-needed fact | Authorized sponsor/admin cleanup of exact still-current leaf |
| Peer claims full state loss and cannot prove original-joiner identity | Retain uncertainty; account equality alone proves no terminal fact | Explicit separately authorized removal if user chooses; no automatic failure cleanup |
| Cleanup is published/selected | Record branch-relative removal evidence | Re-evaluate eligibility and cap before another enrollment |
| Enrollment loses current selection but remains eligible | Record not-selected/retry-blocked, retain recovery state | No replacement Add merely because this pass was lost |
| Old branch becomes permanently ineligible under core rules | Record evidence and retire obsolete attempt safely | Fresh approved session/package; explicit discard/rejoin if locally retained |
| Old branch revives | Recompute exact usable membership and obligations from that branch | Resume only if keys/lineage are usable; otherwise explicit stale-leaf cleanup/recovery |
| Group discarded/deleted, leaf removed, or record invalidated | Revoke pending ack authority for that lineage | No replay ack; fresh authorized recovery where available |

- [ ] Select Welcome authorization by durable KeyPackage purpose, never just intent liveness. Enrollment-only packages
  route to the same-account checks or rejection in approved/expired/cancelled/superseded/invalidated states. Ordinary
  admin packages retain their independent path.
- [ ] Validate exact group, package reference, sponsor account and leaf signature key, intent deadline, proofs, required
  component and every resulting-state invariant before any visible group or package mutation. Invalid Welcomes leave
  both group and consumable KeyPackage state untouched.
- [ ] Make byte-identical duplicate-Welcome acknowledgement recovery mandatory for implementations advertising this
  feature. Preserve the joined-Welcome digest and exact correlation independently of the consumed init key.
- [ ] Define acknowledgement verification at the actual MLS sender: exact added leaf or uninterrupted self-update
  descendant, correct branch and all enrollment fields. Another leaf with the same account or a later Remove/Add is
  insufficient. A bare reusable leaf index is not a lineage identifier.
- [ ] Define the ack-first gate as durable outbound scheduling/publication obligation, not receipt by the sponsor, which
  the joiner cannot observe. Choose and document the existing transport's required publication confirmation boundary;
  retain exact retry material after ambiguous publication. Continue receiving/catching up while blocked. Do not wait
  forever for an unspecified ack-of-ack or silently send chat first.
- [ ] Add a sponsor scheduling margin without claiming guaranteed delivery. Proposed local pilot default: do not begin
  an unexposed Add with less than 120 seconds remaining on the approved deadline; recheck before first exposure. Validate
  that default with platform/signer/delivery measurements and clock policy. Never extend a signed intent in place.
- [ ] Keep the existing approximately 1/2/4/8-minute Welcome retries and 30-minute automatic-retry ceiling as local
  guidance. Clamp attempts to useful lifecycle conditions; expiry alone cannot distinguish late first join from replay
  after success. Missing ack never authorizes deletion. Always report retained uncertainty after stopping automatic work.
- [ ] Specify fresh-pairing reconciliation from D5. A terminal refusal is irrevocable for that attempt, bound to the exact
  correlation and original joining signer, and retained before acknowledgement of refusal. First join and refusal share
  one atomic decision boundary. Missing original-signer proof cannot be repaired by an account-level signature alone.
  Account for sponsor loss/removal; absence of an authorized cleanup actor remains a reported limitation.
- [ ] Remove the local-retirement bypass from automatic replacement enrollment. Only core permanent-ineligibility
  evidence permits treating a losing attempt as unable to return. Explicit local discard/rejoin is necessary for some
  recovery, but never makes another branch ineligible. Define a recovery path for already-retired state that does revive:
  mark leaf unusable, refuse false completion/ack, and require authorized leaf cleanup/re-admission.
- [ ] Define retained-group rejoin with the existing explicit discard boundary, preserving accepted history/provenance
  according to existing policy while never importing the old leaf's cryptographic state into the new membership.
- [ ] Specify retirement bounds for private packages, channel keys, exact Welcome bytes and rejection metadata.
  Once secrets are gone, unknown package references fail closed; metadata compaction must not create an admin fallback,
  duplicate-Add permission, replay ack, or an unbounded retained-record denial of service.

**Validation:** all E/R/A cases and durable boundaries B1–B8. **Commit boundary:** amended lifecycle prose, exact control
encoding for reconciliation, state diagram/table, and vectors. An incomplete negative/reconciliation exchange blocks G1.

## P5: fixtures, registry and independent readiness review

**Files:** affected indexes/registry and conformance fixture location agreed with MDK; proposed LAN owner if selected.
**Consumes:** P1–P4. **Produces:** one pinned implementation baseline with machine-readable byte fixtures.

- [ ] Publish descriptor, Hello, both kind-453 proofs, key schedule, AEAD/AAD, control records, receipt, catalog/selection/
  package/intent batches, reconciliation, and kind-452 ack fixtures. Preserve unsigned-inner-ack semantics; do not turn it
  into an account-signed proof that any sibling could generate.
- [ ] Provide canonical bytes, decoded interpretation, expected hash/signature/decryption result and rejection reason for
  each positive/negative fixture. Include complete transcript traces, not only an isolated HKDF vector.
- [ ] Check candidate identifiers for collisions at adoption. Update registry, owning documents, surface indexes, layout,
  maturity labels and withdrawn-ID handling together. Allocate a new version for incompatible deployed behavior.
- [ ] Resolve carrier ownership/profile, even if the first pilot is single-family. Opaque hints have no inherent device,
  endpoint-safety, group-membership, or consensus authority.
- [ ] Obtain a second implementation's decoding and state-machine review. Review sponsor trust and lack of parent-Commit
  verification on Welcome explicitly; do not claim a joiner independently proves canonical membership.
- [ ] Amend #417, or a clearly identified successor, before adopting/fixing its wire identifiers. Record the exact accepted
  revision that MDK implements; do not merge old #251/#1282 paths as a shortcut.

**Exit:** G1 and a named fixture revision. Run the documentation checks in [validation](validation-and-rollout.md).

[feature]: https://github.com/marmot-protocol/marmot/blob/994ba06878d7093e7ba2cf7f6c34b2fe37bc043b/features/multi-device.md
[review-objections]: https://github.com/marmot-protocol/marmot/pull/417#pullrequestreview-5020898926
[review-followup]: https://github.com/marmot-protocol/marmot/pull/417#pullrequestreview-5025660654
[i1696]: https://github.com/marmot-protocol/mdk/issues/1696
[pr1910]: https://github.com/marmot-protocol/mdk/pull/1910
