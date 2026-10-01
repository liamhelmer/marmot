# Client flows, failure paths and security limits

Status: explanatory diagrams for the non-normative [action plan](README.md), 2026-09-25. They illustrate the proposed
amended behavior, not the unamended #417 implementation. The [protocol plan](protocol-plan.md) owns decisions and the
[validation matrix](validation-and-rollout.md) names the tests. Diagram arrows show messages or decisions, not guaranteed
delivery. Every durable boundary is recoverable after restart.

## 1. One account, independent leaves

An account is a Nostr identity. A leaf is one cryptographic membership in one MLS group, not an attestation that a
particular physical phone exists. A local device label is presentation, not authority. The intended architecture gives
each installation independent device signing material and a distinct leaf in every group it joins.

```mermaid
flowchart TD
    A["Alice account identity"]
    P["Alice phone: independent device state"]
    L["Alice laptop: independent device state"]
    G1["Group 1: phone leaf P1 and laptop leaf L1"]
    G2["Group 2: phone leaf P2 only"]
    B["Bob: different account and independent leaves"]
    A -->|"account proof for device key"| P
    A -->|"account proof for device key"| L
    P --> G1
    L --> G1
    P --> G2
    B --> G1
    B --> G2
    P -. "pairing controls; never copy live MLS state" .-> L
```

**Tradeoff:** per-group opt-in and independent keys preserve group boundaries, but pairing does not automatically enroll
the laptop in Group 2 or deliver historical Group 1 messages. The five-leaf proposal limits authenticated tree entries,
not the number of physical machines secretly holding a copied entry's keys. See diagram 5.

## 2. Happy path: pair, approve, add, join, acknowledge

Both Alice clients already have access to the same account signer. Alice's phone is already a member of the selected
group. The group has explicitly enabled the feature and every current leaf supports it. Bob represents other group
members who validate the Add; he does not approve Alice's ordinary same-account enrollment manually.

```mermaid
sequenceDiagram
    autonumber
    participant S as Alice phone / sponsor
    participant J as Alice laptop / joiner
    participant R as Redundant relay<br/>transport
    participant B as Bob / existing peer
    S->>J: Display signed expiring QR, laptop scans
    J->>J: Verify account, descriptor, proof and expiry
    J->>S: Hello with fresh nonce and account proof
    S->>S: Verify proof, obtain explicit pairing approval
    Note over S,J: D1 authenticates both contributions, derive channel keys
    S->>J: Full-transcript sponsor approval, then encrypted catalog
    J->>J: Verify approval before accepting catalog
    J->>S: Select eligible groups
    J->>J: Persist fresh private KeyPackage and enrollment<br/>purpose
    J->>S: Public KeyPackage for this group
    S->>J: Intent binds group, package, sponsor leaf and<br/>first-exposure deadline
    J->>J: Durably approve exact intent
    J->>S: Enrollment-intent receipt
    S->>S: Persist receipt and exact staged one-Add Commit
    S->>R: Publish Commit before applying locally
    R-->>S: Required publication evidence
    R->>B: Deliver candidate Commit
    B->>B: Validate parent, authority, proofs, path and<br/>resulting state
    S->>R: Deliver matching Welcome through normal<br/>transport
    R->>J: Welcome addressed to the fresh package
    J->>J: Validate purpose and intent, atomically join and<br/>retain ack duty
    Note over J,B: Joiner trusts sponsor-attested branch, peers validate Commit shape
    J->>R: First application payload is enrollment ack
    R-->>J: Required ack-publication evidence releases<br/>outbound gate
    R->>S: Deliver MLS-authenticated ack
    S->>S: Verify exact added leaf lineage and all attempt<br/>fields
    J->>R: Post-join self-update and ordinary traffic
    Note over S,J: Joined and acknowledged are local facts, not global finality
```

**Tradeoffs:** the receipt adds a round trip but prevents an honest sponsor from publishing before durable joiner
approval. Ack-before-update adds a durable publication dependency but makes initial sender attribution unambiguous.
The joiner waits for the defined publication boundary, not an unobservable acknowledgement-of-ack. Other peers can
temporarily see different branches; neither a relay's acknowledgement nor the enrollment ack certifies finality.

**Coverage:** W04/W07, E01/E06 and B1–B7. D1 determines the final channel construction; this sequence does not freeze it.

## 3. Tampered carrier, wrong identity or unauthorized Add

Different checks belong to different recipients. A carrier cannot authorize enrollment. Existing peers can validate an
Add against its parent; a new joiner checks the approved intent and sponsor-attested resulting branch instead.

```mermaid
flowchart TD
    X["Incoming pairing or membership input"] --> K{"Input surface?"}
    K -->|"QR or Hello"| P{"Exact account, proof, transcript and lifetime valid?"}
    P -->|"No"| R1["Reject before group catalog disclosure"]
    P -->|"Yes"| C["Establish authenticated channel"]
    K -->|"Encrypted record"| E{"AEAD, sequence, direction and bounds valid?"}
    E -->|"No"| R2["Reject or terminate; no enrollment side effect"]
    E -->|"Yes"| O{"Transcript-bound approval, package and durable receipt valid?"}
    O -->|"No"| R3["Bounded defer or refuse; honest sponsor sends no Add"]
    O -->|"Yes"| S["Stage only approved per-group work"]
    K -->|"Candidate Commit"| M{"MLS and all resulting-state invariants valid?"}
    M -->|"No"| R4["Existing peer rejects candidate"]
    M -->|"Yes"| I{"Independently valid admin membership authority?"}
    I -->|"Yes"| A2["Accept with privileged priority, including narrow shapes"]
    I -->|"No"| N{"Exact enabled same-account shape or existing narrow flow?"}
    N -->|"Yes"| A1["Apply ordinary authority and its priority"]
    N -->|"No"| R4
```

Examples: a wrong-account KeyPackage cannot use the sibling exception; a non-admin mixed Add/Remove does not qualify;
an admin may still perform a broader operation permitted by ordinary admin policy; no actor bypasses the enabled cap
or malformed component rejection. D1 must protect package/intent/receipt integrity even if the attacker captures the QR:
under unmodified PSK-only, that attacker can forge AEAD records and substitute an old same-account package whose private
key it holds. Local purpose metadata on the honest joiner does not detect that substitution at the sponsor. Authenticated
DH with both keys and the full transcript bound prevents this QR-only attack; it does not repair account-key compromise.
An attacker can still drop traffic, exhaust a bounded channel budget, or force timeout: confidentiality/authentication do
not guarantee availability.

**Sponsor trust limit:** an already authorized malicious sponsor can violate its own UI's consent ceremony or present
a sponsor-attested isolated branch. Other members reject invalid candidate-parent operations, but the joiner cannot
reconstruct all those checks from a Welcome. Receipts prove protocol facts to honest implementations; they do not
provide hardware attestation or make a compromised client honor human approval. Pairing both requires account authority
and trusts a current sponsor; the UI must not claim universal human-consent enforcement.

**Coverage:** W03–W08, U01–U10, C05/C07. Keep this distinction in the threat model and security review.

## 4. A different sibling tries to acknowledge someone else's enrollment

Sharing an account does not authorize a different leaf to complete a specific enrollment. This rule needs real MLS
sender provenance, not the account pubkey or a leaf number supplied inside the message.

```mermaid
sequenceDiagram
    participant S as Sponsor
    participant J as Enrolled laptop leaf L
    participant X as Another Alice leaf X
    X->>S: Ack with copied intent, package and Welcome<br/>fields
    S->>S: MLS authenticates sender as X, not L
    S-->>X: Reject completion, retain pending attempt
    J->>J: Self-update L along uninterrupted lineage
    S->>J: Byte-identical Welcome replay after lost ack
    J->>J: Claim durable recovery budget for attempt/branch/epoch
    J->>S: One recovery ack from authenticated<br/>descendant of L
    S->>S: Verify lineage, branch and every bound field
    Note over S,J: Later Remove/Add at the same index is not a descendant
```

**Tradeoff:** keeping branch-relative lineage and ack budgets costs storage and replay work. Comparing account IDs cannot
enforce this rule. Repeated identical Welcomes coalesce with pending work and cannot trigger another recovery ack in the
same attempt/branch/epoch, including after restart. Exact-byte retries remain finite; lost acks can still leave uncertainty.
A client holding an actual clone of L's secrets is a different problem: see diagram 5.

**Coverage:** A04–A07, E04/E12; M3 implements engine-side validation or durable authenticated provenance.

## 5. Two clients share one leaf's private state

Copying an account secret to another supported signer is not itself copying an MLS leaf. The prohibited operation here
is copying the device's MLS signer, tree/epoch state and sender ratchets so two installations act as the same membership.
Reusing a signature key in a proposed *new* Add can be rejected by uniqueness checks. Secretly cloning an *existing* leaf
does not produce an Add for peers to inspect.

```mermaid
sequenceDiagram
    participant H as Original client holding<br/>leaf L
    participant X as Clone holding the same<br/>leaf L secrets
    participant B as Honest peer Bob
    H-->>X: Out-of-protocol copy of live signer and MLS<br/>state
    Note over H,X: Unsupported operation, no new leaf or Add is visible
    H->>B: Authenticated message as L
    B->>B: Valid crypto identifies L, not physical machine<br/>H
    X->>B: Another authenticated message as L
    alt Generation or replay state conflicts
        B->>B: Enforce MLS replay and ratchet limits
        Note over H,B: Loss, divergence or rejection may appear, not proof of cloning
    else Clone uses a message valid for Bob's current state
        B->>B: May accept, sender is indistinguishable from L
    end
    Note over H,X: Both share compromise exposure and can exercise L authority
```

**What can be enforced:** MDK must refuse supported import/archive paths that resume a copied live device state, generate
fresh device keys on new installation, enforce local single-writer/file-lock discipline, reject duplicate new-leaf keys,
and preserve ordinary replay/ratchet limits. Local locks cannot stop a copied database on another machine. The protocol
cannot generally prove that two authenticated messages came from two physical devices when the credentials are identical.
An ack from the cloned enrolled leaf can authenticate as that leaf; the exact-leaf check only excludes *different* leaves.

**Cryptographic caveat:** MLS derives message keys from a sender's generation and includes a random four-byte reuse guard.
State cloning/rewind can repeat generations and break independent ratchet ownership; it is inaccurate to claim every
pair of cloned sends necessarily repeats the exact AEAD nonce. Random guards mitigate accidental nonce reuse, not shared
leaf impersonation or a coordinated clone. See [RFC 9420 section 6.3.1][rfc-encryption] and
[OpenMLS forward-secrecy guidance][openmls-fs].

**Recovery tradeoff:** once suspected, stop treating the shared leaf as a safely independent device. A healthy separate
sibling or authorized admin removes the shared leaf, which removes the membership used by *both* copies on the selected
branch. Enroll clean replacements with fresh device keys and new leaves. Do not promise selective removal of only one
clone, or that a self-update proves the clone gone. If no healthy authority remains, use explicit admin recovery where
available; otherwise v1 has no in-band recovery. Compromised account-signing authority needs separate account-wide work.

**Required new cases:** X01–X04 in validation. Test the limitation honestly: acceptance of a valid cloned sender is an
expected trust-boundary result, not a test that invents clone detection.

## 6. Lost acknowledgement versus cancelled first admission

The sponsor's observation is identical in both cases: it has an exposed Add and no valid ack. The safe next action is
therefore the same until authenticated reconciliation supplies more evidence.

```mermaid
sequenceDiagram
    participant S as Sponsor
    participant R as Delivery transport
    participant J as Joiner
    S->>R: Published Add and exact Welcome
    alt Approval still live and package secret retained
        R->>J: Welcome, even after sponsor exposure deadline
        J->>J: Durable successful join and ack obligation
        J->>R: Ack
        Note over R,S: Ack is lost or delayed
    else Joiner cancelled or retired required secret before joining
        R->>J: Delayed Welcome
        J->>J: Reject admission, retain refusal evidence
    end
    S->>S: Same observation: published but unacknowledged
    S->>R: Bounded byte-identical Welcome retries
    S->>S: Stop automatic retries, retain uncertainty
    S->>J: Later fresh authenticated pairing and<br/>reconciliation
    alt Joiner proves retained successful membership
        J->>S: Valid exact-lineage recovery ack
        S->>S: Record ack, re-evaluate selected membership
    else Original joiner durably refuses this exact attempt
        J->>S: Terminal proof by original joining key, bound to<br/>fresh session
        S->>S: Stage separate authorized exact-leaf cleanup
    else Joiner remains unavailable
        S->>S: Keep uncertain, no automatic destructive removal
    end
```

**Tradeoff:** conservative uncertainty may occupy a leaf slot and require user/admin recovery. Automatically deleting on
timeout would sometimes evict a successfully joined offline device. The deadline limits first sponsor exposure; delayed
delivery alone does not force refusal while approval and private material remain live. Cancelling after receipt can
still strand a leaf because publication may already have happened; warn before cancellation. Refusal races are serialized
against first join; cleanup is a new Commit, not rollback of history. A successfully joined client leaves through the
existing authorized departure flow, including SelfRemove's admin prerequisites, rather than proving never-joined refusal.
Welcome retries stop when required catch-up material is unavailable or the retry budget ends; that is not proof of failure.

Account authentication alone is not proof this is the original joiner. If full state loss destroyed the original joining
signer, retain uncertainty; the user may request a separately authorized leaf removal without claiming proven failure.

**Coverage:** E07–E11, E13/E15, B4/B5/B8. Reconciliation wire details are a P4/G1 deliverable, not assumed existing messages.

## 7. Concurrent enrollments and branch revival

Two valid Adds from the same parent create alternatives, not a merged set of new leaves. Selection can change while a
losing branch remains inside the core rollback horizon. The plan intentionally chooses conservative retry availability.

```mermaid
flowchart TD
    P["Common parent: account has four leaves"]
    A["Branch A: add laptop; five leaves"]
    B["Branch B: add tablet; five leaves"]
    SA["A selected now; B retained and still eligible"]
    RB["B enrollment marked not-selected; replacement Add blocked"]
    REV["B later wins; re-evaluate exact leaf and key availability"]
    KEYS{"Joiner still has usable B state?"}
    RESUME["Resume B membership and valid obligations"]
    STALE["Unusable retained leaf; no ack or false completion"]
    CLEAN["Authorized exact-leaf cleanup then explicit recovery"]
    OLD["B becomes permanently ineligible by core rules"]
    NEW["Fresh approval and package; enforce current cap and rejoin boundary"]
    P --> A
    P --> B
    A --> SA
    B --> SA
    SA --> RB
    RB --> REV
    REV --> KEYS
    KEYS -->|"Yes"| RESUME
    KEYS -->|"No"| STALE
    STALE --> CLEAN
    RB --> OLD
    OLD --> NEW
```

**Tradeoffs:** neither branch initially exceeds five, so rejecting the losing Welcome solely on leaf count is wrong.
Holding retry until permanent ineligibility can delay progress indefinitely in a quiet group. Local retirement does not
shorten the core horizon. The product reports retry-blocked or offers explicitly authorized reconciliation/removal; it
does not fabricate finality, merge branches, or race a replacement Add merely to make progress look complete.

**Coverage:** U07 and R01–R05. On actual selection, validate each candidate against its own parent and resulting state.

## 8. Removal authority and its limits

```mermaid
flowchart TD
    Q["Request to remove a device"] --> T{"What authority and target?"}
    T -->|"Non-admin same-account sibling; exact other leaf"| S["Narrow sibling Remove with path; ordinary priority"]
    T -->|"Active admin; independently authorized target"| A["Admin removal rules; privileged even for exact narrow shapes"]
    T -->|"Self via sibling operation"| X["Reject; use existing SelfRemove path"]
    T -->|"Other account without admin authority"| X2["Reject"]
    S --> C["Publish, validate and observe selected removal"]
    A --> C
    C --> L["Removal evidence for this group and branch"]
    L --> N["No claim of account-wide or irreversible revocation"]
    L --> K["Cloned target: both copies lose that shared membership"]
```

**Tradeoffs:** reciprocal sibling removal makes normal device management possible without waiting for an admin, but a
compromised current sibling can also remove healthy siblings. Admin membership authority keeps its priority without
padding the Commit. Because admin authority belongs to the account, a compromised sibling of an admin account has the
same priority; this change does not promise the healthy device wins. Authorization does not prove benign intent. Retained
competing branches can undermine apparent permanence under existing convergence. A device label, public publication
slot, or successful removal in one group is not global identity revocation. Keep those limitations visible in the UI.

**Coverage:** A01–A03, I06, X04 and R01–R05. Witnesses remain account-deduplicated even when an account has several leaves.

## Edge-case tradeoff summary

| Choice | Benefit | Cost / limit | User-visible consequence |
| --- | --- | --- | --- |
| Independent leaves | Distinct device authority and cryptographic state | More group leaves, updates and recovery work | Explicit per-group enrollment and removal |
| Exact-leaf ack | Different sibling cannot forge enrollment completion | Retained lineage and branch provenance | Joined, acked and currently selected may differ |
| No timeout deletion | Avoids evicting a joined offline device | Uncertain attempts can occupy slots | Reconcile or request authorized cleanup |
| Wait for permanent branch ineligibility | Avoids replacement enrollment colliding with revival | Quiet groups can remain retry-blocked | Honest blocked status; no finality timer |
| Authenticated DH recommended by D1 | QR capture alone cannot derive channel keys or substitute an offered package | Platform/transcript review; compromised accounts/endpoints remain trusted | Unmodified PSK-only cannot ship: active QR capture permits forged controls and package substitution; later recovery reveals recorded metadata |
| Five-leaf provisional cap | Predictable bound on current tree entries/account | Restricts legitimate fleets; does not count hidden clones | Explicit cap error and remove-before-add flow |
| One-device first invitation | Avoids unauthenticated slot fanout | Selected installation may be unreachable | Risk notice; explicit reinvite recovery |
| LAN carrier | Direct rendezvous without a dedicated relay channel | Permissions, reachability and scoped destination-policy work | Supported-platform declaration and permission errors |
| Sponsor-attested Welcome | Standard assisted MLS join | Joiner cannot prove global branch agreement or all parent rules | Do not equate local join with global finality |
| No clone detector claim | Honest statement of cryptographic identity | Copied credentials can impersonate one leaf | Refuse supported cloning; remove shared leaf and re-enroll clean devices |

[rfc-encryption]: https://www.rfc-editor.org/rfc/rfc9420.html#section-6.3.1
[openmls-fs]: https://book.openmls.tech/forward_secrecy.html
