# Validation, pilot and rollout

Status: non-normative action plan, 2026-09-25. Cases below are required future evidence, not tests executed by committing
this plan. Owners and dependencies are in the [overview](README.md); protocol and implementation contracts are in
[P1–P5](protocol-plan.md) and [M1–M8](mdk-plan.md).

## Test organization

Create focused engine integration tests for same-account membership, enrollment Welcome processing and exact-leaf
acknowledgements; persistent storage tests for the ledger; pairing codec/session tests in the runtime; and application
journeys in the conformance simulator. Suggested new integration targets are `same_account_membership`,
`same_account_enrollment`, and `pairing_session`, with case IDs below retained in test names or documentation.

Extend the existing account/device topology and `adversarial-reliability/multi-device-account/v1` family. Its current
scenario proves multiple same-account leaves can exchange traffic; it does not establish pairing, enrollment authority,
or recovery. Use deterministic in-memory delivery with real MLS crypto and separate account/device keys. Use actual
persistent backend reopen for crash claims. Do not turn all devices into distinct accounts in the harness.

Keep state-machine cases below the runtime independent of wall time. For runtime timing overrides, follow the simulator's
`AGENTS.md`: enable `test-policy-overrides` only for explicitly selected tests; production-timing claims require pinned
production timing. A simulator case passing under instant settlement is not evidence for native deadlines.

## Canonical bytes and channel cases

| ID | Stimulus | Required result | Owner |
| --- | --- | --- | --- |
| W01 | Catalog/selection with 0, 1, 32 and 33 entries; empty final and non-final catalog; package/intent batches with 0, 1, 32 and 33 | Empty catalog accepted only as its sole final batch; empty non-final catalog, empty package/intent and all 33-entry batches rejected; empty final selection accepted | P2/M7 |
| W02 | Minimum valid intent, non-shortest prefix, truncated field, trailing bytes, integer overflow | Valid intent fits corrected vector; malformed encodings reject identically | P2/M7 |
| W03 | Duplicate group/batch/hash, noncatalog selection, package for another group | Reject or apply exactly the defined idempotency rule; never silently change approved meaning | P2/M7 |
| W04 | Correct descriptor, Hello, proofs and full-transcript approval; swap sponsor/joiner proofs or alter role/account/session/version/nonce or DH key in each supported channel option | Match byte fixtures; reflected/altered binding rejects at its verification stage before protected controls are accepted | P2/M7 |
| W05 | Alter AAD length/direction/flags/type, tag or ciphertext | Authentication/canonical validation fails; no allocation or application side effect from unauthenticated input | P2/M7 |
| W06 | Chunk reordering, identical overlap, conflicting overlap, missing bytes and oversized object | Bounded assembly; exact duplicates harmless; conflict/overflow rejects; incomplete object never decoded | M7 |
| W07 | Receipt before durable approval, foreign-session hash, duplicate receipt, 33 hashes | No unauthorized Add; valid repeated receipt is idempotent; bounds enforce | P2/M4/M6 |
| W08 | QR-capture attacker relays honest Hello, substitutes an old valid same-account KeyPackage with known private key, or forges approval/intent/receipt | No substituted Add or forged approval; unmodified PSK-only fails this case and cannot pass D1/G1; account/endpoint compromise documented separately | D1/P2/M7 |
| C01 | Same sequence and same bytes; same sequence different bytes; sequence exhaustion | Ignore duplicate without repeating effects; conflict/exhaustion terminates; never reuse nonce | M7 |
| C02 | One maximum 65,536-byte plaintext catalog record plus tag/framing/metadata arrives before approval; then approval or no approval; multiple records overflow | One maximum encoded record fits the 128 KiB budget and processes once after verified approval; approval can progress when buffering is full; overflow/timeout releases memory with no Add | P2/M7 |
| C03 | Lost receipt, duplicate object retransmission, channel disconnect during a batch | Exact retry or fresh authenticated reconciliation; no duplicate Add | M6/M7 |
| C04 | Wall time moves backward while monotonic elapsed TTL passes | Session expires; signed/display timestamp does not extend lifetime | M7 |
| C05 | Signer refuses, returns altered template/wrong author, replies after expiry or after account switch | Typed refusal/invalid/stale result; no approval, leaked catalog or cross-account mutation | M7 |
| C06 | Restart, QR rotation, supersession, background/suspend and attempted old-session replay | Old channel terminal; fresh secrets/nonces and explicit durable reconciliation | M6/M7 |
| C07 | Four partial objects, aggregate budget overflow, replay budget exhaustion, 16 MiB sequential objects | Enforced local bounds; typed channel closure; durable enrollment obligations survive | M7 |
| C08 | LAN DNS rebinding, disallowed address/interface, metadata endpoint, permission refusal, malicious redirect | No unauthorized dial; permitted address pinned; no shared relay/media safety relaxation | D4/M7 |

## Authority and capability cases

| ID | Stimulus | Required result | Owner |
| --- | --- | --- | --- |
| U01 | Enabled parent, non-admin valid one-Add with required path | Accept, ordinary priority; exact same-account proof validated | P3/M2 |
| U02 | Admin narrow Add/Remove also passes baseline admin rules; same authorized removal expressed by reference or with another baseline-valid operation | All receive privileged priority; no padding needed; same result on send/ingest/replay; self-update/SelfRemove retain dedicated rules | P3/M2 |
| U03 | Non-admin wrong account, referenced Add/Remove or mixed operation | Reject same-account authority; no broad fallback | P3/M2 |
| U04 | Admin removes own sibling and another account's leaf in a baseline-valid mixed removal | Ordinary admin authority accepted with privileged priority | P3/M2 |
| U05 | Missing path; duplicate key/package; duplicate/self/noncurrent removal target | Reject consistently in send, ingest and replay | M2/M3 |
| U06 | Enable with unsupported leaf, including unsupported founder/invitee | Reject enablement/admission; do not evict existing member automatically | M2 |
| U07 | Enabled account at five leaves, sixth Add by admin or sibling | Reject resulting state; concurrent branches are selected, not merged into ten leaves | M2/M6 |
| U08 | Disable with two leaves/account versus after valid sibling removal | First rejects; second passes if every other invariant holds | M2 |
| U09 | `0x800d` dictionary data in each forbidden location | Reject all; capability marker is never a data payload | M2 |
| U10 | Feature absent and ordinary valid admin/self-update/SelfRemove operations | Preserve baseline behavior; disabled feature does not authorize a non-admin enrollment | M2 |
| A01 | Same-account non-admin removal of 1–4 siblings | Accept with ordinary priority; preserve committer and other accounts | M3 |
| A02 | Admin removes one device versus an entire account | Exact requested scope; preserve ordinary admin/last-admin constraints | M3/M8 |
| A03 | Stale target followed by Remove/Add reuse of its index | Reject/reconcile target; never delete replacement occupant | M3 |
| A04 | Another sibling sends correct-looking enrollment ack fields | Reject as wrong authenticated leaf, despite same account | M3/M5 |
| A05 | Enrolled leaf self-updates and responds to duplicate Welcome | Accept only uninterrupted authenticated lineage on applicable branch | M3/M5 |
| A06 | Removed leaf, discarded group or invalidated record produces replay ack | No valid ack/completion | M3/M5 |
| A07 | Same index/account returns through Remove/Add, or ack replayed on another branch | Reject old lineage/correlation; do not conflate index with device | M3/M5 |
| A08 | Two leaves of one account send witnesses on competing-branch test | Count one account identity; retain #991/#1115 regression | M2/V1 |

## Enrollment and recovery cases

| ID | Stimulus | Required result | Owner |
| --- | --- | --- | --- |
| E01 | Valid approved attempt and matching first Welcome | Atomically join and persist ack obligation; no prior chat/self-update | M4/M5 |
| E02 | Admin sponsor, cancelled/superseded attempt or retired package secret; separately exposure deadline alone has passed | Terminal attempt rejects; still-approved late Welcome follows E08; neither case falls back to ordinary admin admission | M4/M5 |
| E03 | Wrong sponsor leaf key, group, account, package, component or proof | Reject without consuming valid package or mutating group | M5 |
| E04 | Valid late duplicate after successful join and package consumption | Recognize exact bytes before dedup; coalesce pending work or claim one recovery ack per attempt/branch/epoch without reprocessing MLS | M5 |
| E05 | Same package reference with changed Welcome/GroupInfo bytes | Reject; preserve legitimate joined record and state | M5 |
| E06 | Post-join maintenance immediately due; ack publish fails | Retain ack obligation and own outbound gate; ingress/backfill remain live | M5 |
| E07 | Join succeeds but all acks are lost | Sponsor retains published-unacknowledged; no automatic Remove/Add | M6 |
| E08 | First Welcome after exposure deadline, approval uncancelled and secret retained; contrast cancellation/secret retirement before delivery | First joins and records ack; second rejects with retained terminal metadata; absent ack never proves either outcome to sponsor | P4/M5/M6 |
| E09 | Fresh authenticated reconciliation proves durable refusal/key loss | Exact attempt becomes terminal; authorized cleanup is separate and branch-relative | P4/M6 |
| E10 | First-join transaction races cancellation/refusal and duplicate delivery | Exactly one durable admission outcome; no contradictory success/refusal facts | M4–M6 |
| E11 | Different same-account device claims refusal; original joiner lost signer; old refusal replayed in new session | No terminal inference without original-joiner key proof bound to fresh challenge; uncertainty or separate explicit removal | P4/M6 |
| E12 | Many byte-identical Welcomes across relays before/after initial ack, after recovery ack, after restart and branch reselection; then a new epoch | One pending obligation coalesces duplicates; at most one fresh recovery ack per attempt/branch/epoch with finite exact-byte retries; budget survives restart/return; new epoch requires valid lineage | P4/M5 |
| E13 | Joined non-admin cancels, then joined active-admin cancels; ack still pending or already published | Preserve join fact; non-admin uses SelfRemove without refusal exchange; admin follows existing prerequisites or authorized removal; Leaving blocks ack traffic under existing rules | P4/M6 |
| E14 | Routine sponsor self-update becomes due between intent creation, approval, receipt and first exposure; restart; urgent rotation changes key | Routine maintenance waits until exposure/safe retirement; hold survives restart and expires with unexposed intent; urgent/key-changing update requires fresh approval | M6 |
| E15 | Group advances beyond Welcome catch-up availability before retry; contrast a complete usable chain still retrievable beyond local anchor and unknown availability | Missing required material stops with catch-up-unavailable; ceiling stops with retry-budget-exhausted; unknown availability stays uncertain; no retention extension, removal or replacement inference | P4/M6 |
| R01 | Enrollment loses one convergence pass, remains inside horizon | Not-selected/retry-blocked, not permanently failed; preserve exact recovery facts | M6 |
| R02 | Losing enrollment later revives with keys retained | Re-evaluate exact membership/ack obligations; no duplicate leaf | M6 |
| R03 | Previously retired/discarded enrollment branch revives without keys | Report unusable stale leaf, no false completion; explicit authorized cleanup/rejoin | M6 |
| R04 | Old branch becomes permanently ineligible; group still retained locally | Fresh package/approval permitted only with explicit discard/rejoin boundary | M6 |
| R05 | Concurrent attempts near cap, sponsor removal, competing cleanup and Add | Respect per-group single-attempt policy and candidate-parent invariants; report no-authority/uncertainty | M2/M6 |
| R06 | One group succeeds, another fails, pairing expires and app restarts | Keep independent results; no cross-group rollback; fresh session reconciles unfinished work | M6/M7 |

## Durable boundary matrix

At each boundary, inject a crash immediately before and after the durable write and immediately before and after the
external side effect. Reopen persistent storage, repeat the input, and compare the visible result and exposed bytes.

| ID | Boundary | Evidence after reopen |
| --- | --- | --- |
| B1 | Private KeyPackage plus purpose persisted before public exposure | Both exist or no public package was exposed; missing purpose fails closed |
| B2 | Approved intent before receipt emission | Receipt never attests to lost approval; repeated receipt is harmless |
| B3 | Sponsor records receipt and staged exact Commit before publication | No Add without receipt; one durable attempt and exact retry bytes |
| B4 | First exposure / partial fanout / ambiguous acknowledgement | No newly generated replacement Add; exposure is not forgotten |
| B5 | Welcome validation, package consumption and group persistence | Invalid join changes nothing; valid join retains exact digests and ack obligation |
| B6 | Ack publication, recovery-budget claim with exact bytes, and own-maintenance gate release | No premature self-update/chat; exact ambiguous retry remains possible within finite budget; duplicate/restart never resets recovery allowance |
| B7 | Verified sponsor ack and selected-membership update | Completion is durable, but later branch loss remains representable |
| B8 | Terminal refusal, cleanup exposure, secret erasure and safe metadata compaction | Refusal cannot later join; cleanup scope persists; erased state does not authorize fallback/replay |

## Shared-leaf and malicious-client cases

These cases test both enforcement and limits from [diagram 5](flows-and-threats.md#5-two-clients-share-one-leafs-private-state).
They must not assert that physical-device cloning can always be detected from MLS messages.

| ID | Stimulus | Required result | Owner |
| --- | --- | --- | --- |
| X01 | Supported import/restore attempts to activate copied live MLS state as a second device | Refuse that import mode; create fresh device keys/state and require normal admission | M4/M8 |
| X02 | Attempt a second local writer; separately run an adversarial copy on another machine | Local lock prevents supported concurrent local access; documentation/test shows it is not a remote clone detector | M4/V1 |
| X03 | Two adversarial clients share one leaf; send both conflicting-generation and otherwise-valid messages/acks | Ordinary MLS replay/ratchet checks remain; valid cloned-leaf traffic may authenticate as that leaf; do not invent physical attribution | M3/V1 |
| X04 | Healthy sibling/admin removes compromised shared leaf while attacker races a sibling Remove; then clean clients enroll after selected removal | Highest independently valid authority sets priority without padding; same admin-account siblings both have admin authority, so no guaranteed healthy winner; selected removal removes both clones; replacements use fresh keys | M3/M6/V1 |

Reject a duplicate signature key in a proposed new leaf independently of X03: cloning an existing leaf has no Add event
to validate. Do not claim that every clone send repeats the exact nonce; retain MLS reuse-guard behavior while testing
shared generation/key ownership and replay conflicts. A compromised sponsor can also ignore local UI approval; test
peer-side rejection separately from joiner reliance on sponsor-attested state, as explained in diagram 3.

## Runtime, bindings and existing-feature regressions

| ID | Stimulus | Required result |
| --- | --- | --- |
| I01 | Sign out A while B has a distinct live public slot | Delete only A's proven publication; B remains retrievable/usable |
| I02 | All deletion targets fail, then wipe/restart/restore connectivity | Exact signed public deletions survive independently and retry without signer |
| I03 | Same-slot replacements versus multiple distinct/unproven slots | Collapse replacements; expose risk for multiple slots; never delete while resolving |
| I04 | Wrong/unreachable initial device chosen; one or many compatible packages | Keep single admission, truthful risk and exact retry; no blind fanout |
| I05 | Welcome on subset of inbox relays; canonical invite returns before fanout | Durable per-recipient transport outcomes; background delivery proceeds |
| I06 | Remove A's leaf while B retains membership/push registration | Correct A departure evidence; B's notifications and group remain usable |
| F01 | Read status after restart or late subscription | Stable attempt reference and all durable facts/allowed actions reconstructed |
| F02 | Cancel before exposure, after receipt/ambiguous exposure and after join | Warn that post-receipt cancellation may strand a cap-consuming leaf; finite typed result, retained reconciliation, authorized departure after join, no false rollback |
| F03 | Pair into mixed enabled/disabled/unsupported groups | Per-group eligibility/partial results; no automatic feature enablement or eviction |
| F04 | Joined device opens chat and enrollment ack arrives | Ack hidden from timeline/unread/notifications; history absence explained |
| F05 | Swift/Kotlin/C calls remove-device and remove-account | Distinct scope and stale-target result survive binding conversion |
| F06 | No local account signer, external signer unavailable, permission denied or QR expired | Actionable typed outcome; no secret transfer or infinite spinner |

## Verification commands

For this documentation-only plan: run `git diff --check`, verify all new relative links, and review the repository's
required boundary scans. No Rust build or product behavior test is needed to validate a planning-document change.

```sh
git diff --check
rg -n 'Rust crate|database table|queue shape|retry worker|local API|test harness|\bengine\b|darkmatter' .
rg -n 'CgkaEngine|PendingStateRef|drain_auto_publish|drain_auto_proposals|confirm_published|publish_failed' .
rg -n 'Nostr kind|event id|relay URL|gift wrap|h tags?' protocol-core
```

Inspect matches; implementation terms in this explicitly non-normative plan are intentional. Future normative changes
must obey the owning surface's placement rules. Confirm all new protocol IDs in registry/indexes and all fixture links.

For implementation, run commands from MDK with the relevant named tests created by the owning package. New target names
below are planned, and must exist before these commands can pass. A filtered command that runs zero tests is not evidence.

```sh
cargo test -p cgka-engine --test same_account_membership
cargo test -p cgka-engine --test same_account_enrollment
cargo test -p cgka-conformance-simulator --test same_account_enrollment
cargo test -p marmot-app pairing
cargo test -p storage-sqlite enrollment
cargo test -p marmot-uniffi enrollment
cargo test -p marmot-c enrollment
cargo test -p cgka-conformance-simulator canonical_vector_fixtures_match_generated_traces
just binding-docs-gate
just fast-ci
```

Use package-appropriate existing capability, storage and publication suites in addition to the new targets. Enable
`test-policy-overrides` only on explicit runtime tests that need it. Run full `just ci` in implementation CI according to
MDK policy. Attach test counts, fixture revision, compiler/dependency revision, failed/retried tests and platform evidence
to each implementation PR. Do not describe isolated reruns as a clean first full-suite run.

## R1: pilot and release

- [ ] Run an opt-in pilot only after G4. Pin Marmot/MDK/client revisions, channel/carrier version, component support and
  allowed platform combinations. Keep unsupported pairs disabled with a clear outcome.
- [ ] Demonstrate two devices of one account and another observing account: enroll selected groups, send both directions,
  restart each side at durable boundaries, lose/recover acks, remove one sibling, and keep the other account unaffected.
- [ ] Exercise empty catalog, more than 32 groups in batches, cap overflow, old client in group, partial success, no
  surviving sponsor, relay outage and native background interruption. Do not substitute simulated carrier success for
  actual mobile permission/network behavior.
- [ ] Collect privacy-safe counts/latencies for attempts, uncertain publication, verified ack, retry-blocked, cleanup,
  timeouts/resource closures and failure categories. Keep any diagnostic correlation opt-in and protected; do not log IDs.
- [ ] Security-review the implemented channel, exact-leaf authority, metadata retention, sponsor trust, refusal races and
  LAN permission boundary. Check tests cover all findings, including inherited branch-relative revocation and reciprocal
  sibling denial of service.
- [ ] Stop new enrollment on any unauthorized ack/removal, state-cloning behavior, purpose downgrade, nonce reuse,
  crash-induced duplicate Add or resource exhaustion escaping the budget. Preserve existing valid membership and recovery
  obligations while investigating; do not erase evidence or automatically disable a multi-leaf group's component.
- [ ] Rollback disables new pairing/enablement in the product while retaining receive/replay/removal support required by
  existing groups. Never withdraw advertised support or downgrade a database while an enabled group needs it. Protocol
  disablement requires the P3 guard; binary downgrade is not a safe substitute.
- [ ] Publish release and integration notes, source/fixture pins, supported platforms, migration and remaining limitations.
  Revisit default enablement only after measured pilot results and a separate decision. No automatic merge or release.

## Deferred work

| Project | Prerequisites / boundary |
| --- | --- |
| Authenticated device directory and all-device first contact | Device/slot authentication, revocation, replacement, expiry, privacy and deterministic cap overflow; resolve #1696 Phase 2 explicitly |
| Account-wide invitation notice/fallback | Defined authenticated notice, privacy and replay model; cannot silently cause alternate-device Adds |
| Account-wide compromise recovery | Authority rotation and cross-group evidence; no global completion claim from this per-group feature |
| Later channel versions | D1 already recommends authenticated DH for initial adoption; any later incompatible construction needs a new version, full-transcript authentication, platform evidence and independent review |
| History/content bootstrap (#1277) | Separate forward-secret transfer envelope, per-group consent, provenance, count/time/byte bounds, retention and deletion semantics; no MLS state import |
| Offline account archive (#1769) | Versioned encrypted portable data, authenticated atomic staging/import and fresh device state; unsupported groups remain history-only/reinvite-required |
| Continuous sync/journals/mesh | Application records only; no `(epoch,timestamp)` merge becomes MLS authority; preserve exporter provenance and erasure policy |
| NIP-46 remote signer (#828) | Independent signer session/permission/restart design; enrollment can use supported existing signer abstractions without implementing NIP-46 |
| Stronger revocation/finality | Separate core protocol work with compatibility/security analysis; no wall-clock/tombstone patch hidden in enrollment |

## Review coverage checklist

- [ ] Original findings covered: vector byte/count error; inherited branch-relative removal; one-Add rationale; no admin
  fallback after intent expiry; PSK transcript/metadata limits; parent-aware classifier; disable guard; send-side checks;
  separate capability advertisement; exact-leaf recovery; non-last-resort packages; receipt ordering; duplicate Welcome;
  ack-before-maintenance; invitation observability limits; carrier specification.
- [ ] Original amendments covered: acknowledgement lineage; ordinary admin removal; account-sync drafts contradiction;
  published-unacknowledged behavior; proof/ack fixtures; bounded pairing memory; no state copying or mesh authority.
- [ ] Subsequent review covered: positive terminality evidence; branch revival after retirement; exact-leaf removal API;
  authenticated ack provenance; earlier approval ordering; admin precedence; scoped LAN policy; incomplete reachability.
- [ ] Discussion objections covered: PSK evidence, cap measurements, invitation notice/directory alternative; stale MDK
  baseline and open issues; closed #1282/#251 status; cleanup/outcome companion trackers; deferred history/archive work.
- [ ] Opus review items 1–13 covered: active QR substitution (W08/D1); role reflection (W04); authority priority (U02/X04);
  replay amplification (E12/B6); fresh ack kind and explicit legacy withdrawal (P1/P5); late admission/cancellation (E08/F02);
  post-join SelfRemove with admin constraints (E13); sponsor maintenance hold (E14); maximum early record (C02);
  useful Welcome retry bound (E15); actor-local transition rows (P4); empty non-final catalog rejection (W01).
