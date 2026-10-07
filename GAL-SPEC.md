# GAL — Grant & Autonomy Lifecycle, Specification

| | |
|---|---|
| **Version** | `0.6.0-draft` |
| **Status** | Draft for Linux Foundation agent-standards discussion. Wire schemas may change before 1.0; see Open Problems and Future Extensions. |
| **Date** | 2026-10-07 |
| **Working group** | LF Edge + Agentic AI Foundation (AAIF) |
| **Author** | Wes Jackson (Red Hat) |
| **Copyright** | © 2026 Red Hat, Inc. |
| **License** | Community Specification License 1.0 · `SPDX-License-Identifier: Community-Spec-1.0`. See `LICENSE.md` for terms and `NOTICE.md` for attribution, acceptance, and patent exclusions. |
| **Sibling specification** | `PTC-SPEC.md`, *PTC: Provenance & Trust Context, Specification* (drafted in parallel). This document cross-references PTC by name and section topic, never by section number. |

GAL specifies the autonomy layer of an agentic control system: an agent's authority to act is
a per-`(principal, action-class)` **grant** whose autonomy level is stored, signed state:
moved **upward** only by a maker≠checker ceremony licensed by a deterministic predicate over
measured evidence, moved **downward** automatically by deterministic demotion triggers, and
enforced per call by a broker the agent cannot write past. Levels-of-autonomy taxonomies name
the rungs; GAL makes the rung itself a signed runtime object that must earn its level and
demotes itself.

---

## 1. Scope and non-goals

### 1.1 In scope

The **Grant** object (§5.1); the **autonomy ladder**: four rungs, three grant levels, and the
actuation discriminator between them (§4.1); the **promotion ceremony**: the only sanctioned
upward path, a deterministic licensing predicate composed with maker≠checker ratification
(§6.4); **automatic demotion**: a closed set of deterministic triggers, the fall-back level,
and hysteresis (§6.7); **record signing**: every level change as a signed, append-only ledger
record (§6.10); the **audit obligation** (§6.11); and the **PTC join**: the
provenance-maturity ceiling on acting rungs (§6.13).

### 1.2 Non-goals (normative exclusions)

- **The per-agent safe-default polarity.** Whether "safe" for a given agent means abstention
  (abstain-is-safe) or a positive deterministic action (a SCRAM/failsafe) MUST be re-derived
  per agent and MUST NOT be specified by this standard or baked into any conforming base
  implementation. A base-level polarity default is a latent safety bug: the polarity that
  protects one domain harms the opposite one. This load-bearing exclusion recurs in
  `lastSafeLevel` (§5.1) and in demotion behavior (§6.7).
- **Domain thresholds.** Every numeric bar and budget referenced by this specification
  (promotion error-rate thresholds, minimum observation counts, error-budget tolerances,
  dwell times) is a deployment knob. Conforming implementations MUST ship such knobs OFF or
  unset by default; an unset knob is no gate. The floor this standard defines is the
  *mechanism* and the ratchet's asymmetry, never a domain number.
- **Policy language.** The expression language of the licensing predicate and of the per-call
  gate is out of scope; the sibling PTC specification records the evaluated-and-declined
  decision on general-purpose policy languages (a differential spike over the full reachable
  input space of the reference gate), and GAL's promotion predicate inherits that decision.
- **Conveying authority in a token.** GAL stores authority in the grant store, changed only
  through signed ledger records (§6.10), rather than conveying it in a delegation token; the
  sibling PTC specification's signed envelope conveys what a receiver needs to verify provenance
  offline. Deployments that need thresholds to travel with the delegation are served by
  delegation specifications, and the two compose: GAL governs the grant lifecycle, the
  delegation chain governs conveyance.

### 1.3 Future extensions (out of scope for this version)

- **Delegation path resolution.** Derivation splits into two halves and only one of them is this
  standard's. *When a path ceases to be valid* is lifecycle, which this standard owns and now
  states: **GAL-39** carries §6.7's rule forward onto derivation, so a derived grant can neither
  outlive nor out-rank the grant it derives from and a demotion at any node stops every path
  through it. That clause is normative and marked NOT YET IMPLEMENTED (§3), because the reference
  implementation has no derived-grant concept yet: a grant names one principal and is checked at
  the enforcement point, so no call there arrives by a path. The marker is the honest way to say
  that; withholding the clause until the code caught up would have understated what this
  specification requires, which is the opposite of what a conformance target is for.

  What remains out of scope is resolving *which* path applies where several reach the same agent.
  That is a property of how authority was conveyed, it belongs to the delegation mechanism, and
  deployments needing it are served by the delegation specifications (§1.2).
- **N>1 quorum ratification and owner-key rotation.** This draft is honest about the N=1
  operator case (§6.4.3, §8.3); quorum ratification and ratifier-key rotation
  mid-evidence-window are tracked for a future version
  ([ptc-gal-reference#33](https://github.com/wjatx/ptc-gal-reference/issues/33)).
- **Evidence weighting for tainted-turn outcomes.** Whether provenance-tainted-turn outcomes
  count toward promotion evidence, at full or discounted weight, is an information-flow-hard
  open problem (§8.1) and is not specified in this version.

---

## 2. Terminology and conformance language

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT",
"RECOMMENDED", "NOT RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted
as described in RFC 2119 [RFC2119] and RFC 8174 [RFC8174] when, and only when, they appear in
all capitals.

| Term | Definition |
|---|---|
| **principal** | The identity tuple an agent acts as (agent identity, skill, user, trust tier). The subject of a grant. |
| **action class** | A class of tool operations sharing a risk shape, derived from declared operation fields (§6.5). The object of a grant. |
| **grant** | The stored, integrity-protected record binding one `(principal, action class)` pair to an autonomy level and its lifecycle state (§5.1). |
| **level / rung** | A rung is a position on the four-position autonomy ladder (§4.1). A level is the stored `Grant.level` value; the level enumeration is complete at the three *acting* rungs. The bottom rung, Recommend, is the absence of a grant and is never a level value. |
| **lastSafeLevel** | The rung automatic demotion falls back to. Always `in-loop` or `on-loop`; never `out-of-loop`. What the safe rung *does* (abstain vs positive safe action) is the per-agent polarity, excluded from this standard (§1.2). |
| **maker** | The identity that proposes a promotion, citing evidence. |
| **checker** | The identity that ratifies a proposed promotion. MUST be a different credential from the maker (§6.4.3). |
| **ceremony** | The recorded maker≠checker act that mutates the grant store: seed, re-seed (re-attestation), propose, ratify, reject, tighten. The grant store's only sanctioned mutation path apart from automatic demotion. |
| **evidence window** | The bounded, period-bucketed span of scoped counters and evidence artifacts a promotion proposal cites. Its span and period granularity are bound into the proposal the checker ratifies (§6.4.2). |
| **demotion trigger** | One of a closed set of deterministic conditions whose firing lowers a grant's level automatically, with no model and no human in the path (§4.2). |
| **high-blast** | The action-class property derived as: effect is a write ∧ the operation crosses an external trust boundary ∧ the operation is not reversible. High-blast classes carry the strictest ceremony requirements (§6.4.4, §6.5). |
| **broker** | The enforcement point: the deterministic mediator that is the agent's only egress, reads the grant per call, and cannot be written past by the agent. GAL's grant enforcer role (§7.1). |
| **envelope** | The deployment's declared bounded region (caps, allowlists, reversibility classes, budgets), content-hashed; the hash binds grants and records to the exact bounds in force (§6.6). |

---

## 3. Document conventions

Normative text is implementation-agnostic. Non-normative callouts marked *Reference
implementation note:* MAY name concrete technology (ptc-gal-reference; §10.2). Field tables pin
field names and abstract types.

**Implementation status markers.** This specification is derived from a running reference
implementation (§10.2) rather than drafted in advance of one, and the normative text is
overwhelmingly a description of mechanism that exists and has been exercised.

Exercised means proven by test. Every promotion, demotion and trigger firing behind this text ran
on evidence a drill produced for that purpose. The reference implementation has no proven-in-use
evidence in the sense of IEC 61508: no grant has been promoted or demoted on observations from a
sustained workload that nobody shaped for the predicate. Its thresholds, its error budgets and the
false-positive rate of its demotion triggers are therefore uncalibrated.

A small number of
clauses are deliberately normative *ahead* of that implementation, because the design question
was settled and the specification tier is the honest place to settle it while the code catches
up. Those clauses are marked individually, wherever a reader can encounter them, with a line of
exactly this form:

> **Implementation status:** NORMATIVE, NOT YET IMPLEMENTED in the reference implementation (tracking: #NNN).

The marker is machine-readable on purpose: its literal prefix `**Implementation status:**` can
be grepped, and `#NNN` is the reference implementation's public tracking issue for the work. A
clause carrying the marker is fully normative (an implementation claiming conformance MUST
satisfy it) and simultaneously a statement that the reference implementation does not yet.
**Absence of the marker means the clause is implemented in the reference implementation.** No
conformance claim, by this project or any other, may cite a marked clause as a shipped control;
a specification clause is a requirement, never evidence that anything enforces it. Where a
marked clause owns a field-table row, the row carries the short inline form
`(not yet implemented, #NNN)`, which means the same thing.

**The inverse marker.** A derived specification drifts in two directions, and the marker above
records only one of them. Where the reference implementation has replaced a mechanism this
specification mandates with one that holds the clause's *purpose* more strongly, the clause is
marked with a line of exactly this form:

> **Implementation status:** NORMATIVE, SATISFIED BY A STRONGER MECHANISM in the reference implementation (tracking: #NNN). <Named mechanism, and why it dominates the one this clause names.>

A table row carries the short inline form `(stronger mechanism, #NNN)`. Four rules govern it,
and the first two are what keep it from becoming a way to launder a defect:

1. **It applies only to a clause that mandates a mechanism, an ordering, or a procedure.** A
   clause stating a *property* cannot be exceeded, only met or not met, because any mechanism
   holding the property conforms. A property-shaped clause the implementation does not hold is a
   defect, and takes the NOT YET IMPLEMENTED marker or a fix.
2. **The marker MUST name the replacing mechanism and state why it dominates.** "Stronger" is
   otherwise an assertion no reader can check, and an unfalsifiable marker is worse than none.
   *Dominates* means it holds every outcome the mandated mechanism guarantees, and forecloses a
   failure the mandated mechanism admits. A mechanism that is merely different is not stronger,
   and its divergence is a defect in the implementation.
3. **The clause remains fully normative and literal conformance remains conformance.** An
   independent implementation that builds exactly what the clause says conforms; the marker warns
   it that a better construction is known and points at where that is being written down.
4. **The marker records that the SPECIFICATION is expected to move**, not the implementation. This
   is the precise inverse of the marker above, and both are transitional: one says the code will
   catch up to the text, the other says the text will catch up to the code. The tracking issue is
   the specification revision, and the honest end state is a clause stating the property with the
   mechanism as one way to hold it, at which point the marker is removed.

**Absence of either marker** means the clause is implemented in the reference implementation as
written.

**Canonical encoding.** Every GAL object defined in §5 MUST be serialized, whenever it is
stored, hashed, or signed, in one canonical JSON profile: **object keys sorted in ascending
code-point order, no insignificant whitespace (compact separators), and ASCII output** (any
non-ASCII character escaped). This is not a presentation preference. §6.9 makes the *stored
bytes* the integrity basis and §6.10 signs a statement over them, so two implementations that
serialize the same object differently cannot verify each other's records. The encoding is
interoperability-critical and is therefore pinned here rather than deferred. The profile is
adjacent to JSON Canonicalization Scheme [JCS], and an implementation MAY use JCS directly;
where the two disagree on a construct, an implementation MUST state which it emits. A
transport MAY re-encode an object in flight, but a verifier MUST verify the stored bytes
verbatim and MUST NOT re-serialize before verifying (§6.9).

A field that is optional in its field table (Required "no") is **omitted** from the canonical
form when its value is null, never emitted as `null`. Schema growth therefore never changes the
bytes of an object that does not use the new field, and integrity indicts tampering rather than
evolution.

*Reference implementation note (non-normative):* the reference implementation emits this
profile for its signed ledger records, and its grant payload does not yet emit the compact
form; that divergence is being corrected.

---

## 4. Enumerations

### 4.1 The autonomy ladder — four rungs, three levels

The ladder has four rungs. The bottom rung is the absence of a grant; the level enumeration is
complete at the three acting rungs and MUST NOT grow a fourth value.

| Rung | `Grant.level` value | Human role | Actuation discriminator |
|---|---|---|---|
| **Recommend** | *(no grant exists)* | Acts entirely in their own systems | The agent holds no grant for the class; the broker default-denies; the agent's advice is only text in its reply. **No staged intent ever exists.** |
| **in-loop** | `"in-loop"` | Approves each act | The agent stages a real brokered call; the broker materializes a durable approval intent and waits; on approval **the broker** executes. |
| **on-loop** | `"on-loop"` | Supervises, can intervene | The broker executes within the envelope; the human observes the audit surface and can tighten or flag. |
| **out-of-loop** | `"out-of-loop"` | Sets the envelope | The broker executes fully within the envelope with no per-act human involvement. |

The observable tell between Recommend and in-loop is the audit trail: Recommend produces no
staged action at all; in-loop produces a staged intent that a human releases. The first
promotion (Recommend → in-loop) is therefore the *creation* of the grant, through the same
recorded ceremony as every later climb.

An in-loop approval is single-use authority, and a single-use resource can be spent by someone
other than its holder. Consumption of an approval MUST be keyed on the approval's own bound
action, the frozen call the human was shown, and MUST NOT be inferred from the approval's
identifier appearing in any execution record; otherwise any party able to write a record can
burn another party's approval without using it. Only the principal the call was frozen for
(the whole identity tuple of §5.1, never the agent identity alone) may release, reject or
otherwise transition it, and the release executes the stored call and nothing re-sent.

The frozen call includes the authority the call was evaluated under. A human approves the
call's visible coordinates under the authority shown at approval time, so two calls with the
same visible fields under different authority are different calls, and a release that runs
under different authority than the one shown breaks what-you-see-is-what-executes even when
every other coordinate matches. An enforcement point MUST be able to verify that the authority
under which a call is released is equivalent to the authority approved; whether that
equivalence is carried with the approval or reconstructed from immutable records of the frozen
call is an implementation choice.

Rejection here is the terminal disposition, the one that closes the approval. A non-terminal
disposition, such as a deferral or a request for changes, is neither release nor rejection: the
approval stays pending and nothing is consumed. Consumption is keyed on that closure, never on
the outcome of execution. Whether a released call executed, failed, or ended in an
indeterminate state is a separate question, and once a release has begun executing, an
indeterminate outcome MUST NOT return the approval to pending, because a retry could
duplicate an effect that did occur.

### 4.2 Demotion triggers (closed set)

Exactly four deterministic conditions may trip automatic demotion. The set is closed in this
specification (§9.1).

| Trigger | Deterministic condition | Maps to `demotionReason` | Default posture |
|---|---|---|---|
| `stale_confidence` | Label-free drift: the distribution of constructed confidence has shifted, voiding the calibration that certified the rung. Derived from a stale evidence artifact as given; the drift detector that *sets* staleness is deployment-tier. | `"pending-evidence"` | Armed per grant (§6.7.4) |
| `corroboration_failure` | A premise failed its k-of-n independent-source quorum: derived deterministically from a typed corroboration record as `agreeing < k`. | `"failing"` | Armed per grant |
| `budget_breach` | An atomic durable counter (error, spend cap, escalation, fallback) blew its per-period bound. | `"failing"` | Armed per grant |
| `false_action` | An authenticated owner flagged an executed operation as wrong. Derived from a durable counter written by the authenticated flag path, never from message content. A single flag suffices. | `"failing"` | Ships OFF; armed per grant |

### 4.3 Ledger record types

Six record types share one append-only ceremony ledger (§5.2). Five record a level change. The
sixth, `reattestation`, records a write that changes none.

| `recordType` | Records | Shape rules |
|---|---|---|
| `promotion` | The maker≠checker ceremony that raises the level | maker ≠ checker REQUIRED; `predicate` REQUIRED non-empty; `triggeredBy` empty; `demotionReason` null; `toLevel` exactly one rung above `fromLevel`, or `in-loop` where `fromLevel` is null |
| `demotion` | Automatic deterministic demotion | `ratifiedBy` is the system demotion-evaluator identity; `triggeredBy` non-empty; `demotionReason` set; `predicate` null; `toLevel` MUST NOT rank above `fromLevel` and MUST NOT be `out-of-loop` |
| `bootstrap` | The sanctioned seed: first creation of a grant outside the propose/ratify ceremony | `fromLevel` null; maker ≠ checker NOT enforced (single-operator seed is sanctioned); `predicate` null; `triggeredBy` empty; `demotionReason` null |
| `tightening` | Voluntary any-level → `in-loop` move (§6.3) | `toLevel` = `"in-loop"`; maker ≠ checker NOT enforced; `predicate` null; `triggeredBy` empty; `demotionReason` null |
| `lapse` | A certification term expired (§6.7.6) | `toLevel` = the grant's `lastSafeLevel`; `ratifiedBy` is the system evaluator identity; `triggeredBy` empty: a lapse is an absence, not a fired condition; `demotionReason` = `"pending-evidence"`; `predicate` null; `toLevel` MUST NOT rank above `fromLevel` and MUST NOT be `out-of-loop` |
| `reattestation` | A grant re-issued at its level under a changed envelope (§6.6) | `fromLevel` and `toLevel` both equal the grant's level; `ratifiedBy` is the identity that re-attested; maker ≠ checker NOT enforced; `predicate` null; `triggeredBy` empty; `demotionReason` null; `certifiedUntil` null |

All six record types are implemented.

A `reattestation` record neither earns a level nor lowers one. A reader that derives a
coordinate's level from its ledger, or looks for the record that raised a grant to its current
level, MUST pass over it: the level and the earning record are the ones the ledger held
immediately before it.

The schema enforces field shape and **one rule of direction**: a `demotion` or `lapse` record
whose `toLevel` ranks above its `fromLevel`, or whose `toLevel` is `out-of-loop`, MUST be refused
at construction, by the writer and by every reader that parses one. These two record types are
signed by the evaluator's key (§6.10), which runs with no human and no model, so a record of
either type that raises a level is authority minted from the side that must only ever lower it.
Leaving the direction to the state machine alone protects the honest write path and nothing
else: a record planted in the store never passes through the state machine. The remaining
transition validity (level ordering, constructibility) is the state machine's obligation (§6.2,
GAL-17), and the continuity of every record with the ledger before it is the auditor's
(§6.10, §6.11).

The same reasoning holds for the issuer's key. A `promotion` record whose `toLevel` is not
exactly one rung above its `fromLevel` MUST be refused at construction, by the writer and by
every reader that parses one (not yet implemented, #159). The ceremony refuses a skip, and a
record that never passed through the ceremony is the case this rule exists for.

### 4.4 Demotion reasons

| `demotionReason` | Meaning | Correct response |
|---|---|---|
| `"failing"` | An error blew a bound; the model/policy is wrong. | Fix it. |
| `"pending-evidence"` | Recent labels are lacking to certify this rung; nothing is proven broken: the certification lapsed. | Gather data. |

The two look identical from outside and demand opposite responses. Implementations MUST keep
them distinct (§6.7.3). In a mixed-trigger demotion, `"failing"` dominates: an actually-blown
bound is the stronger claim than a lapsed certification.

### 4.5 Provenance-maturity levels

The maturity vocabulary is defined by the sibling PTC specification (its provenance-maturity /
rung-ceiling section); GAL consumes it as an ordered enumeration and enforces it as a promotion
gate term (§6.13).

| Maturity | What the mesh can prove | Autonomy it can support |
|---|---|---|
| `taint-bit` | A single derived taint flag propagates | Correct gating *if the sender is trusted* |
| `lineage` | A full (unsigned) provenance chain propagates; the receiver derives its own taint | Approval-gated action |
| `signed-lineage` | The chain is signed and receiver-verified, and the receiver has on record, for every signing key, that the signer's agent cannot reach it; forging a chain takes the key | Eligibility for acting-autonomy rungs |

Ordering is strict: `taint-bit` < `lineage` < `signed-lineage`.

`signed-lineage` carries an **evidence class**, defined by PTC with the tier: `declared` where
the receiver's record of signer key custody is an operator's statement that nothing has
checked, `attested` (reserved in the current PTC draft) where it rests on evidence the receiver
verified. GAL does not define the classes. It carries the class onto whatever the maturity
licenses (§6.13).

---

## 5. Objects

### 5.1 Grant

The unit of authority: per `(principal, action-class)`. Autonomy level is stored state, not a
constant; this record is where it lives and what the lifecycle moves on a ratchet.

| Field | Type | Required | Description |
|---|---|---|---|
| `principal` | Principal | yes | The identity tuple (agent identity, skill, user, tier) this grant is for. |
| `actionClass` | string | yes | The class of action it authorizes (e.g. `"email.send"`, `"payments.transfer"`). Derived per §6.5. |
| `level` | `"in-loop"` \| `"on-loop"` \| `"out-of-loop"` | yes | Current autonomy rung: stored state, moved only by the lifecycle. The enumeration is complete at three values (§4.1). |
| `envelopeHash` | string | yes | Content hash of the envelope in force. The agent cannot widen its own envelope; the hash makes the in-force bounds verifiable and binds the grant to them (§6.6). |
| `promotedBy` | string | yes | The accountable identity that ratified the current level (checker side of maker≠checker), or that last re-attested the grant at it (§6.6). |
| `evidence` | string | yes | Reference to the covered-distribution evidence the promotion cited. An **implementation-defined reference, opaque to GAL**: GAL fixes no URI scheme and no digest binding, and no conforming component may parse or resolve it as a condition of accepting the grant. It MUST be bound into the integrity basis (§6.9): it rides inside the canonically-serialized payload, so the reference cannot be altered after the fact. |
| `ts` | string (ISO-8601 UTC) | yes | The instant of the last write to this grant. It equals the `ts` of the ledger record that write appended (§5.2) and is never backdated. It does not say when the level took effect: after a lapse, enforcement fell at `certifiedUntil` (§6.7.6), and a re-attestation writes the grant without changing its level (§6.6). |
| `lastSafeLevel` | `"in-loop"` \| `"on-loop"` | yes | The rung automatic demotion falls back to. MUST NOT be `"out-of-loop"`. |
| `demotionTriggers` | DemotionTrigger[] | yes | The deterministic conditions armed for this grant, drawn from the closed §4.2 vocabulary. |
| `demotionReason` | null \| `"failing"` \| `"pending-evidence"` | yes | Why currently demoted; null when at full level. |
| `labelLatency` | string (duration) | yes | How long until an action of this class yields ground truth. Caps re-promotion speed (§6.8, §8.2). *Reference implementation note:* validated as an ISO-8601 duration; calendar-ambiguous year/month units are refused. |
| `certifiedUntil` | string (ISO-8601 UTC) \| null | no | The term of the current certification: the instant at and after which this level is no longer certified and the grant lapses (§6.7.6). Null, or absent, means no term; omitted from the canonical form when null (§3). |
| `ownerId` | string | yes | The named human owner accountable for this grant. Every grant has one. |

The grant record carries **no integrity field of its own**. Integrity binds the record's stored
bytes from outside the record (§6.9): a hash carried *inside* the structure it protects cannot
cover the bytes as stored, because computing it changes them.

`DemotionTrigger` is the closed enumeration
`"stale_confidence" | "corroboration_failure" | "budget_breach" | "false_action"`.

Non-normative example:

```json
{
  "principal": {"agentId": "mailer-01", "skill": "correspondence", "user": "ops", "tier": "B"},
  "actionClass": "email.send",
  "level": "on-loop",
  "envelopeHash": "sha256:9f2c4d…",
  "promotedBy": "identity:checker-7",
  "evidence": "evidence/mailer-01/email.send/2026-07-01..2026-07-14",
  "ts": "2026-07-15T14:02:11Z",
  "lastSafeLevel": "in-loop",
  "demotionTriggers": ["budget_breach", "false_action"],
  "demotionReason": null,
  "labelLatency": "P3D",
  "certifiedUntil": null,
  "ownerId": "owner:operations"
}
```

### 5.2 PromotionRecord

The append-only ceremony-ledger record of every write to a grant. Six record types share one
ledger (§4.3); demotions append a `demotion`-typed record on the *same* ledger: demotion is
recorded, never approved. The grant is the balance and the ledger is its journal: no sanctioned
path writes a grant without appending the record that accounts for the write.

| Field | Type | Required | Description |
|---|---|---|---|
| `recordType` | `"promotion"` \| `"demotion"` \| `"bootstrap"` \| `"tightening"` \| `"lapse"` \| `"reattestation"` | yes | The record type; default `"promotion"`. Shape rules per §4.3. |
| `actionClass` | string | yes | The action class of the grant the record accounts for. |
| `principal` | Principal | yes | For whom. |
| `fromLevel` | level \| null | yes | `null` means the Recommend rung: the agent held no grant, and this record's write is the grant's creation (first promotion or bootstrap). |
| `toLevel` | `"in-loop"` \| `"on-loop"` \| `"out-of-loop"` | yes | The resulting level. Equal to `fromLevel` on a `reattestation` record. |
| `evidence` | string | yes | An implementation-defined reference, opaque to GAL, on the same terms as `Grant.evidence` (§5.1), but what it references varies by record type; see below. |
| `predicate` | string \| null | promotion-typed only | The signed promotion predicate authored in advance (the licensing text/result). REQUIRED non-empty on `promotion` records; null otherwise. |
| `proposedBy` | string | yes | Maker identity. |
| `ratifiedBy` | string | yes | Checker identity. On `promotion` records MUST differ from `proposedBy` (§6.4.3). On `demotion` and `lapse` records, the system demotion-evaluator identity. |
| `attestation` | enum \| null | no | **How the two ceremony identities were established**, from a closed vocabulary whose one defined value is `"solo-local"`: ONE operator held both ceremony roles, through two credentials that the same holder could mint. Null, including its absence from any record written before the field existed, means the two identities were established by mutually unmintable credentials, the full maker≠checker guarantee of §6.4.3. The value MUST be **derived** by the issuer from the ceremony identities themselves, never independently asserted by the proposer, so the marker and the identities cannot disagree. This field is what keeps §6.10's non-repudiation claim honest: the record must never imply a review that did not happen, so a record written under the weaker guarantee states it rather than leaving an auditor to assume the stronger one. |
| `envelopeHash` | string | yes | The envelope hash in force when the record was written; bound into the signed statement (§6.10). On a `reattestation` record, the hash the grant was re-issued under. |
| `triggeredBy` | string[] | demotion-typed only | The DemotionTrigger values that fired. Non-empty on `demotion` records; empty otherwise. |
| `demotionReason` | `"failing"` \| `"pending-evidence"` \| null | demotion- and lapse-typed only | Set on `demotion` records; `"pending-evidence"` on `lapse` records (§4.3); null otherwise. |
| `certifiedUntil` | string (ISO-8601 UTC) \| null | no | On a `promotion` record, the certification term the checker ratified; binding the term into the signed record is what makes it part of what the checker is accountable for (§6.10). On a `lapse` record, the term that expired, which is the instant enforcement fell (§6.7.6) (not yet implemented, #165). MUST be null on every other type. Omitted from the canonical form when null (§3). |
| `ts` | string (ISO-8601 UTC) | yes | The instant the writer appended this record, and nothing else; see below. |

> **Implementation status:** NORMATIVE, NOT YET IMPLEMENTED in the reference implementation (tracking: #165). Applies to **`certifiedUntil` on a `lapse` record**; the field on a `promotion` record is implemented.

**What `evidence` carries, per record type.** GAL deliberately does **not** require `evidence`
to be a URI or a digest, because the record types of §4.3 do not carry the same kind of thing
and a uniform format requirement would be violated by half of them on the day it was written. A
`promotion` or `bootstrap` record's `evidence` references the covered-distribution evidence
the level change was licensed by. A `demotion` record's `evidence` carries the human-readable
cause of the demotion: the record exists to say *why the rung fell*, and there is no external
evidence artifact to point at. A `tightening` record's `evidence` is a short free-text
rationale, which an implementation MAY default, since a voluntary tightening is
safety-monotone and cites nothing. A `lapse` record's `evidence` names the expired term, since
a lapse cites the absence of renewal rather than any artifact (§6.7.6); a reader takes the term
from the record's `certifiedUntil` field and never from this string. A `reattestation` record's
`evidence` is a short free-text rationale, which an implementation MAY default. What is uniform, and
what conformance depends on, is that the value is opaque to GAL and integrity-bound with the rest of the record (§6.9, §6.10): an
implementation MUST NOT make accepting a record conditional on parsing it, and MUST NOT permit
it to be edited after signing.

**What `ts` denotes.** One rule holds for every record type. A record's `ts` is the instant its
writer appended it, read from the writer's clock. It MUST NOT be backdated to any event the
record describes, and the same value MUST be written to the grant's `ts` in the same write
(§5.1). A writer MUST stamp a record later than every record already at its coordinate, so `ts`
orders a coordinate's ledger. Where the writer's clock reads at or before the latest record, the
writer advances the stamp past that record, and a record can then run slightly ahead of wall
time.

| `recordType` | `ts` is | `ts` is not |
|---|---|---|
| `promotion` | when the ratified record was written | when the evidence was gathered or the proposal made |
| `bootstrap` | when the seed was written | |
| `tightening` | when the tightening was written | |
| `demotion` | when the evaluator wrote it | when any trigger condition arose |
| `lapse` | when the evaluator wrote it, at or after the explicit evaluation instant | `certifiedUntil`, the instant enforcement fell |
| `reattestation` | when the re-attestation was written | when the envelope changed |

A timestamp does not establish the time of another event that was not separately retained. A
consumer MUST NOT infer from `ts` the instant of a flag, a breach, an evaluation, or any other
event the record does not carry as a field of its own. Where the instant a transition took
effect differs from the instant it was written and the specification needs it, it is carried in
its own typed field, as `certifiedUntil` is on a `lapse` record, and never by adjusting `ts`.
Neither `ts` is an evaluation instant: §6.7.6's rule that the instant is supplied explicitly and
never derived from records is unchanged.

Non-normative example (promotion type):

```json
{
  "recordType": "promotion",
  "actionClass": "email.send",
  "principal": {"agentId": "mailer-01", "skill": "correspondence", "user": "ops", "tier": "B"},
  "fromLevel": "in-loop",
  "toLevel": "on-loop",
  "evidence": "evidence/mailer-01/email.send/2026-07-01..2026-07-14",
  "predicate": "error_rate=0.1667 < threshold=0.2500 over 6 observations …",
  "proposedBy": "identity:maker-3",
  "ratifiedBy": "identity:checker-7",
  "envelopeHash": "sha256:9f2c4d…",
  "triggeredBy": [],
  "demotionReason": null,
  "ts": "2026-07-15T14:02:11Z"
}
```

### 5.3 Evidence artifact contract (provisional)

The licensing predicate and the demotion triggers consume *typed* evidence, never raw model
output. Constructed confidence is a contract, not a logprob read: every consuming function is
pure and deterministic; there is no model in any gate. The shapes below are **provisional**;
field sets may be tightened by a future version.

**ConfidenceArtifact**, the constructed-confidence artifact backing a proposal:

| Field | Type | Description |
|---|---|---|
| `confidence` | number [0,1] | Constructed value, never a raw logprob. |
| `error_prob` | number | The error-budget draw numerator; method-specific, NOT `1 − confidence`. |
| `evidence` | tagged union | Method-specific evidence: self-consistency `{samples, agreement}`, ensemble `{members, agreement}`, or conformal `{coverage, threshold, calibration_size}`. The method vocabulary is a closed catalog. |
| `stale` | boolean | Label-free drift flag; a stale artifact voids the certification and never meets a bar. |
| `computed_at` | string (ISO-8601 UTC) | Construction time. |
| `annotations` | string[] | Attach-only findings (e.g. the evidence reviewer's, §6.4.5); NEVER read by any gate. |

**Error budget**: a per-period bound on cumulative tolerable error, drawn down per decision
as `error_prob × blast_radius`, metered on scoped durable counters. `blast_radius` is the
deployment-declared weight per blast class; a weight table is REQUIRED whenever a budget is
set: the base never invents a domain weight. Breach emits `budget_breach` as a typed demotion
input, never a model judgment. Blast class derives one-way (§6.5): high = write ∧ external ∧
¬reversible (unknown reversibility reads as not-reversible); medium = any other write; low =
read; a deployment MAY tighten a derived class up to high, never lower it.

**DemotionSignal**, emitted at breach detection, consumed by the demotion evaluator (the
evidence machinery emits, never applies):

| Field | Type | Description |
|---|---|---|
| `trigger` | DemotionTrigger | §4.2 vocabulary. |
| `principal` | Principal | Grant coordinate. |
| `action_class` | string | Grant coordinate. |
| `period` | string | The counter-period bucket the breach was metered in. |
| `detail` | string | Audit-safe cause; no payload content. |
| `ts` | string (ISO-8601 UTC) | Emission time. |

**CorroborationRecord**: the typed k-of-n quorum result the `corroboration_failure` trigger
derives from, deterministically. The corroboration pass that *produces* the record (source
selection, agreement judgment, staleness and provenance checks, sanity bounds) is
deployment-tier; this contract owns the shape and its consequence only.

| Field | Type | Description |
|---|---|---|
| `k` | integer ≥ 1 | The quorum: agreeing sources required for the premise to stand. |
| `n` | integer ≥ 1 | Independent sources consulted. |
| `agreeing` | integer ≥ 0 | Sources that agreed **and** were non-stale with valid provenance. |
| `stale_sources` | integer ≥ 0, default 0 | Consulted sources discarded as stale. Audit detail; never adds to `agreeing`. |
| `computed_at` | string (ISO-8601 UTC) | When the corroboration pass ran. |
| `annotations` | string[] | Attach-only findings, the same discipline as `ConfidenceArtifact.annotations`: NEVER read by any gate. |

The field set is **closed**: an implementation MUST reject an unknown key rather than ignore
it, and MUST reject an incoherent record (`k > n`, `agreeing > n`, or `stale_sources > n`)
rather than clamp it. The failure predicate is exactly `failed ⇔ agreeing < k`, and it is the
only computation any gate performs over this record.

**Per-source provenance is deliberately excluded.** A record naming *which* sources agreed is
the obvious extension and this specification declines it: the producer folds staleness and
provenance validity into the `agreeing` count (a stale or provenance-invalid source is simply
not counted, and can therefore never carry a quorum) so the deterministic trigger needs no
source identities to reach the same verdict. Carrying them would put source-identity data into
the base evidence contract, where every consumer and auditor would then have to handle it, in
exchange for nothing the predicate reads. A deployment that needs per-source detail for its own
review keeps it in the deployment-tier corroboration pass that produced the record.

**Label-free drift input**: the `stale_confidence` trigger derives from a stale
ConfidenceArtifact *as given*; the drift detector that sets `stale` is deployment-tier and out
of scope. An unusable or unparseable evidence input MUST cause the demotion evaluator to
refuse loudly, never to conclude "no breach".

---

## 6. Normative behavior

### 6.1 The grant coordinate

The rung is per `(principal, action-class)` state, never per-agent: an agent is a vector of
rungs, one per capability, and "what rung is the agent on?" is a category error. Conforming
implementations MUST NOT expose or act on an agent-scoped autonomy level. The level is
orthogonal to the per-call decision verbs (allow / deny / transform / require-approval /
abstain): level is stored state on the grant, the verb a per-call output of the enforcement
gate, and implementations MUST NOT conflate the two axes. At Recommend the broker
default-denies: absence of a grant is the deny, not a distinct flag; a "blocked" class is
simply an ungranted class.

### 6.2 The ratchet asymmetry

The asymmetry is the load-bearing design property and MUST be preserved:

> Promotion is recorded and human-gated. Demotion is automatic, deterministic, and has no model
> in the loop.

The system can narrow its own envelope without a human in the moment; widening it requires a
recorded ceremony. Any structurally invalid transition (level-skipping outside the sanctioned
re-attestation case, a demotion target of `out-of-loop`, a `demotion` or `lapse` that raises
the level, a fourth level value) MUST be unconstructible: a typed refusal, not a warning.

### 6.3 Tightening is always permitted

Any level → `in-loop` is always permitted: voluntary tightening is safety-monotone and needs
no ceremony, no evidence, and no trigger. It MUST append a `tightening`-typed ledger record
(§4.3) so the level change remains accounted for.

### 6.4 Promotion — the only upward path

#### 6.4.1 The deterministic licensing predicate

The acceptance gate for any upward move is a **pure, deterministic predicate** over windowed,
scoped counters plus the typed evidence artifact. No model output may appear anywhere in the
licensing path. The predicate MUST evaluate, in a fixed order, at least these gate terms:

1. **Provenance-maturity ceiling**: the target rung's required maturity is met (§6.13).
2. **Evidence artifact present**: absence of evidence is not evidence.
3. **Evidence artifact fresh**: a stale artifact voids the certification.
4. **Covered-distribution soundness asserted**: an explicit proposer assertion, recorded on
   the ceremony record (§6.4.2).
5. **Error budget not in breach**: evaluated only when the budget knob is configured; an
   unset knob is no gate (§1.2).
6. **Minimum observation count met**: a thin sample cannot license a promotion.
7. **Error rate below the configured threshold**: computed over the evidence window from
   durable counters (observed erroneous outcomes over observations).

Every ambiguous case (zero observations, sample too small, missing or stale artifact,
unasserted coverage, malformed input) MUST resolve to *ineligible*. The predicate is
fail-safe: it cannot pass on the absence of disconfirming evidence, and callers MUST NOT
override an ineligible result.

#### 6.4.2 Evidence window binding and soundness

The evidence window's span and period granularity MUST be bound into the proposal the checker
ratifies, under the proposal's own integrity mechanism, so the checker sees, and the record
proves, exactly which evidence licensed the change. Evidence MUST come from a *covered*
distribution: gathered where the conditions the promoted class will act under are represented.
A thin observed-accuracy count over an irreversible action class is an unsound predicate and
the ceremony MUST reject it. Covered-distribution soundness enters the predicate as an
explicit proposer assertion recorded on the ceremony record: a human accountability anchor,
not a derived quantity.

*Reference implementation note:* counters are scoped per principal + operation + UTC period; a
writer/reader period mismatch reads disjoint keys and yields zero evidence, failing toward
less authority, never a wrong sum.

#### 6.4.3 Maker≠checker — what it enforces, what it evidences

A promotion is proposed by a maker and ratified by a checker, and the two MUST be different
credentials, compared on credential identity. The standard therefore enforces exactly this:
**no single credential can both propose and ratify** (a compromised agent, automation job, or
leaked key cannot self-promote) and the signed record *evidences*, non-repudiably, which
credentials did each.

It does **not** and cannot enforce two *humans*. With a single operator, maker≠checker is the
same judgment exercised through two credentials, and the record MUST state that truthfully.
Two-human review is an organizational control (separation of duties) that a platform can only
evidence, never enforce; deployments that need it establish distinct credential holders, and
the signed record proves whether that held. Implementations SHOULD make the credential
separation *topological* (a standing trust naming the checker principals) rather than
*ceremonial* (per-ceremony trust mutation, whose revocation may be eventually consistent).

#### 6.4.4 High-blast is always human-ratified

For a high-blast action class (§6.5), a true predicate result is necessary but NEVER
sufficient: a per-instance human ratification is REQUIRED for every promotion. The reason is
control response time, not trust: every control this lifecycle applies *after* a grant is
exercised (the deterministic demotion triggers (§6.7), the independent ledger audit (§6.9), an
off-path observer) bounds the *duration* of a bad grant, never the blast of a single exercise of
it, and a control's response time must be matched to the blast rate of the authority it bounds.
Below the high-blast threshold an act-then-detect-then-demote window is survivable, so a signed
promotion predicate authored in advance MAY stand in as the ratifier (the pre-authorized change
envelope pattern), with its stringency and ceremony scaling to blast radius. At and above it, no
after-the-fact control has a response time matched to a single act; only a human *inside* the
window suffices, and detection MUST NOT be treated as a substitute for ratification. The predicate
result MUST carry the human-ratification requirement
explicitly so the ceremony cannot drop it; an unknown blast class MUST be treated as an error,
never as not-high.

#### 6.4.5 The evidence reviewer may surface, never decide

An OPTIONAL evidence reviewer (if a model, of a different model family than the maker's, so
the pair shares no blind spot) MAY review the evidence bundle during the ceremony. Its
closed-vocabulary findings are attached to the ceremony result for the human ratifier, and
that is all: it MUST NOT be able to license or veto a level change, and a failing or
unreachable reviewer MUST degrade to a recorded reviewer-error finding, never a gate flip in
either direction. This is the surface-vs-decide rule: the model may only surface a concern;
the gate decides, deterministically.

#### 6.4.6 Input integrity of the sanctioned path

When a mutation path is the only sanctioned path to a protected object, every durable input
that path consumes needs the same tamper-evidence as the object itself, or the sanctioned
path IS the bypass. For every ceremony input (stored proposals, evidence artifacts, config
re-read at ratify time) an implementation MUST provide one of: the protected object's own
integrity mechanism, a signature, or full re-validation at the trust boundary (a stored
proposal is input, never authority). Proposals MUST be single-use and expiring.

#### 6.4.7 Write discipline

Ledger and grant writes MUST be conditional (append-only, never overwrite; update-only, never
mint), and **no failure MUST be able to leave a raised grant without its record**: an
interrupted ceremony may leave a record without a raised grant, never the reverse.

Two constructions satisfy this and an implementation MUST state which it provides. **Ordering**
writes the record before the grant mutation it accounts for, so an interruption between the two
lands on the safe side. **Atomicity** commits both as one transaction, so there is no interruption
between them to land anywhere; it holds every outcome the ordering guarantees and additionally
forecloses the partial-failure state the ordering can only bias toward safety. Where a store
offers a transaction, atomicity is RECOMMENDED, and the per-site write orderings that ordering
requires become unnecessary rather than merely redundant.

### 6.5 Action-class derivation

An action class derives from the operation fields the deployment already declares for each
tool operation (effect (read/write), external (crosses a trust boundary), reversible), not
from any base-owned catalog: the base standard owns the *shape* of the classification, the
deployment owns the *content*. High-blast derives from the same fields (write ∧ external ∧
¬reversible, unknown reversibility read conservatively as not-reversible). A deployment MAY
additionally *declare* a class high-blast but MUST NOT un-declare a derived one: the
derivation is tighten-only. A never-grantable hard ceiling, if wanted, is a future
tighten-only envelope knob, not a mechanism of this version.

### 6.6 Envelope binding and re-attestation

Every grant and every ledger record carries the content hash of the envelope in force when it
was written. A grant whose `envelopeHash` does not match the current in-force envelope MUST be
quarantined by the enforcer on every call, loudly, never silently passed (§6.9).

When the envelope changes (any redeploy or tightening that churns the hash), the cure is
**re-attestation**: a ceremony re-issuing the grant at its prior level **under human
ratification**. It carries the prior level rather than restarting at
`lastSafeLevel`: a hard restart would turn every envelope change into an autonomy event, and
the high-blast always-human rule (§6.4.4) already backstops the dangerous classes. An
implementation MUST NOT silently re-mint a grant under a new envelope hash, and re-attestation
MUST refuse when the named configuration it derives from does not match what the enforcer will
load, failing toward writing nothing, never toward minting authority the enforcer will reject.

**Re-attestation appends a record.**

A re-attestation MUST append one `reattestation`-typed ledger record (§4.3), signed under the
issuer role (§6.10), carrying the new `envelopeHash` and the identity that re-attested. The
record and the grant write carry the same `ts` (§5.2) and are subject to the write discipline of
§6.4.7: no failure may leave a re-attested grant without its record. A re-attestation changes
the grant's `envelopeHash`, `promotedBy` and `ts` and nothing else. It MUST NOT change the
level, `lastSafeLevel`, `demotionReason` or the term.

The reason is the ledger's purpose. A re-attestation rewrites the grant under a human's
authority, and a grant rewritten with no record is one the signed ledger cannot explain: nothing
signed says who re-attested it, when, or under which envelope, and the grant's `ts` then matches
no record (§6.11). Earlier drafts held that the ledger records level changes only, so a record
with no level change had no place on it. The record's type is what explains it. A reader passes
over a `reattestation` record when deriving a level (§4.3), and it does not restart dwell
(§6.8).

No write to a grant goes unrecorded. A demotion whose target equals the grant's current level
leaves the level where it was and appends a `demotion` record, as any demotion does. An
implementation MUST NOT provide a path that rewrites a grant and appends no record; fresh
evidence enters a grant through the promotion ceremony. A `reattestation` record MUST be signed
under the issuer role, and the bootstrap override of §6.10 does not apply to it.

What these two rules prevent is a grant brought back to acting authority with nobody
accountable for it. A grant quarantined by an envelope change stays refused until it is
re-attested. If re-attestation could run unsigned, or a second path could rewrite the grant
with no record, then anyone able to reach that path (an operator session with no ceremony key,
a compromised deployment job, a writer on the grant store) could return the grant to its level
and leave the signed ledger showing nothing. The requirement makes that act cost the issuer's
key and leave a record under it.

### 6.7 Demotion — automatic, deterministic, no model

#### 6.7.1 The trigger set and the path

Demotion MUST run when the model is confused and the inputs are suspect: exactly when one
cannot ask a model to decide. So it does not: exactly the four triggers of §4.2, each a
deterministic derivation from typed durable inputs, may trip demotion, and no model call may
appear anywhere on the path. A tripped trigger lowers the grant to `lastSafeLevel`, sets
`demotionReason` per the fixed mapping (§4.2, §4.4, `"failing"` dominating mixed firings),
appends a `demotion`-typed ledger record, and emits an audit event.

#### 6.7.2 Separate identity

Demotion writes MUST run under an identity separate from the agent, and the agent MUST have no
write access of any kind to the grant store: **a grant-store write the agent can reach is a
promotion bypass.** The grant level governs the entire enforcement path; if the agent can
write it, every other control collapses. The separation MUST be structural (an access-control
boundary), not conventional.

#### 6.7.3 Reasons stay distinct

`"failing"` and `"pending-evidence"` MUST be kept distinct end-to-end (§4.4): collapsing them
sends the wrong team at the problem; one demands a fix, the other demands data.

#### 6.7.4 Arming and the flag asymmetry

A trigger fires only for grants listing it in `demotionTriggers`; every non-floor bound ships
OFF (§1.2). `false_action` MUST derive from a durable counter written by an authenticated
owner path, never from message content, and a single flag suffices. The ease gradient points
downward: it is deliberately easier to get demoted than to overcome the safeguards, and no
flag, nor any accumulation of flags, can ever promote anything.

#### 6.7.5 Derivation parity

The `budget_breach` derivation MUST read the same durable counter, key, and comparison that
per-call enforcement uses: the demotion evaluator and the enforcer cannot disagree about
whether a bound blew.

#### 6.7.6 Lapse — a certification has a term

Every trigger in §4.2 asserts that something was **observed**: a bound blew, a quorum failed, a
distribution drifted, an owner flagged. None of them fires when *nothing happens*. A grant promoted
on evidence gathered long ago, whose agent has since been idle, therefore has no path downward: it
is not failing, nothing is proven broken, and it holds its rung indefinitely on a certification
nobody has renewed. The recorded chain says how the authority was acquired; a term is what forces it
to be re-justified.

A grant MAY carry a term (`certifiedUntil`, §5.1). Where it does:

- When the term passes, the grant MUST lapse: fall to `lastSafeLevel`, set `demotionReason` to
  `"pending-evidence"` (the certification lapsed, nothing is proven broken (§4.4)), and append a
  `lapse`-typed ledger record (§4.3). The record MAY carry the expired term as `certifiedUntil`
  (§5.2) (not yet implemented, #165), so a reader has the instant enforcement fell as a field
  beside the instant the record was written. The record's `ts` MUST NOT be set to the term.
- A lapse MUST NOT be recorded as a triggered demotion. `triggeredBy` names conditions that fired;
  a lapse is the *absence* of renewal, not the presence of a signal. Recording it as a trigger would
  leave `triggeredBy` naming a condition that never fired, and would teach every auditor to read a
  demotion record as possibly meaning "nothing happened, on schedule."
- A lapse MUST land on `lastSafeLevel`, never on outright revocation. Revoking authority on a timer
  is a **self-inflicted forced abstention** (see the sibling PTC specification's availability
  section): for a deployment whose polarity is positive-safe-action, a grant that silently expires
  into no-authority produces exactly the harm the agent exists to prevent, on a schedule an
  adversary does not even have to trigger.
- Recovery is the ordinary upward path: the promotion ceremony with fresh evidence (§6.4). An
  implementation MUST NOT auto-renew a term, and the grant holder MUST NOT be able to extend one in
  place; either would make the term self-certifying and thereby vacuous.
- A grant with no term does not lapse. Terms are a deployment knob and ship unset (§9.2): a
  deployment that sets none behaves exactly as it did before this arc existed.

**The evaluation instant.** Whether a term has passed is judged against an explicit instant
supplied to the evaluation, and a term has passed when that instant is at or after
`certifiedUntil`. The instant MUST NOT be derived from the timestamps of the records under
evaluation: not the grant's `ts`, not the latest ledger record, not the audit tape. Deriving it
from records reintroduces the failure this section exists to close, since an idle grant produces
no records and its "now" would stand still.

**Skew fails toward less authority.** Where the evaluation instant is compared with a time
asserted under another party's clock, a declared skew bound MUST be applied in the direction that
confers less authority: a term or an authority-lowering record is treated as effective up to the
bound earlier, and an authority-raising record or a not-before time up to the bound later.

> **Implementation status:** NORMATIVE, NOT YET IMPLEMENTED in the reference implementation (tracking: #175).

What this prevents is authority that outlasts its end by the width of the allowance. A symmetric
leeway honours a lapsed term for as long as the leeway runs, and whoever benefits from a slow
clock gets that window for nothing. Applied one way, the bound costs a grant at most its own width
at the end of a term.

**Enforcement does not wait for the record.** From the instant its term passes, a grant MUST be
enforced at the lower of its stored level and `lastSafeLevel`, whether or not the `lapse` record
has yet been written. The record is the durable account of the transition; the enforcer's
obligation starts at the boundary. The lower of the two, rather than `lastSafeLevel` outright,
because a lapse never raises a level.

**After a lapse.** The lapse writer is idempotent: a grant whose stored level is already at or
below `lastSafeLevel` owes no lapse record. A lapse leaves `certifiedUntil` and `lastSafeLevel`
unchanged. A promotion MUST NOT be built on a grant whose term has passed while its lapse is
unwritten, since it would either raise authority past an expired certification or leave a level
change the ledger cannot explain. The term is set only by the promotion ceremony, carried on the
`promotion` record the checker ratified (§5.2); no other write path may lengthen or remove it.

Grant age is deliberately distinct from the two decay concepts already in this specification.
`stale_confidence` (§4.2) fires when an observed distribution drifts, voiding a calibration; dwell
time (§6.8) damps oscillation between rungs. Neither reaches an idle grant, because neither has
anything to observe.

### 6.8 Hysteresis and re-promotion

Demotion and re-promotion MUST use different thresholds and a dwell time, and re-promotion
MUST require fresh recalibration evidence, never merely "the alarm stopped". Demotion itself
has no dwell: it fires when the condition holds. This prevents a noisy signal from oscillating
the rung. Dwell is measured from the ledger's records and not from the grant's `ts`, and a
`reattestation` record does not restart it: an envelope change is not time spent at a new level.

Re-promotion closes a control loop whose deadtime equals `labelLatency`: with no labels at
action time, re-promotion runs on delayed-label reconciliation: pair each resolved outcome
with the confidence logged at decision time, recompute calibration on the *current*
distribution, then ratchet up. Long label latency caps how fast a class can re-promote, and
therefore how much autonomy it can responsibly hold at all (§8.2).

### 6.9 Grant integrity at rest

Every stored grant MUST be integrity-protected and verified on read. A grant that fails
verification MUST be quarantined loudly: treated as no grant (default-deny) with an explicit,
audit-visible quarantine signal, never silently trusted and never silently dropped. A reader MUST
distinguish *quarantined* from *not found*: they license the same denial but demand opposite
operator responses, and collapsing them hides tampering inside routine absence.

**The integrity basis is the stored bytes.** An implementation MUST serialize the grant once, store
exactly those bytes, bind the integrity value to that byte string, and on read verify the stored
bytes **verbatim, before parsing them**. The integrity value MUST live outside the record it
protects (§5.1): a value carried inside the structure cannot cover the bytes as stored, since
computing it changes them.

The requirement is not merely constructional. A basis that is *re-derived* by parsing and
re-serializing binds the reader's current schema rather than the writer's bytes, so any honest
schema evolution (a widened type, a new optional field, a changed default) makes every
previously-written record fail verification as though it had been tampered with. **Integrity must
indict tampering, never evolution.** A control that cries tamper on legitimate change trains
operators to dismiss it, and then to stop evolving the schema; the resulting freeze presents itself
as prudence. Where a stored-byte basis must nevertheless change, an implementation MUST version the
basis explicitly rather than verify under two bases at once.

### 6.10 Record signing

Every ledger record MUST be signed at write time: a signature envelope (DSSE-style, with
pre-authentication encoding) over an attestation-style statement binding at minimum the record
content, the in-force `envelopeHash`, and the signer identity. The signing key is a workload
identity resolved by the issuing infrastructure, never held by, or reachable from, the agent.
A signed record is the **issuer's** non-repudiable statement of a level change: the identities it
authenticated as proposer and ratifier, the evidence window, the envelope in force. The named
identities do not themselves sign. What a verifier learns is that the holder of the issuer's key
attested those two identities, and the statement is as trustworthy as the issuer that
authenticated them (§8.3, §8.4). An implementation MUST NOT describe a signed record as proof
that the named proposer or ratifier signed anything.

An issuer MUST refuse to store an unsigned record when signing is configured, and MUST refuse
to run at all under half-configured signing (fail toward writing nothing). A tightening is never
refused for want of a key: where this requirement meets §6.3, §6.3 takes precedence, the record is
written unsigned and the audit reports it. An explicit,
recorded operator override MAY permit an unsigned record in bootstrap circumstances; the
record then verifiably carries its unsigned status and audit disposition (§6.11) applies. An
OPTIONAL transparency-log anchor for high-blast promotions is a knob shipping OFF.

**Signing authority is scoped by record type, and the scoping is what makes §6.7.2 hold.**
Records that raise or re-license authority (`promotion`, `bootstrap`, `tightening`, and
`reattestation`) MUST be
signed by the issuer's key. Records the automatic evaluator writes (`demotion`, `lapse`) MUST
be signed by a separate evaluator key, and no identity may hold both. Satisfying "every record
is signed" by giving the evaluator the issuer's key would let the side that runs with no human
and no model mint a promotion, which is the boundary §6.7.2 exists to draw; the requirement to
sign everything and the requirement to separate the identities are only jointly satisfiable
through two keys.

**The evaluator's key may only lower, and only from where the ledger stands.** A record the
evaluator signs MUST start from the level the ledger held immediately before it at that
coordinate: its `fromLevel` MUST equal the `toLevel` of the preceding record, or be absent
where there is none. The direction rule of §4.3 is not sufficient alone. A `demotion` from
`out-of-loop` to `on-loop` lowers on its own terms, and placed on a ledger that stood at
`in-loop` it leaves the derived level raised, under the evaluator's signature. An auditor MUST
report an evaluator record that breaks continuity, at every coordinate that has records whether
or not a grant exists there, and the finding MUST be un-waivable (§6.11): it is a raise the
issuer never signed.

**The issuer's records are held to the same continuity.** A `promotion`, `tightening` or
`reattestation` record
MUST also start from the level the ledger held immediately before it at that coordinate, and an
auditor MUST report one that does not, as an un-waivable finding (not yet implemented, #159).
The ceremony reads the level it moves from, so a break is a record that did not come through the
ceremony, and a promotion that restates its `fromLevel` can hide a skipped rung from the shape
rule of §4.3.

**Issuer standing.** Whether a record was validly signed and whether its signer still has
standing are different questions. A verifier MUST establish, at the evaluation instant and from a
source it pins, that the key which signed the record a grant's level rests on has not been
withdrawn. Where standing is withdrawn or cannot be established within a declared maximum age, the
grant is enforced at `lastSafeLevel`. This rule concerns withdrawal. A key retired on schedule is
not withdrawn, and its rotation does not lower the grants it signed.

> **Implementation status:** NORMATIVE, NOT YET IMPLEMENTED in the reference implementation (tracking: #179).

What this prevents is authority that outlives the authority that granted it. A promotion signed by
an issuer key later found compromised keeps a valid signature, and with no standing check the
grant stays at its level: whoever held the key for a day keeps every grant minted with it. The
fall is to `lastSafeLevel` and not to nothing, for the reason a lapse lands there (§6.7.6).

That fall is the enforcer's, inside the grant's own domain, where `lastSafeLevel` is known. A
verifier in another domain, evaluating a chain that passes through the grant, holds no such level
for it: a conveyed chain declares no fallback. Where that verifier cannot establish the standing of
an issuer on the path, or finds on the grant's ledger a record it must refuse (§4.3, above), the
path confers nothing.

A verifier MUST select the acceptable key set from the record's type, and MUST refuse a record
signed by the other role's key. A key resolvable under both roles is a configuration error and
MUST fail closed rather than resolve in either direction: a key that can sign anything is the
single key this clause forbids, arriving by configuration instead of by design.

**Adopting signing over existing history.** A deployment that adopts record signing after
records exist holds history that cannot be re-signed. That history MUST NOT be excused by
anything the unsigned record says about itself, its timestamp included. On an unsigned record
every field is the claim of whoever wrote the row, so an exemption for "records dated before
adoption" is one a planted record can claim by choosing its date. Earlier drafts of this
specification made exactly that exemption.

Once a deployment declares signing adopted, every record of every type MUST carry a verifying
signature of its role (above), whatever its timestamp. An unsigned record that is honest history
is a true finding whose remediation is deferred, and it is dispositioned the way §6.11
dispositions any such finding: one record at a time, by a signed acknowledgment that binds the
record's stored bytes. The alternative is to re-mint the ledger under signing and archive the
old one as an acknowledged ceremony artifact.

The adoption instant is still declared explicitly, and it is judged against an explicit
evaluation instant, never one derived from the records under audit (the rule §6.7.6 states for
terms). An adoption instant later than the evaluation instant MUST be reported as a finding. It
MUST NOT narrow what is checked: a misconfigured instant that bought a quieter audit would be
the old exemption again, reached by configuration. A deployment that has not declared adoption
MUST be reported as such on every audit, since an audit that checks less than this section
requires would otherwise read as full coverage.

The signing shape is shared with the sibling PTC specification (its signing-shape /
trust-context-envelope sections): same envelope discipline, same workload-identity key
handling, a second statement type on the same machinery. Grant integrity at rest (§6.9) and
change signing are distinct: the first protects the object, the second proves the provenance
of the change.

### 6.11 The audit obligation

An independent auditor (read-only, holding no ceremony credentials) MUST be able to
re-verify the entire ledger: every grant has a ledger counterpart from birth (§6.12), every
record's signature verifies, every transition is structurally valid, every grant's
`envelopeHash` matches an in-force envelope, and no record has been mutated or removed (detecting
a removed record, or a ledger rolled back to an earlier state: not yet implemented, #157). A grant
whose stored level is below the level its ledger derives, with no record explaining the drop, is
a finding: a write that bypassed the ledger fails toward less authority, and is still a write
that bypassed the ledger. So is a grant whose `certifiedUntil` differs from the term on the
promotion record that raised it to its current level.

**The grant's `ts` is the ledger's.**

> **Implementation status:** NORMATIVE, NOT YET IMPLEMENTED in the reference implementation (tracking: #166).

Every sanctioned write stamps the grant and the record it appends with one `ts` (§5.2). An
auditor MUST therefore report a grant whose `ts` differs from the `ts` of the latest ledger
record at its coordinate. A grant `ts` later than every record is a write the ledger does not
explain, whatever that write changed. A grant `ts` earlier than the latest record is a record
whose grant write did not land. This is the one grant-against-ledger check that does not depend
on what the unexplained write changed: the level, term and envelope checks above each catch a
write that moved the field they read, and a write that moved none of them is still a write that
bypassed the ledger.

**What the audit does not do.** The obligation stops at the ledger. Because `evidence` is an
opaque, implementation-defined reference (§5.1, §5.2), an auditor verifies that the reference
is intact and integrity-bound (that the record still cites what it was signed citing) and
does **not** resolve it, fetch it, or re-verify that the cited evidence supports the level
change. An implementation MUST NOT present a passing ledger audit as a re-verification of the
evidence a promotion was licensed by; the licensing judgment was made once, at ratification, by
the accountable checker (§6.4.3), and the audit proves who made it under which bounds, never
that it was correct.

A true finding whose remediation is deferred MUST be dispositioned only by a signed,
append-only acknowledgment artifact: a ceremony record binding the rule, the coordinate, and a
digest of the exact violation detail, so a *new* finding at the same coordinate is never
auto-waived. **Where the finding is about a stored record, the detail MUST bind that record's
stored bytes** (a digest of them, §6.9), so that an acknowledgment excuses those bytes and no
others. A detail naming only the coordinate, type and timestamp is satisfied equally by a
different record written in the same place, and an acknowledged finding would then cover
whatever replaced the record it was issued for. A finding that cannot be bound to stored bytes
MUST be un-waivable. The detail MUST also state the level change the record makes, so the party
acknowledging it sees what authority it sets. A waiver is a ceremony artifact, never a configuration toggle: a waiver mints
"green", which is authority. Only a closed waivable vocabulary may be acknowledged;
integrity-tamper findings MUST be un-waivable. A waiver whose signature cannot be verified
MUST NOT be applied (fail toward red). The audit reports acknowledged findings as
green-with-annotations, never silently green.

**A waiver has a term.**

> **Implementation status:** NORMATIVE, NOT YET IMPLEMENTED in the reference implementation (tracking: #158).

An acknowledgment defers a remediation; it does not cancel one. It MUST carry an expiry instant
set at the ceremony, and an acknowledgment past its expiry MUST NOT be applied: the finding
returns to the report until it is remediated or acknowledged again. Expiry is judged against an
explicit evaluation instant (§6.7.6). This is the rule §6.7.6 applies to a grant, applied to the
artifact that mints "green": a certification nobody has to renew is held indefinitely.

**Removal and rollback.**

> **Implementation status:** NORMATIVE, NOT YET IMPLEMENTED in the reference implementation (tracking: #157).

A signature proves a record is one the issuer or evaluator wrote. It does not prove the record
is still the latest, or that the records before it are all still present. An auditor MUST be
able to detect a record removed from a coordinate's ledger and a ledger restored to an earlier
state, from artifacts a party able to write the store cannot also rewrite. Record signatures
alone do not provide this: every surviving record of a truncated ledger still verifies.

**Reconciliation against the live principal set.**

> **Implementation status:** NORMATIVE, NOT YET IMPLEMENTED in the reference implementation (tracking: #14).

Ledger completeness is necessary and not sufficient: it proves every grant has a story, never
that every acting principal has a grant or that every grant still names a principal that
exists. An auditor MUST additionally be able to reconcile
the grant set against the set of principals the deployment actually runs, reporting **both**
directions: a principal acting without a grant, and a grant whose principal no longer exists. The
first is the shadow-agent case: authority exercised outside the lifecycle entirely, which every
control in this specification is blind to by construction, since a grant that was never issued
cannot be audited. The second is dormant authority waiting for an identity to be reused. Neither is
visible to any check that reads only the store.

Reconciliation is a **report, never an automatic write**. Retiring a grant is a ceremony (§6.3,
§6.4); an auditor able to delete one would hold the authority it audits, and the read-only property
above is what makes its findings trustworthy.

### 6.12 Bootstrap — no orphan grants

Creation of a grant outside the propose/ratify ceremony (the sanctioned bootstrap/seed path)
MUST emit a `bootstrap`-typed ledger record, so every grant has a ledger counterpart from
birth and the audit's no-orphan invariant holds with no exemption.

### 6.13 The PTC join — the provenance-maturity ceiling

A capability's rung may never exceed what the mesh can currently *prove* about its premises.
The sibling PTC specification (its rung-tracks-provenance-maturity section) is normative for
the maturity ladder; GAL operationalizes it strictly as a promotion gate term. **Both acting
rungs (`on-loop` and `out-of-loop`) REQUIRE `signed-lineage` maturity**: a promotion into a
rung whose maturity requirement is unmet MUST be structurally unconstructible; the predicate
evaluates the maturity as a deterministic gate term, not a reviewer's judgment. Unsigned
lineage means approval-gated; no acting-autonomy rung is reachable without receiver-verifiable
signed provenance. **`in-loop` as a target carries no provenance ceiling**: it is the
approval-gated floor, reachable as the Recommend-origin grant-creating first promotion; every
evidence gate of §6.4.1 still applies. PTC supplies the proof; GAL moves the state; the
ceiling is enforced in the predicate, not left to judgment.

**The ceiling inherits the evidence class.** PTC defines `signed-lineage` by a verified
signature together with the receiver's record that each signing key is held where the signer's
agent cannot reach it, and it requires that record to carry an evidence class (§4.5). A
promotion licensed at `signed-lineage` rests on that record, so it rests on the record's class.
The predicate text bound into a `promotion` record whose target is an acting rung MUST state
the maturity it was licensed at and that maturity's evidence class, and the ratifier MUST be
shown both before ratifying. Where the class is `declared`, the ceiling was met on a recorded
assumption about the signing peers, and neither the record nor any conformance claim citing it
may present the maturity as verified.

> **Implementation status:** NORMATIVE, NOT YET IMPLEMENTED in the reference implementation (tracking: #21). Applies to **stating the evidence class in the predicate text and showing it to the ratifier**: the reference implementation's maturity is a proposer assertion that carries no class.

### 6.14 The drill obligation

Demotion is a safety mechanism, and an untested safety mechanism is a liability. Before a
demotion trigger is relied upon, a deployment MUST have exercised it against a live grant and
confirmed: the rung fell to `lastSafeLevel`, the audit event was emitted, the demotion-typed
record was appended, and no model was called on the path. A promotion is likewise not complete
until the promoted grant has acted once under its new level: an unexercised grant can be
operationally dead (e.g. envelope-quarantined) while every ledger row looks correct.

---

### 6.15 The age of an input

An enforcer decides on things it read earlier: the grant, the envelope in force, the keys it
verifies with. For each input a decision depended on that was fetched from a source, the decision
point MUST record the instant as of which it established that input. An input MUST NOT be used
past a maximum age the deployment declares for it. Where a source states the time of its answer,
the as-of instant is the earlier of that time and the time of the fetch.

> **Implementation status:** NORMATIVE, NOT YET IMPLEMENTED in the reference implementation (tracking: #176).

What this prevents is a decision made on something that was true when it was fetched and is false
now: a grant demoted after it was read, an envelope tightened, a key withdrawn. With no maximum
age the old input stays in force for as long as the process that read it keeps running, so
authority removed by a write the enforcer has not re-read is still honoured. With no recorded
instant an auditor cannot tell a decision made on current inputs from one made on stale ones.

## 7. Conformance

### 7.1 Roles

Two conformance roles. The **grant issuer** hosts the ceremony: evaluates the deterministic
predicate, enforces maker≠checker, signs the ledger record, writes the grant store. The
**grant enforcer** (the broker) reads the grant per call, enforces the level's actuation
semantics, runs the demotion evaluation, and structurally cannot be written past by the agent.
One deployment component MAY implement both roles, but the agent MUST be neither. Audit
clauses (GAL-31 to GAL-33, GAL-35, GAL-37, GAL-38 and GAL-40) constrain the artifacts both roles produce and MUST be
satisfiable by a third party holding read-only access.

### 7.2 Issuer clauses

- **GAL-1** A grant SHALL bind exactly one `(principal, action-class)` pair; implementations MUST NOT expose or act on an agent-scoped autonomy level.
- **GAL-2** The `level` enumeration SHALL be complete at `in-loop`, `on-loop`, `out-of-loop`; Recommend SHALL be represented only as the absence of a grant and MUST NOT become a level value.
- **GAL-3** Level SHALL be orthogonal to the per-call decision verbs; no verb outcome may mutate a level and no level may be inferred from a verb.
- **GAL-4** The ceremony (and the automatic demotion path) SHALL be the grant store's only mutation paths (exclusivity of those paths: not yet implemented, #27); the issuer MUST refuse any other write.
- **GAL-5** The promotion licensing predicate SHALL be pure and deterministic with no model output in the path, and SHALL NEVER license a promotion on an input it cannot interpret. **Evidence** that is ambiguous (thin, absent, stale, or uncorroborated) SHALL resolve to ineligible. Input that is **structurally invalid** (a negative counter, a total exceeding its window, an unknown enumeration value) SHALL refuse loudly and SHALL NOT resolve to ineligible, because a broken evidence pipeline reported as ineligible is indistinguishable from an honest denial and directs the proposer to gather more evidence, which can never fix it. A loud refusal SHALL still produce a recorded outcome (a recorded outcome on the ratify path: not yet implemented, #32).
- **GAL-6** The predicate SHALL evaluate at least the seven gate terms of §6.4.1 in a fixed order, with unset knobs constituting no gate.
- **GAL-7** `proposedBy` and `ratifiedBy` on a promotion SHALL be credentials that **a single operator cannot satisfy both of**; the comparison SHALL reject any pair a lone operator can produce, and credential-identity equality is a floor rather than the whole test. Where an identity embeds unstable components (a hostname, a session id), the comparison SHALL additionally reject pairs sharing a ceremony role, since equality alone passes the same operator twice. The record SHALL evidence both non-repudiably. The standard enforces two credentials and evidences, not enforces, two humans.
- **GAL-8** For high-blast classes, a true predicate SHALL carry an explicit per-instance human-ratification requirement that the ceremony MUST NOT drop; an unknown blast class SHALL be an error.
- **GAL-9** High-blast SHALL derive from declared operation fields (write ∧ external ∧ ¬reversible, unknown reversibility read as not-reversible); deployments MAY tighten the derived class, MUST NOT loosen it.
- **GAL-10** Any evidence reviewer SHALL attach findings only; it MUST NOT license or veto, and reviewer failure SHALL degrade to a recorded finding, never a gate flip.
- **GAL-11** Every durable input the ceremony consumes SHALL carry tamper-evidence or be fully re-validated at the trust boundary; proposals SHALL be single-use and expiring.
- **GAL-12** The evidence window's span and period SHALL be bound under the proposal's integrity mechanism; covered-distribution soundness SHALL be recorded as an explicit proposer assertion.
- **GAL-13** Any level → `in-loop` SHALL be permitted without ceremony and SHALL append a `tightening`-typed record.
- **GAL-14** Grant creation outside the ceremony SHALL emit a `bootstrap`-typed record; no grant SHALL lack a ledger counterpart (both, on the runtime bootstrap path: not yet implemented, #27).
- **GAL-15** On envelope change, the grant SHALL be re-attested at its prior level under human ratification, and the re-attestation SHALL refuse on configuration mismatch, failing toward writing nothing. A re-attestation SHALL append exactly one `reattestation`-typed record, signed under the issuer role and carrying the same `ts` as the grant write, and SHALL change nothing on the grant but `envelopeHash`, `promotedBy` and `ts`; a re-attestation SHALL be refused when no issuer signing key is configured; no path SHALL rewrite a grant without appending the record that accounts for the write.
- **GAL-16** Every level change SHALL append exactly one typed record on one append-only ledger; records SHALL never be overwritten, mutated, or removed.
- **GAL-17** Structurally invalid transitions SHALL be unconstructible (typed refusal), including any demotion target of `out-of-loop`. A `demotion` or `lapse` record whose `toLevel` ranks above its `fromLevel`, and a `promotion` record whose `toLevel` is not exactly one rung above its `fromLevel`, SHALL be refused wherever a record is constructed or parsed, not only on the write path (the `promotion` half: not yet implemented, #159).
- **GAL-18** Ledger records SHALL be signed per §6.10 (workload-identity key, envelope hash bound); where signing is configured the issuer SHALL refuse unsigned or half-configured storage absent an explicit, recorded override (refusal by every writer once a ledger has adopted signing, and a durable record of the override: not yet implemented, #162); a tightening SHALL NOT be refused for want of a key.
- **GAL-19** Grant and ledger writes SHALL be conditional, and **no failure SHALL leave a raised grant without its ledger record**. Writing the record before the grant mutation satisfies this by ordering; committing both as one atomic transaction satisfies it by admitting no interruption. An implementation SHALL state which it provides.
- **GAL-20** Promotions to `on-loop` or `out-of-loop` SHALL require `signed-lineage` provenance maturity as a deterministic predicate term; `in-loop` as a target SHALL carry no provenance ceiling. The predicate text on a `promotion` record whose target is an acting rung SHALL state the maturity it was licensed at and that maturity's evidence class as PTC defines it, and the ratifier SHALL be shown both (stating and showing the evidence class: not yet implemented, #21).

### 7.3 Enforcer clauses

- **GAL-21** The enforcer SHALL read the grant per call; absence of a grant SHALL default-deny (Recommend), and Recommend SHALL produce no staged intent.
- **GAL-22** The agent SHALL have no write access to the grant store, enforced structurally; a grant-store write reachable by the agent is a promotion bypass and non-conformant.
- **GAL-23** Demotion SHALL be tripped only by the four triggers of §4.2, each derived deterministically from typed durable inputs, with no model call on the path.
- **GAL-24** A tripped demotion SHALL move the grant to `lastSafeLevel` **or lower, and SHALL NEVER raise its autonomy**; where `lastSafeLevel` ranks above the grant's current level the trip SHALL leave the level unchanged and still append its record. `lastSafeLevel` SHALL never be `out-of-loop`.
- **GAL-25** `demotionReason` SHALL follow the fixed trigger mapping, `"failing"` dominating mixed firings, and the two reasons SHALL never be collapsed.
- **GAL-26** Every demotion SHALL append a `demotion`-typed record (ratified by the system evaluator identity) and emit an audit event (the audit event: not yet implemented, #28); demotion writes SHALL run under an identity separate from the agent.
- **GAL-27** A trigger SHALL fire only for grants listing it in `demotionTriggers`; `false_action` SHALL derive from an authenticated durable counter, a single flag sufficing, and no flag SHALL ever promote.
- **GAL-28** `budget_breach` SHALL be derived from the same durable counter, key, and comparison that per-call enforcement uses; an unusable evidence input SHALL cause a loud refusal, never "no breach" (loud refusal on a counter-period mismatch: not yet implemented, #29).
- **GAL-29** Re-promotion SHALL require fresh recalibration evidence and satisfy dwell-time hysteresis (the hysteresis gate on the promotion path: not yet implemented, #30); demotion SHALL have no dwell.
- **GAL-30** A grant's integrity value SHALL bind its **stored bytes**, SHALL live outside the record it protects, and SHALL be verified verbatim *before* the bytes are parsed. A grant failing verification, or carrying an `envelopeHash` not in force, SHALL be quarantined loudly on every call: treated as no grant, with an audit-visible signal distinguishable from not-found.

- **GAL-34** Where a grant carries a certification term, its expiry SHALL lapse the grant to `lastSafeLevel` with `demotionReason` `"pending-evidence"` and a `lapse`-typed record; a lapse SHALL NOT be recorded as a triggered demotion, SHALL NOT revoke authority outright, and SHALL NOT be auto-renewed or extended in place by the holder. A grant carrying no term SHALL NOT lapse. Term expiry SHALL be judged against an explicit evaluation instant, never one derived from the timestamps of the records under evaluation, and from that instant the grant SHALL be enforced at the lower of its level and `lastSafeLevel` whether or not the lapse record has been written.
- **GAL-36** An in-loop approval SHALL be consumed only by release or terminal rejection of the frozen call it binds, including the authority that call was evaluated under, by the principal (the whole identity tuple) the call was frozen for; a non-terminal disposition SHALL consume nothing; consumption SHALL NOT be inferred from the approval's identifier appearing in any record, nor reversed by an indeterminate execution outcome; and the release SHALL execute the stored call and nothing re-sent, under authority verifiably equivalent to the authority approved.

  > **Implementation status:** NORMATIVE, NOT YET IMPLEMENTED in the reference implementation (tracking: #45). Applies to the **authority-equivalence half only**: a held intent carries no authority coordinate, so release verifies that current authority suffices, not that it is the authority approved. The terminal-rejection and indeterminate-outcome halves hold by construction (no non-terminal disposition exists; a release that fails after claiming the approval stays `approved`); pinning tests for both are requested in #44.
- **GAL-39** A derived grant SHALL NOT outlive or out-rank the grant it derives from, at any depth of derivation. Where a grant is demoted or lapses, every derived grant descending from it SHALL cease to confer authority from that evaluation instant, whether or not a record of the change has been written, and an enforcement point SHALL NOT admit a call whose authority descends through a node in that state.

  > **Implementation status:** NORMATIVE, NOT YET IMPLEMENTED in the reference implementation (tracking: #11).

- **GAL-41** A verifier SHALL establish, at the evaluation instant and from a source it pins, that the key which signed the record a grant's level rests on has not been withdrawn; where standing is withdrawn or cannot be established within a declared maximum age, the grant SHALL be enforced at `lastSafeLevel`. A key retired on schedule is not withdrawn (not yet implemented, #179).
- **GAL-42** Where the evaluation instant is compared with a time asserted under another party's clock, a declared skew bound SHALL be applied in the direction that confers less authority: a term or an authority-lowering record is effective up to the bound earlier, an authority-raising record or a not-before time up to the bound later (not yet implemented, #175).
- **GAL-43** For each fetched input a decision depended on, the decision point SHALL record the instant as of which it established that input, and no input SHALL be used past a maximum age the deployment declares for it; where a source states the time of its answer, the as-of instant SHALL be the earlier of that time and the time of the fetch (not yet implemented, #176).

### 7.4 Audit clauses

- **GAL-31** An independent, read-only party SHALL be able to re-verify the full ledger: signatures, transition validity, envelope binding, the no-orphan invariant, and that no record has been removed and the ledger has not been rolled back (removal and rollback detection: not yet implemented, #157).
- **GAL-32** Audit findings SHALL be dispositioned only by signed, append-only acknowledgment artifacts binding rule + coordinate + violation digest; the waivable vocabulary SHALL be closed, integrity-tamper findings SHALL be un-waivable, and an unverifiable waiver SHALL NOT be applied. A finding about a stored record SHALL bind that record's stored bytes in its detail, and a finding that cannot be so bound SHALL be un-waivable. An acknowledgment SHALL carry an expiry and SHALL NOT be applied past it (the expiry: not yet implemented, #158).
- **GAL-33** Each armed demotion trigger SHALL have been drilled against a live grant before being relied upon, and a promotion SHALL NOT be considered complete until the promoted grant has acted once. **The first act under a promoted grant SHALL be recoverable from the audit record** (the join from audit record to promotion: not yet implemented, #31) by a read-only party: the enforcement point writes it, and the ceremony ledger cannot, since the enforcing component is barred from writing the grant store.
- **GAL-37** Ledger records SHALL be signed under the role their record type names (the issuer's key for `promotion`, `bootstrap`, `tightening` and `reattestation`, a separate evaluator key for `demotion` and `lapse`), and no identity SHALL hold both roles' signing keys. A verifier SHALL select the acceptable keys from the record type, SHALL refuse a record signed by the other role's key, and SHALL refuse a key resolvable under both. Every record after a coordinate's first SHALL start from the level the ledger held immediately before it, whichever role signed it; an auditor SHALL report one that does not, and the finding SHALL be un-waivable (for records signed under the issuer role: not yet implemented, #159).
- **GAL-38** Where record signing is adopted over existing history, no record SHALL be excused from the signing requirement by its own timestamp or any other field it carries; unsigned history SHALL be dispositioned per record by acknowledgment (GAL-32) or re-minted. The adoption instant SHALL be explicit and judged against an explicit evaluation instant; one not yet in force SHALL be a finding and SHALL NOT narrow what is checked; and an undeclared adoption SHALL be reported on every audit.
- **GAL-40** A ledger record's `ts` SHALL be the instant its writer appended it, later than every record already at its coordinate and never backdated, and the same value SHALL be written to the grant in the same write. An auditor SHALL report a grant whose `ts` differs from the `ts` of the latest ledger record at its coordinate (the audit rule: not yet implemented, #166).
- **GAL-35** An auditor SHALL be able to reconcile the grant set against the live principal set in both directions (principals acting without a grant, and grants whose principal no longer exists) and SHALL report rather than write; retiring a grant remains a ceremony.

  > **Implementation status:** NORMATIVE, NOT YET IMPLEMENTED in the reference implementation (tracking: #14).

### 7.5 Clause origin mapping (non-normative)

| Clause | Origin |
|---|---|
| GAL-1 | grant-lifecycle §"The ladder"; GAL.md §4; SCHEMAS §1 |
| GAL-2 | grant-lifecycle §"The ladder"; SCHEMAS §1 `level` note |
| GAL-3 | SCHEMAS §1 `level` note; GAL.md §4 |
| GAL-4 | GAL.md §5 (ceremonies as the only sanctioned mutation path); grant-lifecycle §"write-protection seam" |
| GAL-5 | GAL.md §5; deterministic-gate.md; predicate design (fail-safe) |
| GAL-6 | predicate gate order (provenance / artifact / fresh / covered / budget / observations / rate) |
| GAL-7 | GAL.md §8 (#202: two credentials, evidences two humans) |
| GAL-8 | grant-lifecycle §Promotion (high-blast always human); predicate `requires_human_ratification` |
| GAL-9 | GAL.md §3 (actionClass derives; tighten-only high-blast); SCHEMAS §"BlastClass derivation" |
| GAL-10 | grant-lifecycle §Promotion (evidence reviewer); deterministic-gate.md (surface-vs-decide) |
| GAL-11 | grant-lifecycle §"input-integrity corollary" |
| GAL-12 | GAL.md §9 (covered assertion recorded); multi-period window binding (#207/#212) |
| GAL-13 | GAL.md §4 (any-level → in-loop); SCHEMAS §7 `tightening` |
| GAL-14 | GAL.md §5 (`seed` emits bootstrap record); SCHEMAS §7 `bootstrap`; L2 |
| GAL-15 | GAL.md §5 (re-attestation carries prior level); grant-lifecycle §audit (#199 refusal seam). Reversed in 0.5.0-draft: re-attestation appends a record (§6.6), built in the reference implementation under #164. Amended in 0.5.1-draft: the rule is no write without a record, and the record is always issuer-signed. |
| GAL-16 | SCHEMAS §7 (append-only ledger); L2 |
| GAL-17 | L1 (transition unconstructibility); grant-lifecycle §ladder |
| GAL-18 | GAL.md §8; tce-signing-shape.md; grant-lifecycle §audit (refuse-unsigned) |
| GAL-19 | GAL.md §11 Phase 4 (conditional writes, no raised grant without its record); L6 |
| GAL-20 | PTC.md §9; predicate `REQUIRED_PROVENANCE_MATURITY`; GAL.md §9 |
| GAL-21 | grant-lifecycle §"Recommend → in-loop line" |
| GAL-22 | grant-lifecycle §"write-protection seam" |
| GAL-23 | grant-lifecycle §Demotion (exactly four, deterministic); L3 |
| GAL-24 | grant-lifecycle §ladder (`lastSafeLevel`); L5 |
| GAL-25 | grant-lifecycle §"Record why you demoted"; L4 |
| GAL-26 | grant-lifecycle §Demotion (demotion-typed record; separate identity); SCHEMAS §7 |
| GAL-27 | grant-lifecycle §triggers 4 (`false_action`, ease gradient, ships OFF) |
| GAL-28 | L8 (budget-breach derivation parity); #192 refuse-on-unusable-input |
| GAL-29 | grant-lifecycle §Hysteresis; L7 |
| GAL-30 | GAL.md §3 (HMAC + loud quarantine); grant-lifecycle §audit (`GRANT_ENVELOPE_IN_FORCE`); L6 |
| GAL-31 | GAL.md §11 (grant-integrity audit); grant-lifecycle §"audit instrument" |
| GAL-32 | grant-lifecycle §"audit instrument" (#196 acknowledgment ceremony) |
| GAL-33 | grant-lifecycle §"This must be drilled"; lessons doctrine ("a promotion is not done until the promoted grant acts once") |
| GAL-34 | Five Eyes *Careful adoption of agentic AI services* (2026-05-01), the "expiry timers and recorded grant chains" pairing; `docs/references/five-eyes-agentic-guidance.md` FE-1. The evaluation-instant sentence answers an implementer finding against the IETF WIMSE cross-org delegation draft (finding 4, expired authority kept alive through quiet periods). |
| GAL-35 | Five Eyes *Careful adoption of agentic AI services*, "periodically reconcile the registry against the live set of agents"; `docs/references/five-eyes-agentic-guidance.md` FE-2. Normative ahead of the reference implementation (§3; tracking #14). |
| GAL-36 | GAL §4.1; reference implementation `broker/runtime/pep.py` (`_owns_intent`, the full-principal ownership check) and `approval/`; an implementer finding against the IETF WIMSE cross-org delegation draft (approvals burned by a third party citing their identifier). |
| GAL-37 | GAL §6.10 (this revision); §6.7.2's separate evaluator identity read together with §6.10's sign-everything requirement. Reference implementation `broker/grants/record_signing.py` (`RECORD_TYPE_SIGNING_ROLE`, `verify_record_by_type`) and the IAM namespace split in `infra/lib/identity-stack.ts`. |
| GAL-38 | GAL §6.10; reference implementation `RECORD_SIGNING_EPOCH`. Rewritten in 0.3.0-draft: the exemption by instant it first described was claimable by a planted unsigned record. |
| GAL-39 | §6.7 read forward onto derivation; `auto-agents/book/ch41` §"Demotion propagates down the chain". Answers the lifecycle half of an implementer finding against the IETF WIMSE cross-org delegation draft (finding 3, a broken path neutralizing a valid one); the other half, resolving WHICH path applies where several reach one agent, belongs to the delegation mechanism and is out of scope (§1.3). Normative ahead of the reference implementation (§3; tracking #11). |
| GAL-40 | GAL §5.2 and §6.11 (0.5.0-draft); reference implementation `broker/grants/ledger_clock.py` (`next_ledger_ts`). Answers an outside question on what a ledger record's `ts` denotes ([ptc-gal-standards#2](https://github.com/wjatx/ptc-gal-standards/issues/2)). The audit rule is normative ahead of the reference implementation (§3; tracking #166). |
| GAL-41 | GAL §6.10 (0.6.0-draft). The verifier-side statement is `draft-jackson-wimse-evaluation` §3.5; this is the lifecycle's half, for withdrawal and not rotation. |
| GAL-42 | GAL §6.7.6 (0.6.0-draft). The asymmetric form is stated for a verifier in `draft-jackson-wimse-evaluation` §3.2; prior art for inputs that fail toward less authority is the lease (Gray and Cheriton, 1989). |
| GAL-43 | GAL §6.15 (0.6.0-draft). Companion to PTC's clause of the same content. |

### 7.6 Implementation Conformance Statement

The clause tables of §7.2–§7.4 constitute an **implementation conformance statement proforma**
in the sense of ISO/IEC 9646-7. This section defines how one is completed and what a conformance
claim consists of. The convention is shared with the companion PTC specification, which mirrors
it in its §8.4.

**A conformance claim is a completed statement, not an assertion.** An implementation claiming
conformance SHALL publish, for each role it claims, a support answer against every clause bound
to that role. Answers are drawn from a closed set:

| Answer | Meaning |
|---|---|
| `Y` | Supported. The implementation satisfies the clause as written. |
| `N` | Not supported. The implementation does not satisfy the clause; the claim for that role fails. |
| `N-A` | Not applicable. Permitted **only** where the clause carries a conditional predicate that the implementation does not meet. |

**Every clause is mandatory within its role.** This specification defines no optional clauses:
claiming a role claims all of that role's clauses, and `N-A` is never available merely because a
capability was not built. Where a clause permits a deployment choice (GAL-9's allowance to tighten
a derived class, for example), the permission is *inside* a mandatory clause and does not make the
clause optional. An implementation that answers `N` anywhere is not a conforming implementation of
that role and SHALL NOT describe itself as one; it may of course describe itself accurately, and
a published statement with an `N` in it is more useful to a reader than a withheld one.

**The reference implementation's statement is derived, never hand-written.** Its answers follow
mechanically from the implementation-status markers of §3: an unmarked clause answers `Y`, a clause
marked NOT YET IMPLEMENTED answers `N`. Deriving rather than authoring means the statement cannot
drift from the markers, and the markers cannot drift from the clause list, because both are
extracted from this document.

**Traceability is bidirectional, and the second direction is the load-bearing one.** Every clause
SHALL trace *downward* to a conformance test or an explicit recorded gap, and every conformance
test SHALL trace *upward* to the clause id it exercises. The downward direction answers "is this
requirement verified"; the upward direction answers "does this verification correspond to any
requirement", which is the only direction that detects a test suite accumulating confidence about
something the specification never asked for. The objective is borrowed from DO-178C §5.5 and §6.5.
**Only the objective**: this specification adopts no part of that standard's process, assurance
levels, coverage criteria, or tool qualification, and an implementation's conformance here is not
a claim about airborne-software certification.

---

## 8. Security considerations and open problems

### 8.1 Evidence poisoning

Promotion evidence is accumulated from live behavior, and that behavior can be shaped. Two cases
behave differently and must not be conflated, because a mitigation for the first does nothing
about the second.

**Tainted grooming.** An injected input shapes behavior and the turn carries taint. Excluding
tainted-turn outcomes starves evidence (an availability lever: poison inputs to freeze a
capability at a low rung); including them lets an adversary who can taint a turn shape what the
window records. The likely direction is taint-aware evidence windows with weighting as deployment
policy, but this is information-flow-hard (the same class of problem as declassification) and
this specification deliberately does not specify it (§1.3). Deployments SHOULD treat evidence
windows overlapping known-tainted activity with suspicion at ratification; the covered-distribution
assertion (§6.4.2) is the accountability anchor for exactly this judgment.

**Untainted grooming.** A patient adversary, or ordinary drift with no adversary at all, can
produce a *clean* behavioral record that the predicate rewards. No turn is tainted, so there is
nothing for a taint-aware window to weight and the SHOULD above does not fire. Deployments MUST
NOT treat the absence of taint in an evidence window as evidence that the window was not groomed.

The structural reason is worth stating plainly, because it bounds what any evidence predicate can
be trusted to do. Taint is a ratchet: it accumulates, the subject cannot lower it, and it therefore
cannot be farmed. An evidence window that licenses promotion on accumulated clean behavior is the
opposite shape, a credit mechanism whose state the subject improves through its own conduct. Any
control whose state the subject can improve is farmable by a subject patient enough to improve it
honestly first.

This specification does not resolve untainted grooming, and no weighting scheme closes it, because
there is no signal to weight. What bounds it is the ceremony rather than the predicate: promotion
requires maker≠checker ratification (§6.4) that the accumulating party cannot supply, so groomed
evidence must still pass a second credential. Deployments SHOULD therefore treat the promotion
predicate as a filter and never as an authorization, and ratifiers SHOULD read a clean evidence
window as the absence of recorded trouble rather than as positive evidence of trustworthiness.

### 8.2 Label latency bounds autonomy

Re-promotion's control loop has deadtime equal to `labelLatency`. Domains with very slow
ground truth may never soundly accumulate the recalibration evidence that `out-of-loop`
requires. **That is a correct result, not a defect**: a domain in which you cannot tell for
months whether an action was right is a domain in which full autonomy cannot be evidenced.
This specification prefers an honest ceiling to an unsound promotion, and states the
consequence normatively expecting it to be contested.

### 8.3 Who ratifies the ratifier

`ownerId` is the root of trust for ratification. At a single named owner this is clean; the
N>1 fan (quorum ratification, key rotation mid-window, delegation) is unspecified in this
version (§1.3). The honest bound (§6.4.3): the standard enforces two *credentials* and evidences two
*humans*. Against a malicious sole operator it provides non-repudiation, not prevention.

### 8.4 The enforcement point is load-bearing

Every property here assumes the enforcer cannot be bypassed: the agent holds no credentials,
its only egress is the broker, and the grant store is structurally write-protected from it
(GAL-22). A deployment that lets the agent reach a tool directly, or reach the grant store
with any write, has no autonomy lifecycle, only autonomy paperwork.

---

## 9. Extensibility

### 9.1 The trigger set is closed in this specification

The demotion-trigger vocabulary is exactly the four values of §4.2; deployments arm a subset
per grant via `demotionTriggers` and MUST NOT extend the vocabulary. The two derivation-heavy
triggers (`stale_confidence`, `corroboration_failure`) deliberately consume *typed evidence as
given*: the detectors producing the evidence are deployment-tier and freely extensible, so
domain-specific demotion logic lives in what a deployment feeds the triggers, not in new
trigger values. Widening the vocabulary changes what every auditor must understand a demotion
record to mean: a specification change, not a knob.

The time-based **lapse** arc (§6.7.6) is deliberately not a fifth trigger, for the same reason
stated from the other side. A trigger record asserts an observation; a lapse asserts that no
renewal occurred. Collapsing them would leave `triggeredBy` naming a condition that never fired,
and would blur the single distinction the demotion vocabulary exists to preserve: whether
something is broken or merely uncertified.

### 9.2 Knobs versus floor

The deployment extension surface is the knob inventory, under the floor-vs-knob rule: the
non-negotiable floor is the mechanism and its asymmetry (the ceremony as the only upward path,
deterministic demotion, the signed ledger, the structural write-protection); every bar,
budget, threshold, dwell, reviewer, and transparency anchor is a deployment knob shipping OFF.
A conforming implementation MUST NOT require modification of the base implementation to
express a deployment's policy stance, and MUST NOT present a control as enabled when it gates
nothing.

---

## 10. Version history and references

### 10.1 Version history

| Version | Date | Notes |
|---|---|---|
| `0.1.0-draft` | 2026-07-24 | Initial draft for Linux Foundation agent-standards discussion. |
| `0.2.0-draft` | 2026-07-25 | Adds the time-based **lapse** arc (§6.7.6, `certifiedUntil`, the `lapse` record type, GAL-34): a certification has a term, and its expiry is deliberately *not* a fifth demotion trigger, since a lapse asserts an absence rather than an observation. Adds reconciliation of the grant set against the live principal set in both directions (§6.11, GAL-35), the shadow-agent case no store-only check can see. Corrects §5.1: the grant carries no integrity field of its own, and §6.9/GAL-30 now state the stored-bytes basis (a hash inside the structure it protects cannot cover the bytes as stored) plus the rule that integrity must indict tampering, never schema evolution. |
| `0.2.1-draft` | 2026-07-29 | Introduces the **implementation-status marker** (§3) and applies it to the two clauses that are normative ahead of the reference implementation, GAL-34 / §6.7.6 (the lapse arc, tracking #255) and GAL-35 / §6.11 (two-direction reconciliation, tracking #256), so no reader can mistake either for a shipped control. Records the shipped `attestation` field on `PromotionRecord` (§5.2), which states when one operator held both ceremony roles and so keeps §6.10's non-repudiation claim honest. Pins the canonical JSON encoding for all GAL objects (§3), previously deferred despite §6.9's stored-bytes integrity basis making it interoperability-critical. Resolves the `evidence` reference format as an opaque, integrity-bound, implementation-defined string (§5.1, §5.2) and pins the `CorroborationRecord` field set (§5.3), including the deliberate exclusion of per-source provenance. Adds the audit's honest limit (§6.11): it verifies signatures, never the cited evidence. Corrects §4.3/§5.2, which said "four record types" over a five-row table. |
| `0.2.2-draft` | 2026-08-03 | Corrects §8.1 (Evidence poisoning), whose mitigation direction and normative SHOULD were both keyed on taint while the attack they name does not require it (#342). Grooming a promotion needs no tainted turn: a patient adversary, or drift with no adversary, can produce a clean behavioral record that the predicate rewards, leaving a taint-aware evidence window nothing to weight. §8.1 now separates tainted from untainted grooming, keeps the existing taint-aware guidance scoped to the first, adds a MUST NOT against reading absence of taint as absence of grooming, and states the structural asymmetry that motivates both: taint is a ratchet and cannot be farmed, whereas an evidence window rewarding accumulated clean behavior is a credit mechanism whose state the subject improves through its own conduct. Names maker≠checker ratification (§6.4), not the predicate, as what bounds the untainted case, and directs ratifiers to read a clean window as absence of recorded trouble rather than as positive evidence of trustworthiness. No clause, schema, or wire change; §8.1 carries no conformance clause. |
| `0.2.3-draft` | 2026-09-19 | The lapse arc is implemented: removes the NOT YET IMPLEMENTED markers from §4.3, §5.1, §6.7.6 and GAL-34. §6.7.6 and GAL-34 gain the **evaluation instant** (explicit, never derived from records), enforcement at the lower of level and `lastSafeLevel` from the boundary without waiting for the record, and the after-lapse rules. §5.2 adds `lapse` to `recordType` and the ratified `certifiedUntil` to promotion records, and fixes `demotionReason`'s type scope, which contradicted §4.3's lapse row. §3 pins that a null optional field is omitted from the canonical form; `Grant.certifiedUntil` becomes optional accordingly. §4.1 and new clause GAL-36 make in-loop approval consumption keyed on the bound call and the whole principal. §6.11 makes an unexplained level drop, and a term that differs from the ratified one, audit findings. §1.2 adds delegation path resolution and token-conveyed authority as non-goals. The evaluation-instant rule, GAL-36 and the delegation non-goal respond to implementer findings against the IETF WIMSE cross-org delegation draft. |
| `0.2.4-draft` | 2026-09-19 | Scopes signing authority by record type in §6.10, with clauses GAL-37 and GAL-38. §6.10 had required every ledger record to be signed while §6.7.2 required the automatic evaluator to be a separate identity, and the reference implementation signed only promotions. The two requirements are jointly satisfiable only through two keys: an issuer key for the records that raise or re-license authority and a separate evaluator key for demotion and lapse, with verification selecting the acceptable keys from the record type and refusing a key resolvable under both. §6.10 also states how a deployment adopts signing over history it cannot re-sign: an explicit recorded instant, its extent reported rather than passed silently, and an instant not yet in force reported as a finding. |
| `0.2.5-draft` | 2026-09-19 | Moves delegation path resolution out of §1.2 (normative exclusions) into §1.3 (future extensions), and narrows it. 0.2.3 had excluded "path resolution and per-path fail-closed semantics" as one item; the second half is lifecycle, which this standard owns, and §6.7's demotion rule already determines when authority along a path stops being valid. The exclusion as written disclaimed a rule this specification is positioned to state. §1.3 now separates the two: resolving *which* path applies where several reach one agent belongs to the delegation mechanism, while a derived grant neither outliving nor out-ranking the grant it derives from is reserved for the version that adds derived grants. Conveying authority in a token stays a §1.2 non-goal, unchanged. No clause, schema, or wire change. |
| `0.2.6-draft` | 2026-09-20 | Rewrites §10.2 to name the public reference implementation, [ptc-gal-reference](https://github.com/wjatx/ptc-gal-reference), which the section did not previously mention: it named a private repository a reader cannot open. Restates the claim as one about the CODE rather than about a deployment the authors operate, and gives the reader the procedure to check it from a checkout with no cloud account. Removes the assertion that the grant-integrity audit runs "continuously in CI on two environments": that workflow is manually triggered and had not run in eight weeks. The end-to-end deployment drills are retained but relabelled as recorded history rather than reader-verifiable evidence. Non-normative section; no clause, schema, or wire change. |

| `0.2.7-draft` | 2026-09-21 | Adds **GAL-39**: a derived grant may neither outlive nor out-rank the grant it derives from, and a demotion or lapse at any node stops every path descending through it from the evaluation instant, whether or not the record has been written. Normative ahead of the reference implementation and marked accordingly (§3; tracking #11). §1.3 is rewritten: 0.2.5 had articulated this rule in prose and then declined to state it as a clause "because no conforming implementation has a derived grant to apply it to", which set this specification's requirement to the reference implementation's current coverage. §3's marker convention exists precisely so a settled design question can be stated at the specification tier while the code catches up, and GAL-35 already used it; withholding GAL-39 understated what GAL requires. §1.3 now retains only the half that is genuinely not ours, resolving which path applies where several reach one agent. Answers the lifecycle half of an implementer finding against the IETF WIMSE cross-org delegation draft. |

| `0.2.8-draft` | 2026-09-21 | Repoints every implementation-status marker at an issue in the PUBLIC reference implementation. The markers previously cited the private working tracker: of the twenty issues cited across both specifications, nineteen resolved only in a repository no reader of the published text can open, while §3 states that `#NNN` is "the reference implementation's public tracking issue for the work". A marker's credibility rests on that pointer being chaseable, since the marker is what lets a clause be normative ahead of the code without overclaiming. Nineteen issues were filed in ptc-gal-reference and the live citations renumbered; historical changelog entries keep their original numbers, because they record what was true when written. No clause text, schema, or wire change. |

| `0.2.9-draft` | 2026-09-21 | Repoints §1.3's N>1 ratifier entry at [ptc-gal-reference#33](https://github.com/wjatx/ptc-gal-reference/issues/33). The previous citation was worse than unreachable: it named an issue in a private tracker that described unrelated work, a channel-routing fan whose "N" counts mapped principals rather than ratifiers. A reader who could open it would have been misled, and a reader who could not was told nothing. The new issue states the three questions §8.3 actually groups under the N>1 fan (quorum ratification, ratifier-key rotation inside an evidence window, and delegated ratification) and distinguishes the last from agent-to-agent delegation of action authority, which is GAL-39. Non-normative; no clause, schema, or wire change. |

| `0.2.10-draft` | 2026-09-29 | Amends §4.1 and **GAL-36** to agree with `draft-jackson-wimse-evaluation-02` §3.1 and §4.3, which cites GAL-36 as the source of its consumption rule and had moved past it. Three changes. The frozen call includes the authority it was evaluated under, and an enforcement point must be able to verify that the authority a call is released under is equivalent to the authority approved; the mechanism is left open. Rejection means only the terminal disposition, so a deferral or a request for changes consumes nothing. Consumption is keyed on closure, never on execution outcome, so an indeterminate outcome after release does not return the approval to pending. The authority-equivalence half is marked NOT YET IMPLEMENTED (tracking: [ptc-gal-reference#45](https://github.com/wjatx/ptc-gal-reference/issues/45)); the other two hold in the reference implementation by construction. Normative change to one clause; no schema or wire change. |
| `0.3.0-draft` | 2026-10-02 | Corrects defects in the ledger's integrity contract, each found by attacking the reference implementation and each a gap in this specification as well as in the code. (1) §4.3 left a record's direction to the state machine, so a `demotion` or `lapse` that raised a level was constructible by anything that did not pass through it; the direction is now a shape rule (§4.3, §6.2, **GAL-17**). (2) §6.10 adds continuity: a record the evaluator signs starts from the level the ledger held immediately before it, and a break is an un-waivable finding (**GAL-37**). (3) §6.10 and **GAL-38** exempted records dated before signing was adopted, which an unsigned planted record could claim by choosing its date; no record is now excused by anything it says about itself, and unsigned history is acknowledged per record or re-minted. (4) §6.11 and **GAL-32** require a finding about a stored record to bind that record's stored bytes, so an acknowledgment cannot cover a record that replaced the one it was issued for. (5) §6.10 no longer says a signed record proves who proposed and who ratified: it is the issuer's statement about identities it authenticated. Adds two requirements marked NOT YET IMPLEMENTED: an acknowledgment carries a term (§6.11, GAL-32), and removal and rollback are detectable (§6.11, **GAL-31**), which §6.11 already asserted and record signatures do not provide. Editorial, with no change of requirement: the body text no longer uses dashes as punctuation, and the inline marker's short form changes with that, separating the issue number with a comma (`(not yet implemented, #NNN)`) where it used a dash. |
| `0.3.1-draft` | 2026-10-02 | Closes a gap in the ledger's transition validity for records the issuer signs, found while triaging the reference implementation's markers. 0.3.0 made direction a shape rule and continuity an audit rule for the evaluator's records only; a `promotion` that skips a rung was refused by the ceremony but constructible and parseable outside it, and a `promotion` or `tightening` that broke continuity with the ledger went unreported. §4.3 and **GAL-17** now refuse a `promotion` whose `toLevel` is not exactly one rung above its `fromLevel` wherever a record is constructed or parsed, and §6.10 and **GAL-37** extend continuity to every record after a coordinate's first. Both are marked NOT YET IMPLEMENTED (tracking: [ptc-gal-reference#159](https://github.com/wjatx/ptc-gal-reference/issues/159)). Normative change to two clauses; no schema or wire change. |
| `0.4.0-draft` | 2026-10-03 | Follows PTC `0.4.0-draft`, which redefines the `signed-lineage` tier: a verified signature no longer suffices, and the receiver must have on record, per signing key, that the signer holds it where its own agent cannot reach it, with the evidence class of that record (`declared` or `attested`). §4.5 restates the tier, which had said "the receiver cannot be lied to", and introduces the evidence class. §6.13 and **GAL-20** make the ceiling inherit it: a promotion into an acting rung states the maturity it was licensed at and that maturity's evidence class in its predicate text, and the ratifier is shown both, so a ceiling met on a `declared` record is never presented as verified. That requirement is normative ahead of the reference implementation and marked (tracking #21). No object or wire change. |
| `0.5.0-draft` | 2026-10-04 | The ledger journals every write to a grant, and a timestamp says one thing. Answers [ptc-gal-standards#2](https://github.com/wjatx/ptc-gal-standards/issues/2), which asked what a ledger record's `ts` denotes for each record type. (1) §5.2 states one rule with a per-type table: `ts` is the instant the writer appended the record, later than every record at its coordinate, never backdated, and it does not establish the time of any event not separately retained. §5.1 had defined the grant's `ts` as "when this level took effect", which did not hold after a lapse or a re-attestation; it is now the instant of the last write, equal to the record's. §6.7.6's evaluation-instant rule is unchanged. (2) **GAL-15** and §6.6 are reversed: re-attestation appends a record, of a sixth type, `reattestation` (§4.3), issuer-signed, with no predicate, no triggers and no level change. 0.2-series drafts said it appended none, for two reasons. The first was that the ledger records level changes, so an entry no transition explains had no place on it; the record's type is what explains it, and without one the signed ledger could not say who rewrote the grant beside it. The second was that a record "would make the re-promotion reference ambiguous", a phrase this specification never defined. It came from the reference implementation's name for the level a later promotion climbs back toward, and the concern it stands for is a reader mistaking a same-level record for the one that earned the level; §4.3 now tells every such reader to pass over the type. The same clause forbids any other same-level rewrite of a grant. (3) A `lapse` record may carry `certifiedUntil`, the term that expired, so the instant enforcement fell is a field and no reader has to parse `evidence` for it (§5.2, §6.7.6). (4) New audit clause **GAL-40** (§6.11): a grant's `ts` equals the `ts` of the latest ledger record at its coordinate. (5) §6.8: dwell is measured from ledger records and a re-attestation does not restart it. The `reattestation` record, the lapse field and the audit rule are marked NOT YET IMPLEMENTED (tracking: [ptc-gal-reference#164](https://github.com/wjatx/ptc-gal-reference/issues/164), [#165](https://github.com/wjatx/ptc-gal-reference/issues/165), [#166](https://github.com/wjatx/ptc-gal-reference/issues/166)). Also corrects §10.2's clause count, which had not been updated since the clause set grew. |
| `0.5.1-draft` | 2026-10-07 | The `reattestation` record is built in the reference implementation ([ptc-gal-reference#164](https://github.com/wjatx/ptc-gal-reference/issues/164)), so its NOT YET IMPLEMENTED markers come off in §4.3, §5.2, §6.6, §6.10 and **GAL-15**. Building it found that GAL-15's last clause, "no other path SHALL rewrite a grant at an unchanged level", contradicted §4.3, which lets a `demotion` record's `toLevel` equal its `fromLevel`: a demotion that fires on a grant already at its floor rewrites the grant at an unchanged level and appends a `demotion` record. The clause and §6.6 now state the rule that was meant: no write to a grant goes unrecorded. Two requirements are added to the same clause. A `reattestation` record is always signed under the issuer role, and a re-attestation is refused when no issuer signing key is configured; the bootstrap override of §6.10 does not extend to it, because re-attestation returns a quarantined grant to acting authority and an unsigned record of that names nobody who can be held to it. §6.6 states what the two rules prevent. The markers for [#165](https://github.com/wjatx/ptc-gal-reference/issues/165) and [#166](https://github.com/wjatx/ptc-gal-reference/issues/166) are unchanged. |
| `0.6.0-draft` | 2026-10-07 | Three clauses are added, each stated with what it prevents and each marked NOT YET IMPLEMENTED, and two earlier rulings are published. (1) **GAL-41** (§6.10), issuer standing: a verifier establishes that the key behind a grant's level has not been withdrawn, and a grant whose issuer's standing is withdrawn or unknown is enforced at `lastSafeLevel`. Validity when signed and standing when evaluated are different questions, and only the first was asked. Rotation is not withdrawal. (2) **GAL-42** (§6.7.6): clock skew is applied toward less authority and never symmetrically. (3) **GAL-43** (new §6.15): every fetched input a decision depended on has a recorded as-of instant and a declared maximum age. (4) **GAL-18** gains the qualifier §6.10 always carried, that the issuer refuses unsigned storage *where signing is configured*, and is marked for the two things the reference implementation does not yet do: refuse in every writer once a ledger has adopted signing, and leave a durable record of the override. A tightening is never refused for want of a key; where GAL-13 and GAL-18 meet, GAL-13 takes precedence. (5) §3 states that the lifecycle is proven by test and has no proven-in-use evidence in the sense of IEC 61508, so its thresholds and the false-positive rate of its triggers are uncalibrated. Follows PTC `0.5.0-draft`, which carries the companion clauses on skew and input age. |

### 10.2 Reference implementation

**[ptc-gal-reference](https://github.com/wjatx/ptc-gal-reference)** implements both conformance
roles: its ceremony command set is the reference issuer, its broker the reference enforcer. Its
L1–L8 lifecycle conformance suite seeds this specification's conformance tests.

**The claim here is about the code, not about any deployment its authors operate.** Clone the
implementation, clone these specifications into a `spec/` directory at its root, and run its
test suite: the
tests that compare specification text against the shipped schemas then execute rather than skip,
and its continuous integration does exactly that on every commit. Across both specifications 42 of
96 conformance clauses are marked normative ahead of the implementation and individually tracked
(§3); none has been outgrown by the implementation. What a reader can check from a checkout, with
no cloud account and no credential, is the whole of what is claimed here.

The lifecycle has separately been drilled end-to-end against deployed infrastructure: a real
principal promoted on real approval evidence under maker≠checker with a distinct least-privilege
identity per leg, all four demotion triggers fired against real grants, re-attestation after a real
envelope change, and a full-arc terminal proof (evidence → propose → ratify → exercise → induced
demotion → re-climb → exercise) completed 2026-07-15. Those runs are recorded history, not
reproducible from a checkout, and are offered as such rather than as evidence a reader can verify.

### 10.3 References

- **[RFC2119]** Bradner, S., "Key words for use in RFCs to Indicate Requirement Levels",
  BCP 14, RFC 2119, March 1997.
- **[RFC8174]** Leiba, B., "Ambiguity of Uppercase vs Lowercase in RFC 2119 Key Words",
  BCP 14, RFC 8174, May 2017.
- **[SHERIDAN]** Sheridan, T. B. and Verplanck, W. L., "Human and Computer Control of Undersea
  Teleoperators", MIT Man-Machine Systems Laboratory, 1978. The origin of the
  levels-of-autonomy vocabulary the ladder adopts.
- **[J3016]** SAE International, "Taxonomy and Definitions for Terms Related to Driving
  Automation Systems for On-Road Motor Vehicles". The descriptive-taxonomy prior art: a label
  on a system, not stored state an enforcement point reads per call.
- **[PCCP]** U.S. FDA, "Marketing Submission Recommendations for a Predetermined Change
  Control Plan for Artificial Intelligence-Enabled Device Software Functions". The regulatory
  precedent for pre-authorized change inside a declared envelope; GAL's pre-authored signed
  predicate is this pattern as a runtime object.
- **[ODD]** Operational Design Domain (SAE J3016; BSI PAS 1883). A capability claim holds
  only inside the declared domain; GAL's covered-distribution requirement is its lifecycle form.
- **[JCS]** Rundgren, A., Jordan, B., and Erdtman, S., "JSON Canonicalization Scheme (JCS)",
  RFC 8785, June 2020. The canonical-JSON profile §3 is adjacent to and MAY be satisfied by.
- **[DSSE]** Dead Simple Signing Envelope, https://github.com/secure-systems-lab/dsse.
- **[IN-TOTO]** in-toto Attestation Framework, https://github.com/in-toto/attestation.
- **[PTC-SPEC]** *PTC: Provenance & Trust Context, Specification*, `PTC-SPEC.md` (sibling
  draft): normative for the provenance-maturity ladder (§6.13 here), the signing shape (§6.10
  here), and the deterministic per-call gate the enforcer implements.
