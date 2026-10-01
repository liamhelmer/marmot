# Discussion and evidence register

Status: non-normative research/coordination record, checked 2026-09-25. Refresh statuses before implementation. A historical
issue is evidence of a concern, not proof the current code still has the defect. Automated reviews are attributed as
reviews and do not establish maintainer consensus.

## Pinned sources

| Source | Revision / significance |
| --- | --- |
| Marmot master | `26fa6a6972d7b4325cb3d105ffdd41a1ceda2bb0`; also the plan's fork base |
| [Marmot #417](https://github.com/marmot-protocol/marmot/pull/417) | `994ba06878d7093e7ba2cf7f6c34b2fe37bc043b`; open, unmerged reviewed proposal |
| [MDK implementation inspected](https://github.com/marmot-protocol/mdk/tree/31a3722577805b49925142069f98037c9e75d6a2) | `31a3722577805b49925142069f98037c9e75d6a2`; file/line evidence in earlier review |
| MDK live master observed during review | `9d87fe1f1f7217c0e38e7bba2343220322cd78fc`; not substituted for the pinned code audit |
| [MDK v0.10.4 release](https://github.com/marmot-protocol/mdk/releases/tag/v0.10.4) | Released 2026-09-20; source `fcc85edd8dbd07c8293c899ee52230f72c54c897`. Stable compatibility baseline checked 2026-09-25; supporting multi-device client releases remain to be named at G4. |
| Local input reviews | `multi-device-path-forward.md`, `astra-path-forward.md`, `fable-path-forward.md`; findings incorporated here, not required checkout dependencies |

The released-baseline compatibility check inspected v0.10.4's
[authorization/ordering classifier](https://github.com/marmot-protocol/mdk/blob/fcc85edd8dbd07c8293c899ee52230f72c54c897/crates/cgka-engine/src/app_components.rs#L1410),
[Welcome capability/admin checks](https://github.com/marmot-protocol/mdk/blob/fcc85edd8dbd07c8293c899ee52230f72c54c897/crates/cgka-engine/src/group_lifecycle.rs#L1283),
and [account-wide sign-out cleanup](https://github.com/marmot-protocol/mdk/blob/fcc85edd8dbd07c8293c899ee52230f72c54c897/crates/marmot-app/src/runtime/mod.rs#L3698).
This was source inspection, not an interoperability test. The overview records the compatibility boundaries; K01–K05
provide the required future release evidence without replacing the earlier implementation audit.

## Current design agreement and unresolved objections

| Source | Observed status | Substance / action |
| --- | --- | --- |
| [#417 architecture review](https://github.com/marmot-protocol/marmot/pull/417#pullrequestreview-5016433734) | Aug. 25, earlier head | Supports separating enrollment, content and carrier; rejects mesh `(epoch,timestamp)` merge as MLS authority. Preserve that boundary in P1 and deferred sync. Recheck earlier defects against final amended head. |
| [#417 substantive objections](https://github.com/marmot-protocol/marmot/pull/417#pullrequestreview-5020898926) | Aug. 25, reviewed head | Requests Noise/authenticated-DH consideration or concrete platform evidence and restrictions; measurements or different cap; directory admission or invitation notice/fallback. Answer in D1–D3, not by citing an approval label. |
| [#417 subsequent automated review](https://github.com/marmot-protocol/marmot/pull/417#pullrequestreview-5025660654) | Aug. 26, same head | Calls the three product choices open; supports duplicate-Welcome, exact-leaf acknowledgement and drafts corrections. P2–P4 cover concrete amendments. |
| [#417 latest approval](https://github.com/marmot-protocol/marmot/pull/417#pullrequestreview-5121449101) | Approved Sept. 5, same head | Brief audit marker, no substantive resolution text. PR still unmerged; retain both the approval and earlier concerns accurately. |
| [MDK #1518](https://github.com/marmot-protocol/mdk/pull/1518) | Merged Aug. 23 | Architecture aligns, but baseline details differ: 1–4 Adds, separate capabilities, joiner QR, DH, Welcome fallback and losing-branch explanation. P1 reconciles the document rather than treating all details as agreed. |
| [MDK #1518 losing-Welcome review](https://github.com/marmot-protocol/mdk/pull/1518#discussion_r3838348933) | Aug. 23 review | Correctly notes sibling branches are selected, not merged; both can satisfy five-leaf limit. P4/R01–R04 define actual recovery. |
| [MDK #1199](https://github.com/marmot-protocol/mdk/pull/1199) | Merged July 30 | Earlier non-normative exploration; useful history, superseded where #1518/#417 differ. |

## Active and adjacent MDK work

| Source | Observed status | Action / scope distinction |
| --- | --- | --- |
| [#1275 interactive QR pairing](https://github.com/marmot-protocol/mdk/issues/1275) | Open | Canonical older design thread; External Commit/group-material transfer conflicts with current direction. Update its scope/reference in P1 when authorized, keeping useful UX/security questions. |
| [#1276 rotating session lifecycle](https://github.com/marmot-protocol/mdk/issues/1276) | Open | Reuse single-use, expiration, supersession, restart and bounded-concurrency concepts; do not inherit old payload or global-session assumptions without protocol support. |
| [#1282 session implementation closure](https://github.com/marmot-protocol/mdk/pull/1282#issuecomment-5207604718) | Closed unmerged Aug. 6 | Maintainer said “not now.” Source of reusable ideas, not an adopted prerequisite. |
| [#1282 clock review](https://github.com/marmot-protocol/mdk/pull/1282#issuecomment-5202214008) and [fix response](https://github.com/marmot-protocol/mdk/pull/1282#issuecomment-5202411653) | Aug. 6 | Wall-clock rollback extended bearer TTL; response added monotonic expiry. Carry regression C04 into M7. Rotation scheduling also needs explicit ownership. |
| [#1696 invitation loss](https://github.com/marmot-protocol/mdk/issues/1696) | Open | Phase 1 gives multi-slot evidence; Phase 2 assumes a directory and all-device initial admission absent from #417. D3/M1 preserve the distinction. |
| [#1696 production follow-up](https://github.com/marmot-protocol/mdk/issues/1696#issuecomment-5617620528) | Sept. 10 | Several missing invites included single-slot recipients. Multi-slot selection is a concrete risk, not a complete explanation of all invitation failures. Retain #1706 correlation work. |
| [#1697 device cleanup](https://github.com/marmot-protocol/mdk/issues/1697) | Open | M1 implements proven local-slot cleanup; directory-dependent orphan retirement and account-wide revocation remain separate. |
| [#1570 deletion retry after wipe](https://github.com/marmot-protocol/mdk/issues/1570) | Open | Persist exact signed public events outside destroyed account state; part of M1, not an optional best-effort retry. |
| [#1641 recipient Welcome outcomes](https://github.com/marmot-protocol/mdk/issues/1641) | Open | Reuse typed account-scoped ingest/catch-up outcomes in M1/M6/M8. A sender's successful Commit is not recipient join. |
| [#1706 per-recipient publication outcomes](https://github.com/marmot-protocol/mdk/issues/1706) | Open | Durable fanout/route/package evidence is separate from authenticated receipt; integrate M1/M6. |
| [#1910 compatible KeyPackage ranking](https://github.com/marmot-protocol/mdk/pull/1910) | Merged Sept. 18 | Preserve group compatibility, slot supersession and temporary client preference. It improves selection, not authenticated device discovery. |
| [#1277 history sync](https://github.com/marmot-protocol/mdk/issues/1277#issuecomment-5201017713) | Open; blocker investigation Aug. 6 | Missing per-device envelope/provenance and policy decisions support deferral. Replaying via ordinary group messages would broadcast duplicated history. |
| [#1769 account archives](https://github.com/marmot-protocol/mdk/issues/1769) | Open | Supports portable accepted data with fresh device state, atomic import and typed admission outcomes. No live database/ratchet clone. Separate from enrollment delivery. |
| [#828 NIP-46 signer](https://github.com/marmot-protocol/mdk/issues/828) | Open | Cross-device signer availability is related but separate. Existing signer abstractions suffice for supported v1 pairings; do not imply NIP-46 is already provided. |
| [#1531 import preflight](https://github.com/marmot-protocol/mdk/issues/1531) | Closed | Preserve fail-closed migration/consent boundaries while updating onboarding; a pairing offer does not make all legacy groups or histories portable. |
| [#1741 onboarding cancellation](https://github.com/marmot-protocol/mdk/issues/1741) | Closed | Useful precedent: cancel UI progress without forgetting already exposed publication. Apply that distinction to M6. |
| [#991 witness identity](https://github.com/marmot-protocol/mdk/issues/991), [#1115 tests](https://github.com/marmot-protocol/mdk/pull/1115) | Closed / merged | Reuse account-deduplication regression coverage; do not count leaves as separate witnesses. |
| [#1092 consumed packages](https://github.com/marmot-protocol/mdk/issues/1092), [#1093 self-update scheduling](https://github.com/marmot-protocol/mdk/issues/1093) | Closed | Existing maintenance/lifecycle behavior must integrate with purpose-bound packages and ack-first scheduling, not be reimplemented blindly. |

## Historical alternatives and regression evidence

| Source | Observed disposition | Relevant lesson |
| --- | --- | --- |
| [Marmot #40 client-id review](https://github.com/marmot-protocol/marmot/pull/40#issuecomment-3910451039) | Closed unmerged May 2 | Public client IDs leak correlation and leave ghost devices/revocation gaps. Supports rejecting blind slot fanout; future directory must solve these problems. |
| [Marmot #44 invite reachability](https://github.com/marmot-protocol/marmot/pull/44#issuecomment-3941828218) | Closed unmerged July 3 | Newest-device invitation loss predates #417; changing enrollment mechanics does not eliminate it. |
| [Marmot #44 compromise concern](https://github.com/marmot-protocol/marmot/pull/44#issuecomment-4074530529) and [response](https://github.com/marmot-protocol/marmot/pull/44#issuecomment-4075829845) | Historical discussion | Trying untrusted clients with production identities/groups expands compromise exposure. Opt-in helps admission choices but cannot make compromised authorized clients trustworthy. |
| [MDK #251 External Commit implementation](https://github.com/marmot-protocol/mdk/pull/251) | Closed unmerged | Do not merge old join/PSK/authorization paths; review only isolated crypto helper patterns if reused. |
| [Marmot Discussion #27](https://github.com/marmot-protocol/marmot/discussions/27#discussioncomment-15547751) | Only Discussion found; last substantive reply Jan. 20 | Parent references alone do not solve conflict/finality. Later irrelevant activity is not renewed design agreement. |
| [Marmot #389 group-event-key export](https://github.com/marmot-protocol/marmot/issues/389#issuecomment-5050738977) | Closed not planned July 22 | Old non-normative construction needed revisiting. Current no-key-transfer boundary avoids importing that design. |
| [Marmot #393 lifecycle](https://github.com/marmot-protocol/marmot/issues/393#issuecomment-5057880991) | Closed, addressed by #408 | Candidate-parent authority, resulting-state invariants and account/leaf separation remain core boundaries. |
| [Marmot #391 Welcome security](https://github.com/marmot-protocol/marmot/issues/391#issuecomment-5057880226) | Closed | Atomic Welcome handling and first-contact trust limits matter; do not generalize pairing approval into an unadopted mandatory ordinary-invite consent protocol. |
| [Marmot #237 removal evidence](https://github.com/marmot-protocol/marmot/issues/237), [#282 push cleanup](https://github.com/marmot-protocol/marmot/issues/282) | Closed July 23 | Regression requirements: leaf removal does not mean account disappearance or deletion of surviving sibling push state. |
| [Marmot #81 push-token collision](https://github.com/marmot-protocol/marmot/issues/81#issuecomment-4871800075) | Closed fixed June 29 | Existing canonical token key includes leaf index. Reuse sibling coverage; do not report the old defect as current. |
| [Marmot #380 retained secrets](https://github.com/marmot-protocol/marmot/issues/380#issuecomment-5038231095) | Closed July 23 | Authenticated retained-branch replay needs private retained material, not just public tree/Commit bytes. Preserve explicit retention/security tradeoffs. |

## Opus action-plan review disposition

Review supplied 2026-09-25 against the six planning files and pinned master above. These are corrections to the plan;
future implementation cases remain unchecked, and this change neither adopts protocol rules nor withdraws IDs on master.

| Item | Disposition and reason | Work / validation |
| --- | --- | --- |
| 1. QR-secret capture and package substitution | Accepted. PSK-only permits active record forgery, not just passive metadata recovery. Recommend authenticated DH and block unmodified PSK-only; a PSK alternative needs independently authenticated commitments under a session-authorized key the QR-only attacker lacks. | D1, P2, M7, W08; threat diagram 3 |
| 2. Reflected account proofs | Plan clarified; the pinned #417 already has distinct roles and descriptor/joiner-context bindings, so sharing an account key alone does not demonstrate reflection. Preserve exact template checks for both channel options and require staged full-transcript authentication. | D1/P5, M7, W04 |
| 3. Admin priority by shape | Accepted. Choose the highest independently valid authority so narrow admin membership operations need no padding. Admin authority is account-scoped: a compromised sibling of the same admin account also has privileged authority; no healthy-winner guarantee follows. | D6/P3, M2, U02/X04; diagrams 3/8 |
| 4. Welcome replay amplification | Accepted. Coalesce pending acks and persist at most one fresh recovery ack per attempt/branch/epoch, with finite exact-byte retries and no reset on restart or branch return. | P4, M5, E04/E12/B6 |
| 5. Kind 452 collision | Accepted. It remains the old local join-authorization proof kind and must never identify an MLS-carried ack. Allocate a fresh ack kind and retain 452 as withdrawn/reserved at adoption. | P5, registry/fixture consistency check |
| 6. Legacy draft still live | Accepted. Describe it as superseded by this plan, and explicitly schedule withdrawal of 0x800a, 452, 0xf2f0/0xf2ef and the associated External Commit/join-PSK/key-transfer flow in owners/registry. | P1/P5; overview constraints |
| 7. Late Welcome and undefined deadline | Accepted with the suggested exposure-only deadline. Five-minute default lifetime with at least 120 seconds remaining at staging/exposure; late admission remains possible while approval and secrets are live. Separate bounded secret retention covers the retry schedule; warn that post-receipt cancellation can strand a leaf. | P4, M4/M6/M8, E08/F02; diagram 6 |
| 8. SelfRemove after successful join | Accepted with existing authorization constraints. Non-admin joiners can use SelfRemove; active admins require the existing admin-policy prerequisites or an independently authorized removal. Never-joined refusal proofs cannot represent post-join departure. | D5/P4, M6, E13 |
| 9. Sponsor maintenance invalidates intent | Accepted. Hold routine own-leaf maintenance from intent creation to exposure/safe retirement; reconstruct after restart. Urgent rotation still proceeds with fresh approval for unexposed work. | M6, E14 |
| 10. Pre-approval buffer too small | Accepted. Budget 128 KiB including ciphertext/tag/framing/metadata and require the maximum encoded record to fit; approval must still progress when early buffering is full. | M7, C02 |
| 11. Welcome retries and retained history | Qualified fix. Stop retries when required catch-up material is unavailable and report why, without extending retention. An old Welcome does not itself preclude catch-up if a complete usable Commit chain remains retrievable; local anchor age alone is insufficient. | P4/M6, E15 |
| 12. Mixed observers in transition table | Accepted. Name each actor and distinguish local observations from authenticated cross-peer evidence. | P4 |
| 13. Empty non-final catalog case | Accepted. Explicitly reject it; the empty catalog is a sole final batch. | P2, W01 |

## Follow-up discipline

- [ ] Link the final amended proposal and this action plan from relevant trackers when their maintainers accept the scope.
  Merely writing this plan does not authorize posting updates or closing others' issues.
- [ ] Distinguish superseded designs, fixed regressions, active implementation work and unresolved product choices in each
  future PR description. Do not label all historical multi-device issues as blockers.
- [ ] Split directory/revocation assumptions out of #1696 Phase 2/#1697 where #417 is incorrectly expected to provide them.
- [ ] Keep history/archives and signer projects independent; link prerequisites without claiming enrollment delivers them.
- [ ] Refresh the evidence register at G0/G1 and before pilot release, with dates and source revisions.

Marmot #421/#422 (relay discovery migration) and #418/#419 (consensual purge) were also inspected. They are not established
enrollment prerequisites. Purge/retention semantics need fresh review when history/content synchronization is designed.
