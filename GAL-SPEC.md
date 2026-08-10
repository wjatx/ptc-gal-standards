# GAL — Grant & Autonomy Lifecycle, Specification

| | |
|---|---|
| **Version** | `0.2.2-draft` |
| **Status** | Draft for Linux Foundation agent-standards discussion. Wire schemas may change before 1.0; see Open Problems and Future Extensions. |
| **Date** | 2026-08-03 |
| **Working group** | LF Edge + Agentic AI Foundation (AAIF) |
| **Author** | Wes Jackson (Red Hat) |
| **Copyright** | © 2026 Red Hat, Inc. |
| **License** | Community Specification License 1.0 · `SPDX-License-Identifier: Community-Spec-1.0`. See `LICENSE.md` for terms and `NOTICE.md` for attribution, acceptance, and patent exclusions. |
| **Sibling specification** | `PTC-SPEC.md` — *PTC: Provenance & Trust Context, Specification* (drafted in parallel). This document cross-references PTC by name and section topic, never by section number. |

GAL specifies the autonomy layer of an agentic control system: an agent's authority to act is
a per-`(principal, action-class)` **grant** whose autonomy level is stored, signed state —
moved **upward** only by a maker≠checker ceremony licensed by a deterministic predicate over
measured evidence, moved **downward** automatically by deterministic demotion triggers, and
enforced per call by a broker the agent cannot write past. Levels-of-autonomy taxonomies name
the rungs; GAL makes the rung itself a signed runtime object that must earn its level and
demotes itself.

---

## 1. Scope and non-goals

### 1.1 In scope

The **Grant** object (§5.1); the **autonomy ladder** — four rungs, three grant levels, and the
actuation discriminator between them (§4.1); the **promotion ceremony** — the only sanctioned
upward path, a deterministic licensing predicate composed with maker≠checker ratification
(§6.4); **automatic demotion** — a closed set of deterministic triggers, the fall-back level,
and hysteresis (§6.7); **record signing** — every level change as a signed, append-only ledger
record (§6.10); the **audit obligation** (§6.11); and the **PTC join** — the
provenance-maturity ceiling on acting rungs (§6.13).

### 1.2 Non-goals (normative exclusions)

- **The per-agent safe-default polarity.** Whether "safe" for a given agent means abstention
  (abstain-is-safe) or a positive deterministic action (a SCRAM/failsafe) MUST be re-derived
  per agent and MUST NOT be specified by this standard or baked into any conforming base
  implementation — a base-level polarity default is a latent safety bug: the polarity that
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

### 1.3 Future extensions (out of scope for this version)

- **N>1 quorum ratification and owner-key rotation.** This draft is honest about the N=1
  operator case (§6.4.3, §8.3); quorum ratification and ratifier-key rotation
  mid-evidence-window are tracked for a future version (reference-implementation tracking
  issue #187).
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
implementation note:* MAY name concrete technology (safe-agents; §10.2). Field tables pin
field names and abstract types.

**Implementation status markers.** This specification is derived from a running reference
implementation (§10.2) rather than drafted in advance of one, and the normative text is
overwhelmingly a description of mechanism that exists and has been exercised. A small number of
clauses are deliberately normative *ahead* of that implementation, because the design question
was settled and the specification tier is the honest place to settle it while the code catches
up. Those clauses are marked individually, wherever a reader can encounter them, with a line of
exactly this form:

> **Implementation status:** NORMATIVE, NOT YET IMPLEMENTED in the reference implementation (tracking: #NNN).

The marker is machine-readable on purpose: its literal prefix `**Implementation status:**` can
be grepped, and `#NNN` is the reference implementation's public tracking issue for the work. A
clause carrying the marker is fully normative — an implementation claiming conformance MUST
satisfy it — and simultaneously a statement that the reference implementation does not yet.
**Absence of the marker means the clause is implemented in the reference implementation.** No
conformance claim, by this project or any other, may cite a marked clause as a shipped control;
a specification clause is a requirement, never evidence that anything enforces it. Where a
marked clause owns a field-table row, the row carries the short inline form
`(not yet implemented — #NNN)`, which means the same thing.

**The inverse marker.** A derived specification drifts in two directions, and the marker above
records only one of them. Where the reference implementation has replaced a mechanism this
specification mandates with one that holds the clause's *purpose* more strongly, the clause is
marked with a line of exactly this form:

> **Implementation status:** NORMATIVE, SATISFIED BY A STRONGER MECHANISM in the reference implementation (tracking: #NNN). <Named mechanism, and why it dominates the one this clause names.>

A table row carries the short inline form `(stronger mechanism — #NNN)`. Four rules govern it,
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
   mechanism as one way to hold it — at which point the marker is removed.

**Absence of either marker** means the clause is implemented in the reference implementation as
written.

**Canonical encoding.** Every GAL object defined in §5 MUST be serialized, whenever it is
stored, hashed, or signed, in one canonical JSON profile: **object keys sorted in ascending
code-point order, no insignificant whitespace (compact separators), and ASCII output** (any
non-ASCII character escaped). This is not a presentation preference. §6.9 makes the *stored
bytes* the integrity basis and §6.10 signs a statement over them, so two implementations that
serialize the same object differently cannot verify each other's records — the encoding is
interoperability-critical and is therefore pinned here rather than deferred. The profile is
adjacent to JSON Canonicalization Scheme [JCS], and an implementation MAY use JCS directly;
where the two disagree on a construct, an implementation MUST state which it emits. A
transport MAY re-encode an object in flight, but a verifier MUST verify the stored bytes
verbatim and MUST NOT re-serialize before verifying (§6.9).

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

### 4.2 Demotion triggers (closed set)

Exactly four deterministic conditions may trip automatic demotion. The set is closed in this
specification (§9.1).

| Trigger | Deterministic condition | Maps to `demotionReason` | Default posture |
|---|---|---|---|
| `stale_confidence` | Label-free drift: the distribution of constructed confidence has shifted, voiding the calibration that certified the rung. Derived from a stale evidence artifact as given; the drift detector that *sets* staleness is deployment-tier. | `"pending-evidence"` | Armed per grant (§6.7.4) |
| `corroboration_failure` | A premise failed its k-of-n independent-source quorum: derived deterministically from a typed corroboration record as `agreeing < k`. | `"failing"` | Armed per grant |
| `budget_breach` | An atomic durable counter (error, spend cap, escalation, fallback) blew its per-period bound. | `"failing"` | Armed per grant |
| `false_action` | An authenticated owner flagged an executed operation as wrong. Derived from a durable counter written by the authenticated flag path — never from message content. A single flag suffices. | `"failing"` | Ships OFF; armed per grant |

### 4.3 Ledger record types

Five record types share one append-only ceremony ledger (§5.2).

| `recordType` | Records | Shape rules |
|---|---|---|
| `promotion` | The maker≠checker ceremony that raises the level | maker ≠ checker REQUIRED; `predicate` REQUIRED non-empty; `triggeredBy` empty; `demotionReason` null |
| `demotion` | Automatic deterministic demotion | `ratifiedBy` is the system demotion-evaluator identity; `triggeredBy` non-empty; `demotionReason` set; `predicate` null |
| `bootstrap` | The sanctioned seed: first creation of a grant outside the propose/ratify ceremony | `fromLevel` null; maker ≠ checker NOT enforced (single-operator seed is sanctioned); `predicate` null; `triggeredBy` empty; `demotionReason` null |
| `tightening` | Voluntary any-level → `in-loop` move (§6.3) | `toLevel` = `"in-loop"`; maker ≠ checker NOT enforced; `predicate` null; `triggeredBy` empty; `demotionReason` null |
| `lapse` *(not yet implemented — #255)* | A certification term expired (§6.7.6) | `toLevel` = the grant's `lastSafeLevel`; `ratifiedBy` is the system evaluator identity; `triggeredBy` empty — a lapse is an absence, not a fired condition; `demotionReason` = `"pending-evidence"`; `predicate` null |

> **Implementation status:** NORMATIVE, NOT YET IMPLEMENTED in the reference implementation (tracking: #255).

The `lapse` row above, and only that row, is normative ahead of the reference implementation
(§3, §6.7.6). The other four record types are implemented.

The schema enforces field *shape* only; transition validity (level ordering, constructibility)
is the state machine's obligation (§6.2, GAL-17).

### 4.4 Demotion reasons

| `demotionReason` | Meaning | Correct response |
|---|---|---|
| `"failing"` | An error blew a bound; the model/policy is wrong. | Fix it. |
| `"pending-evidence"` | Recent labels are lacking to certify this rung; nothing is proven broken — the certification lapsed. | Gather data. |

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
| `signed-lineage` | The chain is signed and receiver-verified; the receiver cannot be lied to | Eligibility for acting-autonomy rungs |

Ordering is strict: `taint-bit` < `lineage` < `signed-lineage`.

---

## 5. Objects

### 5.1 Grant

The unit of authority: per `(principal, action-class)`. Autonomy level is stored state, not a
constant; this record is where it lives and what the lifecycle moves on a ratchet.

| Field | Type | Required | Description |
|---|---|---|---|
| `principal` | Principal | yes | The identity tuple (agent identity, skill, user, tier) this grant is for. |
| `actionClass` | string | yes | The class of action it authorizes (e.g. `"email.send"`, `"payments.transfer"`). Derived per §6.5. |
| `level` | `"in-loop"` \| `"on-loop"` \| `"out-of-loop"` | yes | Current autonomy rung — stored state, moved only by the lifecycle. The enumeration is complete at three values (§4.1). |
| `envelopeHash` | string | yes | Content hash of the envelope in force. The agent cannot widen its own envelope; the hash makes the in-force bounds verifiable and binds the grant to them (§6.6). |
| `promotedBy` | string | yes | The accountable identity that ratified the current level (checker side of maker≠checker). |
| `evidence` | string | yes | Reference to the covered-distribution evidence the promotion cited. An **implementation-defined reference, opaque to GAL**: GAL fixes no URI scheme and no digest binding, and no conforming component may parse or resolve it as a condition of accepting the grant. It MUST be bound into the integrity basis (§6.9) — it rides inside the canonically-serialized payload — so the reference cannot be altered after the fact. |
| `ts` | string (ISO-8601 UTC) | yes | When this level took effect. |
| `lastSafeLevel` | `"in-loop"` \| `"on-loop"` | yes | The rung automatic demotion falls back to. MUST NOT be `"out-of-loop"`. |
| `demotionTriggers` | DemotionTrigger[] | yes | The deterministic conditions armed for this grant, drawn from the closed §4.2 vocabulary. |
| `demotionReason` | null \| `"failing"` \| `"pending-evidence"` | yes | Why currently demoted; null when at full level. |
| `labelLatency` | string (duration) | yes | How long until an action of this class yields ground truth. Caps re-promotion speed (§6.8, §8.2). *Reference implementation note:* validated as an ISO-8601 duration; calendar-ambiguous year/month units are refused. |
| `certifiedUntil` *(not yet implemented — #255)* | string (ISO-8601 UTC) \| null | yes | The term of the current certification: the instant after which this level is no longer certified and the grant lapses (§6.7.6). `null` means no term. |
| `ownerId` | string | yes | The named human owner accountable for this grant. Every grant has one. |

> **Implementation status:** NORMATIVE, NOT YET IMPLEMENTED in the reference implementation (tracking: #255). Applies to `certifiedUntil` only; every other field of `Grant` is implemented.

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
  "ownerId": "owner:wes"
}
```

### 5.2 PromotionRecord

The append-only ceremony-ledger record of every grant level change. Five record types share one
ledger (§4.3); demotions append a `demotion`-typed record on the *same* ledger — demotion is
recorded, never approved.

| Field | Type | Required | Description |
|---|---|---|---|
| `recordType` | `"promotion"` \| `"demotion"` \| `"bootstrap"` \| `"tightening"` | yes | The record type; default `"promotion"`. Shape rules per §4.3. |
| `actionClass` | string | yes | The class whose level changed. |
| `principal` | Principal | yes | For whom. |
| `fromLevel` | level \| null | yes | `null` means the Recommend rung: the agent held no grant, and this record's write is the grant's creation (first promotion or bootstrap). |
| `toLevel` | `"in-loop"` \| `"on-loop"` \| `"out-of-loop"` | yes | The resulting level. |
| `evidence` | string | yes | An implementation-defined reference, opaque to GAL, on the same terms as `Grant.evidence` (§5.1) — but what it references varies by record type; see below. |
| `predicate` | string \| null | promotion-typed only | The signed promotion predicate authored in advance (the licensing text/result). REQUIRED non-empty on `promotion` records; null otherwise. |
| `proposedBy` | string | yes | Maker identity. |
| `ratifiedBy` | string | yes | Checker identity. On `promotion` records MUST differ from `proposedBy` (§6.4.3). On `demotion` records, the system demotion-evaluator identity. |
| `attestation` | enum \| null | no | **How the two ceremony identities were established**, from a closed vocabulary whose one defined value is `"solo-local"`: ONE operator held both ceremony roles, through two credentials that the same holder could mint. Null — including its absence from any record written before the field existed — means the two identities were established by mutually unmintable credentials, the full maker≠checker guarantee of §6.4.3. The value MUST be **derived** by the issuer from the ceremony identities themselves, never independently asserted by the proposer, so the marker and the identities cannot disagree. This field is what keeps §6.10's non-repudiation claim honest: the record must never imply a review that did not happen, so a record written under the weaker guarantee states it rather than leaving an auditor to assume the stronger one. |
| `envelopeHash` | string | yes | The envelope hash in force when the record was written; bound into the signed statement (§6.10). |
| `triggeredBy` | string[] | demotion-typed only | The DemotionTrigger values that fired. Non-empty on `demotion` records; empty otherwise. |
| `demotionReason` | `"failing"` \| `"pending-evidence"` \| null | demotion-typed only | Set on `demotion` records; null otherwise. |
| `ts` | string (ISO-8601 UTC) | yes | Record timestamp. |

**What `evidence` carries, per record type.** GAL deliberately does **not** require `evidence`
to be a URI or a digest, because the record types of §4.3 do not carry the same kind of thing
and a uniform format requirement would be violated by half of them on the day it was written. A
`promotion` or `bootstrap` record's `evidence` references the covered-distribution evidence
the level change was licensed by. A `demotion` record's `evidence` carries the human-readable
cause of the demotion — the record exists to say *why the rung fell*, and there is no external
evidence artifact to point at. A `tightening` record's `evidence` is a short free-text
rationale, which an implementation MAY default, since a voluntary tightening is
safety-monotone and cites nothing. A `lapse` record's `evidence` names the expired term, since
a lapse cites the absence of renewal rather than any artifact (§6.7.6). What is uniform, and
what conformance depends on, is that the value is opaque to GAL and integrity-bound with the rest of the record (§6.9, §6.10): an
implementation MUST NOT make accepting a record conditional on parsing it, and MUST NOT permit
it to be edited after signing.

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

**ConfidenceArtifact** — the constructed-confidence artifact backing a proposal:

| Field | Type | Description |
|---|---|---|
| `confidence` | number [0,1] | Constructed value — never a raw logprob. |
| `error_prob` | number | The error-budget draw numerator; method-specific, NOT `1 − confidence`. |
| `evidence` | tagged union | Method-specific evidence: self-consistency `{samples, agreement}`, ensemble `{members, agreement}`, or conformal `{coverage, threshold, calibration_size}`. The method vocabulary is a closed catalog. |
| `stale` | boolean | Label-free drift flag; a stale artifact voids the certification and never meets a bar. |
| `computed_at` | string (ISO-8601 UTC) | Construction time. |
| `annotations` | string[] | Attach-only findings (e.g. the evidence reviewer's, §6.4.5); NEVER read by any gate. |

**Error budget** — a per-period bound on cumulative tolerable error, drawn down per decision
as `error_prob × blast_radius`, metered on scoped durable counters. `blast_radius` is the
deployment-declared weight per blast class; a weight table is REQUIRED whenever a budget is
set — the base never invents a domain weight. Breach emits `budget_breach` as a typed demotion
input, never a model judgment. Blast class derives one-way (§6.5): high = write ∧ external ∧
¬reversible (unknown reversibility reads as not-reversible); medium = any other write; low =
read; a deployment MAY tighten a derived class up to high, never lower it.

**DemotionSignal** — emitted at breach detection, consumed by the demotion evaluator (the
evidence machinery emits, never applies):

| Field | Type | Description |
|---|---|---|
| `trigger` | DemotionTrigger | §4.2 vocabulary. |
| `principal` | Principal | Grant coordinate. |
| `action_class` | string | Grant coordinate. |
| `period` | string | The counter-period bucket the breach was metered in. |
| `detail` | string | Audit-safe cause; no payload content. |
| `ts` | string (ISO-8601 UTC) | Emission time. |

**CorroborationRecord** — the typed k-of-n quorum result the `corroboration_failure` trigger
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
it, and MUST reject an incoherent record — `k > n`, `agreeing > n`, or `stale_sources > n` —
rather than clamp it. The failure predicate is exactly `failed ⇔ agreeing < k`, and it is the
only computation any gate performs over this record.

**Per-source provenance is deliberately excluded.** A record naming *which* sources agreed is
the obvious extension and this specification declines it: the producer folds staleness and
provenance validity into the `agreeing` count — a stale or provenance-invalid source is simply
not counted, and can therefore never carry a quorum — so the deterministic trigger needs no
source identities to reach the same verdict. Carrying them would put source-identity data into
the base evidence contract, where every consumer and auditor would then have to handle it, in
exchange for nothing the predicate reads. A deployment that needs per-source detail for its own
review keeps it in the deployment-tier corroboration pass that produced the record.

**Label-free drift input** — the `stale_confidence` trigger derives from a stale
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
default-denies: absence of a grant is the deny, not a distinct flag — a "blocked" class is
simply an ungranted class.

### 6.2 The ratchet asymmetry

The asymmetry is the load-bearing design property and MUST be preserved:

> Promotion is recorded and human-gated. Demotion is automatic, deterministic, and has no model
> in the loop.

The system can narrow its own envelope without a human in the moment; widening it requires a
recorded ceremony. Any structurally invalid transition (level-skipping outside the sanctioned
re-attestation case, a demotion target of `out-of-loop`, a fourth level value) MUST be
unconstructible — a typed refusal, not a warning.

### 6.3 Tightening is always permitted

Any level → `in-loop` is always permitted: voluntary tightening is safety-monotone and needs
no ceremony, no evidence, and no trigger. It MUST append a `tightening`-typed ledger record
(§4.3) so the level change remains accounted for.

### 6.4 Promotion — the only upward path

#### 6.4.1 The deterministic licensing predicate

The acceptance gate for any upward move is a **pure, deterministic predicate** over windowed,
scoped counters plus the typed evidence artifact. No model output may appear anywhere in the
licensing path. The predicate MUST evaluate, in a fixed order, at least these gate terms:

1. **Provenance-maturity ceiling** — the target rung's required maturity is met (§6.13).
2. **Evidence artifact present** — absence of evidence is not evidence.
3. **Evidence artifact fresh** — a stale artifact voids the certification.
4. **Covered-distribution soundness asserted** — an explicit proposer assertion, recorded on
   the ceremony record (§6.4.2).
5. **Error budget not in breach** — evaluated only when the budget knob is configured; an
   unset knob is no gate (§1.2).
6. **Minimum observation count met** — a thin sample cannot license a promotion.
7. **Error rate below the configured threshold** — computed over the evidence window from
   durable counters (observed erroneous outcomes over observations).

Every ambiguous case — zero observations, sample too small, missing or stale artifact,
unasserted coverage, malformed input — MUST resolve to *ineligible*. The predicate is
fail-safe: it cannot pass on the absence of disconfirming evidence, and callers MUST NOT
override an ineligible result.

#### 6.4.2 Evidence window binding and soundness

The evidence window's span and period granularity MUST be bound into the proposal the checker
ratifies, under the proposal's own integrity mechanism, so the checker sees — and the record
proves — exactly which evidence licensed the change. Evidence MUST come from a *covered*
distribution: gathered where the conditions the promoted class will act under are represented.
A thin observed-accuracy count over an irreversible action class is an unsound predicate and
the ceremony MUST reject it. Covered-distribution soundness enters the predicate as an
explicit proposer assertion recorded on the ceremony record — a human accountability anchor,
not a derived quantity.

*Reference implementation note:* counters are scoped per principal + operation + UTC period; a
writer/reader period mismatch reads disjoint keys and yields zero evidence — failing toward
less authority, never a wrong sum.

#### 6.4.3 Maker≠checker — what it enforces, what it evidences

A promotion is proposed by a maker and ratified by a checker, and the two MUST be different
credentials, compared on credential identity. The standard therefore enforces exactly this:
**no single credential can both propose and ratify** — a compromised agent, automation job, or
leaked key cannot self-promote — and the signed record *evidences*, non-repudiably, which
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
exercised — the deterministic demotion triggers (§6.7), the independent ledger audit (§6.9), an
off-path observer — bounds the *duration* of a bad grant, never the blast of a single exercise of
it, and a control's response time must be matched to the blast rate of the authority it bounds.
Below the high-blast threshold an act-then-detect-then-demote window is survivable, so a signed
promotion predicate authored in advance MAY stand in as the ratifier — the pre-authorized change
envelope pattern — with its stringency and ceremony scaling to blast radius. At and above it, no
after-the-fact control has a response time matched to a single act; only a human *inside* the
window suffices, and detection MUST NOT be treated as a substitute for ratification. The predicate
result MUST carry the human-ratification requirement
explicitly so the ceremony cannot drop it; an unknown blast class MUST be treated as an error,
never as not-high.

#### 6.4.5 The evidence reviewer may surface, never decide

An OPTIONAL evidence reviewer — if a model, of a different model family than the maker's, so
the pair shares no blind spot — MAY review the evidence bundle during the ceremony. Its
closed-vocabulary findings are attached to the ceremony result for the human ratifier, and
that is all: it MUST NOT be able to license or veto a level change, and a failing or
unreachable reviewer MUST degrade to a recorded reviewer-error finding, never a gate flip in
either direction. This is the surface-vs-decide rule: the model may only surface a concern;
the gate decides, deterministically.

#### 6.4.6 Input integrity of the sanctioned path

When a mutation path is the only sanctioned path to a protected object, every durable input
that path consumes needs the same tamper-evidence as the object itself — or the sanctioned
path IS the bypass. For every ceremony input (stored proposals, evidence artifacts, config
re-read at ratify time) an implementation MUST provide one of: the protected object's own
integrity mechanism, a signature, or full re-validation at the trust boundary (a stored
proposal is input, never authority). Proposals MUST be single-use and expiring.

#### 6.4.7 Write discipline

Ledger and grant writes MUST be conditional (append-only, never overwrite; update-only, never
mint), and **no failure MUST be able to leave a raised grant without its record** — an
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
tool operation — effect (read/write), external (crosses a trust boundary), reversible — not
from any base-owned catalog: the base standard owns the *shape* of the classification, the
deployment owns the *content*. High-blast derives from the same fields (write ∧ external ∧
¬reversible, unknown reversibility read conservatively as not-reversible). A deployment MAY
additionally *declare* a class high-blast but MUST NOT un-declare a derived one: the
derivation is tighten-only. A never-grantable hard ceiling, if wanted, is a future
tighten-only envelope knob, not a mechanism of this version.

### 6.6 Envelope binding and re-attestation

Every grant and every ledger record carries the content hash of the envelope in force when it
was written. A grant whose `envelopeHash` does not match the current in-force envelope MUST be
quarantined by the enforcer on every call — loudly, never silently passed (§6.9).

When the envelope changes (any redeploy or tightening that churns the hash), the cure is
**re-attestation**: a ceremony re-issuing the grant at its prior level **under human
ratification**. It appends no ledger record, deliberately: the ceremony ledger records level
*changes*, and a re-attestation changes no level. Appending one would put an entry on the ledger
that no level transition explains, and would make the re-promotion reference ambiguous. It
carries the prior level rather than restarting at
`lastSafeLevel` — a hard restart would turn every envelope change into an autonomy event, and
the high-blast always-human rule (§6.4.4) already backstops the dangerous classes. An
implementation MUST NOT silently re-mint a grant under a new envelope hash, and re-attestation
MUST refuse when the named configuration it derives from does not match what the enforcer will
load — failing toward writing nothing, never toward minting authority the enforcer will reject.

### 6.7 Demotion — automatic, deterministic, no model

#### 6.7.1 The trigger set and the path

Demotion MUST run when the model is confused and the inputs are suspect — exactly when one
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
sends the wrong team at the problem — one demands a fix, the other demands data.

#### 6.7.4 Arming and the flag asymmetry

A trigger fires only for grants listing it in `demotionTriggers`; every non-floor bound ships
OFF (§1.2). `false_action` MUST derive from a durable counter written by an authenticated
owner path — never from message content — and a single flag suffices. The ease gradient points
downward: it is deliberately easier to get demoted than to overcome the safeguards, and no
flag — nor any accumulation of flags — can ever promote anything.

#### 6.7.5 Derivation parity

The `budget_breach` derivation MUST read the same durable counter, key, and comparison that
per-call enforcement uses — the demotion evaluator and the enforcer cannot disagree about
whether a bound blew.

#### 6.7.6 Lapse — a certification has a term

> **Implementation status:** NORMATIVE, NOT YET IMPLEMENTED in the reference implementation (tracking: #255).

Every trigger in §4.2 asserts that something was **observed**: a bound blew, a quorum failed, a
distribution drifted, an owner flagged. None of them fires when *nothing happens*. A grant promoted
on evidence gathered long ago, whose agent has since been idle, therefore has no path downward — it
is not failing, nothing is proven broken, and it holds its rung indefinitely on a certification
nobody has renewed. The recorded chain says how the authority was acquired; a term is what forces it
to be re-justified.

A grant MAY carry a term (`certifiedUntil`, §5.1). Where it does:

- When the term passes, the grant MUST lapse: fall to `lastSafeLevel`, set `demotionReason` to
  `"pending-evidence"` — the certification lapsed, nothing is proven broken (§4.4) — and append a
  `lapse`-typed ledger record (§4.3).
- A lapse MUST NOT be recorded as a triggered demotion. `triggeredBy` names conditions that fired;
  a lapse is the *absence* of renewal, not the presence of a signal. Recording it as a trigger would
  leave `triggeredBy` naming a condition that never fired, and would teach every auditor to read a
  demotion record as possibly meaning "nothing happened, on schedule."
- A lapse MUST land on `lastSafeLevel`, never on outright revocation. Revoking authority on a timer
  is a **self-inflicted forced abstention** (see the sibling PTC specification's availability
  section): for a deployment whose polarity is positive-safe-action, a grant that silently expires
  into no-authority produces exactly the harm the agent exists to prevent, on a schedule an
  adversary does not even have to trigger.
- Recovery is the ordinary upward path — the promotion ceremony with fresh evidence (§6.4). An
  implementation MUST NOT auto-renew a term, and the grant holder MUST NOT be able to extend one in
  place; either would make the term self-certifying and thereby vacuous.
- A grant with no term does not lapse. Terms are a deployment knob and ship unset (§9.2): a
  deployment that sets none behaves exactly as it did before this arc existed.

Grant age is deliberately distinct from the two decay concepts already in this specification.
`stale_confidence` (§4.2) fires when an observed distribution drifts, voiding a calibration; dwell
time (§6.8) damps oscillation between rungs. Neither reaches an idle grant, because neither has
anything to observe.

### 6.8 Hysteresis and re-promotion

Demotion and re-promotion MUST use different thresholds and a dwell time, and re-promotion
MUST require fresh recalibration evidence — never merely "the alarm stopped". Demotion itself
has no dwell: it fires when the condition holds. This prevents a noisy signal from oscillating
the rung.

Re-promotion closes a control loop whose deadtime equals `labelLatency`: with no labels at
action time, re-promotion runs on delayed-label reconciliation — pair each resolved outcome
with the confidence logged at decision time, recompute calibration on the *current*
distribution, then ratchet up. Long label latency caps how fast a class can re-promote, and
therefore how much autonomy it can responsibly hold at all (§8.2).

### 6.9 Grant integrity at rest

Every stored grant MUST be integrity-protected and verified on read. A grant that fails
verification MUST be quarantined loudly — treated as no grant (default-deny) with an explicit,
audit-visible quarantine signal — never silently trusted and never silently dropped. A reader MUST
distinguish *quarantined* from *not found*: they license the same denial but demand opposite
operator responses, and collapsing them hides tampering inside routine absence.

**The integrity basis is the stored bytes.** An implementation MUST serialize the grant once, store
exactly those bytes, bind the integrity value to that byte string, and on read verify the stored
bytes **verbatim, before parsing them**. The integrity value MUST live outside the record it
protects (§5.1): a value carried inside the structure cannot cover the bytes as stored, since
computing it changes them.

The requirement is not merely constructional. A basis that is *re-derived* by parsing and
re-serializing binds the reader's current schema rather than the writer's bytes, so any honest
schema evolution — a widened type, a new optional field, a changed default — makes every
previously-written record fail verification as though it had been tampered with. **Integrity must
indict tampering, never evolution.** A control that cries tamper on legitimate change trains
operators to dismiss it, and then to stop evolving the schema; the resulting freeze presents itself
as prudence. Where a stored-byte basis must nevertheless change, an implementation MUST version the
basis explicitly rather than verify under two bases at once.

### 6.10 Record signing

Every ledger record MUST be signed at write time: a signature envelope (DSSE-style, with
pre-authentication encoding) over an attestation-style statement binding at minimum the record
content, the in-force `envelopeHash`, and the signer identity. The signing key is a workload
identity resolved by the issuing infrastructure — never held by, or reachable from, the agent.
A signed record makes a level change non-repudiable: who proposed, who ratified, over which
evidence window, under which envelope.

An issuer MUST refuse to store an unsigned record when signing is configured, and MUST refuse
to run at all under half-configured signing (fail toward writing nothing). An explicit,
recorded operator override MAY permit an unsigned record in bootstrap circumstances; the
record then verifiably carries its unsigned status and audit disposition (§6.11) applies. An
OPTIONAL transparency-log anchor for high-blast promotions is a knob shipping OFF.

The signing shape is shared with the sibling PTC specification (its signing-shape /
trust-context-envelope sections): same envelope discipline, same workload-identity key
handling, a second statement type on the same machinery. Grant integrity at rest (§6.9) and
change signing are distinct: the first protects the object, the second proves the provenance
of the change.

### 6.11 The audit obligation

An independent auditor — read-only, holding no ceremony credentials — MUST be able to
re-verify the entire ledger: every grant has a ledger counterpart from birth (§6.12), every
record's signature verifies, every transition is structurally valid, every grant's
`envelopeHash` matches an in-force envelope, and no record has been mutated or removed.

**What the audit does not do.** The obligation stops at the ledger. Because `evidence` is an
opaque, implementation-defined reference (§5.1, §5.2), an auditor verifies that the reference
is intact and integrity-bound — that the record still cites what it was signed citing — and
does **not** resolve it, fetch it, or re-verify that the cited evidence supports the level
change. An implementation MUST NOT present a passing ledger audit as a re-verification of the
evidence a promotion was licensed by; the licensing judgment was made once, at ratification, by
the accountable checker (§6.4.3), and the audit proves who made it under which bounds, never
that it was correct.

A true finding whose remediation is deferred MUST be dispositioned only by a signed,
append-only acknowledgment artifact — a ceremony record binding the rule, the coordinate, and a
digest of the exact violation detail, so a *new* finding at the same coordinate is never
auto-waived. A waiver is a ceremony artifact, never a configuration toggle: a waiver mints
"green", which is authority. Only a closed waivable vocabulary may be acknowledged;
integrity-tamper findings MUST be un-waivable. A waiver whose signature cannot be verified
MUST NOT be applied (fail toward red). The audit reports acknowledged findings as
green-with-annotations, never silently green.

**Reconciliation against the live principal set.**

> **Implementation status:** NORMATIVE, NOT YET IMPLEMENTED in the reference implementation (tracking: #256).

Ledger completeness is necessary and not sufficient: it proves every grant has a story, never
that every acting principal has a grant or that every grant still names a principal that
exists. An auditor MUST additionally be able to reconcile
the grant set against the set of principals the deployment actually runs, reporting **both**
directions — a principal acting without a grant, and a grant whose principal no longer exists. The
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
rung whose maturity requirement is unmet MUST be structurally unconstructible — the predicate
evaluates the maturity as a deterministic gate term, not a reviewer's judgment. Unsigned
lineage means approval-gated; no acting-autonomy rung is reachable without receiver-verifiable
signed provenance. **`in-loop` as a target carries no provenance ceiling** — it is the
approval-gated floor, reachable as the Recommend-origin grant-creating first promotion; every
evidence gate of §6.4.1 still applies. PTC supplies the proof; GAL moves the state; the
ceiling is enforced in the predicate, not left to judgment.

### 6.14 The drill obligation

Demotion is a safety mechanism, and an untested safety mechanism is a liability. Before a
demotion trigger is relied upon, a deployment MUST have exercised it against a live grant and
confirmed: the rung fell to `lastSafeLevel`, the audit event was emitted, the demotion-typed
record was appended, and no model was called on the path. A promotion is likewise not complete
until the promoted grant has acted once under its new level — an unexercised grant can be
operationally dead (e.g. envelope-quarantined) while every ledger row looks correct.

---

## 7. Conformance

### 7.1 Roles

Two conformance roles. The **grant issuer** hosts the ceremony: evaluates the deterministic
predicate, enforces maker≠checker, signs the ledger record, writes the grant store. The
**grant enforcer** — the broker — reads the grant per call, enforces the level's actuation
semantics, runs the demotion evaluation, and structurally cannot be written past by the agent.
One deployment component MAY implement both roles, but the agent MUST be neither. Audit
clauses (GAL-31 to GAL-33 and GAL-35) constrain the artifacts both roles produce and MUST be
satisfiable by a third party holding read-only access.

### 7.2 Issuer clauses

- **GAL-1** A grant SHALL bind exactly one `(principal, action-class)` pair; implementations MUST NOT expose or act on an agent-scoped autonomy level.
- **GAL-2** The `level` enumeration SHALL be complete at `in-loop`, `on-loop`, `out-of-loop`; Recommend SHALL be represented only as the absence of a grant and MUST NOT become a level value.
- **GAL-3** Level SHALL be orthogonal to the per-call decision verbs; no verb outcome may mutate a level and no level may be inferred from a verb.
- **GAL-4** The ceremony (and the automatic demotion path) SHALL be the grant store's only mutation paths; the issuer MUST refuse any other write.
- **GAL-5** The promotion licensing predicate SHALL be pure and deterministic with no model output in the path, and SHALL NEVER license a promotion on an input it cannot interpret. **Evidence** that is ambiguous — thin, absent, stale, or uncorroborated — SHALL resolve to ineligible. Input that is **structurally invalid** — a negative counter, a total exceeding its window, an unknown enumeration value — SHALL refuse loudly and SHALL NOT resolve to ineligible, because a broken evidence pipeline reported as ineligible is indistinguishable from an honest denial and directs the proposer to gather more evidence, which can never fix it. A loud refusal SHALL still produce a recorded outcome.
- **GAL-6** The predicate SHALL evaluate at least the seven gate terms of §6.4.1 in a fixed order, with unset knobs constituting no gate.
- **GAL-7** `proposedBy` and `ratifiedBy` on a promotion SHALL be credentials that **a single operator cannot satisfy both of**; the comparison SHALL reject any pair a lone operator can produce, and credential-identity equality is a floor rather than the whole test. Where an identity embeds unstable components (a hostname, a session id), the comparison SHALL additionally reject pairs sharing a ceremony role, since equality alone passes the same operator twice. The record SHALL evidence both non-repudiably. The standard enforces two credentials and evidences — not enforces — two humans.
- **GAL-8** For high-blast classes, a true predicate SHALL carry an explicit per-instance human-ratification requirement that the ceremony MUST NOT drop; an unknown blast class SHALL be an error.
- **GAL-9** High-blast SHALL derive from declared operation fields (write ∧ external ∧ ¬reversible, unknown reversibility read as not-reversible); deployments MAY tighten the derived class, MUST NOT loosen it.
- **GAL-10** Any evidence reviewer SHALL attach findings only; it MUST NOT license or veto, and reviewer failure SHALL degrade to a recorded finding, never a gate flip.
- **GAL-11** Every durable input the ceremony consumes SHALL carry tamper-evidence or be fully re-validated at the trust boundary; proposals SHALL be single-use and expiring.
- **GAL-12** The evidence window's span and period SHALL be bound under the proposal's integrity mechanism; covered-distribution soundness SHALL be recorded as an explicit proposer assertion.
- **GAL-13** Any level → `in-loop` SHALL be permitted without ceremony and SHALL append a `tightening`-typed record.
- **GAL-14** Grant creation outside the ceremony SHALL emit a `bootstrap`-typed record; no grant SHALL lack a ledger counterpart.
- **GAL-15** On envelope change, the grant SHALL be re-attested — prior level, under human ratification, appending **no** ledger record, since no level changed — and the re-attestation SHALL refuse on configuration mismatch, failing toward writing nothing.
- **GAL-16** Every level change SHALL append exactly one typed record on one append-only ledger; records SHALL never be overwritten, mutated, or removed.
- **GAL-17** Structurally invalid transitions SHALL be unconstructible (typed refusal), including any demotion target of `out-of-loop`.
- **GAL-18** Ledger records SHALL be signed per §6.10 (workload-identity key, envelope hash bound); the issuer SHALL refuse unsigned or half-configured storage absent an explicit, recorded override.
- **GAL-19** Grant and ledger writes SHALL be conditional, and **no failure SHALL leave a raised grant without its ledger record**. Writing the record before the grant mutation satisfies this by ordering; committing both as one atomic transaction satisfies it by admitting no interruption. An implementation SHALL state which it provides.
- **GAL-20** Promotions to `on-loop` or `out-of-loop` SHALL require `signed-lineage` provenance maturity as a deterministic predicate term; `in-loop` as a target SHALL carry no provenance ceiling.

### 7.3 Enforcer clauses

- **GAL-21** The enforcer SHALL read the grant per call; absence of a grant SHALL default-deny (Recommend), and Recommend SHALL produce no staged intent.
- **GAL-22** The agent SHALL have no write access to the grant store, enforced structurally; a grant-store write reachable by the agent is a promotion bypass and non-conformant.
- **GAL-23** Demotion SHALL be tripped only by the four triggers of §4.2, each derived deterministically from typed durable inputs, with no model call on the path.
- **GAL-24** A tripped demotion SHALL move the grant to `lastSafeLevel` **or lower, and SHALL NEVER raise its autonomy**; where `lastSafeLevel` ranks above the grant's current level the trip SHALL leave the level unchanged and still append its record. `lastSafeLevel` SHALL never be `out-of-loop`.
- **GAL-25** `demotionReason` SHALL follow the fixed trigger mapping, `"failing"` dominating mixed firings, and the two reasons SHALL never be collapsed.
- **GAL-26** Every demotion SHALL append a `demotion`-typed record (ratified by the system evaluator identity) and emit an audit event; demotion writes SHALL run under an identity separate from the agent.
- **GAL-27** A trigger SHALL fire only for grants listing it in `demotionTriggers`; `false_action` SHALL derive from an authenticated durable counter, a single flag sufficing, and no flag SHALL ever promote.
- **GAL-28** `budget_breach` SHALL be derived from the same durable counter, key, and comparison that per-call enforcement uses; an unusable evidence input SHALL cause a loud refusal, never "no breach".
- **GAL-29** Re-promotion SHALL require fresh recalibration evidence and satisfy dwell-time hysteresis; demotion SHALL have no dwell.
- **GAL-30** A grant's integrity value SHALL bind its **stored bytes**, SHALL live outside the record it protects, and SHALL be verified verbatim *before* the bytes are parsed. A grant failing verification, or carrying an `envelopeHash` not in force, SHALL be quarantined loudly on every call — treated as no grant, with an audit-visible signal distinguishable from not-found.

- **GAL-34** Where a grant carries a certification term, its expiry SHALL lapse the grant to `lastSafeLevel` with `demotionReason` `"pending-evidence"` and a `lapse`-typed record; a lapse SHALL NOT be recorded as a triggered demotion, SHALL NOT revoke authority outright, and SHALL NOT be auto-renewed or extended in place by the holder. A grant carrying no term SHALL NOT lapse.

  > **Implementation status:** NORMATIVE, NOT YET IMPLEMENTED in the reference implementation (tracking: #255).

### 7.4 Audit clauses

- **GAL-31** An independent, read-only party SHALL be able to re-verify the full ledger: signatures, transition validity, envelope binding, and the no-orphan invariant.
- **GAL-32** Audit findings SHALL be dispositioned only by signed, append-only acknowledgment artifacts binding rule + coordinate + violation digest; the waivable vocabulary SHALL be closed, integrity-tamper findings SHALL be un-waivable, and an unverifiable waiver SHALL NOT be applied.
- **GAL-33** Each armed demotion trigger SHALL have been drilled against a live grant before being relied upon, and a promotion SHALL NOT be considered complete until the promoted grant has acted once. **The first act under a promoted grant SHALL be recoverable from the audit record** by a read-only party — the enforcement point writes it, and the ceremony ledger cannot, since the enforcing component is barred from writing the grant store.
- **GAL-35** An auditor SHALL be able to reconcile the grant set against the live principal set in both directions — principals acting without a grant, and grants whose principal no longer exists — and SHALL report rather than write; retiring a grant remains a ceremony.

  > **Implementation status:** NORMATIVE, NOT YET IMPLEMENTED in the reference implementation (tracking: #256).

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
| GAL-15 | GAL.md §5 (re-attestation carries prior level); grant-lifecycle §audit (#199 refusal seam) |
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
| GAL-34 | Five Eyes *Careful adoption of agentic AI services* (2026-05-01), the "expiry timers and recorded grant chains" pairing; `docs/references/five-eyes-agentic-guidance.md` FE-1. Normative ahead of the reference implementation (§3; tracking #255). |
| GAL-35 | Five Eyes *Careful adoption of agentic AI services*, "periodically reconcile the registry against the live set of agents"; `docs/references/five-eyes-agentic-guidance.md` FE-2. Normative ahead of the reference implementation (§3; tracking #256). |

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
months whether an action was right is a domain in which full autonomy cannot be evidenced —
this specification prefers an honest ceiling to an unsound promotion, and states the
consequence normatively expecting it to be contested.

### 8.3 Who ratifies the ratifier

`ownerId` is the root of trust for ratification. At a single named owner this is clean; the
N>1 fan — quorum ratification, key rotation mid-window, delegation — is unspecified in this
version (§1.3). The honest bound (§6.4.3): the standard enforces two *credentials* and evidences two
*humans*. Against a malicious sole operator it provides non-repudiation, not prevention.

### 8.4 The enforcement point is load-bearing

Every property here assumes the enforcer cannot be bypassed: the agent holds no credentials,
its only egress is the broker, and the grant store is structurally write-protected from it
(GAL-22). A deployment that lets the agent reach a tool directly, or reach the grant store
with any write, has no autonomy lifecycle — only autonomy paperwork.

---

## 9. Extensibility

### 9.1 The trigger set is closed in this specification

The demotion-trigger vocabulary is exactly the four values of §4.2; deployments arm a subset
per grant via `demotionTriggers` and MUST NOT extend the vocabulary. The two derivation-heavy
triggers (`stale_confidence`, `corroboration_failure`) deliberately consume *typed evidence as
given* — the detectors producing the evidence are deployment-tier and freely extensible, so
domain-specific demotion logic lives in what a deployment feeds the triggers, not in new
trigger values. Widening the vocabulary changes what every auditor must understand a demotion
record to mean: a specification change, not a knob.

The time-based **lapse** arc (§6.7.6) is deliberately not a fifth trigger, for the same reason
stated from the other side. A trigger record asserts an observation; a lapse asserts that no
renewal occurred. Collapsing them would leave `triggeredBy` naming a condition that never fired,
and would blur the single distinction the demotion vocabulary exists to preserve — whether
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
| `0.2.0-draft` | 2026-07-25 | Adds the time-based **lapse** arc (§6.7.6, `certifiedUntil`, the `lapse` record type, GAL-34): a certification has a term, and its expiry is deliberately *not* a fifth demotion trigger, since a lapse asserts an absence rather than an observation. Adds reconciliation of the grant set against the live principal set in both directions (§6.11, GAL-35) — the shadow-agent case no store-only check can see. Corrects §5.1: the grant carries no integrity field of its own, and §6.9/GAL-30 now state the stored-bytes basis (a hash inside the structure it protects cannot cover the bytes as stored) plus the rule that integrity must indict tampering, never schema evolution. |
| `0.2.1-draft` | 2026-07-29 | Introduces the **implementation-status marker** (§3) and applies it to the two clauses that are normative ahead of the reference implementation — GAL-34 / §6.7.6 (the lapse arc, tracking #255) and GAL-35 / §6.11 (two-direction reconciliation, tracking #256) — so no reader can mistake either for a shipped control. Records the shipped `attestation` field on `PromotionRecord` (§5.2), which states when one operator held both ceremony roles and so keeps §6.10's non-repudiation claim honest. Pins the canonical JSON encoding for all GAL objects (§3), previously deferred despite §6.9's stored-bytes integrity basis making it interoperability-critical. Resolves the `evidence` reference format as an opaque, integrity-bound, implementation-defined string (§5.1, §5.2) and pins the `CorroborationRecord` field set (§5.3), including the deliberate exclusion of per-source provenance. Adds the audit's honest limit (§6.11): it verifies signatures, never the cited evidence. Corrects §4.3/§5.2, which said "four record types" over a five-row table. |
| `0.2.2-draft` | 2026-08-03 | Corrects §8.1 (Evidence poisoning), whose mitigation direction and normative SHOULD were both keyed on taint while the attack they name does not require it (#342). Grooming a promotion needs no tainted turn: a patient adversary, or drift with no adversary, can produce a clean behavioral record that the predicate rewards, leaving a taint-aware evidence window nothing to weight. §8.1 now separates tainted from untainted grooming, keeps the existing taint-aware guidance scoped to the first, adds a MUST NOT against reading absence of taint as absence of grooming, and states the structural asymmetry that motivates both: taint is a ratchet and cannot be farmed, whereas an evidence window rewarding accumulated clean behavior is a credit mechanism whose state the subject improves through its own conduct. Names maker≠checker ratification (§6.4), not the predicate, as what bounds the untainted case, and directs ratifiers to read a clean window as absence of recorded trouble rather than as positive evidence of trustworthiness. No clause, schema, or wire change; §8.1 carries no conformance clause. |

### 10.2 Reference implementation

**safe-agents** (controlled-agents) implements both conformance roles: its ceremony command set is
the reference issuer, its broker the reference enforcer. The lifecycle is live-drilled
end-to-end: a real principal promoted on real approval evidence under maker≠checker with a
distinct least-privilege identity per leg; all four demotion triggers fired live against real
grants; re-attestation exercised after a real envelope change; the independent
grant-integrity audit running continuously in CI on two environments; and a full-arc terminal
proof (evidence → propose → ratify → exercise → induced demotion → re-climb → exercise)
completed 2026-07-15. Its L1–L8 lifecycle conformance suite seeds this specification's
conformance tests.

### 10.3 References

- **[RFC2119]** Bradner, S., "Key words for use in RFCs to Indicate Requirement Levels",
  BCP 14, RFC 2119, March 1997.
- **[RFC8174]** Leiba, B., "Ambiguity of Uppercase vs Lowercase in RFC 2119 Key Words",
  BCP 14, RFC 8174, May 2017.
- **[SHERIDAN]** Sheridan, T. B. and Verplanck, W. L., "Human and Computer Control of Undersea
  Teleoperators", MIT Man-Machine Systems Laboratory, 1978 — the origin of the
  levels-of-autonomy vocabulary the ladder adopts.
- **[J3016]** SAE International, "Taxonomy and Definitions for Terms Related to Driving
  Automation Systems for On-Road Motor Vehicles" — the descriptive-taxonomy prior art: a label
  on a system, not stored state an enforcement point reads per call.
- **[PCCP]** U.S. FDA, "Marketing Submission Recommendations for a Predetermined Change
  Control Plan for Artificial Intelligence-Enabled Device Software Functions" — the regulatory
  precedent for pre-authorized change inside a declared envelope; GAL's pre-authored signed
  predicate is this pattern as a runtime object.
- **[ODD]** Operational Design Domain (SAE J3016; BSI PAS 1883) — a capability claim holds
  only inside the declared domain; GAL's covered-distribution requirement is its lifecycle form.
- **[JCS]** Rundgren, A., Jordan, B., and Erdtman, S., "JSON Canonicalization Scheme (JCS)",
  RFC 8785, June 2020 — the canonical-JSON profile §3 is adjacent to and MAY be satisfied by.
- **[DSSE]** Dead Simple Signing Envelope, https://github.com/secure-systems-lab/dsse.
- **[IN-TOTO]** in-toto Attestation Framework, https://github.com/in-toto/attestation.
- **[PTC-SPEC]** *PTC — Provenance & Trust Context, Specification*, `PTC-SPEC.md` (sibling
  draft): normative for the provenance-maturity ladder (§6.13 here), the signing shape (§6.10
  here), and the deterministic per-call gate the enforcer implements.
