# Multi-device implementation plan

> **For agentic workers:** implement the work packages in dependency order using
> `superpowers:executing-plans`, or `superpowers:subagent-driven-development` when that execution method is selected.
> Checkboxes track future work, not claims that the work has shipped. Read the owning repository's instructions first.

**Status:** non-normative action plan; prepared 2026-09-25. No protocol identifiers are assigned by this directory.

**Goal:** deliver opt-in, per-group enrollment and removal of additional devices of one Marmot account, with independent
MLS state, recoverable publication, and truthful client outcomes.

**Architecture:** a current account leaf sponsors an ordinary MLS Add of a fresh, same-account KeyPackage. An
authenticated pairing channel coordinates enrollment; the ordinary Welcome transport delivers admission material.
Every device retains independent cryptographic state. Group membership follows ordinary Marmot convergence.

**Technology:** MLS/OpenMLS, Marmot application components and authorization proofs, Nostr account identities and
Welcome delivery, MDK Rust session/runtime/storage layers, and native UniFFI/C consumers. Pairing channel and carrier
choices are subject to the decision gates below.

**Specification input:** [Marmot PR #417][pr417] at `994ba06878d7093e7ba2cf7f6c34b2fe37bc043b`, particularly its
[feature][spec-feature] and [component][spec-component]. These are proposal inputs, not the adopted behavior on master.

This directory consolidates the 2026-09-25 path-forward proposal and subsequent protocol, implementation, and discussion
review. It is self-contained; execution does not require the local Fable/Astra review documents. The implementation
details here are intentionally non-normative planning material. Move final protocol rules into their owning surfaces;
keep Rust APIs, storage design, scheduling, and test-runner details in MDK or `implementation-model.md`.

## Read order and deliverables

1. This overview: scope, gates, dependencies, and work-package sequence.
2. [Client flows and threats](flows-and-threats.md): eight diagrams covering happy path, tampering, cloned leaves,
   acknowledgement uncertainty, branch revival and removal; each explains what can and cannot be enforced.
3. [Protocol amendments and decisions](protocol-plan.md): decisions D1–D6, tasks P1–P5, authorization matrix, state model.
4. [MDK implementation work](mdk-plan.md): tasks M1–M8, source locations, interfaces, persistence, integration.
5. [Validation and rollout](validation-and-rollout.md): fixture and adversarial cases, commands, gates, pilot and rollback.
6. [Discussion and evidence register](discussion-register.md): agreement, contradictions, historical dispositions, follow-up.

The plan commit is documentation only. Implementation, upstream spec amendment, tracker updates, and release are future
work. Do not interpret a checked-in plan, an approved review, or a merged documentation PR as implemented support.

## Global constraints

- Each account-device owns independent MLS signer/state, KeyPackage private material, and sender ratchets. Never clone
  a SQLCipher/OpenMLS store, live or historical epoch secrets, pending MLS operations, or ratchets onto another device.
- Use same-account inline Add/Remove authorization. The External Commit, join-PSK, transferred group-event-key and
  `0x800a` path is superseded by this plan, but remains a live draft on the pinned master. P1/P5 must explicitly withdraw
  it in the owning documents and registry before adoption. Withdrawn wire values remain reserved and never reassigned.
- Treat `0x800d`, kind `453`, and new pairing record numbers as candidate assignments from #417 until the registry and
  amended owning documents agree. Kind `452` is already allocated to the old local join-authorization proof; the
  MLS-carried enrollment ack requires a new, collision-free kind. No implementation silently reinterprets old values.
- The starting v1 recommendation is one inline Add per Commit, one to four sibling Removes per Commit, and at most five
  leaves per account while enabled. D2 evaluates the cap before freezing it; do not independently choose a different
  cap in a client. A changed cap also requires review of all related bounds and fixtures.
- Enablement is opt-in per group by its creator/admin. Every current leaf supports the component before enablement;
  disabling requires at most one leaf per account in resulting state. Never automatically evict unsupported members.
- Pairing coordinates enrollment only. It neither establishes global finality nor changes branch selection. Do not add
  wall-clock rejection, transport arrival order, device tombstones, or mesh timestamps to consensus.
- No v1 history/content/attachment/account-secret transfer over the enrollment channel. No account-wide revocation,
  automatic all-device first-contact admission, public device directory, or global removal-completion claim.
- A joiner already controls the same account through its local key or supported external signer. Pairing does not
  provision that authority. An unavailable signer is a typed failure, not an excuse to export the account secret.
- A pairing KeyPackage is fresh, single-purpose, not last-resort, and never publicly relay-published. Persist purpose
  before exposing its public bytes. Purpose survives intent expiry/cancellation and cannot downgrade to admin joining.
- Preserve publish-before-apply and exact-byte retry. A relay acknowledgement, local selected membership, an enrollment
  acknowledgement, and global finality are different facts; v1 provides no global-finality certificate.
- Keep logs/forensics free of account/group IDs, keys, QR secrets, raw pairing bytes, relay URLs, and message content.
  Use bounded outcomes and aggregate counters. Protocol data required for recovery stays in protected local storage.
- Existing active groups, account removal, SelfRemove, push state, ordinary invitations, and unknown optional component
  preservation retain their established semantics except for explicitly reviewed protocol amendments.
- Distinct leaves are a supported-client requirement, not remote physical-device attestation. A malicious client that
  already holds a cloned leaf secret can impersonate that leaf. Do not promise clone detection or selective revocation
  of only one physical copy; [diagram 5](flows-and-threats.md#5-two-clients-share-one-leafs-private-state) explains recovery.

## Compatibility and coordinated upgrades

This plan changes no running client. The released compatibility baseline is MDK `v0.10.4`, checked on 2026-09-25;
the [evidence register](discussion-register.md#pinned-sources) pins its source and the Marmot spec revision. No released
version is identified here as multi-device-capable. P1/M8 must name and test the first supporting releases before rollout.

| Boundary | Required behavior |
| --- | --- |
| Feature absent from a group | Upgraded clients continue interoperating with the released baseline. Preserve ordinary invitations, authorization, Commit priority, replay and unknown optional data. Do not apply the new sibling authority or five-leaf cap globally. |
| Enablement in an existing group | Coordinate upgrades of every current member leaf across all accounts, not just sponsor and joiner. Each upgraded member must publish its support in an accepted own-leaf update; installing a binary or publishing a new KeyPackage alone does not update existing membership. A leaf without that support blocks enablement even while offline; do not evict it automatically. Already-supporting members need not be online simultaneously. |
| Creation or admission into an enabled group | Every founding or later member must support the required component. Baseline clients cannot join; keep groups that need them on the existing feature set. |
| Pairing | Sponsor and joiner must use compatible approved pairing and carrier versions as well as supporting the group feature. An unamended #417 pairing implementation is not presumed compatible with the amended transcript, receipts or ack kind. |
| Rollback | Stop new enrollment while retaining support for enabled groups, retained branches and recovery obligations. Removing the group requirement first needs at most one leaf per account; it does not by itself make an older binary or database downgrade safe. |

There is also an account-wide operational boundary outside group negotiation: baseline MDK sign-out/wipe can delete a
sibling's public KeyPackages. M1 cannot change an older installation's behavior. Before an account enters the pilot,
upgrade all its known installations to builds with device-scoped cleanup, including installations outside the target
group. Upgrade before using old sign-out/wipe as a retirement step. Unknown installations remain a documented risk to
future invitation delivery; neither group capability negotiation nor relay discovery proves they are absent.

Version pins belong at these existing gates:

| Stage | What to pin or coordinate |
| --- | --- |
| P1 / G0 | Record the released MDK and actual client builds used as compatibility controls; retain `v0.10.4` as this review's baseline and add newer releases explicitly. |
| P5 / G1 | Pin the amended Marmot revision, allocated identifiers, pairing/carrier versions and fixture revision before implementation claims interoperability. |
| M1–M8 / G2–G4 | Pin each candidate client's MDK dependency and matching native libraries, generated bindings/headers and storage migration version. Run the mixed-version cases against the released controls. |
| M8 / G4, then R1 | Publish the tested minimum and allowed builds for each client family/platform, including the M1 cleanup fix. Coordinate account installations and all group members, observe their signed capability updates, then let an admin explicitly enable the group before pairing. |
| R1 / G5 and later updates | Keep pilot builds pinned until replacement builds pass the compatibility matrix. A rollback target must retain required feature and storage support; do not fall back to `v0.10.4` for enabled groups. |

Different clients need compatible protocol support, not identical app version numbers. The release matrix records which
build combinations have been tested; signed capabilities establish member support, and an admin authorizes enablement.
Neither a version label nor a single-client test substitutes for
[mixed-version validation](validation-and-rollout.md#mixed-version-compatibility).

## Decisions before wire freeze

The architecture is the recommended direction; the following choices still require explicit recorded conclusions.
Review history is not consensus by implication. [D1–D6](protocol-plan.md#decision-gates) specify evidence and completion.

| ID | Decision | Starting recommendation | Completion evidence |
| --- | --- | --- | --- |
| D1 | Pairing authentication and forward secrecy | Authenticated ephemeral DH; unmodified PSK-only fails the active QR-capture case | Platform/library/size evidence, transcript binding, active substitution tests, payload exclusions, security review response |
| D2 | Per-account leaf cap | Keep five provisionally | Measurements at 1/2/5/16 leaves; resource and abuse analysis; explicit cap decision |
| D3 | Initial invitation reachability | One compatible selected KeyPackage; preserve and expose uncertainty | Documented limitation and recovery UX; separate directory/notice proposal instead of promising #417 solves it |
| D4 | Carrier interoperability | One specified LAN profile, if safe/platform-feasible | Two implementations or explicit single-family pilot; pairing-specific destination policy |
| D5 | Lost/expired enrollment and branch revival | Retain uncertainty; no destructive timeout inference; wait for permanent ineligibility before replacement | Exact transition table, durable reconciliation, adversarial traces |
| D6 | Admin authorization precedence | Highest priority granted by independently valid authority; admin membership changes remain privileged even in narrow form | Cross-surface authorization matrix and matched ingest/replay fixtures |

If D1 or D2 changes the recommendation, update the whole plan and proposal fixtures together before adoption. Do not
ship incompatible interpretations under one version while calling the difference a client preference.

## Work packages and dependencies

Ownership below names responsibilities, not individuals who have accepted assignments. Assign an accountable maintainer
and reviewer to each package when implementation starts. Each row should be a separately reviewable change or small
stack with its own passing acceptance evidence.

| Package | Deliverable | Responsible area | Depends on | Gate |
| --- | --- | --- | --- | --- |
| P1 | Record decisions and reconcile competing documents/trackers | Protocol + product + security | Source refresh | G0 |
| P2 | Correct bounded encoding and complete pairing control ordering | Protocol/codec | P1 D1/D4 | G1 |
| P3 | Authorization, capability lifecycle, admin precedence | Protocol/MLS | P1 D2/D6 | G1 |
| P4 | Enrollment lifecycle, terminal evidence, replay and branch recovery | Protocol/recovery | P2, P3, D5 | G1 |
| P5 | Fixtures, identifiers, carrier ownership, independent spec review | Protocol/conformance | P2–P4 | G1 |
| M1 | Device-scoped KeyPackage cleanup and invitation observability | MDK app/account | Can start now | G2 |
| M2 | Context-aware classification and unconditional resulting-state checks | MDK engine | P3 pinned | G2 |
| M3 | Exact-leaf removal and acknowledgement authentication | MDK engine/traits | M2, P4 pinned | G2 |
| M4 | Purpose-bound KeyPackages and durable enrollment storage | MDK engine/storage | P4 pinned | G2 |
| M5 | Atomic Welcome join, replay ack, maintenance gate | MDK engine/session | M3, M4 | G2 |
| M6 | Publication, reconciliation, branch and cleanup orchestration | MDK app/session | M1, M3–M5 | G3 |
| M7 | Pairing runtime, signer integration, one bounded carrier | MDK app/transport | P2/P5, M4; integrate after M6 | G3 |
| M8 | Typed bindings, group opt-in, device-specific client flows | MDK bindings + clients | M1–M7 | G4 |
| V1 | Full fault matrix, independent codecs and platforms | Conformance/security | Incremental from P2 | G3/G4 |
| R1 | Limited pilot, observation, rollback and documented release | Release + clients | G4 | G5 |

M1 and the general send-validation repair can start independently of #417. A spike may target an explicitly pinned
amended proposal before adoption, but it does not advertise production support or publish experimental protocol bytes
to normal user groups. Cross-package parallel work needs agreed interfaces from [the MDK plan](mdk-plan.md).

## Gates and review focus

| Gate | Evidence required to pass |
| --- | --- |
| G0: design ready | D1–D6 recorded, stale descriptions mapped, scope and residual risks acknowledged |
| G1: spec ready | Correct canonical bytes, complete transition/authority rules, fixtures and independent review; registry consistency |
| G2: engine ready | Send/ingest/replay parity, exact leaf authority, atomic persistence, no premature capability advertising |
| G3: integration ready | Durable publication/recovery, bounded pairing/carrier, two codecs, fault matrix, compatible routing and released-baseline interoperability |
| G4: pilot ready | Native bindings/platform journeys, tested client version matrix and coordinated upgrade path, truthful UI, narrow LAN policy if used, security review |
| G5: release ready | Pilot evidence, documented limitations, release/integration notes and safe rollback procedure |

Review focus maps to explicit cases in [validation](validation-and-rollout.md):

1. Successful join with lost ack versus failed join: no automatic destructive cleanup (E07–E10).
2. Losing branch revives after local state was discarded: no false enrollment completion (R01–R04).
3. Another leaf of the same account forges a plausible ack or a leaf index is reused: reject (A04–A07).
4. Approval/catalog reorder, signer delays, or a clock rollback: bounded, explicit outcomes (C02–C06).
5. Two devices sign out, receive invitations, or remove siblings concurrently: preserve unaffected devices (I01–I06).

## Completion and deferred work

Enrollment is complete for a group only in the locally observed, explicitly reported sense defined in P4. A successful
pilot requires selected devices to send and receive after enrollment, crash recovery without duplicate Adds, usable
per-group removal, and evidence for every gate. Finishing this plan does not mean finishing device-directory, history,
archive, remote-signer, or global-compromise recovery projects.

Deferred work has explicit owners and entry conditions in [validation and rollout](validation-and-rollout.md#deferred-work).
Revisit default enablement only after the pilot, not as a hidden fallback when pairing finds no eligible groups.

[pr417]: https://github.com/marmot-protocol/marmot/pull/417
[spec-feature]: https://github.com/marmot-protocol/marmot/blob/994ba06878d7093e7ba2cf7f6c34b2fe37bc043b/features/multi-device.md
[spec-component]: https://github.com/marmot-protocol/marmot/blob/994ba06878d7093e7ba2cf7f6c34b2fe37bc043b/app-components/same-account-membership-v1.md
