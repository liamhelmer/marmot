# Security and completeness review of the multi-device idea

Status: non-normative review, 2026-09-30. This document identifies defects and unresolved requirements in the
`ideas/multi-device.md` proposal at `07da8ffbfa7aa6c1e73e8e326085d0c47dc298e7`, then compares concrete failure scenarios
with the [alternate multi-device action plan][plan]. It does not adopt either design or claim that planned safeguards
are implemented.

The alternate plan was inspected at `1fa5eadf726fed9b2e17db4291744eac62dcb85b`. General references use the
`liamhelmer/marmot` `features/multi-device` branch so they follow future revisions. References to exact source lines use
the inspected commit so the cited evidence does not move. References to the reviewed idea always identify its exact
revision and source lines.

Each item states the defect, the capabilities and sequence needed to expose it, the alternate plan's response, and
the remaining gap. The statuses mean:

- **Addressed in the plan:** concrete safeguards and validation work are specified; execution and verification remain.
- **Avoided by scope:** the alternate v1 excludes the behavior that creates the exposure; the broader feature is unsolved.
- **Partially addressed:** a safeguard or truthful limitation exists, but part of the scenario remains possible.
- **Deferred:** the alternate plan explicitly leaves the required protection to another project.

Account-key compromise is an accepted limitation of the idea. Findings involving that capability concern its additional
detection and approval promises, rather than an expectation that possession of the account key ceases to confer account
authority. Other scenarios require only a stolen device, a compromised current leaf, unreliable delivery, or honest
clients acting concurrently. A missing rule in an idea is a design gap, not evidence of an exploit in a shipped client.

## 1. A known publication slot is treated as evidence of a known device

**Defect.** The detection rule does not require cryptographic continuity between an approved installation and later
packages in its slot. The important signal is a slot outside the roster; a familiar client tag adds no independent
device authentication. [Reviewed rule][idea-detection].

**Exposure.** An attacker with account-signing authority reads an approved device's public slot, publishes a newer valid
KeyPackage there, and copies its client tag. The attacker owns the replacement's private keys. A client that recognizes
devices by slot alone can miss the new installation; if it uses that recognition for gap filling, it can add the attacker
to existing chats as though the approved installation had simply refreshed its package. This does not require guessing
the random slot id: it is public after publication.

**Alternate plan response — avoided by scope for enrollment.** Enrollment uses a package delivered in an authenticated
pairing session, an approved group-specific intent, and a durable receipt. A public slot replacement does not authorize
that operation. [D1 and P2][protocol] supply the binding; [M1][mdk] treats local publication ownership separately from relay
discovery. The approved package exchange is shown at [flow 2, line 58][flow-package].

**Remaining gap.** An authenticated account-wide device directory, slot replacement authorization, and recognized-device
alerts remain [deferred][deferred]. No diagram specifically covers recognized-slot takeover. The plan does not protect
ordinary invitations from account-key compromise or prove that every installation has been detected.

## 2. Verifying one device can import an unverified second roster

**Defect.** The merge rule expands approval of one device into admission of its other device-group members without
requiring separate verification of those members. [Reviewed merge rule][idea-merge].

**Exposure.** Alice checks her laptop's code. Its separate device group also contains an unexpected installation, through
earlier account compromise, a compromised laptop, or a previous mistaken link. Absorbing that whole roster can import
the unexpected device into the phone's trust set. Automatic enrollment or history transfer can then disclose chats
Alice intended to expose only to the laptop. Verifying the laptop's package does not verify the other members' keys.

**Alternate plan response — avoided by scope.** The alternate v1 enrolls a particular joiner into selected groups; it
does not implement roster absorption. Each group/package operation has its own consent record. The separation is
illustrated at [flow 1, line 32][flow-scope], and the receipt is shown at [flow 2, line 60][flow-consent].

**Remaining gap.** There is no secure device-group merge protocol in this plan. A future merge must define which device
keys are individually approved, how conflicting rejection/removal records are preserved, and how access scope survives
the merge. A malicious sponsor already trusted to enroll siblings remains a trust boundary; see
[flow 3, line 120][flow-sponsor].

## 3. Enrollment and history opt-outs lack distributed enforcement

**Defect.** The approval screen offers enrollment and history choices, but automatic adding and gap filling do not
define how every sibling retains and honors those choices. [Reviewed choices][idea-consent] and
[automatic adding][idea-bulk].

**Exposure.** Alice disables "Add to all my chats" while linking a laptop. Later, another honest sibling sees the laptop
missing from a chat and fills the gap, granting access anyway. Similarly, one sibling can honor "Bring chat history"
being off while another interprets the approved roster entry as permission to send history. If historical content is
sent as ordinary plaintext inside a shared device group, every member can decrypt it regardless of a target-specific
toggle. These failures require no compromised account key.

**Alternate plan response — avoided by scope, with explicit enrollment consent.** The joiner selects groups and durably
approves exact intents before the sponsor publishes. There is no unrestricted roster-based gap filling or v1 history
transfer. [P2][protocol] and [flow 2, line 56][flow-selection] describe the selected-group boundary.

**Remaining gap.** Persistent cross-device chat inclusion/exclusion policy and history permissions remain unsolved for
the broader account-sync feature. [Deferred history work][deferred] requires separate consent and transfer policy.
No diagram specifically covers conflicting opt-out state between siblings or recipient-specific history encryption.

## 4. Publication deletion is insufficient to revoke a device

**Defect.** The proposal overstates the relationship between deleting a slot, detecting a new package, and excluding a
removed device from later admission. [Reviewed removal guarantee][idea-removed] and [deletion flow][idea-removal].

**Exposure.** A thief retains an unconsumed, still-valid published last-resort KeyPackage and its private material from a
stolen device. Alice removes that device and revokes its external-signer session. A different inviter obtains the cached
package from a relay that missed or ignored deletion and sends a new Welcome to it. The thief can decrypt that Welcome
without creating or publishing a new package or obtaining another account signature. A new admission needs a Welcome;
reuse does not necessarily need a newly generated KeyPackage. It cannot arise from old MLS state alone.

**Alternate plan response — partially addressed.** Enrollment-only packages cannot silently downgrade to ordinary admin
admission after cancellation; [P4][protocol] keeps their purpose and rejection records. Removal is explicitly per group
and branch, rather than account-wide. That limitation appears at [flow 8, line 302][flow-removal].

**Remaining gap.** Ordinary public packages still need authenticated revocation/discovery semantics for stronger exclusion
from future invitations. [Directory, compromise-recovery and stronger-revocation work][deferred] is deferred. Flow 8
covers the limitation, not this exact cached-package re-invitation. [NIP-09][nip09] makes deletion best effort; the
[reviewed package lifecycle][key-lifecycle] permits last-resort reuse within its bounds.

## 5. Conflicting decisions and rollback can leave irreversible disclosure behind

**Defect.** "First decision recorded" does not specify a common authenticated order or how decisions interact with branch
changes and actions in other groups. [Reviewed conflict rule][idea-decisions].

**Exposure.** Two honest siblings approve and reject the same sign-in while disconnected. Without a defined ordering rule,
they can interpret different decisions as first. Alternatively, an approval is acted on and later loses branch selection.
Chat Adds and transferred history may already have been exposed. Withdrawing the original device-group message cannot
undo the disclosure or atomically roll back membership in unrelated chats. No malicious relay forgery is needed; delay
and concurrency suffice.

**Alternate plan response — addressed in the enrollment plan.** [P2/P4][protocol] bind approval to an exact operation and
durable receipt, serialize first join against terminal refusal, retain exposure uncertainty, and track branch eligibility.
[M6][mdk] keeps results independent per group. Lost-ack uncertainty is shown at [flow 6, line 205][flow-uncertainty]; branch
revival is shown at [flow 7, line 253][flow-branches].

**Remaining gap.** The plan supplies no global finality or total ordering for a future account roster. It avoids history
disclosure during enrollment, but revocation of already disclosed content cannot be implemented by rollback. Reconciliation
bytes, transitions and tests remain P4/G1 deliverables. The broader approval-versus-rejection race has no dedicated diagram.

## 6. Sibling authorization needs complete commit and Welcome validation

**Defect.** Enabling a new non-admin membership path without defining its exact authority at every validation boundary
leaves both privilege-expansion and interoperability risks. [Reviewed sibling path][idea-sibling].

**Exposure.** A compromised non-admin member creates a nominal sibling operation containing an unrelated Add, another
account's Remove, or a policy update. A validator that authorizes the whole Commit from one qualifying operation accepts
unauthorized changes. Separately, an honest sibling Add can be accepted by existing peers yet rejected by the joiner if
Welcome validation still requires an admin inviter. A malicious trusted sponsor can also present a valid-looking isolated
branch; account equality alone does not establish the intended chat's canonical continuation.

**Alternate plan response — addressed in the plan, with an explicit sponsor-trust limit.** [P3][protocol] requires narrow
inline shapes, candidate-parent checks, identity/key validation, and independent ordinary admin authority for broader
operations. P4 routes Welcomes by package purpose and validates the exact sponsor and intent. Rejection of mixed non-admin
operations is covered at [flow 3, line 111][flow-authority]; sponsor trust is explicit at
[flow 3, line 120][flow-sponsor].

**Remaining gap.** Owning documents and implementations must actually receive the P3/P4 amendments; a plan is not that
change. A new joiner still trusts its sponsor and does not independently verify every parent-state rule or global agreement.
The [reviewed Welcome rules][welcome-rules] already describe bootstrap limitations; the alternative acknowledges them.

## 7. A leaf number does not establish device ownership or durable identity

**Defect.** The proposed leaf announcement lacks a verified device-to-leaf binding and branch-relative lineage rules.
[Reviewed mapping proposal][idea-mapping].

**Exposure.** A sibling caches "laptop is leaf 7." That leaf is removed; a healthy tablet later occupies position 7. A stale
removal targets the tablet. A compromised sibling can also lie about which same-account leaf it owns, or send a plausible
completion message for another leaf. Authenticating the device-group sender proves who made the assertion, not ownership
of the asserted chat leaf.

**Alternate plan response — addressed for per-group operations.** [M3][mdk] binds removal targets to the expected parent
and exact lineage, revalidates before exposure, and authenticates acknowledgements from the actual MLS sender. A Remove/Add
breaks lineage even when the index and account match. The wrong-sibling acknowledgement and reused-index boundary appear
at [flow 4, line 138][flow-wrong-ack] and [flow 4, line 146][flow-index]. Validation cases A03–A07 cover the distinction.

**Remaining gap.** The plan does not build the broader private device-to-chat directory. Nor can lineage distinguish two
physical copies holding the same actual leaf secrets; see [flow 5, line 182][flow-clone]. Those limitations must remain
separate from stale-index protection.

## 8. The short-code security argument lacks an enforced pairing session

**Defect.** The collision argument assumes a short attempt window without specifying session freshness, cancellation,
expiration, exact hashed bytes, or an immutable approval target. [Reviewed code][idea-code] and
[collision argument][idea-collision].

**Exposure.** Alice leaves a pairing attempt pending. An account-key attacker has a longer offline target-search window
than the claimed few minutes. If it finds a matching truncated code and delivery hides the legitimate candidate, the
approver can see only the attacker's matching code, so duplicate-code detection does not help. A separate race occurs
if approval verifies one package but adding subsequently resolves a newer package from the same slot. This is a missing
security argument and binding requirement, not a claim that a 44-bit target collision has been demonstrated practical.

**Alternate plan response — addressed in direction; D1 remains open.** [D1/P2][protocol] require authentication of both
participants' contributions and the complete session transcript, with package commitments and receipts. [M7][mdk] adds
signed expiry, monotonic elapsed limits, supersession and fail-closed restart. Transcript checks appear at
[flow 3, line 94][flow-transcript]. The alternate QR mechanism has its own active-substitution risk, explicitly covered
at [flow 3, line 113][flow-qr]; it is not safe merely because it replaces four words with a QR.

**Remaining gap.** Final channel construction, canonical transcript bytes, platform evidence and security acceptance are
D1/P2/P5 gates. An unmodified QR-PSK-only construction cannot pass them. No diagram specifically analyzes offline grinding
of the original word code, and account/endpoint compromise remains outside the pairing guarantee.

## 9. Bulk joins can lose Welcomes when their shared package is rotated

**Defect.** The bulk flow depends on one last-resort package surviving several joins without reconciling the replacement
and private-key deletion lifecycle. [Reviewed bulk flow][idea-bulk].

**Exposure.** Fifteen chat Welcomes target one package. The laptop receives the first, publishes a replacement, and erases
the old init key as required by the [reviewed lifecycle][key-lifecycle]. The other fourteen delayed Welcomes are now
undecryptable even though their Adds may already have inserted leaves. Extending key retention to rescue them increases
the exposure of every recorded Welcome under that shared key; a later self-update does not erase those recordings.

**Alternate plan response — avoided for enrollment.** [M4/M5][mdk] use fresh enrollment-only, non-last-resort packages and
atomic single-use consumption. Each group receives its own package, as shown at [flow 2, line 58][flow-package]. Losing a
particular package does not retire the packages for the other groups. [P4][protocol] preserves refusal and exposure evidence
and treats cleanup as a separate authorized operation; see [flow 6, line 219][flow-cancellation].

**Remaining gap.** Delayed delivery, package loss and unavailable catch-up history can still strand an individual leaf.
The proposed response is bounded retry and truthful uncertainty, not guaranteed completion. Ordinary public last-resort
invitations retain their existing tradeoff. No diagram shows the original multi-Welcome shared-package rotation itself.

## 10. An external signer does not automatically neutralize a stolen installation

**Defect.** Keeping the raw account key elsewhere does not by itself specify revocation of the stolen client's delegated
signing/decryption session or its existing MLS authority. [Reviewed signer claim][idea-signer].

**Exposure.** A thief obtains a phone's persistent signer-session credentials and MLS state, but not the nsec. While the
signer still honors that session, the thief can request permitted account signatures and inbox decryption. Merely deleting
the device's roster entry does not terminate that capability. Conversely, revoking the signer session does not itself
remove already authorized MLS leaves or erase local messages. Recovery needs both boundaries addressed.

**Alternate plan response — partially addressed; session revocation deferred.** [M7/M8][mdk] require account access through
supported signer abstractions, verify signer results, reject stale/refused callbacks, and prohibit account-secret transfer.
Per-group removal supplies a separate membership operation. The distinction between account authority and leaf compromise
is explained at [flow 5, line 194][flow-clone-recovery].

**Remaining gap.** [NIP-46 session, permission and restart design][deferred] remains independent work. The plan does not
specify remote revocation of another installation's signer session, coordination with group cleanup, or a combined recovery
completion condition. No diagram specifically covers the stolen signer-session scenario. [NIP-46][nip46] distinguishes
session permissions and termination from physical storage of the user key.

## 11. Discovery cannot guarantee timely detection, and alerts need validation

**Defect.** The detection promise does not define its delivery assumptions, discovery completeness, or sufficient evidence
for a key-compromise warning. [Reviewed detection guarantee][idea-detection] and [prompt triggers][idea-prompts].

**Exposure.** A relay or partition hides an unauthorized account-signed slot from Alice while another inviter sees it;
Alice receives no prompt until the evidence becomes reachable. Separately, if clients warn on an unrecognized Welcome
reference before establishing what it authenticates, an outsider can send unsolicited or malformed inbox traffic and
cause false compromise warnings. A rotated or retired legitimate package can also be unfamiliar to a sibling. A public
client tag is a publisher's claim, not vendor attestation.

**Alternate plan response — partially addressed.** [D3][protocol] and [M1][mdk] retain first-contact reachability uncertainty and
avoid using public-slot fanout as enrollment authority; M1 records unattributed-slot delivery risk without pruning or
retargeting packages. Pairing validates before disclosure or enrollment side effects. The availability limit appears at
[flow 3, line 117][flow-availability], and first-invitation uncertainty at [flow summary, line 326][flow-first-contact].

**Remaining gap.** An authenticated device-discovery and anomaly-alert protocol is [deferred][deferred]. Neither plan can
guarantee detection under permanent suppression. No diagram covers discovery suppression or false compromise prompts;
carrier denial of service is related coverage only. Warning criteria must distinguish authenticated compromise evidence
from unfamiliar delivery and malformed input.

## 12. Automatic inactivity removal can evict a live device

**Defect.** The inactivity policy does not define safe evidence, clock handling or authority for removal under incomplete
delivery. Its time limits are a retention policy, not proof that a device is abandoned. [Reviewed inactivity rule][idea-inactive].

**Exposure.** A legitimate device remains in use while its device-group heartbeat is suppressed or its background process
is suspended. Another sibling reaches the removal threshold and evicts it. A restarted clock or differently retained
liveness records can cause siblings to disagree. Automatic removal can also encounter an admin/last-admin constraint or
no surviving authorized remover. The result is loss of access or an unfulfillable cleanup promise, not evidence of key theft.

**Alternate plan response — avoided for v1 liveness cleanup.** The alternate enrollment plan does not use the idea's
account-wide inactivity eviction. [D5/P4][protocol] refuse to equate silence or a timeout with failed admission and keep
cleanup separately authorized. The analogous joined-but-offline case is explicit at [flow 6, line 238][flow-timeout].

**Remaining gap.** A future inactivity policy still needs authenticated evidence, user-visible policy consent, clock and
partition handling, authority checks, and recovery behavior. Enrollment uncertainty is not a substitute for that policy.
No diagram specifically covers heartbeat-based eviction of an established device.

## 13. Sign-out ordering does not guarantee network removal before erasure

**Defect.** Ordering local publication steps does not order acceptance across independent groups or establish durable
completion before local state is wiped. [Reviewed sign-out flow][idea-signout].

**Exposure.** A device publishes its device-group departure, then chat departures. A sibling receives the chat change first
and gap-fills the still-rostered device. Alternatively, a chat advances before committing the departure proposal; the
proposal becomes stale after the departing client has wiped the state needed to refresh it. A crash can also erase the
signing capability before publication cleanup has durable retry bytes. Remaining leaves or publications outlive the local
sign-out, and an admin departure may additionally require a prior policy change.

**Alternate plan response — partially addressed.** [M1][mdk] persists exact device-owned deletion events and target
obligations outside the account data being erased. [M6/M8][mdk] separate pending publication from observed membership and
preserve existing authorized departure constraints. Eliminating roster-based gap filling avoids that particular re-add
race. The required self-departure path is shown at [flow 8, line 298][flow-selfremove].

**Remaining gap.** M1 addresses publication cleanup, not a complete cross-group departure-before-wipe protocol. Durable
chat-leave obligations, stale-proposal recovery after wipe, the last device-group member, and global sign-out completion
are not established by this plan. No diagram covers the full distributed sign-out sequence; flow 8 is authority coverage.

## 14. History defaults require a separate security and retention contract

**Defect.** Offering history by default before defining transfer authorization, provenance, erasure and content bounds
leaves a promised confidentiality-sensitive behavior unspecified. [Reviewed history default][idea-consent] and
[deferred transfer mechanics][idea-history].

**Exposure.** An honest sibling exports locally retained old messages and media keys to a newly linked device despite an
expired retention policy or a narrower consent choice. If account-level encryption alone is used for an archive, later
account-key compromise exposes it. If live or historical MLS state is copied to recover history, installations can share
leaf secrets and lose independent ratchet ownership. The original idea prohibits state copying and nsec-only backups;
the defect is the missing transfer/import contract needed to enforce those boundaries.

**Alternate plan response — avoided during enrollment; history deferred.** [D1][protocol] excludes messages, drafts,
attachments, archives, account secrets and MLS state from the pairing channel. [Deferred history/archive projects][deferred]
require separate consent, provenance, limits and fresh device state. The absence of history bootstrap is explicit at
[flow 1, line 32][flow-scope], and the shared-leaf hazard at [flow 5, line 158][flow-state-copy].

**Remaining gap.** There is no implemented or finalized historical-content envelope, recipient permission model, retention
enforcement, media-key handling, or recovery-secret design here. Flow 5 covers state-copy hazards; retention-bypassing
history export and archive-key compromise have no dedicated diagrams. These exclusions must survive later scope expansion.

## 15. Feature readiness must be established for existing leaves, not installed apps

**Defect.** The readiness direction lacks the exact enablement, disabling and mixed-version rules needed to make the
new authority interoperable. [Reviewed readiness requirement][idea-sibling] and [open component question][idea-readiness].

**Exposure.** An admin enables sibling Adds because users installed new binaries or published compatible public packages,
while their existing chat leaves still advertise old capabilities. Those members cannot process the enabled behavior.
Another failure occurs when a group disables the feature while several leaves per account still depend on its semantics,
or rolls back to a binary that cannot process retained feature-bearing branches. Account-wide cleanup remains vulnerable
to an older installation that can delete siblings' publications.

**Alternate plan response — addressed in the plan.** The [compatibility matrix][compatibility] requires accepted own-leaf
capability updates from every current leaf, explicit group enablement, controlled founding/later admission, and an
at-most-one-leaf-per-account disable guard. Rollback retains receive/recovery support. K01–K05 require mixed-version
evidence, including older account installations. Group prerequisites appear at [flow 2, line 38][flow-prerequisites].

**Remaining gap.** Actual minimum client builds, coordinated rollout and passing compatibility evidence remain future
gates. No diagram covers the complete upgrade/rollback sequence; flow 2 states its prerequisites. Installing one upgraded
client does not satisfy them.

## 16. A leaf cap alone does not bound duplicate work, churn or retained branches

**Defect.** The cap does not specify complete resulting-state enforcement, duplicate suppression or the resource budget
needed around discovery and concurrent enrollment. [Reviewed cap][idea-cap] and [gap-fill race][idea-gap-race].

**Exposure.** Two honest siblings add the same missing device from the same parent, creating competing candidates and
duplicate delivery work. Near the cap, concurrent valid Adds produce alternative full branches; treating their leaves as
an accumulated set gives an incorrect overflow result. A compromised authorized sibling can repeatedly add/remove devices
or extend competing branches while staying below the current-tree cap. An account-key holder can publish many candidate
slots, and an outsider can flood a pairing carrier. None of those workloads is bounded merely by limiting selected leaves.

**Alternate plan response — partially addressed.** [P3][protocol] and [M6][mdk] require resulting-state checks and per-group attempt
coordination; [M7][mdk] sets record, object, buffering, replay and session budgets. D2 explicitly requires churn and retained-
branch measurements before freezing the provisional five-leaf cap. Concurrent alternatives are shown at
[flow 7, line 253][flow-branches]; bounded carrier exhaustion at [flow 3, line 117][flow-availability].

**Remaining gap.** D2 measurements and demonstrated limits remain pending. The plan does not guarantee progress against
an indefinitely active compromised member or adversarial carrier. Public candidate-discovery and notification budgets
still need the future directory/alert design. No diagram covers sustained below-cap churn or public-slot flooding.

## Follow-up required before describing the alternate plan as a fix

- Complete D1 and the canonical pairing/reconciliation contracts. The QR-substitution scenario is a security gate, not an
  optional enhancement. Use the [validation matrix][validation] to require active-substitution and reflection evidence.
- Amend the owning protocol surfaces and reconcile the superseded draft through P1/P5. This branch still contains an
  [older feature draft][old-feature]; the action plan itself does not amend its wire rules or assign new identifiers.
- Implement and verify the exact-leaf, durable-purpose, publication, cancellation and branch-revival safeguards. The
  validation document contains future acceptance cases, not evidence that those tests have run.
- Track account-wide directory/revocation, signer-session recovery, history/archives, inactivity policy and distributed
  sign-out as explicit gaps. Per-group enrollment cannot stand in for those projects.

Two residual limits apply even after those safeguards: an already compromised current sibling can exercise the proposed
reciprocal removal authority ([flow 8, line 307][flow-reciprocal]), and a malicious client holding an actual clone of a leaf
can impersonate that leaf ([flow 5, line 182][flow-clone]). The original idea already requires independent leaves; these
are threat boundaries to preserve, not a claim that either design authorizes cloning. Recovery may require removing the
shared leaf and enrolling clean replacements. It cannot promise selective removal of only one physical copy or universal
progress when no healthy authorized actor remains.

[plan]: https://github.com/liamhelmer/marmot/tree/features/multi-device/features/multi-device
[protocol]: https://github.com/liamhelmer/marmot/blob/features/multi-device/features/multi-device/protocol-plan.md
[mdk]: https://github.com/liamhelmer/marmot/blob/features/multi-device/features/multi-device/mdk-plan.md
[validation]: https://github.com/liamhelmer/marmot/blob/features/multi-device/features/multi-device/validation-and-rollout.md
[deferred]: https://github.com/liamhelmer/marmot/blob/features/multi-device/features/multi-device/validation-and-rollout.md#deferred-work
[compatibility]: https://github.com/liamhelmer/marmot/blob/features/multi-device/features/multi-device/README.md#compatibility-and-coordinated-upgrades
[old-feature]: https://github.com/liamhelmer/marmot/blob/features/multi-device/features/multi-device.md
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
[welcome-rules]: https://github.com/marmot-protocol/marmot/blob/07da8ffbfa7aa6c1e73e8e326085d0c47dc298e7/protocol-core/joining.md#L42-L102
[nip09]: https://github.com/nostr-protocol/nips/blob/master/09.md#client-usage
[nip46]: https://github.com/nostr-protocol/nips/blob/master/46.md#requested-permissions
[flow-package]: https://github.com/liamhelmer/marmot/blob/1fa5eadf726fed9b2e17db4291744eac62dcb85b/features/multi-device/flows-and-threats.md#L58
[flow-consent]: https://github.com/liamhelmer/marmot/blob/1fa5eadf726fed9b2e17db4291744eac62dcb85b/features/multi-device/flows-and-threats.md#L60
[flow-selection]: https://github.com/liamhelmer/marmot/blob/1fa5eadf726fed9b2e17db4291744eac62dcb85b/features/multi-device/flows-and-threats.md#L56
[flow-scope]: https://github.com/liamhelmer/marmot/blob/1fa5eadf726fed9b2e17db4291744eac62dcb85b/features/multi-device/flows-and-threats.md#L32
[flow-sponsor]: https://github.com/liamhelmer/marmot/blob/1fa5eadf726fed9b2e17db4291744eac62dcb85b/features/multi-device/flows-and-threats.md#L120
[flow-removal]: https://github.com/liamhelmer/marmot/blob/1fa5eadf726fed9b2e17db4291744eac62dcb85b/features/multi-device/flows-and-threats.md#L302
[flow-uncertainty]: https://github.com/liamhelmer/marmot/blob/1fa5eadf726fed9b2e17db4291744eac62dcb85b/features/multi-device/flows-and-threats.md#L205
[flow-branches]: https://github.com/liamhelmer/marmot/blob/1fa5eadf726fed9b2e17db4291744eac62dcb85b/features/multi-device/flows-and-threats.md#L253
[flow-authority]: https://github.com/liamhelmer/marmot/blob/1fa5eadf726fed9b2e17db4291744eac62dcb85b/features/multi-device/flows-and-threats.md#L111
[flow-wrong-ack]: https://github.com/liamhelmer/marmot/blob/1fa5eadf726fed9b2e17db4291744eac62dcb85b/features/multi-device/flows-and-threats.md#L138
[flow-index]: https://github.com/liamhelmer/marmot/blob/1fa5eadf726fed9b2e17db4291744eac62dcb85b/features/multi-device/flows-and-threats.md#L146
[flow-clone]: https://github.com/liamhelmer/marmot/blob/1fa5eadf726fed9b2e17db4291744eac62dcb85b/features/multi-device/flows-and-threats.md#L182
[flow-transcript]: https://github.com/liamhelmer/marmot/blob/1fa5eadf726fed9b2e17db4291744eac62dcb85b/features/multi-device/flows-and-threats.md#L94
[flow-qr]: https://github.com/liamhelmer/marmot/blob/1fa5eadf726fed9b2e17db4291744eac62dcb85b/features/multi-device/flows-and-threats.md#L113
[flow-cancellation]: https://github.com/liamhelmer/marmot/blob/1fa5eadf726fed9b2e17db4291744eac62dcb85b/features/multi-device/flows-and-threats.md#L219
[flow-clone-recovery]: https://github.com/liamhelmer/marmot/blob/1fa5eadf726fed9b2e17db4291744eac62dcb85b/features/multi-device/flows-and-threats.md#L194
[flow-availability]: https://github.com/liamhelmer/marmot/blob/1fa5eadf726fed9b2e17db4291744eac62dcb85b/features/multi-device/flows-and-threats.md#L117
[flow-first-contact]: https://github.com/liamhelmer/marmot/blob/1fa5eadf726fed9b2e17db4291744eac62dcb85b/features/multi-device/flows-and-threats.md#L326
[flow-timeout]: https://github.com/liamhelmer/marmot/blob/1fa5eadf726fed9b2e17db4291744eac62dcb85b/features/multi-device/flows-and-threats.md#L238
[flow-selfremove]: https://github.com/liamhelmer/marmot/blob/1fa5eadf726fed9b2e17db4291744eac62dcb85b/features/multi-device/flows-and-threats.md#L298
[flow-state-copy]: https://github.com/liamhelmer/marmot/blob/1fa5eadf726fed9b2e17db4291744eac62dcb85b/features/multi-device/flows-and-threats.md#L158
[flow-prerequisites]: https://github.com/liamhelmer/marmot/blob/1fa5eadf726fed9b2e17db4291744eac62dcb85b/features/multi-device/flows-and-threats.md#L38
[flow-reciprocal]: https://github.com/liamhelmer/marmot/blob/1fa5eadf726fed9b2e17db4291744eac62dcb85b/features/multi-device/flows-and-threats.md#L307
