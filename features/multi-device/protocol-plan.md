# Protocol amendments and decision gates

Status: non-normative action plan, 2026-09-25. Read the [overview](README.md) first. Requirements below describe changes
to propose and verify, not newly adopted Marmot rules. Proposed type/record names remain provisional until P5.

## Source and ownership map

Use [#417 at the reviewed revision][feature] for the starting proposal. Some named files exist only on that branch;
do not mistake their absence on master for endorsement of its older draft. P1/P5 explicitly withdraw that draft on adoption.

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

### D1: authentication, forward secrecy and the channel payload boundary

Owner: protocol/security with Android, Apple, and desktop maintainers. Evidence: the
[Aug. 25 #417 review][review-objections] and [subsequent review][review-followup].

- [ ] Compare PSK-only and authenticated ephemeral DH using the actual supported platform stacks. Record available
  primitives/libraries, added dependencies/binary footprint, mobile lifecycle constraints, implementation/review effort,
  and license/support considerations. A generic claim that Noise is large is insufficient evidence.
- [ ] Describe QR capture during pairing, later QR-secret recovery, active carrier alteration, account-key compromise,
  external-signer authorization, and memory capture separately. State what each permits and what remains MLS-protected.
- [ ] Treat active QR capture as an enrollment-integrity attack, not only later metadata disclosure. A QR-secret holder
  on the carrier path can derive PSK-only record keys, alter approvals/intents/receipts and substitute a valid same-account
  KeyPackage whose private material it holds. Joiner-local package purpose cannot let the sponsor detect substitution.
  Recommend authenticated ephemeral DH on this evidence. Unmodified PSK-only cannot pass D1/G1; any retained PSK option
  needs independently authenticated package-hash commitments and receipts bound to the complete session transcript under
  a key the QR-only attacker lacks. A signature by the substituted package's own key alone is insufficient: an attacker
  holding an old device key could make it. Specify how that key is authorized by the authenticated joining proof, and
  protect approval/intent integrity as well. Record remaining QR-compromise exposure explicitly.
- [ ] Require exact proof-domain, role, account, session and transcript validation for every channel option. #417 already
  separates sponsor and joining-device kind-453 templates; preserve those checks and test reflection in both directions.
  The descriptor proof binds the descriptor; the Hello proof binds it and the joiner's fresh contribution. Authenticate
  the complete negotiated transcript in the sponsor's post-Hello approval before accepting catalog data; an initial QR
  signature cannot cover a later joiner nonce. Define exact signed bytes and transcript stages without circular hashes.
- [ ] For the recommended DH construction, authenticate both ephemeral keys, roles, session/account binding, nonces,
  descriptor and negotiated version through these proofs and final approval; validate peer keys/shared secret and erase
  ephemeral private material. The previous one-sided DH mix-in does not bind the joiner key sufficiently. A QR holder
  without account-signing or endpoint secrets must not forge records or replace either key in a relayed honest handshake.
- [ ] Restrict plaintext types to catalog IDs, selections, public KeyPackages, intents, approvals/receipts, and bounded
  reconciliation controls. No messages, drafts, attachments, archive sections, account secrets, group-event keys or MLS
  state. PSK-only also exposes recorded enrollment metadata on later QR-secret recovery; DH does not repair compromised
  account authority or endpoints, and neither construction prevents carrier denial of service.
- [ ] Record the decision and reviewer rationale. Any later incompatible channel construction uses a new version;
  future history transfer needs a separately reviewed forward-secret channel and content policy of its own.

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
  through that attempt. Reserve refusal proofs for attempts that never joined. After joining, use the existing
  [SelfRemove flow](../../protocol-core/member-departure.md) when authorized, retaining successful-join history. Active
  admins must follow that flow's admin-policy prerequisites or request an independently authorized exact-leaf removal;
  do not add a reconciliation message or bypass the admin/last-admin constraints to leave.
- [ ] Separate terminal *admission* from group *cleanup*: a refusal does not undo an exposed Add. Cleanup is a new
  authorized Remove with its own publication and branch-relative outcome.
- [ ] Leave unreachable peers in a truthful uncertain state. Stopping retries is finite; determining remote failure is
  not guaranteed. No destructive timeout is introduced to make the UI appear settled.

**Exit:** the P4 transition table covers positive, negative, unknown, crash, expiry, and branch-revival traces.

### D6: admin precedence

Owner: protocol/MLS. Adopt the P3 matrix after checking every owning document, particularly the ordinary admin-policy
rules. Evaluate every applicable authority against the candidate parent and use the highest priority that valid authority
grants. A narrow shape does not suppress independently valid admin membership authority, and padding/reference form must
not be needed to obtain it. Preserve dedicated self-update/SelfRemove priority; admin status alone does not reclassify
those operations or bypass resulting-state invariants. Admin authority is account-scoped: a compromised sibling of an
admin account has it too. This rule does not promise the healthy sibling wins that race.

## P1: reconcile the design record

**Files:** owning documents in the map above; MDK `docs/explorations-of-multi-device.md`; linked trackers as separately
authorized maintainer work. **Consumes:** [discussion register](discussion-register.md). **Produces:** D1–D6 decisions,
an amended proposal revision, and a migration/terminology table.

- [ ] Pin current upstream heads, #417 head, supported MDK/client versions and all referenced issue states. Preserve
  the reviewed baselines so changed conclusions can be explained without silently replacing evidence.
- [ ] Record the [compatibility boundaries and version gates](README.md#compatibility-and-coordinated-upgrades), starting
  with released MDK `v0.10.4` and named client builds as controls. Keep new authorization/cap rules behind negotiated
  enablement; any change to valid baseline acceptance or branch ordering needs separate compatibility/version review.
- [ ] Inventory the older live draft on pinned master: `0x800a`, kind `452`, `0xf2f0`/`0xf2ef`, External Commit admission,
  join PSK and group-event-key transfer. Record them as superseded by this plan, not already withdrawn on master. The
  adoption change must mark them withdrawn in `foundation/registries.md`, `features/multi-device.md` and
  `app-components/multi-device-join-authorization-v1.md`, with matching indexes/references. Preserve ID tombstones and
  historical meanings; do not withdraw shared MLS External Commit/PSK primitives or adopted kind `450`/`451` proofs.
- [ ] Reconcile MDK #1518's one-to-four Adds, separate capabilities, joiner-displayed QR, DH handshake, admin fallback,
  and incorrect losing-Welcome leaf-count explanation with the amended proposal. Clearly label abandoned alternatives.
- [ ] Record #1275/#1276 as historical design inputs where they require External Commits or old payloads. Record #1282
  and #251 as closed, unmerged sources of ideas, not ready-to-merge implementation dependencies.
- [ ] Resolve `account-sync.md`'s conflicting treatment of drafts in its own future content policy. Do not silently add
  drafts/history to enrollment to resolve the documentation contradiction.
- [ ] Record the accepted branch-relative revocation limitation and new reciprocal sibling-removal authority. The core
  rollback exposure is inherited; the ability of a compromised non-admin sibling to remove healthy siblings is still a
  feature-specific threat that the pilot must explain and test.

**Validation:** each disagreement maps to a decision/task, and P5 tracks every still-live legacy reference for withdrawal
at adoption. **Commit boundary:** non-normative reconciliation/decision record before protocol adoption.

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
- [ ] Bind offered package hashes and receipt/intent correlation to the D1-authenticated transcript. Define the DH
  approval/key-confirmation bytes, or the independently authenticated commitments required for a PSK alternative, in
  the encoding owner and fixtures. Valid AEAD under a QR-derived key alone does not meet this requirement.
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

**Validation:** W01–W08 and C01–C06, including 32/33-entry boundaries, empty non-final catalog rejection, approval reorder,
lost receipt and restart. **Commit boundary:** byte rules and fixtures together; no allocated identifiers with incomplete
state semantics.

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
| Enabled parent; exact valid same-account one-Add shape; no independent admin authority | Current sponsor leaf | Ordinary |
| Enabled parent; exact valid same-account sibling-Remove shape; no independent admin authority | Current same-account leaf | Ordinary |
| Either exact narrow membership shape also independently authorized by admin policy | Evaluate both authorities; retain the higher valid priority | Privileged |
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
acknowledgement obligation/evidence and recovery budget, added leaf lineage, local key availability, selected membership
and branch eligibility. The intent deadline bounds the sponsor's first Add exposure, not first Welcome admission.
Keep cleanup publication separate from terminal admission. A received ack remains historical evidence if membership later
loses selection; do not overwrite it with a misleading never-joined state.

| Observer / actor | Trigger | Durable action and resulting status | Permitted next operation |
| --- | --- | --- | --- |
| Joiner | Package created | Persist private bundle plus enrollment-only purpose before exposure | Send its public bytes for the named group/session |
| Joiner; sponsor on receipt | Valid intent approved | Joiner persists exact approval/deadline; sponsor persists authenticated receipt | Emit receipt; sponsor records it before Add |
| Sponsor | Deadline/cancellation before any Commit exposure, including staged Add | Record definitely-not-exposed outcome and apply safe staged rollback | Fresh work after rollback; reconcile retirement with joiner; no claim ambiguous exposure was cancelled |
| Joiner | Deadline passes without a known terminal outcome | Retain approval and package secret within explicit retention bounds; no inference about sponsor exposure | Accept a later matching Welcome while attempt is live; fresh pairing can reconcile |
| Joiner | Explicit cancellation/refusal before successful join | Atomically retain terminal rejection metadata; retire secrets when safe | Prove refusal in fresh reconciliation; warn that an exposed Add may need cleanup |
| Sponsor | Add may have reached any peer | Preserve exact bytes and durable correlation; publication pending/uncertain | Reconcile/retry exact operation; do not create another Add |
| Joiner | First matching Welcome, including after exposure deadline, with live approval and package secret | Atomically validate/join, retain digests, record ack obligation | Ack before other outbound application traffic or own update |
| Joiner | First Welcome after durable refusal/cancellation or secret retirement | Reject under enrollment-purpose rules | No admin fallback; reconcile without asserting sponsor already knows refusal |
| Joiner | Successful join, byte-identical Welcome replay | Recognize durable success before generic dedup; coalesce pending ack work | At most one recovery ack per attempt/branch/epoch from valid lineage; never re-consume package |
| Joiner | Same package reference, changed Welcome bytes | Reject without mutation | Retain legitimate receipt/record |
| Sponsor | Ack absent after retry budget, or useful catch-up material unavailable | Published-but-unacknowledged with stop reason | Stop automatic Welcome retries; preserve bounded evidence and uncertainty |
| Joiner | User wants to leave after successful join | Retain join fact; start authorized existing departure flow | SelfRemove subject to admin prerequisites, or independently authorized exact-leaf removal; no refusal proof |
| Sponsor / cleanup actor | Fresh reconciliation with original-joiner proof establishes durable refusal, including lost init material | Record durable admission terminality, separate cleanup-needed fact | Authorized sponsor/admin cleanup of exact still-current leaf |
| Sponsor | Peer claims full state loss and cannot prove original-joiner identity | Retain uncertainty; account equality alone proves no terminal fact | Explicit separately authorized removal if user chooses; no automatic failure cleanup |
| Each observer locally | Cleanup is published/selected | Record branch-relative removal evidence | Re-evaluate eligibility and cap before another enrollment |
| Each observer locally | Enrollment loses current selection but remains eligible | Record not-selected/retry-blocked, retain recovery state | No replacement Add merely because this pass was lost |
| Each observer locally | Old branch becomes permanently ineligible under core rules | Record evidence and retire obsolete attempt safely | Fresh approved session/package; explicit discard/rejoin if locally retained |
| Each observer locally | Old branch revives | Recompute exact usable membership and obligations from that branch | Resume only if keys/lineage are usable; otherwise explicit stale-leaf cleanup/recovery |
| Joiner; sponsor on authenticated evidence | Group discarded/deleted, leaf removed, or record invalidated | Revoke pending ack authority for that lineage | No replay ack; fresh authorized recovery where available |

Rows describing both peers are separate local transitions linked by authenticated messages, not shared storage. A
sponsor's definitely-not-exposed fact is not observable from silence at the joiner; a joiner's refusal is not observable
from a missing ack at the sponsor.

- [ ] Select Welcome authorization by durable KeyPackage purpose, never just intent liveness. Enrollment-only packages
  route to the same-account checks or rejection in every state, including after exposure-deadline expiry, cancellation,
  supersession or invalidation. Ordinary admin packages retain their independent path.
- [ ] Validate exact group, package reference, sponsor account and leaf signature key, durable approval/admission state,
  proofs, required component and every resulting-state invariant before any visible group or package mutation. Invalid Welcomes leave
  both group and consumable KeyPackage state untouched.
- [ ] Make byte-identical duplicate-Welcome acknowledgement recovery mandatory for implementations advertising this
  feature. Preserve the joined-Welcome digest and exact correlation independently of the consumed init key.
- [ ] Coalesce duplicates while an initial or recovery ack is pending. After initial publication, allow at most one fresh
  recovery ack per `(attempt, branch identity, MLS epoch)` with currently authorized lineage. Persist the budget claim
  and exact ack bytes atomically before exposure, including across restart and branch reselection; identical relay
  redelivery cannot reset it or create fresh work. Use finite exact-byte publication retries within that obligation's
  original budget. A later epoch may allow one new recovery ack, not one per Welcome delivery. Test repeated multi-relay
  replays before/after initial publication, budget exhaustion, restart and branch return.
- [ ] Define acknowledgement verification at the actual MLS sender: exact added leaf or uninterrupted self-update
  descendant, correct branch and all enrollment fields. Another leaf with the same account or a later Remove/Add is
  insufficient. A bare reusable leaf index is not a lineage identifier. Existing Leaving/removed send prohibitions still
  apply; ack recovery never blocks an otherwise authorized departure or permits application traffic while Leaving.
- [ ] Define the ack-first gate as durable outbound scheduling/publication obligation, not receipt by the sponsor, which
  the joiner cannot observe. Choose and document the existing transport's required publication confirmation boundary;
  retain exact retry material after ambiguous publication. Continue receiving/catching up while blocked. Do not wait
  forever for an unspecified ack-of-ack or silently send chat first.
- [ ] Define the signed intent deadline as the latest sponsor first-exposure time. Proposed local pilot default: five
  minutes from intent creation, with at least 120 seconds remaining both when staging an unexposed Add and immediately
  before first exposure. Validate these defaults with platform/signer/delivery measurements and clock policy. Never
  extend a signed intent in place. A delayed first Welcome may join after that deadline if the approval is uncancelled,
  the package secret remains available and all other admission checks pass. The joiner cannot independently prove when
  the sponsor exposed the Add; this remains part of sponsor trust, not a wall-clock consensus rule.
- [ ] Keep the existing approximately 1/2/4/8-minute Welcome retries and 30-minute automatic-retry ceiling as local
  guidance from first exposure. Before each retry, check whether the Welcome's Add epoch still has a usable catch-up
  path to the selected branch within the core retained-history and transport availability rules. Stop and report
  `catch-up-unavailable` when required material is no longer available; do not extend cryptographic retention to keep
  retries alive. Crossing the local retained anchor is not alone proof that catch-up is impossible if a complete usable
  Commit chain is still retrievable. Unknown availability is reported as uncertain, not as guaranteed catch-up. Deadline
  expiry alone does not stop a useful retry or prove failure; missing ack never authorizes deletion or a replacement Add.
- [ ] Set bounded package-secret retention separately from the exposure deadline, covering the approved deadline plus
  the 30-minute retry ceiling and 120-second delivery margin in the pilot. Explicit cancellation or unrecoverable key
  loss can end admission earlier. Warn before cancelling after receipt: the sponsor may already have published, leaving
  a stranded leaf that consumes a cap slot until separately authorized cleanup. Retention expiry also preserves this
  uncertainty and rejection metadata; it is not remote removal evidence.
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

- [ ] Publish descriptor, Hello, kind-453 role proofs and full-transcript approval, key schedule, AEAD/AAD, control records,
  receipt, catalog/selection/package/intent batches, reconciliation, and newly allocated ack-kind fixtures. Preserve
  unsigned-inner-ack semantics; do not turn it into an account-signed proof that any sibling could generate.
- [ ] Provide canonical bytes, decoded interpretation, expected hash/signature/decryption result and rejection reason for
  each positive/negative fixture. Include complete transcript traces, not only an isolated HKDF vector.
- [ ] Check candidate identifiers for collisions at adoption. Update registry, owning documents, surface indexes, layout,
  maturity labels and withdrawn-ID handling together. Allocate a new version for incompatible deployed behavior.
- [ ] Allocate a fresh kind for the MLS-carried enrollment ack; never reuse `452`. Complete P1's withdrawal of the old
  multi-device draft in the same adoption change: retain `0x800a`, `452`, `0xf2f0` and `0xf2ef` as withdrawn/reserved with
  their historical meanings, and remove live External Commit/join-PSK/group-event-key-transfer instructions from the
  feature/component owners. Keep kind `452` classified as a non-published local signing template. Add a fixture/registry
  check that rejects interpreting its old signed proof as the new unsigned MLS ack, and resolves every old reference.
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
