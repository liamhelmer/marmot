# Full-scope security review of the multi-device idea

Status: non-normative review and proposed changes, 2026-09-30. This is an alternative response to the same sixteen
findings in the [earlier review][prior-review], rather than an amendment to the narrower implementation plan.

**Recommendation:** keep account-wide linking and finish its authorization, reconciliation, and recovery contracts.
Removing the device group, invitation fanout, gap filling, history, or account-wide removal from the deliverable would
avoid some exposures without fixing the proposed feature. The changes below retain those behaviors.

## Reviewed sources and scope

The reviewed [idea][idea] is pinned to `07da8ffbfa7aa6c1e73e8e326085d0c47dc298e7`; this was also upstream `master` when
checked on 2026-09-30. The earlier review is pinned to fork commit `cdfe3aeb249401a94ca00b64cd3c03f256f904fa`.
Its [alternate plan][narrow-plan] supplies useful safeguards, but explicitly excludes several account-wide behaviors.
This review proposes how to complete those behaviors, not how to remove them from a first version.

The retained scope includes:

- Independent MLS leaves and state for every installation, with no primary device.
- Remote discovery and human-confirmed linking, the private device group, and merging independently created groups.
- Account-wide chat enrollment, persistent opt-outs, automatic gap filling, all-device invitations and account decisions.
- Device management, rejection, removal across chats, inactivity policy, and complete local sign-out.
- Authorized history and small-state synchronization, media-key handling, and backups protected by a separate secret.
- The idea's ten-leaf-per-account limit; measurement can inform its implementation without silently reducing it to five.

Recovering when every device and recovery secret is lost remains outside the original scope. Defining backup protection
does not promise recovery in that case. Nothing here adopts wire values, final component encodings, or shipped support.
Each proposed remedy has acceptance evidence still to produce.

Account-key compromise remains the idea's stated boundary: someone holding the raw nsec can act as the account and
bootstrap another identity claim. A compromised approved sibling can also disclose content it already knows, and actual
copies of one leaf's private state cannot be distinguished cryptographically. The remedies protect honest clients from
substitution, stale evidence, unauthorized scope expansion, and unsafe recovery; they do not revoke information an
attacker already possesses. Availability and completion assume reachable healthy authorized actors and eventual delivery
of the relevant finite input, as in [Marmot convergence][convergence].

## Shared changes needed by the findings

These are design requirements for the owning surfaces, not new schemas in this review.

**Installation identity.** Give each installation a fresh device signing key independent of its account signer and of
its per-chat MLS keys. Pairing binds that key to the account, the random publication slot, and the exact approved
KeyPackage. Authenticate refreshes and key rotations through that binding. Keep labels, hardware details, and the
device-to-chat map private. A public slot or copied client tag is only discovery data.

**Account control records.** Carry explicit, bounded, authenticated operations in the device group: enrollment grants,
consent changes, invitation decisions, retirement, merge manifests, and acknowledgements. Bind each operation to its
issuer's verified device-group leaf, device-key generation, operation ID, authenticated parent/checkpoint, target, and
scope. Authenticate their issuer and authority in that source state before retaining a denial; an outsider's signed
claim or a newly invented device group is not a trusted revocation. Commit a small control checkpoint/root and the
ordered references to bounded operation batches in an owning GroupContext component. Fetching a referenced record does
not accept it until its bytes, authority and checkpoint validate. Retain denial/retirement evidence and already exposed
operations across branch changes. These records authorize honest-device orchestration; MLS Commits still determine
membership in each group. They do not create global finality or change chat branch scoring.

**Durable authorization.** An enrollment grant names the recipient device key, pairing transcript, current consent
revision, and permitted chats/history. A worker rechecks the grant and known denials before exposing an Add, a Welcome,
or a history chunk. Changing a grant creates a new revision; discovering a familiar slot never creates one. A revoked
grant cannot be revived by replay or by selecting an older device-group branch. New approval of a retired installation
requires an explicit fresh pairing and generation, not a delayed old approval.

**Distributed operation tracking.** One user action creates an account operation with separate obligations for each
chat, the device group, public discovery, and the external signer. Track prepared, exposed, selected, recipient-verified,
revoked, and unresolved facts separately. Reconcile and compensate after branch changes. Report partial progress instead
of treating local publication as an atomic account-wide transaction.

The following items preserve the earlier review's numbering. Each states an attack or failure precondition, a concrete
sequence, the proposed repair, and evidence needed to accept it.

## 1. A known slot does not authenticate a known installation

**Defect.** The [discovery rule][idea-detection] recognizes a slot without requiring continuity of the approved device's
keys. Client tags are self-reported.

**Attack scenario.** Mallory has Alice's account-signing authority, reads her phone's public slot, and publishes a newer
valid KeyPackage into it, copying the client tag. Mallory owns the package's private keys. An honest sibling resolves the
known slot to its newest package and gap-fills Mallory into existing chats without a new approval.

**Proposed changes.** Bind the approved installation key and slot during authenticated pairing. Every refreshed package
has a device-key signature over the account, slot, generation, exact package reference, capabilities, and bounded
validity. Preserve the MLS signature and account-identity proof; the extra proof serves a different purpose. Accept a
device-key rotation only with a continuity certificate signed by the old key, or after explicit fresh pairing if that
key is lost. An account-signed replacement without continuity becomes an unrecognized candidate, even in a familiar slot.

Gap filling and history delivery resolve a verified grant to exact device-authenticated package bytes. If a relay
replaces the slot, they reject the substitution and fetch the approved material through the device group. Ordinary
first-contact invitations remain possible to unlinked account-authorized devices, with the distinction in item 4.

**Acceptance evidence.** Replay valid refreshes out of order, replace a known slot with an attacker package and copied
tag, and rotate keys with and without continuity. No replacement without device continuity receives existing-chat access
or history from an honest sibling. This does not prevent an nsec holder from creating a separate account-authorized sign-in.

## 2. Linking one device cannot silently approve its whole roster

**Defect.** The [absorption rule][idea-merge] treats verification of one device as verification of its other members.

**Attack scenario.** Alice verifies her laptop. Its independently created device group includes Mallory's installation,
introduced during a previous compromise or mistaken approval. The phone absorbs the laptop's roster and sends Mallory
chat Welcomes and history even though Alice checked only the laptop.

**Proposed changes.** Keep group merging, but exchange a signed merge manifest listing both source group checkpoints,
every device key and generation, prior approval provenance, consent, and all known rejection/retirement records. A source
group's membership is evidence of its history, not approval by the destination group. The approver verifies each newly
trusted key directly, or approves an explicit list of keys vouched for by a currently trusted device under a recorded
delegation policy. The UI identifies every imported installation; unverifiable entries remain pending and receive no
destination secrets. Mutual proof of device-key possession is required even for a delegated entry.

Resolve consent by the intersection of existing permissions; preserve denials. Name the surviving group in the signed
manifest. Serialize incompatible merges against their exact parent manifests; stale plans are rebuilt and approved
again, not applied to a changed roster. Join each approved device through an ordinary fresh MLS Add. Retire the old group
only after its authorized obligations are reconciled and all migrating reachable devices acknowledge the new state;
offline devices retain a migration obligation. Do not erase the losing group's evidence on first publication.

**Acceptance evidence.** Merge two groups with an extra unverified member, conflicting opt-outs, a retired key, an offline
device, and concurrent competing merges. No implicit trust expansion or denial loss occurs; the approved devices still
merge and enroll across the union of permitted chats. A malicious device explicitly trusted to vouch for others remains
an acknowledged delegation risk.

## 3. Consent must follow the device across enrollment, gap filling, and history

**Defect.** The [approval toggles][idea-consent] have no distributed enforcement contract for [automatic adding][idea-bulk].

**Attack scenario.** Alice links a laptop with chat enrollment or history disabled. Her tablet was offline, later sees a
roster member missing from a chat, and fills the gap or exports history. Alternatively, the phone puts target-specific
history plaintext in the shared device group, where every sibling can read it.

**Proposed changes.** Store a versioned grant for each recipient device, with separate existing-chat, future-chat,
history, media, and small-state permissions. An explicit all-chats choice is a durable wildcard plus explicit exceptions;
a disabled toggle is a denial, not missing data. Define which roster devices can change those grants, require recorded
user consent for widening them, and have the recipient acknowledge the exact revision. Carry the policy through merges
and catch-up. An offline sibling obtains the relevant grant and denial checkpoint before it begins automatic work.

Require each enrollment or gap-fill task to cite its grant revision and chat. Revalidate before publication and before
any secret-bearing transfer. Known denial overrides an older grant regardless of delivery order; missing or conflicting
policy pauses the task. History is additionally encrypted to the permitted recipient using item 14's channel, even when
the device group carries its ciphertext. Membership in that group alone never grants history permission.

**Acceptance evidence.** Exercise both toggles independently, wildcard exceptions, permission changes during a transfer,
offline catch-up and merge. An honest sibling never widens recorded consent. Delivery suppression can hide a later
revocation until it is received; the UI and contract state that cutover limit, and disclosure already made is recorded.

## 4. Slot deletion is cleanup, not cryptographic revocation

**Defect.** The [removal promise][idea-removed] and [deletion steps][idea-removal] do not cover cached reusable packages.

**Attack scenario.** A thief keeps a stolen device's still-valid last-resort package and init key. Alice removes it and
revokes its signer session. An unrelated inviter fetches a cached package from a relay that ignored deletion and encrypts
a new Welcome to it. The thief decrypts without a new package or account signature.

**Proposed changes.** Retire the installation key/generation, all outstanding package references, and every known chat
leaf. Gossip authenticated retirement records privately to siblings; publish a minimal account-authenticated revocation
or admission-status object for inviters, without labels or chat IDs. Known retirement dominates stale positive discovery
records. Preserve exact device-scoped deletion events, but never use deletion acknowledgement as proof of exclusion;
[NIP-09][nip09] does not provide that guarantee.

To strengthen future admission, introduce a versioned fresh admission authorization bound to the inviter's challenge,
exact package, proposed group, operation ID, and expiry. A linked device's own signature is insufficient after theft:
the authorization also needs a non-target approved sibling or an external signer enforcing that installation's active
status. That co-authorizer also checks the linked target's future-chat grant and the account's decision for this exact
invitation; active status alone cannot override an enrollment opt-out or decline. Bind the grant revision and scoped
consent commitment in the admission proof. A sole raw-key installation can authorize ordinary admission with the account
key; stealing it is the accepted account-compromise case. Unlinked installations retain the idea's ordinary
account-authorized first-contact path, but
need a fresh account authorization; possession of cached package private material alone is insufficient. The account
authority used here is assumed uncompromised or effectively session-revoked.

Bind package issuance to linked or unlinked admission purpose and keep that classification through rotation and local
restart. A known linked or retired generation cannot fall back to the unlinked path to evade its grant or retirement.
Inviters verify the carried account-authorized purpose/binding; they do not infer it from an absent discovery response.
Creating a genuinely new installation identity still requires fresh account authority and normal discovery/approval;
an attacker that retains that authority is within the account-compromise boundary.

An inviter validates authorization before encrypting the Welcome, not just when the recipient later joins. Redundant
discovery and signed status checkpoints improve availability and detect conflicting views, but signatures cannot prove
that an eclipsed inviter has the latest view. For linked devices, co-authorizers refuse on known retirement and pause
when status cannot be refreshed. Bound authorization lifetime and state the maximum stale-authorization window. Expiry
is checked when preparing new admission; it does not invalidate a previously valid historical Commit during replay.

**Acceptance evidence.** Invite with cached packages after signer revocation, replay an old authorization for another
group or challenge, suppress retirement on one relay, and try approval by the stolen target alone. No new Welcome is
encrypted without the required fresh authority. Test future-chat opt-outs and linked-to-unlinked downgrade attempts as
well. Previously issued authorizations and legacy inviters remain exposure windows to report; capability rollout in
item 15 gates the stronger guarantee.

## 5. Conflicting decisions need explicit ordering and irreversible-effect accounting

**Defect.** [First decision recorded][idea-decisions] has no common ordering or account-wide rollback semantics.

**Attack scenario.** Disconnected siblings approve and reject the same sign-in or invitation. Each acts on its local
first message. Another approval later loses the device-group branch after chat Adds and history export have occurred.
Deleting the approval cannot retract those secrets or undo unrelated chat Commits.

**Proposed changes.** Define the decision reducer over authenticated operation IDs and causal parent references. Fold
decisions in the selected device-group Commit history and define exact ordering of records within one authenticated
batch; never use relay timestamps or arrival order. A rejection or retirement known for a target generation overrides
an outstanding approval for that generation. After a completed approval, a new rejection is an explicit retirement.
Retain authenticated denials and an exposure ledger outside rollback of the displayed roster, so selecting an older
branch cannot restart automatic enrollment or export. A new approval requires a fresh generation and consent.

Before irreversible history export, obtain a transcript-bound authorization receipt from the recipient and a durable
checkpoint acknowledgement from each device in the current authorizing roster. These are automatic policy checks, not
additional human approvals. Each acknowledges the exact decision and known-denial set, and durably locks that decision;
concurrent refusal prevents a complete receipt set. Keep the roster fixed for that operation. A new decision after that
barrier is a revocation, not a retroactive claim that export never happened. Offline devices can delay this barrier:
linking and catch-up remain supported, but unsafe export stays pending. A more available quorum variant would need its
own stated Byzantine fault bound, intersecting quorums, membership-change protocol and safety evidence before replacing
this conservative barrier.

Bind the barrier to a certified authorizing-roster checkpoint, consent revision and immutable export-manifest digest.
Each roster member durably records at most one compatible outcome for that barrier ID. A merge, retirement, key rotation
or conflicting control branch does not silently replace the roster or release those locks. Stop an unfinished export
and either obtain an explicit authenticated lock-transfer/cancellation acknowledgement from every required old member,
or keep it pending. A replacement roster can start a fresh barrier only with the reconciled old exposure/lock state;
it cannot declare a previously missing acknowledgement received. Persist the complete receipt set before export and
mark revoked barriers ineligible for fresh chunks after that fact is known. Acceptance includes conflicting roster
checkpoints, cancellation and resumption after merge, rotation, retirement and branch revival. This is an application
authorization barrier, not an extra vote in MLS branch selection.

For each exposed chat operation, reconcile its actual selected membership and issue independently authorized exact-leaf
compensating Removes when approval is withdrawn. Do not call those compensations a transaction rollback. Display possible
exposure even when removal succeeds. Invitation decline uses the same account operation to remove every known account
leaf and prohibit honest gap filling; hidden leaves remain unresolved obligations until discovered.

**Acceptance evidence.** Partition siblings, reorder approve/reject, crash after acknowledgements, and revive an older
device-group or chat branch. A denied grant never automatically restarts, incompatible pre-export decisions cannot both
pass the barrier, and the ledger retains every possible disclosure. Later revocation cannot erase a completed transfer.

## 6. Same-account authority needs validation at every MLS boundary

**Defect.** The [non-admin sibling path][idea-sibling] lacks complete Commit and Welcome authority rules.

**Attack scenario.** A compromised non-admin constructs a sibling Commit containing another account's Add/Remove or a
policy change. A validator authorizes the whole Commit because one operation qualifies. Conversely, peers accept an
honest sibling Add but the joiner rejects its Welcome because it still expects an admin inviter. A sponsor can also
present a valid but isolated chat branch.

**Proposed changes.** Specify narrow same-account Add and Remove forms in the owning component/core documents. Validate
the complete proposal list against the authenticated candidate parent: account bindings, package signatures, device
scoped admission proofs, supported capabilities, key uniqueness, resulting-state ten-leaf cap, and admin-policy constraints.
Every operation needs its own valid authority. A sibling path cannot authorize unrelated changes. Independently valid
admin authority remains available and retains its established priority; admin status is account-scoped.

Separate private orchestration policy from consensus-visible validity. Carry a self-contained scoped authorization
with the chat operation, authenticated by the sponsor's chat leaf and the target's bound package/receipt keys, naming
the exact candidate parent, account, recipient, package, operation and consent commitment. Any required account/signer
attestation also travels with its verifiable chain. Peers can validate those bytes without joining the private device
group or fetching its roster. A consent commitment records the scope asserted by the trusted sponsor; it does not prove
every private preference to outsiders. Protect the private grant separately by item 3's honest-client checks.

New preparation stops on known denial. Receiving or replaying an already exposed Commit validates its carried proof
against its authenticated parent and original authorization context, never against the receiver's latest private roster,
current wall clock or latest grant. A later denial schedules compensation; it does not retroactively make a valid chat
Commit invalid. Retained original authorization evidence is sufficient for independent replay. Removing this distinction
would let different private discovery views split chat consensus.

Bind enrollment packages and receipts to the exact sponsor leaf/lineage, target device, chat, consent, and package bytes.
The joiner checks the authorized sibling Welcome path, package purpose, authenticated group context and exact expected
admission. Ordinary admin invitations have their own fresh-admission checks from item 4; failed or canceled enrollment
packages cannot downgrade into that path. Apply the same rules to creation, local preparation, inbound processing,
replay and retained-branch revival. Account control records are evidence for these checks, not a substitute for MLS.

**Acceptance evidence.** Reject mixed non-admin Commits and wrong-purpose Welcomes; accept valid bulk/account enrollment;
verify admin precedence and identical send/ingest/replay decisions. Cross-check a new joiner's group checkpoint with a
healthy sibling when available and record uncertainty otherwise. A malicious authorized sponsor can still lie about
its isolated view, as the [existing bootstrap rules][joining] recognize.

## 7. Leaf positions need authenticated installation binding and lineage

**Defect.** The proposed [leaf announcements][idea-mapping] treat an index as durable device identity.

**Attack scenario.** A sibling remembers that the laptop owns leaf 7. The laptop is removed and a tablet later occupies
7; a delayed removal evicts the tablet. A compromised sibling can also announce that it owns another device's leaf or
forge a completion message using only the common account identity.

**Proposed changes.** Maintain the private account-wide device-to-chat directory, but require two connected proofs: the
verified device key signs the chat, exact parent/checkpoint, admission/package reference and claimed leaf; an application
acknowledgement from the actual chat leaf authenticates the same binding. Challenge-bind the proof to the requesting
sibling to prevent reuse in a different context. Authenticate the MLS sender, not an inner account field. Group equality
and leaf number alone are insufficient.

Track a membership lineage from its authenticated Add through legitimate own-leaf updates. Remove/Add starts a new
lineage even at the same index with the same account. Bind every removal and completion acknowledgement to its expected
parent and lineage; re-resolve against the new parent before preparing replacement bytes. An inconsistent or missing
proof leaves the mapping unresolved, rather than authorizing a guessed removal. Normal same-account removal authority
still applies; proving ownership does not grant cross-account removal authority.

**Acceptance evidence.** Reuse an index, rotate a leaf key, replay a delayed announcement, forge another sibling's ack,
and revive the old branch. Only the intended lineage is affected. Real clones of that lineage remain indistinguishable;
recovery removes the shared lineage and enrolls clean independent devices, not one purported physical copy.

## 8. Four words must authenticate a fresh two-party session

**Defect.** The [code derivation][idea-code] and [collision argument][idea-collision] rely on a short window without
enforcing freshness or binding subsequent approval to immutable bytes.

**Attack scenario.** An account-key attacker observes the laptop's public package and has days to search for another
package with its 44-bit truncated code while Alice leaves the prompt pending. Suppressing the legitimate candidate
defeats duplicate-code detection. Independently, approval checks one package but an Add later resolves the slot again
and uses a substituted package. This is a missing security bound, not evidence that a 44-bit target search is practical.

**Proposed changes.** Keep remote four-word confirmation, but derive it from a reviewed short-authentication-string
protocol over an ephemeral authenticated key exchange, rather than a static public package hash. Use a commitment
phase that fixes both parties' contributions before the peer's unpredictable contribution is revealed. Bind account,
both installation keys, roles, fresh nonces, ephemeral keys, exact package hashes, session ID and negotiated version in
the transcript. Human comparison on the two independently held screens authenticates that transcript; DH by itself
and account signatures shared by both installations do not.

Freeze the transcript and package bytes before either approval. Use one-use session IDs, explicit cancellation and
supersession, a fixed short deadline measured with monotonic elapsed time, bounded attempts, and fresh challenges after
restart. Late signer callbacks cannot revive a canceled session. Final grant, recipient receipt, and channel key
confirmation cover the same complete transcript. Use forward-secret, role-separated record keys with authenticated
sequence numbers and bounded framing. A QR can convey the same session commitment as an optional convenience, but QR
capture alone cannot authorize package substitution; prove that property for the chosen construction.

**Acceptance evidence.** Independent implementations agree on canonical transcript/word-list fixtures. Exercise
commitment grinding, reflection, MITM substitution, duplicate candidates, QR capture, expiry, clock rollback, restart
and package refresh after approval. Record the security bound for the chosen protocol and allowed attempts, not merely
the number of words. An audited protocol/profile and cryptographic review are required before wire freeze; this review
does not invent or certify a new key exchange. No account-wide function requires a LAN-only carrier.

## 9. Bulk enrollment needs independent packages and recoverable Welcome obligations

**Defect.** [Bulk enrollment][idea-bulk] shares one last-resort package without reconciling the [init-key deletion
bound][key-lifecycle].

**Attack scenario.** Fifteen Welcomes use one package. The laptop processes the first, publishes its replacement and
deletes the old init key. Fourteen delayed Welcomes can no longer be decrypted, while the corresponding Adds may already
exist. Keeping that shared key indefinitely exposes every recorded Welcome if it is later stolen; self-updates cannot
erase those recordings.

**Proposed changes.** Keep one installation publication slot for discovery, but use a fresh, single-use, purpose-bound
package for each chat enrollment. Exchange the bounded set over the authenticated device channel. Discovery slot
rotation and those private enrollment packages have separate lifecycles. Persist each package's purpose, recipient,
chat and operation before releasing its bytes, and consume it atomically with durable Welcome processing. Delete its
init key after successful consumption. A local self-update remains useful but is not the confidentiality argument.

For all-device first-contact invitations, obtain distinct per-invitation packages using item 4's fresh authorization
exchange; maintain bounded prekeys for asynchronous reachability where feasible. If a last-resort fallback is offered,
state its delayed-delivery/key-retention tradeoff and track that device's pending invitation separately. Do not extend
the existing deletion bound to rescue fanout. A failed fallback is repaired with a fresh authorized Add, after resolving
or removing the old lineage, not by silently accumulating ghost leaves.

Track exact Add/Welcome bytes, publication, selected branch, recipient ack and refusal per chat. Retry already exposed
bytes exactly. Loss of an init key does not prove the Add failed; timeout does not authorize automatic destructive
cleanup. Reconcile outstanding attempts before creating replacements, including losing branches that can still revive.

**Acceptance evidence.** Delay fourteen of fifteen Welcomes, rotate the public slot, lose one private package, reorder
acks, and revive a losing Add. Other chats still join. The failed chat remains repairable or truthfully unresolved,
without excessive init-key retention or duplicate selected leaves. [MLS package reuse guidance][mls-reuse] supports
separating these lifecycles.

## 10. Stolen-device recovery must revoke signer sessions and MLS authority

**Defect.** The [external-signer claim][idea-signer] conflates keeping the nsec elsewhere with terminating delegated access.

**Attack scenario.** A thief steals the phone's signer-session credentials and MLS state. The signer continues honoring
its signing and inbox-decryption requests after Alice deletes the phone from the roster. Conversely, terminating the
session alone leaves the stolen chat leaves able to read or send until effective removal.

**Proposed changes.** Give each installation its own signer client key/session. Bind its session identifier privately to
the verified device identity and request only the permissions needed by that installation. The recovery workflow opens
the signer's authenticated management UI or uses an explicitly supported authenticated management protocol to revoke the
lost installation's session. Do not presume [NIP-46][nip46] defines remote revocation of another client's session: its
client logout is not such a security boundary. Unsupported signer management is an explicit unresolved recovery step,
with instructions to revoke at the signer, not a false success result.

The same account operation retires the device, invalidates outstanding grants/admission authorizations, issues exact
leaf removals in all known chats and the device group, and retires its public packages. The signer records revocation
durably, rejects stale queued requests and reconnection without renewed user authorization, and returns authenticated
completion evidence where supported. Verify delayed signer responses against the operation and current permissions.

**Acceptance evidence.** After revocation, copied client credentials fail signing/decryption requests across signer
restart; delayed replies do not resume enrollment. Separately prove selected removal of each stolen lineage and show
remaining blocked chats. No screen claims recovery complete merely because one boundary succeeded. Raw nsec theft
still requires account migration; neither session revocation nor device removal erases previously retained content.

## 11. Discovery and warnings need authenticated evidence and delivery assumptions

**Defect.** The [detection guarantee][idea-detection] and [prompt triggers][idea-prompts] lack completeness and warning rules.

**Attack scenario.** A relay hides an unauthorized slot from Alice but serves it to an inviter. Alice sees no prompt.
An outsider then floods her inbox with malformed Welcomes naming unfamiliar package references, provoking false
key-compromise warnings. A legitimate sibling's retired or rotated package can also look unfamiliar.

**Proposed changes.** Keep account-wide discovery and all-device invitations. Query and subscribe to redundant relay
sets with bounded periodic reconciliation; exchange signed observed-slot/package checkpoints and anomaly evidence inside
the device group. Validate account/event signatures, package bytes, identity proof, lifetime, capability and device
continuity before classifying a discovery candidate. Existing and retired package inventories distinguish a legitimate
rotation from a new installation. A client tag is a display hint, never vendor attestation.

For a Welcome, authenticate its envelope and any attributable evidence before raising a security alert. An unfamiliar
reference alone cannot prove someone signed a new account package. Fetch and validate that package or another actual
account-signing artifact; otherwise report unverified delivery, not confirmed key compromise. Deduplicate prompts by
verified installation/package evidence, cap candidate work and aggregate flooding alerts. Distinguish unrecognized
account-authorized sign-in, failed known-device continuity, reappearance of a retired generation, and invalid input.

Inviters fan out to every discovered valid compatible authorized device up to ten, in deterministic protocol-defined
order. Preserve accept/decline as an account operation; unseen or unavailable devices catch up via consent-aware gap
filling. When discovery is incomplete or authorization is unavailable, record pending targets and reconcile instead of
claiming all installations were reached.

**Acceptance evidence.** Hide one relay's slots, reconnect an offline sibling, replay legitimate retired packages and
flood malformed inbox inputs. A reachable verified anomaly produces one meaningful prompt; unauthenticated traffic
does not prove account compromise. Detection is eventual under stated reachability assumptions, not guaranteed during
permanent eclipse. This corrects the promise while retaining the detection feature.

## 12. Inactivity cleanup needs a consented policy, not a claim of abandonment

**Defect.** The [30/90-day rule][idea-inactive] lacks evidence, clock, partition and removal-authority semantics.

**Attack scenario.** A live tablet's heartbeats are suppressed or background execution stops. The phone's clock jumps
forward and it evicts the tablet. Different siblings disagree about its activity or ignore a keep-while-inactive choice;
another cleanup attempt tries to remove the last authorized admin without satisfying the departure rules.

**Proposed changes.** Retain inactivity reminders and automatic cleanup, but make the policy explicit at enrollment:
silence may cause policy-based retirement, not prove theft or physical abandonment. Store a signed policy revision,
the keep override, heartbeat sequence/challenge and acknowledged activity checkpoints. A newer known override or activity
record cancels a pending retirement. Show both authenticated reported activity and when it was last observed; a device's
wall-clock timestamp cannot establish another device's inactivity.

At the threshold, create a proposed retirement bound to the exact device generation, policy revision and acknowledged
last-activity checkpoint. Probe over independent paths, distribute the proposal, and allow a specified grace period for
the target or a sibling to provide newer evidence. Use monotonic elapsed accounting with conservative restart recovery;
clock rollback, missing history, divergent checkpoints or widespread transport outage pauses automation. Time schedules
the operation locally and never changes historical MLS validity or branch selection. Fix the exact grace and clock
bounds in the policy document before adoption; clients cannot choose divergent interop rules silently.

If the policy permits retirement after the bounded grace, carry out the same durable account-wide removal workflow as
manual removal, subject to current authority/admin constraints. Explain that an isolated but live device can still be
retired under this consented policy; avoiding that in every partition would require abandoning automatic timeout-based
cleanup. The keep override preserves access for deliberately offline devices, and explicit re-pairing restores access.

**Acceptance evidence.** Exercise suspension, selective heartbeat loss, multi-relay outage, future timestamps, restart,
keep-override races, and last-admin cases. Unsafe clock/history states defer automation; valid consented policy still
performs automatic cleanup. The UI records policy retirement rather than claiming confirmed compromise.

## 13. Sign-out is a durable distributed departure, not publication followed by hope

**Defect.** The [sign-out sequence][idea-signout] orders local sends, not acceptance across groups or recoverable cleanup.

**Attack scenario.** Chat departure arrives before device-group departure; a sibling gap-fills the still-approved phone
back into the chat. The phone then wipes before a stale SelfRemove can be refreshed. A crash can also erase its signer
before public deletion bytes are prepared. On the last device, no sibling remains to finish any of these tasks.

**Proposed changes.** Begin with an authenticated retiring record for the installation generation and a durable manifest
of chat lineages, device-group membership, slot cleanup and signer-session obligations. Retiring immediately suppresses
new honest enrollment, invitation acceptance, gap filling and exports once observed. Siblings acknowledge the record
and take ownership of each recoverable cleanup obligation before the departing device erases their only copy. Retirement
is deny-only control state; it never becomes approval again because a device-group branch changes.

Ordering transport sends still cannot stop a sibling that has not received retirement. Therefore the sign-out contract
tracks Adds already in flight, keeps retirement evidence for reconciliation, and removes any revived or late-added
lineage. It reports pending network departure until the required actors observe the record and relevant chats settle
under the stated convergence assumptions. No temporary roster gap authorizes resurrection.

For normal completion, retain the minimum protected signing/leave state until selected membership removals are observed;
stop sending ordinary chat content during that interval. Use [existing SelfRemove][departure], refreshing epoch-bound
proposals before erasure, or an authorized sibling's exact-lineage Remove. A versioned, epoch-independent self-departure
authorization is another possible repair for post-wipe retry: the departing leaf signs an irrevocable intent naming its
chat and membership lineage, and remaining members can remove only that lineage in later parents. Such a new authority
needs its own canonical proof, replay and capability rules; it cannot bypass current admin constraints or target a reused
index. Do not pretend existing SelfRemove already has that property.

Prepare exact public deletion events and recoverable delivery obligations before revoking the signer. Keep only the
bounded non-secret retry material needed after final erasure. On the last device, finish each chat departure with a
remaining member or keep it pending; transfer last-admin authority first. Close a one-member device group through an
explicitly specified terminal close/disband operation, retaining already prepared retirement evidence for delivery.
If Alice chooses immediate local erasure while offline, show that local sign-out succeeded and network removal remains
unresolved; do not describe it as complete remote revocation.

**Acceptance evidence.** Reorder every departure, crash before and after handoff/erasure, make SelfRemove stale, revive
an Add, and sign out the last device or concurrent admin siblings. Every remaining obligation has durable retry or an
explicit blocked result. Per-device public cleanup never deletes a sibling's publications.

## 14. History, small-state sync and backups need a recipient-specific contract

**Defect.** The [history default][idea-consent] promises sensitive disclosure before [transfer and archive rules][idea-history]
exist. The idea prohibits MLS-state copying and nsec-only backups, but does not define how to enforce those boundaries.

**Attack scenario.** An honest sibling exports messages or media keys after their retention deadline, despite the
recipient's opt-out, or to every member of the device group. A later nsec compromise decrypts an account-encrypted
archive. Copying old MLS secrets for convenience creates cloned leaf state and defeats independent ratchet ownership.
A malicious exporter can also relabel an invented message as authentic old history.

**Proposed changes.** Keep history as part of account-wide linking, using a separately versioned content-transfer profile
bound to the established device identities and the exact history grant. Establish a fresh forward-secret channel or
recipient-specific envelope under fresh DH-derived transfer keys. Only the authorized recipient can decrypt it; the
device group can carry coordination and encrypted chunks. A pairing-control carrier does not implicitly authorize
content types. The default-on UI summarizes the scope and requires explicit link confirmation; recorded choices govern
every sibling and every retry.

Use an authenticated export manifest naming the exporter, recipient, chat, consent revision, source checkpoint, retained
time range, content hashes, retention policy and expiry. Recheck retention and known revocation before each chunk. Send
only content still permitted for export, including separately scoped media keys; apply the stricter known policy when
histories disagree. Bound chunk sizes, total transfer, decompression, object counts and resumptions. Authenticate chunk
indices/hashes and receipt state to prevent mix-and-match imports. Erase transfer keys and staging material on completion,
expiry or cancellation, while preserving non-secret exposure evidence.

Import into an archive namespace, separate from live MLS state. Preserve original source references where verifiable
and identify the exporter's attestation when independent original-message authentication is unavailable. A manifest
signature authenticates the exporter, not every claimed historical sender. Never import leaf signers, init keys,
epoch secrets, group-event keys, pending operations or sender ratchets. New membership uses fresh leaves. If a source
cannot provide trustworthy historical provenance, mark that content accordingly rather than fabricating live events.

Small-state sync has separately bounded, consented record types, such as read markers and pins, with defined causal
merge and deletion semantics. It does not inherit permission to transfer message drafts or content merely by sharing
a channel. Backups encrypt content under a fresh archive key protected by a high-entropy recovery secret independent
of the nsec, or an explicitly reviewed threshold recovery scheme. Account identity authenticates ownership but cannot
alone unwrap the archive. Record expiry, media access and restored-content provenance; restore no live MLS state.

**Acceptance evidence.** Run recipient opt-outs, retention expiry mid-transfer, revoked consent during resume, corrupt
chunks, oversized/compressed input, forged sender attribution and later nsec compromise of recorded archives. Independent
live leaves remain independent after import. A compromised exporter can still leak what it already retained, and a
compromised recipient can keep delivered content; the protocol cannot force remote erasure.

## 15. Readiness and rollback must cover every current leaf and account installation

**Defect.** The [readiness direction][idea-readiness] and [sibling path][idea-sibling] do not define enable/disable or migration.

**Attack scenario.** An admin enables the feature because people installed a new app or published a new public package,
but their existing leaves still advertise old capabilities. Those leaves cannot process sibling operations. Disabling
while multiple leaves remain changes authority beneath them; rolling back the app breaks retained feature branches.
An older installation signs out and deletes siblings' packages across the account.

**Proposed changes.** Version the group readiness component and each new device-control, admission and transfer profile.
Every current leaf publishes support in an accepted own-leaf update before explicit admin enablement. Capability on a
new package does not update an existing member. Founding and later leaves satisfy the same requirement. Upgrade a device
group's control profile only after its current leaves support it; unknown required record types block that upgrade
instead of being silently treated as no policy. Never evict unsupported devices automatically to make rollout easier.

Define the component's exact activation effects: narrow sibling authority, per-device admin departure when another leaf
of that admin account remains, resulting-state cap, and required enrollment proofs. Admin self-departure checks the
resulting parent state, not a stale claim that a sibling exists. The last leaf still follows last-admin rules. Disabling
requires at most one leaf per account and completion or explicit migration of all obligations using the old authority.

Maintain compatible receive/replay/recovery support and protected storage across rollback. A stopped feature prevents
new work; it does not discard old obligations or decode old bytes under new meanings. Upgrade all known account
installations to device-scoped publication cleanup before activating account-wide workflows. Strong cached-package
revocation in item 4 additionally requires upgraded inviters; mark legacy invites as having weaker guarantees rather
than implying a group capability can police outsiders.

**Acceptance evidence.** Test independently implemented clients, old/new existing leaves, offline leaves, profile
upgrades, disable attempts with multiple leaves, retained-branch replay and rollback. Publish a tested minimum-build
matrix before release. Capability support establishes protocol readiness; a version string alone does not.

## 16. Ten leaves do not bound discovery, retries, competing branches, or churn

**Defect.** The [ten-leaf limit][idea-cap] does not define all resource bounds or resolve the [gap-fill race][idea-gap-race].

**Attack scenario.** Two honest siblings add the same missing device from the same parent. Near ten leaves, competing
branches are each valid but combining their leaves gives a false overflow. A compromised sibling churns Add/Remove
operations indefinitely below the cap. An account-key attacker publishes thousands of slots; an outsider floods the
pairing carrier or history-transfer decoder.

**Proposed changes.** Retain the ten-leaf cap and enforce it on each complete candidate resulting tree, never on a union
of alternate branches. State separate limits for Add/Remove batch sizes, package sets, discovery candidates, decoded
bytes, signatures, active sessions, transfer work and stored operation receipts. Use a per-chat/device-generation
idempotency key plus exact parent/package references. A device-group gap-fill claim reduces duplicate preparation, but
is a bounded scheduling lease, not exclusive authority or a branch-selection rule. Revalidate membership immediately
before publishing, coordinate bounded retries and retain duplicate-exposure evidence for cleanup.

Reject malformed or unauthenticated traffic early; deduplicate verified candidates before prompting. Never discard a
required valid convergence dependency merely because a local queue is full. Apply backpressure, suspend new local work,
or report resource-blocked recovery while preserving the existing deterministic core semantics. Any validity-affecting
churn limit would need an explicitly negotiated protocol rule, not a private per-client rate limit. An authorized
attacker that indefinitely extends valid branches can still defeat convergence liveness; isolating/removing that actor
or stopping the session is the recovery boundary.

**Acceptance evidence.** Measure CPU, memory, storage, fanout and replay at one, two and ten leaves, with multiple accounts,
concurrent gap filling, sustained below-cap churn and candidate/decoder flooding. Exercise buffer exhaustion during
recovery and fair resumption after backpressure. Freeze interoperable bounds only with evidence; reducing the product
to five devices or single-device invitations is not this review's proposed fix.

## Proposed change packages and completion evidence

All packages belong to this full-scope deliverable. Dependency order sequences the work; it does not defer account-wide
features to an unspecified later project. The narrow plan remains useful implementation input where its safeguards
match these requirements, but its exclusions are not acceptance criteria for this alternative.

| Package | Owning surfaces to amend or add | Findings | Acceptance gate |
| --- | --- | --- | --- |
| A. Installation and admission identity | Foundation identity, KeyPackages, authorization proofs and canonical encoding; Nostr publication/discovery binding | 1, 4, 7, 8, 9, 11 | Exact signed bytes, key continuity, remote pairing transcript, fresh admission and cached-package attack fixtures |
| B. Account control and device group | New versioned app components for readiness/roster policy as needed; a feature-level account-linking flow and an owned application-record definition | 2, 3, 5, 11, 12 | Deterministic control reducer, denial persistence, merge/catch-up transitions, consent and pre-export barrier traces |
| C. Chat membership and fanout | Same-account membership component; core group setup, joining, departure, publish lifecycle, convergence interaction and durability | 6, 7, 9, 13, 15, 16 | Send/ingest/replay parity, exact lineage authority, per-chat recovery, all-device invitation and gap-fill journeys |
| D. Retirement and signer recovery | Account-linking feature, device-group records, signer-management integration and transport-owned revocation/publication rules | 4, 10, 12, 13 | Manual/automatic removal and last-device sign-out fault matrix; signer and network outcomes reported separately |
| E. Content and backup sync | Account-sync feature; owned transfer/manifest encoding, retention/media references, and each supported carrier profile | 3, 5, 14 | Recipient isolation, retention/provenance, consent races, archive secrecy and independent-leaf restoration |
| F. Interoperability and bounded rollout | Foundation registries/conformance, affected surface indexes and non-normative client release documentation | 15, 16; cross-cutting | Independent fixtures/clients, ten-leaf resource measurements, version matrix and rollback evidence |

Foundation owns reusable identity/proof/encoding rules; components own their direct versioned bytes; core owns MLS
transitions; transports own outer envelopes, publishing and fetch rules. The feature describes the user journeys and
references those owners. Final adoption updates `layout.md`, each affected surface index, and registries together, with
private-use component IDs and new IDs for breaking component versions. This review assigns none. Implementation
architecture stays in implementation documentation.

For every package, produce exact bounded encodings, positive and rejection fixtures, prior-state/authority checks,
durable retry boundaries, and traces for crash, offline catch-up, concurrency and branch revival. Obtain an independent
security/protocol review of the pairing, control barrier and history profiles; a documentation PR is not that evidence.
Release only after all six packages' end-to-end evidence covers account linking, two-group merge, enrollment with
opt-outs, invitation accept/decline, gap filling, recipient-specific history, device removal and last-device sign-out.

Completion claims have explicit meanings:

- **Linked:** the exact installation has an authenticated approved grant and recorded device-group admission; chat
  enrollment and history have their own visible progress.
- **Enrolled:** that chat's current selected branch includes the verified target lineage and the target acknowledged
  the exact admission. A relay receipt alone is insufficient; later branch changes reopen reconciliation.
- **Retired:** honest siblings retain the denial and stop authorizing new work; chat removal, public cleanup and signer
  revocation remain independently tracked.
- **Cleanup complete for the known scope:** every enumerated obligation has verified evidence under the stated delivery,
  authority and convergence assumptions. It is not proof of unknown chats, permanent global finality or remote erasure.
- **Locally signed out:** protected local account/content state is erased. This can precede network cleanup only with
  an explicit unresolved outcome and recoverable non-secret obligations.

These contracts retain the account-wide product while correcting promises that a serverless asynchronous system cannot
unconditionally make. They close the identified design gaps by supplying enforceable authority, state and recovery
requirements; they do not claim to eliminate account-key compromise, malicious already-authorized disclosure, or
permanent denial of service.

[prior-review]: https://github.com/liamhelmer/marmot/blob/cdfe3aeb249401a94ca00b64cd3c03f256f904fa/features/multi-device/review-07da8ffbfa7aa6c1e73e8e326085d0c47dc298e7.md
[idea]: https://github.com/marmot-protocol/marmot/blob/07da8ffbfa7aa6c1e73e8e326085d0c47dc298e7/ideas/multi-device.md
[narrow-plan]: https://github.com/liamhelmer/marmot/blob/cdfe3aeb249401a94ca00b64cd3c03f256f904fa/features/multi-device/README.md
[convergence]: ../../protocol-core/convergence.md
[joining]: https://github.com/marmot-protocol/marmot/blob/07da8ffbfa7aa6c1e73e8e326085d0c47dc298e7/protocol-core/joining.md#L42-L102
[departure]: ../../protocol-core/member-departure.md
[idea-detection]: https://github.com/marmot-protocol/marmot/blob/07da8ffbfa7aa6c1e73e8e326085d0c47dc298e7/ideas/multi-device.md#L44-L55
[idea-merge]: https://github.com/marmot-protocol/marmot/blob/07da8ffbfa7aa6c1e73e8e326085d0c47dc298e7/ideas/multi-device.md#L126-L129
[idea-consent]: https://github.com/marmot-protocol/marmot/blob/07da8ffbfa7aa6c1e73e8e326085d0c47dc298e7/ideas/multi-device.md#L146-L148
[idea-bulk]: https://github.com/marmot-protocol/marmot/blob/07da8ffbfa7aa6c1e73e8e326085d0c47dc298e7/ideas/multi-device.md#L187-L195
[idea-removed]: https://github.com/marmot-protocol/marmot/blob/07da8ffbfa7aa6c1e73e8e326085d0c47dc298e7/ideas/multi-device.md#L48-L49
[idea-removal]: https://github.com/marmot-protocol/marmot/blob/07da8ffbfa7aa6c1e73e8e326085d0c47dc298e7/ideas/multi-device.md#L251-L262
[idea-decisions]: https://github.com/marmot-protocol/marmot/blob/07da8ffbfa7aa6c1e73e8e326085d0c47dc298e7/ideas/multi-device.md#L168-L171
[idea-sibling]: https://github.com/marmot-protocol/marmot/blob/07da8ffbfa7aa6c1e73e8e326085d0c47dc298e7/ideas/multi-device.md#L199-L201
[idea-mapping]: https://github.com/marmot-protocol/marmot/blob/07da8ffbfa7aa6c1e73e8e326085d0c47dc298e7/ideas/multi-device.md#L327-L330
[idea-code]: https://github.com/marmot-protocol/marmot/blob/07da8ffbfa7aa6c1e73e8e326085d0c47dc298e7/ideas/multi-device.md#L102-L104
[idea-collision]: https://github.com/marmot-protocol/marmot/blob/07da8ffbfa7aa6c1e73e8e326085d0c47dc298e7/ideas/multi-device.md#L161-L164
[idea-signer]: https://github.com/marmot-protocol/marmot/blob/07da8ffbfa7aa6c1e73e8e326085d0c47dc298e7/ideas/multi-device.md#L54-L55
[idea-prompts]: https://github.com/marmot-protocol/marmot/blob/07da8ffbfa7aa6c1e73e8e326085d0c47dc298e7/ideas/multi-device.md#L154-L159
[idea-inactive]: https://github.com/marmot-protocol/marmot/blob/07da8ffbfa7aa6c1e73e8e326085d0c47dc298e7/ideas/multi-device.md#L264-L268
[idea-signout]: https://github.com/marmot-protocol/marmot/blob/07da8ffbfa7aa6c1e73e8e326085d0c47dc298e7/ideas/multi-device.md#L270-L290
[idea-history]: https://github.com/marmot-protocol/marmot/blob/07da8ffbfa7aa6c1e73e8e326085d0c47dc298e7/ideas/multi-device.md#L344-L347
[idea-readiness]: https://github.com/marmot-protocol/marmot/blob/07da8ffbfa7aa6c1e73e8e326085d0c47dc298e7/ideas/multi-device.md#L339-L340
[idea-cap]: https://github.com/marmot-protocol/marmot/blob/07da8ffbfa7aa6c1e73e8e326085d0c47dc298e7/ideas/multi-device.md#L225-L232
[idea-gap-race]: https://github.com/marmot-protocol/marmot/blob/07da8ffbfa7aa6c1e73e8e326085d0c47dc298e7/ideas/multi-device.md#L204
[key-lifecycle]: https://github.com/marmot-protocol/marmot/blob/07da8ffbfa7aa6c1e73e8e326085d0c47dc298e7/foundation/key-packages.md#L86-L112
[nip09]: https://github.com/nostr-protocol/nips/blob/0046368a747c5c25ae2bec28bae0e537744c8f10/09.md
[nip46]: https://github.com/nostr-protocol/nips/blob/0046368a747c5c25ae2bec28bae0e537744c8f10/46.md
[mls-reuse]: https://www.rfc-editor.org/rfc/rfc9420.html#section-16.8
