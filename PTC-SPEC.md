# PTC — Provenance & Trust Context, Specification

**Version:** 0.5.1-draft
**Date:** 2026-10-07
**Status:** Draft for Linux Foundation agent-standards discussion. Wire schemas may change before
1.0; see Open Problems and Future Extensions.
**Working group:** LF Edge + Agentic AI Foundation (AAIF)
**Author:** Wes Jackson (Red Hat)
**Copyright:** © 2026 Red Hat, Inc.
**License:** Community Specification License 1.0 · `SPDX-License-Identifier: Community-Spec-1.0`
See `LICENSE.md` for terms and `NOTICE.md` for attribution, acceptance, and patent exclusions.

## 1. Scope and non-goals

PTC (Provenance & Trust Context) specifies a **connectivity-orthogonal trust layer** for agentic
systems: a signed trust-context object (sender class, append-only provenance chain, and taint
labels on a Biba integrity lattice) that travels **with data across every agent and tool
boundary** and is consumed by a **deterministic, model-free gate** at each. MCP is how agents
reach tools; A2A is how agents reach agents; PTC is how trust travels across both.

### 1.1 In scope

- The **trust-context object** (envelope schema, provenance-chain entry, taint derivation, and
  signing envelope (§5)) and its **carriage rule**: the object rides orthogonally to transport
  (an MCP `_meta` field, an A2A extension, a native channel binding) and is verifiable
  independently of it (§4).
- The **gate contract**: the deterministic decision function at each boundary, its closed verb
  set, and the confinement of model judgment to the suspicion-only direction (§6.8–§6.9).
- The **signing contract**: DSSE-enveloped, in-toto-style attestation of the provenance chain,
  keyed to a broker-held workload identity (§6.6–§6.7).
- The **provenance-maturity ladder** and its join to the autonomy lifecycle (§7); **conformance
  roles and numbered clauses** (§8).

### 1.2 Non-goals

- **Transport.** MCP and A2A are adopted, not competed with. PTC defines no wire binding of its
  own; every transport binding is implementation-tier. This includes **transport peer
  authentication**: PTC deliberately places non-repudiation at the *message* layer, a DSSE-signed
  chain (§6.6), rather than at the session layer (mutual TLS). The two are not equivalent and the
  choice is not a claim that transport authentication is unnecessary. A signature survives the
  relay hops that terminate a TLS session, which is the property a multi-zone mesh needs; mutual
  TLS in turn authenticates a peer that PTC's chain does not otherwise identify. A deployment
  SHOULD do both. This specification constrains only the former.
- **Policy language.** PTC requires the gate to be a pure, deterministic function with a specific
  verb alphabet and a stated expressiveness floor (§6.8); it does not mandate a policy language.
  Cedar and OPA/Rego were evaluated by differential test against the reference gate over its full
  reachable input space (all 73,728 reachable fact combinations) and declined with a published
  record: Cedar cannot checkably express first-match ordering, cannot express the value-producing
  `transform` verb, and **has no quantifier**, so existential reasoning over provenance records and
  over chain hops, this specification's own subject matter, is inexpressible in it. Rego
  reproduces the gate exactly but forfeits the purity and analyzability that motivated the question.
  The requirement is the behavioral contract, not the engine.
- **Model-screening internals.** PTC constrains what a content screen may *do* (refuse or pass,
  never bless; §6.9); the classifier inside a screen is out of scope.

### 1.3 Explicitly out of scope for this version (future extensions)

- **Cross-relay per-signer attribution.** In this version signing is **per-envelope**: a relay
  re-packages the envelope, so upstream signatures are not carried forward; upstream hops still
  ride as lineage. Attributing each intermediate signer *across* a relay requires nested per-hop
  attestations (tracked as reference-implementation issue [#13](https://github.com/wjatx/ptc-gal-reference/issues/13)).
- **Selective content disclosure.** The provenance chain is a contentless index; drill-down to
  upstream source content with least-disclosure semantics (SD-JWT-VC) is reserved.
- **Key-custody attestation at enrolment.** Tier 3 rests on a receiver's record of where a
  signer's key sits (§7.1), and in this version that record is `declared`: the receiver checks
  nothing. Evidence a receiver can check is a future extension. Its shape is a signing key
  generated in hardware or a managed key service that attests the key cannot be exported, with
  the attestation checked once, when the key is enrolled; TPM key certification, platform key
  attestation and WebAuthn attestation are the precedents. An attestation of non-exportability
  alone will not be sufficient, because an agent that can make the key's holder sign has no need
  to export anything: the attestation has to cover the key's use policy, binding signing to the
  broker's code identity. The evidence class `attested` (§7.1) is reserved for this.
- **Remote attestation of the signer's deployment.** Evidence checked at enrolment does not
  detect a signer whose arrangement later drifts from what was enrolled. Fresh evidence per
  session, in the manner of the RATS architecture (RFC 9334) with claims carried in an Entity
  Attestation Token (RFC 9711), is reserved.

## 2. Terminology and conformance language

The key words MUST, MUST NOT, REQUIRED, SHALL, SHALL NOT, SHOULD, SHOULD NOT, RECOMMENDED, MAY, and
OPTIONAL are to be interpreted as described in RFC 2119 as clarified by RFC 8174, when and only
when they appear in all capitals.

**Implementation status markers.** This specification is derived from a running reference
implementation rather than drafted ahead of one, and its normative text is overwhelmingly a
description of mechanism that exists and has been exercised. Where a clause is deliberately
normative *ahead* of that implementation, because the design question is settled and the
specification tier is the honest place to settle it while the code catches up, it is marked
wherever a reader can meet it, with a line of exactly this form:

> **Implementation status:** NORMATIVE, NOT YET IMPLEMENTED in the reference implementation (tracking: #NNN).

That line is the template. In a real marker `#NNN` is a number: the issue in the reference
implementation's public tracker (<https://github.com/wjatx/ptc-gal-reference/issues>) where the
work to implement the clause is recorded. Where a clause is only partly unimplemented, the marker
goes on to say which part it applies to. The literal prefix `**Implementation status:**` is fixed
so that the markers can be found by a text search.

A table row carries the short inline form `(not yet implemented, #NNN)`, which means the same
thing; inside the parentheses it may first name the part of the clause it applies to. A marked clause is fully normative: an implementation claiming conformance MUST satisfy
it, and the marker is simultaneously a statement that the reference implementation does not yet.
**Absence of the marker means the clause is implemented in the reference implementation**, which
is what makes its presence worth anything, and no conformance claim, by this project or any
other, may cite a marked clause as a shipped control. A specification clause is a requirement,
never evidence that anything enforces it.

**The inverse marker.** Drift runs both ways, and a clause the reference implementation has
outgrown takes the mirror form:

> **Implementation status:** NORMATIVE, SATISFIED BY A STRONGER MECHANISM in the reference implementation (tracking: #NNN). <Named mechanism, and why it dominates the one this clause names.>

A table row carries the short inline form `(stronger mechanism, #NNN)`. It applies **only** to a
clause mandating a mechanism, an ordering, or a procedure, never to one stating a property: a
property can only be met or not met, so a property-shaped clause the implementation does not hold
is a defect rather than a supersession. The marker MUST name the replacing mechanism and say why
it dominates, meaning it holds every outcome the mandated mechanism guarantees and forecloses a
failure that mechanism admits; merely different is not stronger. The clause stays fully normative
and literal conformance remains conformance. Unlike the marker above, this one records that the
*specification* is expected to move, and its tracking issue is the specification revision.

**Absence of either marker** means the clause is implemented as written. The convention is shared
with the companion GAL specification, which defines it in full in its §3.

| Term | Definition |
|---|---|
| **principal** | The identity under which an agent acts and against which authority, budgets, and audit records are scoped. An envelope addresses exactly one target principal. |
| **turn** | The unit of taint accumulation: a bounded span of agent activity whose identity is minted and owned by the broker, never declared by the agent. Taint accumulates within a turn and is cleared only by broker- or harness-owned rollover. |
| **broker** | The deterministic enforcement point that mediates every tool call an agent makes. The agent holds no credentials; its only egress is the broker. The broker holds the signing key, stamps provenance, and writes audit records under an identity the agent cannot reach. |
| **gate** | A deterministic decision function at an agent or tool boundary, evaluating a call or message against pre-resolved facts. The **airlock** (the receiving-side gate pipeline that validates, verifies, trust-maps, deduplicates, screens, and stamps an inbound envelope) and the broker's policy decision point are both gates. |
| **sender class** | The receiver-derived classification of who addressed the agent: `owner`, `peer-agent`, or `external`. A floor on treatment, never a grant of content trust. |
| **provenance chain** | The ordered, append-only, non-strippable sequence of hop entries recording where content came from and what each zone judged about its own source. A replayable index, not a content log. |
| **taint** | The derived integrity property of content or of a turn, fixed by the trust of its sources through deterministic lookup, never by model judgment of content. Non-strippable within a turn. |
| **endorsement / declassification** | The explicit, audited, configuration-declared act of raising a source's integrity above the floor, exempting its reads from taint. The only mechanism by which lower-integrity data may influence a higher-integrity action. |
| **trust zone** | One broker's span of control: the agent(s), airlock, and stores governed by a single enforcing broker. Provenance hops and signatures are per-zone. A zone is named by a **zone id**, an opaque string compared exactly. Two deployments MUST NOT share a zone id, and two environments of one agent are two deployments (§6.7). |
| **envelope** | The typed trust-context object (§5) that any inbound or outbound signal becomes; it crosses zone boundaries. |

## 3. Enumerations (closed vocabularies)

Unless a table below states otherwise, these vocabularies are **closed**: an implementation MUST
NOT extend them (see §10).

### 3.1 Sender classes

Closed set of three. "Unmapped" is deliberately **not** a fourth class: it is the absence of a
mapping, and it short-circuits the pipeline as a drop.

| Class | Who | What verified authenticity raises | Receiver hop label |
|---|---|---|---|
| `owner` | the target principal's accountable human | the full command surface: commands, questions, approval responses | `trusted` |
| `peer-agent` | an authenticated peer agent the receiver has pre-declared | may wake the worker and deliver signals addressed to the mapped principal | `trusted`: the *hop* is authentic; chain-inherited taint persists untouched |
| `external` | a verified-but-external sender | the minimum: delivery to the mapped principal, subject to every downstream gate | `untrusted`: always taints |

The hop label is a fixed function of the class, not a per-entry configuration knob. What varies
per deployment is *membership* (which identities map to which class), never the label semantics.

### 3.2 Biba integrity levels

Higher is more trusted. The sender classes map onto the lattice: `owner` → `USER`, `peer-agent` →
`AGENT`, `external` → `UNTRUSTED_WEB`; a tool-connector read enters at `TOOL_OUTPUT`.

| Level | Meaning |
|---|---|
| `SYSTEM` | platform/configuration provenance; the enforcement machinery itself |
| `USER` | the accountable owner |
| `AGENT` | an authenticated peer agent |
| `TOOL_OUTPUT` | content returned by a tool/connector read |
| `UNTRUSTED_WEB` | external, unendorsed content |

Enforcement in this version is the lattice's **two-level projection**: a turn is either at-or-above the
external-write integrity floor (`tainted = false`) or below it (`tainted = true`). The five-level
vocabulary and the no-write-up rule are normative; full multi-level enforcement (per-action
minimum levels, receiver-derived per-hop levels) is deferred to a later version (§7).

### 3.3 Decision verbs

The gate's closed verb alphabet. Exactly five; `transform` is value-producing (it emits a
substituted operation plus clamped arguments, not a boolean).

| Verb | Meaning |
|---|---|
| `allow` | execute the call as asked |
| `deny` | refuse the call |
| `transform` | execute a safer variant: a substituted operation plus clamped arguments (argument clamping: not yet implemented, #16) |
| `require_approval` | hold the call for out-of-band human ratification before execution |
| `abstain` | decline to decide; surface to a human without executing |

> **Implementation status:** NORMATIVE, NOT YET IMPLEMENTED in the reference implementation
> (tracking: #16). Applies to the **argument-clamping half of `transform` only**: the reference
> gate substitutes the operation and passes arguments through byte-for-byte. Operation
> substitution is implemented and exercised; the other four verbs are unaffected.

Grant levels (`in-loop` / `on-loop` / `out-of-loop`) are **not** PTC vocabulary: they are the
autonomy rungs of the companion GAL specification, orthogonal to the five verbs; PTC constrains
them only through the maturity ladder (§7).

### 3.4 Provenance hop labels

| Label | Meaning |
|---|---|
| `trusted` | the appending zone judged its own source trusted under its own trust map |
| `untrusted` | the appending zone judged its own source untrusted; taints every downstream turn |

### 3.5 Drop reasons

The closed reason vocabulary for recorded drops at the receiving airlock.

| Reason | Emitting gate |
|---|---|
| `authenticity_failed` | transport verification |
| `malformed` | schema validation, including the wire-form checks of §5.7 |
| `audience_mismatch` | audience check |
| `chain_signature_missing` | chain verification (when enabled) |
| `chain_signature_invalid` | chain verification (when enabled) |
| `chain_signer_unknown` | chain verification (when enabled) |
| `expired` | expiry check |
| `unmapped` | trust mapping |
| `principal_mismatch` | trust mapping |
| `screen_refused` | content screen (when enabled) |

A drop record MAY carry a `detail` machine code (pattern `^[a-z][a-z0-9_]{0,63}$`, never free
text). Deduplicated replays are silent no-ops, not drops.

### 3.6 Screen refuse codes

The base reserves exactly two codes; implementations extend the vocabulary **in configuration or
source, never at runtime** (see §10).

| Code | Meaning |
|---|---|
| `injection_suspected` | the canonical suspicion verdict |
| `screen_error` | reserved for the fail-closed seam backstop: an exception escaping the screen resolves as this refusal |

### 3.7 Attribution bases (observer role)

| Basis | Attributed to | Throttle-eligible |
|---|---|---|
| `signed-chain` | the verified full-cover signer's key id | yes |
| `transport-token` | the transport-layer identity that actually sent the bytes | yes |
| `unattributable` | nobody (reported, never counted toward a throttle) | no |
| `principal` | a broker principal + operation (flood signals; no sender identity) | no |

### 3.8 Observer remediation vocabulary

Closed set of five machine codes, each naming a concrete lever; a report suggests, a human
enacts; the vocabulary deliberately contains no shed, deny, or drop action (PTC-41):

`remove_trust_map_entry` · `rotate_channel_token` · `unenroll_verify_key` ·
`review_screen_config` · `review_pending_approvals`

## 4. The carriage rule

The trust-context object MUST ride orthogonally to transport and MUST be verifiable independently
of it. Conforming carriages include an MCP `_meta` field, an A2A extension, and a native channel
binding; no field of the object may close over a wire technology. The open vocabularies
(`channel_type`, provenance `source` schemes) exist precisely so transports stay implementation-tier.

## 5. The trust-context object

The object is a typed envelope (called **EventTrigger** in the reference implementation) that any
inbound or outbound signal becomes. One record type crosses the zone boundary at two seams: a
sending broker's outbound path emits it; a receiving airlock re-validates it, runs its gates, and
hands the stamped envelope to the worker.

### 5.1 Envelope

| Field | Type | Required | Description |
|---|---|---|---|
| `schema_version` | integer | yes | Wire versioning; two zones deploy independently. `1` in this version. |
| `event_id` | string | yes | Stable, sender-chosen idempotency key. Dedupe scope is `(sender.channel_identity, event_id)`; also the cross-zone audit join key. |
| `principal` | string | yes | The target principal the event addresses. The receiver MUST verify it serves this principal; a mismatch is a drop, never a re-route. |
| `audience` | string | yes | The zone id of the one receiver the envelope is addressed to. Set by the sending broker from its own configuration of the peer, never by the agent (broker authorship of this field: not yet implemented, #15). A receiver MUST refuse an envelope whose `audience` is not its own zone id, compared exactly (§6.1). |
| `sender` | SenderIdentity | yes | Who, at the transport layer, plus verification evidence (§5.2). |
| `payload` | object | yes | The parsed, normalized payload: size-bounded; never the raw original. |
| `payload_digest` | string | no | `"sha256:<hex>"` over the raw original artifact: the algorithm name `sha256`, a colon, and exactly 64 lowercase hexadecimal digits, with nothing before or after. One raw original has one spelling. REQUIRED when `payload_ref` is set. |
| `payload_ref` | string | no | Opaque pointer to the stored raw original; fetched only via a brokered (taint-tracked) read. |
| `provenance` | ProvenanceEntry[] | yes | Append-only chain, minimum length 1. Taint derives from this (§5.4). |
| `chain_signatures` | ChainSignature[] | yes | Signatures over the envelope as it left the signing zone (§5.5, §6.6); `[]` on an unsigned envelope. At most 8. |
| `sender_class` | enum §3.1 | no | RECEIVER-owned. Absent (null) on the wire. A receiver MUST discard any inbound value before any gate reads the envelope; the field is set only by the receiver's trust-map gate. |
| `ts` | string | yes | ISO-8601 UTC creation time. |
| `expiry` | string | yes | Hard TTL. Expired envelopes MUST drop before any budget-spending gate. |

> **Implementation status:** NORMATIVE, NOT YET IMPLEMENTED in the reference implementation (tracking: #15). Applies to **broker authorship of `audience` only**, with the other broker-set fields of §6.5: the reference connector sends the envelope it is handed, so the field holds what the caller wrote. The receiver's check of the field (§6.1, PTC-45) is implemented.

### 5.2 SenderIdentity

| Field | Type | Required | Description |
|---|---|---|---|
| `channel_type` | string | yes | Open, namespaced vocabulary (`"telegram"`, `"peer-agent"`, …), never a closed transport enum. |
| `channel_identity` | string | yes | Transport-level identity (chat id, From-domain, publisher id): opaque; normalized by the receiving adapter. When the chain is signed, its canonicalized value is bound into the signature (§6.6). |
| `evidence` | string[] | yes | `"{scheme}:{result}"` checks stamped by the adapter that PERFORMED the verification (e.g. `"dkim:pass"`). A receiver MUST NOT treat sender-asserted evidence as its own. |

### 5.3 ProvenanceEntry (one hop)

| Field | Type | Required | Description |
|---|---|---|---|
| `zone` | string | yes | The agent/airlock zone that appended this entry. |
| `source` | string | yes | Namespaced source id, `{scheme}:{value}` (e.g. `"email:vendor.example.com"`, `"channel:telegram"`, `"peer:email-agent"`, `"connector:{tool}.{op}"`). |
| `evidence` | string[] | yes | The authenticity checks this zone performed for this hop. |
| `label` | enum §3.4 | yes | This zone's judgment of ITS OWN source under ITS OWN trust map. |
| `ts` | string | yes | ISO-8601 UTC. |

### 5.4 Taint (derived — there is no field)

There is deliberately **no writable taint field** on the envelope. The envelope-level taint is a
derived property:

> An envelope is tainted **iff any provenance entry carries `label: "untrusted"`**.

Inside a zone, taint is held on the broker-owned turn, not on the envelope: at ingestion each
provenance `source` is fed into the turn, and the turn's taint state is what the gate reads and
the audit surface records; source-based, path-recorded, non-strippable (§6.3). On the brokered
call the gate evaluates, this state appears as `taint: { tainted: boolean, sources: string[] }`
alongside the turn's ingested-source record (`session.ingestedSources`), populated by the
broker, never model-supplied.

### 5.5 ChainSignature

| Field | Type | Required | Description |
|---|---|---|---|
| `key_id` | string | yes | The signing broker's workload-identity key id; a receiver resolves it to a public key to verify. |
| `zone` | string | yes | The zone that signed (attribution). MUST equal the `zone` of the last provenance entry. |
| `covers` | integer | yes | The number of provenance entries the signature commits to. MUST equal `len(provenance)`: every signature covers the whole chain (§6.6). |
| `payload_type` | string | yes | The DSSE `payloadType` bound into the signed pre-authentication encoding. MUST be `application/vnd.in-toto+json`. |
| `sig` | string | yes | The Ed25519 signature over the DSSE PAE of the in-toto-style statement (§6.6), as the canonical padded base64 (RFC 4648 §4) of exactly 64 bytes. A receiver MUST reject any other spelling of the same bytes, so that one signature has one wire form. |

### 5.6 Example

An envelope carrying a supplier invoice for payment approval, as received, before the receiver's
own hop is stamped. The scenario is deliberately consequential: the action the payload invites
(releasing a payment) is worth more than the effort of forging the email that carried it, which is
the case taint exists for.

```json
{
  "schema_version": 1,
  "event_id": "inv-8842-a1",
  "principal": "example-agent",
  "audience": "payments-agent-production",
  "sender": { "channel_type": "webhook", "channel_identity": "peer:email-agent",
              "evidence": ["hmac:pass"] },
  "payload": { "invoice_ref": "INV-4471", "action": "approve_payment", "amount": "12400.00" },
  "payload_digest": "sha256:9f2c…",
  "payload_ref": "artifact://email-agent/raw/8842",
  "provenance": [
    { "zone": "email-agent", "source": "email:vendor.example.com", "evidence": ["dkim:pass"],
      "label": "untrusted", "ts": "2026-07-24T14:03:01Z" },
    { "zone": "email-agent", "source": "peer:email-agent", "evidence": [],
      "label": "untrusted", "ts": "2026-07-24T14:03:02Z" }
  ],
  "chain_signatures": [
    { "key_id": "email-agent-broker-1", "zone": "email-agent", "covers": 2,
      "payload_type": "application/vnd.in-toto+json", "sig": "base64…" }
  ],
  "sender_class": null,
  "ts": "2026-07-24T14:03:02Z",
  "expiry": "2026-07-24T15:03:02Z"
}
```

The envelope is tainted (an `untrusted` entry exists) and non-strippably so: nothing the sender
did after reading the email removed the entry. Note what `dkim:pass` does and does not buy: the
mail was genuinely sent by the domain it claims, and the content is still untrusted, because
authenticity of a sender is not trustworthiness of content. `sender_class` is null because only the
receiver may set it. `audience` names the one receiver the sending broker addressed, and the
signature covers it, so the same bytes delivered to any other receiver are refused there (§6.1).

### 5.7 Wire form

A signature commits to bytes, so an envelope has exactly one serialized form and a bounded size.

- **One form.** The wire form is a JSON object in ASCII, with every non-ASCII character escaped.
  A Producer MUST refuse to send, and a Receiver MUST refuse to forward, an envelope whose wire
  form does not parse back to an envelope equal to the one serialized. A value JSON cannot
  carry unchanged (a non-finite number, a non-string object key) is rewritten in transit, and a
  signature over the in-process value would then cover content the receiver never sees.
- **Bounded.** The wire form of an envelope a Producer sends MUST NOT exceed 196,608 bytes, and
  a Receiver MUST refuse a larger one. A Receiver that forwards an accepted envelope adds its
  own provenance entry and the sender class; the forwarded form MUST NOT exceed 262,144 bytes.
  The gap between the two ceilings is reserved for the receiver's own additions.
  `chain_signatures` holds at most 8 entries.
- **Refused before acceptance.** A Receiver applies these checks at schema validation, before
  any gate that records the message as seen. An envelope that passes every gate and then
  cannot be forwarded would otherwise claim its idempotency key and be lost.

## 6. Normative behavior

### 6.1 Ingestion stamping

On accepting an inbound envelope, the receiving airlock MUST:

1. Re-validate the envelope against the schema (§5), including the wire form (§5.7), and discard
   any inbound `sender_class`. The receiver trusts nothing but what the envelope carries and
   what its own gates re-verify; no store is shared between zones.
2. Refuse an envelope whose `audience` is not the receiver's own zone id, compared exactly, as
   an `audience_mismatch` drop. This check MUST run before chain verification (§6.7) and before
   deduplication. The signer addressed the envelope to another receiver and did nothing wrong,
   so the drop record carries no verification evidence and names no signer; and a misdelivered
   envelope claims no idempotency key. The check runs whether or not verification is enabled.
3. Resolve `(sender.channel_type, sender.channel_identity)` through its own trust map: a
   deterministic, configuration-derived, exact-match lookup; never a model call, never content
   inspection. An unmapped identity, however well-authenticated, is a **drop**: no envelope, no
   screening spend, no reply toward the sender; always recorded, PII-safely (identity as digest).
4. Append **exactly one** receiver provenance entry, whose `label` is the fixed function of the
   resolved sender class (§3.1), and set `sender_class`, overwriting any inbound value: exactly
   one stamped envelope per deduplicated message (dedupe key `(sender.channel_identity,
   event_id)`); replays are silent no-ops.
5. Feed every provenance `source` into the broker-held receiving turn before the worker acts.

**What the sender learns.** A refusal at the airlock (an unmapped identity, a principal the
receiver does not serve, an envelope addressed to another receiver, an expired envelope, a failed
verification, a screen refusal) discloses no gate, entry, path, signature or evaluated chain state
beyond what the receiver already publishes. The refusal is recorded as a drop, where the operator
reads it. What a sender learns beyond that depends on two things the receiver already knows:
whether it authenticated and mapped the sender before it evaluated, and whether the refusal is
about authority or about content.

- Toward a sender the airlock authenticated and mapped before it evaluated, a refusal of
  authority MUST be distinguishable as permanent or as transient, and as nothing more. A refusal
  is permanent when the airlock evaluated the envelope and would refuse it again if it were
  presented unchanged. It is transient when the airlock could not evaluate, because an input it
  needs (a key source, a status source) was unavailable.
- Toward every other sender, every delivered envelope SHOULD receive the same response, for
  example the transport's success status, so that the response itself carries no verdict.
- A screen refusal MUST be indistinguishable from acceptance toward every sender.

> **Implementation status:** NORMATIVE, NOT YET IMPLEMENTED in the reference implementation (tracking: #174). Applies to the **permanent-or-transient classification toward an authenticated and mapped sender**; the uniform response toward every other sender, and a screen refusal that reads as acceptance, are implemented.

What this prevents is the error oracle, and it has two attackers. A stranger probing the ingress
learns the receiver's trust configuration from any answer that differs between refusals, and a
"transient" answer tells it that an attack on a key source or a status source is working, so a
stranger learns nothing. A mapped peer that has been compromised can authenticate, and what it
wants is feedback on content, which payload the screen let through, so no sender ever receives a
verdict on content. A legitimate mapped peer needs one thing to do its work, whether to retry or
to stop, and that fact describes the receiver's own state and not the sender's chain. This is the
rule verifier-side guidance states for an authenticated caller
(`draft-jackson-wimse-evaluation` §5).

Taint derivation at ingestion is **deterministic and never model-judged**: a source taints the
receiving turn if *either* its chain label is `untrusted` *or* the receiver's own input trust map
does not trust it.

### 6.2 The one-way rule

> **No output of trust-mapping, or of any gate, may lower the taint derived from the provenance
> chain.** A gate may only *add* the receiver's own entry and set the receiver-owned
> `sender_class`. No resolution result renders a tainted envelope clean.

Three consequences, each a conformance clause (§8):

1. **Authenticity never cleans.** An `owner` or `peer-agent` resolution over a chain containing an
   `untrusted` origin appends a `trusted` hop, and the envelope stays tainted, because derived
   taint is any-untrusted over the whole chain. Hop authenticity is orthogonal to content trust.
2. **Sender-asserted labels are a floor, never a grant.** An `untrusted` entry taints the
   receiving turn even where the receiver's map would trust that source; a `trusted` entry earns
   nothing the receiver's map does not grant: a compromised peer stamping `trusted` launders nothing.
3. **Screens refuse or pass, never bless** (§6.9).

### 6.3 Non-strippability and the append-only chain

- The provenance chain is **append-only**: a hop appends via a frozen additive path; no operation
  edits or removes prior entries.
- Turn taint is **non-strippable**: once a turn is tainted it cannot be un-tainted within its
  lifetime. The flag has no setter; the gate derives every call's taint from the broker-held turn,
  ignoring any value the model supplies. Turn identity is broker-owned: the agent supplies
  neither the turn id nor a rollover signal, so it cannot launder taint by declaring a fresh turn.
- Taint is cleared **only** by broker/harness-owned turn rollover or session expiry. There is
  deliberately no taint-clearing API reachable by the agent; model output and later clean reads
  in the same turn MUST NOT clear taint.

**Delegation does not launder.** A context an agent causes to exist outside its zone, or to run
later, MUST begin with a turn at least as tainted as the turn that caused it. Delegation MUST NOT
yield a cleaner turn than the delegator's.

> **Implementation status:** NORMATIVE, NOT YET IMPLEMENTED in the reference implementation (tracking: #177).

What this prevents is an injection laundered through delegation. An agent that has read planted
content cannot make an external write without a hold (§6.4). If it can hand the task to a worker
in another zone, to a queue another agent drains, or to its own later run, and that context starts
clean, then the planted content has only to ask for the hand-off, and every component does what it
was built to do. Broker-owned turn identity closes the move of declaring a fresh turn; this closes
the same move made through a second context. A publish to a peer is already covered: it is an
external write, held on a tainted turn, and it carries the turn's sources with it (§6.5).

### 6.4 No-write-up and audited declassification

The Biba rule: data at integrity level *L* MUST NOT flow into an action requiring a level *> L*
without an explicit, **audited endorsement**. In the two-level projection, the structural cut:

> **A tainted turn's external write escalates.** When the turn is tainted and the requested
> operation is an external write, the gate MUST route the call through the deployment's polarity
> seam to `require_approval` where a human is reachable, `deny` where none is. The cut is
> **grant-independent** (no autonomy rung routes around it), floor, never tunable off; and
> `transform` MUST NOT silently downgrade a tainted write.

Tainted **reads** are allowed and audited: they are what taints. Internal (non-boundary-crossing)
writes are not taint-gated by this floor.

**A destination the agent composed.** A read is not always only a read. Where an operation takes
an argument that names where the request goes (an address, a host, a path on another system) and
the agent composed that argument, a tainted turn makes the destination something an attacker could
have chosen. An implementation MUST provide a control under which, on a tainted turn, a call whose
destination the agent composed is treated as an external write and routed through the escalation
above. Which argument of an operation is its destination is declared in reviewed configuration, as
the operation's other facts are. A deployment MAY leave the control off.

> **Implementation status:** NORMATIVE, NOT YET IMPLEMENTED in the reference implementation (tracking: #178).

What this prevents is request forgery steered by injected content: the agent reads text an
attacker planted, the text supplies an address, and a tool that holds network position the
attacker lacks fetches it. The control trusts no destination. It asks who could have chosen this
one. It does not stop request forgery inside a tool server: a server that follows a redirect, or
resolves a name to an internal address, does so after the gate has decided, and only the component
that opens the connection, or a network boundary around it, can check where a request lands. A
list of approved destinations checked at the gate is defeated the same way, which is why this
specification defines none. Containment closes that gap, and this control narrows it.

**Endorsement** is the consumer's trusted-sources declaration: naming a source (e.g.
`connector:{tool}.{op}`) as trusted asserts a **declassification**: "raise this source's ingest
above the integrity floor for this agent." Endorsement MUST be declared in reviewed
configuration, never by the model or agent at runtime; it endorses a *source*, never individual
content; it can only ever *raise* an ingest's level; every exemption lands on the audit surface (§9).

### 6.5 Outbound stamping

The agent supplies the *intent* of an outbound publish; the broker constructs the *envelope*:

- The agent authors `event_id`, target `principal`, and `payload`, and nothing that could launder
  taint or forge provenance. The broker sets `audience`, `sender`, `provenance`, `ts`, and
  `expiry`. `audience` comes from the broker's own configuration of the peer it is sending to:
  the agent chooses which declared peer to address and never writes the zone id.
  `sender_class` is absent on the wire; a sender never asserts its own class.
- **The sending zone's provenance entry is broker-stamped from the sending turn's taint state,
  never agent-authored.** A tainted turn yields an `untrusted` entry the agent can neither remove
  nor relabel; the label is a deterministic function of the turn, never a model judgment.
- **Lineage, not a collapsed bit.** A fresh origination carries the turn's actual ingested taint
  sources into the outbound chain as `untrusted` origin hops ahead of the sending zone's own hop,
  so the receiver re-derives taint from the true origin and a human sees it. A relay appends its
  hop and never edits the upstream chain; sources already carried inbound are not duplicated.
- **A publish is an external write.** On a tainted turn it rides the standing escalation cut
  (§6.4); no publish-specific rule exists. The sender's only path to emit tainted content across
  a zone boundary is the honest, human-gated one.
- The transport credential that authenticates to the peer is held by the broker, never the agent.

### 6.6 Signing

Signing turns the chain from *asserted* into *authenticated*: non-repudiation of *who asserted a
hop*, never a taint mechanism, never a correctness proof, never a substitute for the receiver's
own trust map.

- **Shape.** The signature is an **Ed25519** signature over a **DSSE pre-authentication encoding**
  (`DSSEv1 <len> <payloadType> <len> <payload>`) of a canonical-JSON (sorted keys, no whitespace,
  ASCII) **in-toto-style statement**. The statement binds **the whole envelope**, in four parts:

  1. `subject`: a canonical hash of the actual inline `payload`, always, which the verifier
     recomputes from the envelope it was handed. No other field can stand in for it. When the
     envelope names a raw original, a second subject binds its `payload_digest` and, when set,
     its `payload_ref`, so a reference or digest cannot be attached to, changed on, or stripped
     from a signed envelope. That the bytes behind the reference match the digest is the
     dereferencing zone's check, not the signature's.
  2. The ordered provenance hops, every one of them.
  3. The signature's own `key_id`/`zone`, so attribution is non-malleable.
  4. **Every other field of the envelope**, with exactly two exceptions: `sender_class`, which
     is the receiver's to set and which the receiver discards on arrival, and
     `chain_signatures`, which are the signatures themselves.

  The fourth part is defined by exclusion on purpose. A statement that enumerates the fields it
  signs leaves out every field nobody thought to add, and leaves out, silently, every field a
  later version introduces. A Producer MUST sign, and a Receiver MUST verify, the envelope minus
  the named exceptions; a field added to the envelope is bound from the day it exists, and
  removing a field from the signed set is a versioned change to this specification (§10).

  What that binding buys, field by field: `event_id` and `sender.channel_identity` are the two
  halves of the dedupe key, so an exact replay dedupes and any mutation of either breaks the
  signature and classifies as forgery, attributed to transport only, never the impersonated
  signer. `expiry` cannot be extended. `audience` cannot be rewritten to another receiver.
  `ts` and `sender.evidence`, which receivers record, cannot be altered. `sender.channel_identity`
  is bound in its canonical form (surrounding whitespace removed, case-folded), because the
  receiving adapter canonicalizes it before the gate runs and an honest sender's spelling must
  still verify. The `principal` is bound for a second reason: a receiver that does not share the
  sender's log cannot recover the on-behalf-of principal from the chain, so a statement that
  omitted it would authenticate an action without saying whom it was taken for. This is the
  distinction OAuth 2.0 Token Exchange (RFC 8693) draws between its `sub` and `act` claims.
- **Refusal to sign.** A Producer MUST refuse to sign an envelope the statement cannot name
  exactly: a payload that does not survive its own JSON serialization unchanged, a `payload_ref`
  with no `payload_digest`, or a `payload_digest` outside the one spelling of §5.1. A Receiver
  treats the same condition as a failed signature. Signing less than the whole envelope is never
  the fallback.
- **Identifiers.** The DSSE `payloadType` MUST be `application/vnd.in-toto+json`, and the
  statement's `_type` MUST be `https://in-toto.io/Statement/v1`. These are in-toto's own
  registered identifiers [IN-TOTO], adopted rather than minted: a PTC statement is an in-toto
  Statement, and a verifier that already understands the framework MUST NOT have to learn a
  second envelope spelling to parse one.

  The `predicateType` is a different matter, because the predicate is PTC's own. This
  specification pins its **shape** and not its value: a `predicateType` MUST be an absolute
  URI, MUST carry an explicit version segment, and MUST identify exactly one statement kind;
  a single URI MUST NOT cover two predicate shapes, and a change to a predicate's fields MUST
  bump its URI rather than redefine one in place, so a verifier can reject a shape it does not
  know instead of mis-parsing it. The **namespace itself is to be assigned by the standards
  body** adopting this specification; it is deliberately left open here, because a vendor
  domain in a normative identifier makes every conforming implementation depend on one
  organization's DNS.

  *Reference implementation note (non-normative):* the reference implementation currently emits
  predicate types under `https://safe-agents.dev/…`, versioned and one per statement kind
  (provenance chain, promotion record, audit acknowledgment, tool admission). That is a vendor
  namespace serving as a placeholder and is expressly **not** proposed as the normative value.
- **Broker-keyed.** The private key is the broker's workload identity (SPIFFE/WIMSE substrate
  RECOMMENDED; DID fallback), resolved by the broker at start-up, never present in the agent
  image and never a caller assertion: **the agent cannot sign.** This is a requirement on the Producer,
  and nothing in a chain shows a Receiver whether it is met. What a Receiver records about it,
  and what follows from that record, is §7.1.
- **Per-envelope, full cover only.** The sending broker signs the envelope as it leaves, with
  the full chain (`covers = len(provenance)`), including preserved upstream hops. A signature
  over a prefix of the chain is not a valid signature in this version: a statement over a prefix
  says nothing about the hops after it, and it is byte-identical to a full-cover statement for
  the envelope with those later hops removed, so a party holding no key could cut a chain back
  to it. The verifier requires a signature's `zone` to equal the `zone` of the last provenance
  entry; inbound signatures are not carried across a relay (§1.3).
- **One key, one zone.** A signing key MUST NOT be shared between zones. Two environments of
  one agent are two zones (§2) and sign with two keys. Optional transparency-log anchoring of high-value chains (Sigstore/Rekor-style)
  is a deployment knob, OFF by default.

### 6.7 Receiver verification

Verification is a receiver-side knob that **MAY be off**; the consequence is expressed by the
maturity ladder (§7), not hidden: unsigned traffic caps the mesh at the unsigned-lineage tier,
and observer attribution degrades to transport-token strength.

**The audience check is not part of verification** and does not turn off with it: a receiver
refuses an envelope addressed elsewhere whether or not it verifies signatures (§6.1).

When verification is enabled, it MUST fail closed:

- An envelope with no signatures, an unknown signer, a `covers` other than `len(provenance)`, a
  `payload_type` other than the one DSSE type (§5.5), a signature whose `zone` is not the `zone`
  of the last provenance entry, or an invalid signature MUST be rejected: dropped and loudly
  quarantined, before any budget-spending gate (trust map, dedupe, screen). Every *present*
  signature must pass every check; one valid signature does not excuse another that fails.
- **A verification key is scoped, and the scope is the receiver's own configuration.** Being
  known to the receiver does not let a key speak for every peer. For each key it enrols, a
  receiver MUST record the one zone id the key may sign for and the set of
  `sender.channel_identity` values, in canonical form, that an envelope signed by that key may
  claim. A signature from a known key naming a different zone, or over an envelope whose sender
  identity is outside the key's set, MUST be rejected as an invalid signature. A key enrolled
  with no scope MUST be refused at configuration load, as MUST a key id enrolled twice. Without
  the scope, any enrolled peer could sign an envelope naming another peer's zone and identity,
  and the receiver would record it as verified.
- **A verification key carries a custody record.** For each key it enrols, a receiver MUST also
  record whether the signer holds the private key in agent-separated custody, and the evidence
  class of that record (§7.1). A key enrolled without a custody record MUST be refused at
  configuration load. The record has no bearing on whether a signature verifies: a chain signed
  by a key recorded outside agent-separated custody passes or fails on the checks in this
  section, and §7.1 holds it to tier 2.
- **Zone ids must be distinct for the audience check to mean anything.** Two receivers that
  share a zone id and enrol the same signer key accept each other's traffic, and nothing a
  receiver can compute locally detects that. A deployment MUST give each receiver its own zone
  id (§2); this specification states the requirement and cannot enforce it (§9).
- An envelope the receiving airlock constructs itself from a transport-authenticated request,
  as the owner channel does, crosses no zone boundary and carries no chain signature. Chain
  verification does not apply to it. The exemption MUST key on the receiver's own adapter, fixed
  in its image-baked configuration, never on anything the request carries.
- Misconfigured verification (a configured but unusable key source) MUST fail closed loudly,
  never silently degrade to no verification.
- A successful verification is recorded as evidence of a check **performed** (e.g. `sig:pass` in
  the receiver's hop evidence) only when the gate actually ran on this envelope and passed, never
  because verification is merely configured, and never for an unverified chain. Downstream consumers (§6.11) MUST NOT infer, recompute, or override these fields.
- Where every signature on a verified chain was made by a key recorded in agent-separated
  custody, the receiver records that too, under the same rule (only when the gate ran on this
  envelope and passed) and in a form that names the evidence class (e.g. `custody:declared`).
  That entry is evidence of a record **consulted**, never of a check performed on the peer.
- Verification authenticates lineage; it does **not** clean taint. The receiver re-derives taint
  from the chain regardless of signature status.

### 6.8 The deterministic gate

The gate contract is the invariant that makes everything above enforceable:

> **The model may only SURFACE a concern. It never sets a decision verb, never suppresses taint,
> and never modifies its own grant.** Every security-relevant fact the gate keys on comes from
> code and configuration, never from what a model said.

- The decision function MUST be **pure and deterministic**: no I/O, no model call, no side
  effects; the same inputs always produce the same decision. All environmental facts are resolved
  *before* the decision into a closed, pre-computed fact set; no fact may be model-derived.
- The verb alphabet is exactly the five verbs of §3.3. Rule evaluation is **first-match** over an
  ordered rule set; the matched rule's identity is recorded on the audit surface. A write no rule
  permits falls through to **default-deny**.
- The one model-authored input, the call's arguments, MUST be opaque to the verb: varying the
  arguments, including overt "this is pre-approved" injection text, never changes the verb.
- A model MAY surface findings, request a stricter outcome (`abstain` with escalation), or raise
  suspicion in a screen; it MUST NOT authorize, loosen, or decide. The model may tighten, never
  loosen: it is never the judge in the dangerous direction.

The safe-default polarity (whether an unresolvable case resolves to abstention or to a positive
safe action) is deliberately NOT fixed by this specification: it is re-derived per deployment at
a single, identifiable seam. Baking one polarity into the base is a latent safety bug for
deployments with the opposite harm profile. The seam is not merely a configuration convenience:
because a forced abstention is itself an attacker objective for one of the two polarities, the
gate's escalation is a *consequence to be monitored*, not an outcome to be trusted (§6.12).

This has a consequence for any policy language a deployment might substitute for the reference
gate. A language whose outcome alphabet is two-valued (permit / forbid) cannot express `transform`,
which must emit a substituted operation plus clamped arguments
(argument clamping: not yet implemented, #16), and for the same reason it cannot host the
polarity seam, since a positive safe action is a positive action *with arguments*. The
value-producing verb and the caller-owned polarity seam are one requirement seen twice. A
conforming gate language MUST therefore support a value-producing outcome, and MUST support
existential quantification over provenance records and over chain hops: the Biba direction of
§6.4 and the chain reasoning of §5.3 and §6.7 are both quantified shapes, and a language without a
quantifier cannot state this specification's own subject matter.

### 6.9 The content screen (the one model-judged gate)

A deployment MAY run a content screen (injection classifier) at the receiving airlock. It is the
**only** gate that may consult a model, and it is confined to the suspicion-only direction:

- **A screen refuses or passes; it never blesses.** A pass changes nothing: not the envelope,
  not the chain, not the derived taint, not the sender class. A pass is **contentless** (no
  reason, no score, nothing downstream could cache as a safety assertion).
- A refusal's reason is a **closed-vocabulary machine code** (§3.6), never free text: model
  free-text derived from an untrusted message is itself untrusted content, and writing it to the
  trusted drop log would hand an injected payload a path into the audit trail.
- The seam MUST fail closed: an exception escaping the screen resolves as a `screen_error`
  refusal, recorded and dedupe-marked. An implementation may choose fail-open *inside* the
  screen; what is forbidden is the silent version, where infrastructure failure is
  indistinguishable from a clean verdict.
- The screen ships OFF; taint and every floor property hold identically with or without it:
  defense in depth, never a taint dependency. Ordering MUST keep expired, unmapped, and replayed
  messages away from the screen, so screening spend is bounded by novel, authenticated, mapped
  traffic.

### 6.10 Tool discovery is untrusted input (the MCP posture)

Where an agent's tools are discovered from a tool server at connect time (MCP), the discovery
response is **untrusted input**: the server names its own tools, writes its own descriptions and
schemas, and can change any of them between connections. A conforming host MUST NOT treat an
advertised tool list as an authorization:

- **Two-key admission.** A tool is callable only when two independent facts, from configuration
  layers of different power, line up: (1) an **image-baked, code-reviewed declaration** of the
  `(server_id, tool_name)` namespace with the host-assigned effect classification the gate reads,
  never the server's self-description; and (2) an **integrity-protected registry activation**
  recording the definition-hash a human admitted. The mutable layer can select and tighten, never
  mint a callable; missing either key fails toward uncallable.
- **The signed definition set.** The admitted hash MUST cover **every advertised field** of the
  definition, as canonical-JSON `sha256`: the always-present core `(server_id, tool_name,
  input_schema, description)` plus each metadata field the server actually advertised. A field the
  server does not advertise MUST be **excluded from the preimage**, never serialized as a null:
  the host cannot distinguish an absent field from one advertised as null, so neither may the hash.
  A field appearing, changing, or vanishing IS drift. Two properties follow from the exclusion, and
  both are load-bearing: a definition advertising no metadata hashes identically to the core-only
  basis, so no deployment is forced through an **empty-delta re-vet** (a drift alarm with nothing
  to show teaches operators to dismiss the alarm that matters); and additive growth of the
  definition schema moves no hash until a server actually advertises the new field, so tracking an
  evolving tool-description format is not a fleet-wide re-admission event. `description` is
  deliberately in the set: it is the model-facing injection surface, and a description-only change
  IS drift, as is a flipped annotation hint or a rewritten output schema.
- **Drift quarantines.** At every connect the host re-hashes the live advertised definition. A
  mismatch drops the tool to uncallable, surfaces loudly exactly once, and only a fresh human
  ceremony re-admits. An undeclared or unadmitted discovered tool is uncallable and surfaces as a
  quarantine-class finding.
- **Admission is a ceremony**: two distinct verified credential identities (maker ≠ checker;
  self-admission structurally impossible) and an append-only, signed admission record binding who
  admitted which tool at which hash.
- **Responses taint** by default under source id `connector:{server_id}.{tool_name}` (§6.4).
  Per-tool endorsement is legal **only** for tools declaring structured (schema-bounded) output,
  refused at configuration load otherwise: free text is exactly the injection surface
  endorsement must not wave through.
- **Reconnect is a new discovery.** A restarted or reconnected server is a new server: discovery
  evaluation re-runs in full before any call; no verdict or finding is carried across a death,
  which surfaces as a typed, attributed failure, never an indistinct timeout.
- **Capability-naming configuration is image-baked only.** Anything that names code to run or an
  endpoint to reach (spawn commands, server URLs) is honored only from the reviewed, image-baked
  declaration, structurally out of reach of any runtime-mutable store.

*Reference implementation note (non-normative):* the reference host's operator tooling renders
description drift verbatim, in full: the countermeasure is against reviewer complacency, not invisibility.

### 6.11 The off-path observer (campaign correlation)

A deployment SHOULD run an off-path observer correlating the airlock's recorded exhaust (drops,
screen refusals, approval-queue flood signals) into attributed campaigns for human review. It:

- MUST NOT gate: it holds no on-path seam and cannot refuse, delay, or throttle a request. Its
  output is a report and a suggested remediation from the closed vocabulary (§3.8); applying a
  remediation is a human ceremony.
- MUST attribute at recorded strength (§3.7). Signer (`signed-chain`) attribution is legal only
  for reasons **both signature-bound and dedupe-capped** (in this version, exactly
  `screen_refused`); forgery-class reasons attribute to the transport identity **only, never the
  claimed signer**, or the observer becomes a reflected-denial-of-service weapon against the
  impersonated party; and pre-dedupe reasons cap at transport-token strength regardless of
  verification, or a captured genuinely-signed envelope could be replayed to accrue an unbounded
  attributed count against an innocent signer.
- MUST treat unattributable events as terminal (reported, never throttle-counted); MUST be
  PII-safe (digests, key ids, machine codes, counts, timestamps, opaque refs; never raw identity
  or content); and MUST NOT shed: the remediation vocabulary contains no shed/deny/drop action.

### 6.12 Availability: forced abstention and queue amplification

Every control above is an **integrity** control: it prevents an agent from being made to do the
wrong thing. An adversary who cannot achieve that can still achieve something valuable: making the
agent do **nothing**. Poison an input, and the no-write-up floor (PTC-27) escalates the tainted turn
exactly as specified; the agent falls silent. The escalation is correct. Whether the resulting
silence is *safe* is not a property this specification can decide, and a conforming deployment MUST
NOT treat "escalated" as "handled".

**Polarity decides the harm.** Under **abstain-is-safe** polarity, a forced abstention is the
attacker's whole objective: the agent's job was to act on demand, and silencing it is the win. Under
**positive-safe-action** polarity, a forced abstention causes the exact harm the agent exists to
prevent: a silenced monitor does not raise the alarm it was deployed to raise. The safe response to
a poisoned input is escalation *to a human*, not abstention; those are the same thing only when a
human is actually reached. Two requirements follow.

**Liveness: the forced-abstention half.**

- A deployment whose safe-default polarity is positive-safe-action MUST operate a **liveness
  contract**: a declared expected output and a deadline, monitored independently of the agent.
- The monitor MUST be **deterministic**. It observes *that* an expected output did not occur by its
  deadline; it MUST NOT judge *why*. No model may sit on this path. The temptation is a smarter
  monitor that decides whether a given abstention was legitimate; that reintroduces a probabilistic
  classifier onto the safety path (§6.8) and hands the attacker a second thing to fool. A timestamp
  comparison cannot be prompt-injected.
- The sign of life MUST be an artifact the agent can neither forge nor suppress without actually
  failing to act: the enforcement point's own audit or ledger append of the declared operation. A
  self-report from the agent is not a sign of life.
- A deployment whose polarity is abstain-is-safe MAY omit the monitor. The polarity derivation is
  per-deployment; an implementation MUST NOT fix it on the deployment's behalf (§6.8).

**Queue de-amplification: the escalation half.**

- Escalating to a human is the safe response to a poisoned input. An adversary able to trigger many
  calls converts one injection into a **flood of approval requests**: a denial of service against
  human attention, and, worse, cover under which the one approval that matters goes unread.
- A conforming deployment MUST de-amplify and MUST NOT shed. Identical pending approvals (same
  principal, operation, and arguments) SHOULD coalesce onto one; coalescing is safe under both
  polarities because it never hides a genuinely distinct approval. A queue-depth threshold MAY raise
  an alarm, and when it does the deployment MUST still hold every intent.
- Shedding under flood (denying, dropping, or auto-resolving queued approvals) re-enters the
  forced-abstention harm above through the mitigation itself, and MUST NOT be a default. This is the
  same rule the off-path observer obeys at its own layer (PTC-41), and it generalizes: **the
  remediation vocabulary of a control that fires under attack must not contain the attacker's
  objective.**

**An approval that expires undecided.** A held approval MAY expire. Its expiry period MUST NOT
shorten as the queue grows. An approval that expires without a human decision MUST be recorded as
expired, MUST be distinguishable on the audit record from a human denial, and MUST NOT count as a
human decision in any evidence the grant lifecycle reads. Where a queue-depth alarm is raised, each
approval that expires while it is raised MUST be reported to the approver, by count at least.

> **Implementation status:** NORMATIVE, NOT YET IMPLEMENTED in the reference implementation (tracking: #163).

What this prevents is shedding by another trigger. A timer is outside the rule against shedding
under load, because time fires it and load does not, and under a flood its effect is the same: the
one approval that mattered ages out behind the noise, and the record shows a hold followed by
nothing. An attacker who can raise a flood then gets the denial this section forbids, silently and
on a schedule. Expiry is kept, because approving a stale request is its own harm. What is forbidden
is an expiry that leaves no trace, or that reads as a person having said no.

### 6.13 Time: skew, and the age of an input

A gate decides at an instant, on inputs it fetched earlier, some of them stamped by a clock that
is not its own. Two rules keep either from extending authority.

**Skew fails toward less authority.** Where the evaluation instant is compared with a time
asserted under another party's clock, a declared skew bound MUST be applied in the direction that
confers less authority: an expiry or an authority-lowering record is treated as effective up to
the bound earlier, and an authority-raising record or a not-before time up to the bound later.

> **Implementation status:** NORMATIVE, NOT YET IMPLEMENTED in the reference implementation (tracking: #175).

What this prevents is authority that outlasts its end by the width of the allowance. A symmetric
leeway, which is what token libraries usually offer, honours an expired envelope for as long as
the leeway runs, and an attacker who can hold a message back, or who benefits from a slow clock,
gets that window for nothing. Applied one way, the bound costs an honest sender at most its own
width.

**Every fetched input has an age.** For each input a decision depended on that was fetched from a
source, the decision point MUST record the instant as of which it established that input. An input
MUST NOT be used past a maximum age the deployment declares for it. Where a source states the time
of its answer, the as-of instant is the earlier of that time and the time of the fetch.

A decision is good no longer than its shortest-lived input. The decision point MUST record the
earliest instant at which any input the decision depended on passes its maximum age, and a
decision MUST NOT be relied on past that instant: a call held on it, or carried on from it, is
evaluated again.

> **Implementation status:** NORMATIVE, NOT YET IMPLEMENTED in the reference implementation (tracking: #176).

What this prevents is a decision made on something that was true when it was fetched and is false
now: a verification key withdrawn after it was loaded, a manifest from which a tool has since been
removed, a trust map that has since been tightened. With no maximum age the old input stays in
force for as long as the process that read it keeps running, so a party that lost access by a
configuration change keeps it until a restart nobody scheduled. With no recorded instant an
auditor cannot tell a decision made on current inputs from one made on stale ones. Taking the
earlier of the two times follows the skew rule above: a replayed old answer carries its old time
and ages out, and a source whose clock runs fast, or that lies, cannot make its answer look fresher
than the moment it was fetched.

The decision-level instant prevents authority exercised on a decision that has outlived what
it was computed from. A call is permitted and held; while it waits, the key that verified its
chain is withdrawn; the hold is released on the strength of the earlier decision. Every input
was in date when it was read, and the action still runs on a withdrawn key. This is the rule a
validating resolver and a certification path follow: a result lives no longer than the
shortest-lived thing it was computed from.

## 7. The provenance-maturity ladder

A capability's autonomy is bounded by what the mesh can currently *prove* about its premise. The
ladder is normative:

| Tier | Provenance maturity | What it enables |
|---|---|---|
| 1 | **taint bit** propagates | correct gating *if you trust the sender*: low-blast autonomy against simulated or sandboxed effects only |
| 2 | full **lineage** in the chain (real origin sources carried, §6.5) | the receiver derives its own taint; a human sees the true origin: real-world action *with human approval* |
| 3 | **signed** lineage, receiver verification ON (§6.6–§6.7), every signing key recorded in agent-separated custody (§7.1) | a chain can be forged only by a holder of its signing key, and the receiver has on record that the signer's agent is not one: autonomous cross-mesh high-blast action becomes eligible |

**The join to GAL.** A capability's autonomy rung MUST NOT exceed the mesh's current provenance
maturity. The companion GAL specification (Grant & Autonomy Lifecycle, drafted in parallel as
this document's sibling) defines the grant levels, the evidence-licensed promotion ceremony, and
the deterministic demotion triggers; its promotion predicate MUST enforce this ceiling (the
acting rungs require the signed-lineage tier for high-blast external action) rather than leaving
it to reviewer judgment (see GAL-SPEC.md, the promotion-predicate and rung-ceiling sections).
This is the correct expression of what the trust layer can currently
prove. Signing moves the rung, but only the provenance ceiling, and that ceiling is not the only
bound. What the trust layer proves is *premise*, not *response time*: no provenance maturity
shortens the window between a high-blast act and its detection. So tier 3 makes high-blast
autonomous action *eligible* (it lifts the provenance ceiling) while GAL's independent
requirement that every high-blast promotion be human-ratified (GAL-SPEC.md §6.4.4) still holds:
after-the-fact controls bound a compromise's *duration*, not a single act's blast, and a control's
response time must be matched to the blast rate of the authority it bounds. Signing raises the
ceiling; it does not place a human inside a high-blast window, and detection MUST NOT substitute
for one.

### 7.1 Tier 3 and signer key custody

A signature excludes whoever does not hold the key. Tier 3 depends on the signer's own agent
being among the excluded. §6.6 requires that (the agent cannot sign, PTC-11), but as a property
of the Producer, and a chain looks the same to a Receiver whether or not the Producer has it.
Where the signing key can be read by the signer's agent, as when a broker and its agent run as
one operating-system user, an agent under injection can produce a chain that verifies. The
signature then excludes every party except the one it is there to exclude, and verification
passes regardless. Tier 3 is therefore defined by a verified signature together with what the
receiver has on record about where the signing key sits.

- **Agent-separated custody.** A signer holds its key in agent-separated custody when the
  private key is stored, and every use of it is controlled, on the far side of a boundary the
  signer's agent cannot cross: the agent can neither read the key nor cause a signature to be
  made other than by the broker's own outbound stamping (§6.5). The boundary is one the signer's
  platform enforces (a separate operating-system or workload identity, a key service whose
  signing permission only the broker holds, a hardware-held key with a use policy), never one
  the signer's own code enforces on itself.
- **The custody record.** For each verification key it enrols, a Receiver MUST record, in its
  own configuration and beside the key's scope (§6.7), whether the signer holds the key in
  agent-separated custody, and the **evidence class** of that record. A key with no custody
  record MUST be refused at configuration load.
- **Evidence classes.** The vocabulary is closed, with two values.
  - `declared`: the receiver's operator recorded the custody when the key was enrolled, from
    what was established with the peer's operator out of band. The Receiver has verified
    nothing. A declared record turns an assumption about the peer into a written one that a
    gate can read and an audit can question. An implementation MUST NOT record, report or
    describe a declared record as verification of the peer.
  - `attested`: reserved for custody evidence a receiver can check (§1.3). This version defines
    no procedure that produces it, and a Receiver MUST refuse an `attested` record at
    configuration load.
- **What reaches tier 3.** A chain is at tier 3 only when verification ran on its envelope and
  passed (§6.7), and every signature on it was made by a key recorded in agent-separated
  custody. A verified chain signed by a key not so recorded is at tier 2. It remains verified,
  and observer attribution (§6.11) is unchanged; it MUST NOT be counted as tier 3.
- **A deployment's tier is the lowest tier among the chains it accepts.** A deployment is at
  tier 3 only when verification is enabled and every verification key it enrols is recorded in
  agent-separated custody. Enrolling one key recorded otherwise places the deployment at tier
  2, because a grant's rung is held per capability, not per envelope, and nothing downstream of
  the airlock separates what that key's traffic influenced from the rest.
- **The class travels with the tier.** Whatever consumes tier 3 inherits the evidence class
  under it. A statement that a deployment is at tier 3 MUST name the class, and where the
  records under it differ, the weakest. In this version that is always `declared`.

**What tier 3 claims under `declared`.** It claims that forging a chain takes the signing key,
and that the receiver has written down, per key, that the signer's agent does not have it. It
does not claim the receiver has checked. It also says nothing about a signer whose broker is
itself compromised, which holds the key by design and is outside what any custody arrangement
addresses (§9).

## 8. Conformance

### 8.1 Conformance roles

| Role | Description |
|---|---|
| **PTC Producer** | A sending broker: constructs, stamps, and (when signing) signs outbound envelopes. |
| **PTC Receiver** | A receiving airlock + gate: validates, verifies, trust-maps, screens, stamps, and ingests inbound envelopes; enforces the gate contract on the resulting turns. |
| **PTC Tool Host** | A broker hosting discovered tools (MCP): enforces the discovery-as-untrusted-input posture. |
| **PTC Observer** | An off-path correlator over the receiver's recorded exhaust. |

An implementation claims conformance per role. A **PTC Relay** is a Producer and Receiver
composed: it MUST satisfy both roles' clauses, appending and re-signing per §6.5–§6.6, never
editing upstream hops.

### 8.2 Conformance clauses

One table; the **Role** column names the conformance role each clause binds ("All" = every role).

| # | Role | Clause | Origin |
|---|---|---|---|
| PTC-1 | All | Taint is derived: no writable taint field exists on the envelope; envelope taint is any-untrusted over the chain. | SCHEMAS C3 |
| PTC-2 | All | The provenance chain is append-only: hops append via a frozen additive path; no operation edits or removes prior entries (non-removal: not yet implemented, #17). | SCHEMAS C3 |
| PTC-3 | All | The raw original is never embedded: `payload` is parsed and bounded; the raw original sits behind `payload_ref` + `payload_digest` and is fetched only via a brokered, taint-tracked read (the indirection and its brokered read: not yet implemented, #18). | SCHEMAS C2 |
| PTC-4 | All | Transport-neutral: no field closes over a wire technology; `channel_type` and `source` are open, namespaced vocabularies. | SCHEMAS C8 |
| PTC-5 | All | `expiry` is a hard TTL, checked deterministically before any budget-spending gate. | SCHEMAS C6 |
| PTC-6 | All | `event_id` joins the audit surfaces of both zones for end-to-end replay (the sending-zone half of the join: not yet implemented, #19). | SCHEMAS C7 |
| PTC-7 | Producer | Outbound provenance is broker-stamped from the sending turn's taint state; the agent supplies no provenance and no label (broker stamping on the shipped call path: not yet implemented, #15); a tainted turn yields a non-strippable `untrusted` entry; relaying appends and never edits. | PUBLISH P3; TRUST-MAPPING (sender side) |
| PTC-8 | Producer | The agent authors intent, not identity: it supplies `event_id`, target `principal`, `payload`; the broker sets `audience`, `sender`, `provenance`, `ts`, `expiry` (broker authorship of these fields: not yet implemented, #15); `sender_class` is absent on the wire. | PUBLISH P4 |
| PTC-9 | Producer | Lineage, not a collapsed bit: a fresh origination carries the turn's actual ingested taint sources as origin hops (origin-hop derivation: not yet implemented, #15); a relay does not duplicate sources already in the chain. | PUBLISH P8 |
| PTC-10 | Producer | A cross-zone publish is an external write; on a tainted turn it escalates through the standing no-write-up cut; no publish-specific rule exists. | PUBLISH P2; TAINT §5 |
| PTC-11 | Producer | The agent holds no transport credential and no signing key; both are broker-held and broker-resolved (broker signing on the shipped call path: not yet implemented, #15). | PUBLISH P1; SIGNING S3 |
| PTC-12 | Producer | The signature is Ed25519 over the DSSE PAE of a canonical in-toto-style statement binding the whole envelope: the inline payload hash as a subject, the raw-original digest and reference as a second subject when the envelope carries them, the ordered hops, the signer's own `key_id`/`zone`, and every other envelope field except `sender_class` and `chain_signatures`, with `sender.channel_identity` in canonical form. The signed set is defined by exclusion and is not narrowable. A Producer refuses to sign an envelope the statement cannot name exactly. The DSSE `payloadType` is `application/vnd.in-toto+json` and the statement `_type` is `https://in-toto.io/Statement/v1`; a `predicateType` is an absolute, explicitly versioned URI identifying exactly one statement kind. | SIGNING S1, S1b |
| PTC-13 | Producer | Signing is per-envelope with full-chain cover only (`covers = len(provenance)`; a prefix signature is invalid); attribution is non-malleable (a signature's `zone` must equal the `zone` of the last provenance entry); inbound signatures are not carried across a relay; a signing key is not shared between zones. | SIGNING S2 |
| PTC-14 | Receiver | Exactly one stamped envelope per deduplicated message (exactly-once under concurrency: not yet implemented, #20), dedupe key `(sender.channel_identity, event_id)`; replays are silent no-ops. | SCHEMAS C1 |
| PTC-15 | Receiver | A `principal` the receiver does not serve is a drop, never a re-route. | SCHEMAS (principal); TRUST-MAPPING |
| PTC-16 | Receiver | `sender_class` is receiver-derived only: absent on the wire, set solely by the receiver's trust-map gate, unconditionally overwriting any inbound value. | SCHEMAS C4 |
| PTC-17 | Receiver | Trust mapping is a deterministic, configuration-derived, exact-match lookup; unmapped identities drop: silently toward the sender, always recorded, identity as digest never raw. | TRUST-MAPPING |
| PTC-18 | Receiver | The one-way rule: no gate output lowers chain-derived taint; authenticity never cleans; an `external` hop always taints. | TRUST-MAPPING (one-way rule) |
| PTC-19 | Receiver | Sender-asserted chain labels are a floor, never a grant: a source taints the receiving turn if its chain label is `untrusted` OR the receiver's own input trust map does not trust it. | TRUST-MAPPING (consequence 2) |
| PTC-20 | Receiver | Every provenance source is fed into the broker-held receiving turn before the worker acts. | SCHEMAS C5 |
| PTC-21 | Receiver | Enabled verification fails closed: an envelope with no signature, an unknown signer, a `covers` other than the full chain, a `payload_type` or `sig` outside the one form of §5.5, or an invalid signature is rejected and loudly quarantined before any budget-spending gate; every present signature must pass every check; misconfigured verification fails closed loudly. | SIGNING S4, S5 |
| PTC-22 | Receiver | Verification MAY be off; with it off, unsigned traffic passes and the deployment's maturity tier (§7) and observer attribution strength degrade accordingly: stated consequences, never silent ones (deriving the tier from whether verification is configured and from the custody records of §7.1: not yet implemented, #21). | SIGNING S5 |
| PTC-23 | Receiver | Evidence-of-check is stamped only by the gate that performed the check, only when it ran; downstream consumers never fabricate, infer, or override it. | SIGNING (gate evidence); WATCHDOG W8 |
| PTC-24 | Receiver (gate) | The decision function is pure, deterministic, and model-free; facts are pre-resolved into a closed fact set with no model-derived field; evaluation is first-match over an ordered rule set; the matched rule is recorded (recording the matched rule: not yet implemented, #22); unmatched writes default-deny. | deterministic-gate |
| PTC-25 | Receiver (gate) | The verb alphabet is exactly `allow` / `deny` / `transform` / `require_approval` / `abstain`; `transform` produces a substituted operation plus clamped arguments (argument clamping: not yet implemented, #16); model-authored arguments never change the verb. | deterministic-gate |
| PTC-26 | Receiver (gate) | Turn taint is source-based and non-strippable; turn identity is broker-owned; taint clears only by broker/harness-owned rollover; there is no agent-reachable clearing path. | TAINT §1, §3, §4 |
| PTC-27 | Receiver (gate) | No-write-up is floor: a tainted turn's external write escalates through the polarity seam (`require_approval` where a human is reachable, `deny` otherwise), grant-independently, with no silent `transform` downgrade; the cut is never tunable off. | TAINT §5, §1.1 |
| PTC-28 | Receiver (gate) | Endorsement (declassification) is configuration-declared, per-source, audited (the audit event on application: not yet implemented, #23), and raise-only; never model- or agent-declared, never per-content, never a lowering operator. | TAINT §1.1, §2 |
| PTC-29 | Receiver (gate) | A content screen refuses or passes, never blesses: a pass is contentless and changes nothing; a refusal's reason is a closed-vocabulary machine code; an escaped screen exception fails closed as `screen_error`; ordering keeps expired/unmapped/replayed traffic away from the screen (a replayed copy under concurrent delivery: not yet implemented, #20). | SCREENING; TRUST-MAPPING (consequence 3) |
| PTC-30 | Tool Host | Discovery is untrusted input: a tool is callable only via two-key admission: an image-baked declaration of the `(server_id, tool_name)` namespace + host-assigned effect classification, AND an integrity-protected activation at a matching admitted hash; the mutable layer can select and tighten, never mint; missing either key is uncallable. | MCP-HOST doctrine, M3, M4, M13 |
| PTC-31 | Tool Host | The admitted hash covers every advertised definition field as canonical-JSON `sha256`: the core `(server_id, tool_name, input_schema, description)` plus each metadata field the server advertised; a non-advertised field is excluded from the preimage, never serialized as null, so a metadata-less definition hashes identically to the core-only basis. The set is not narrowable. | MCP-HOST M1 |
| PTC-32 | Tool Host | A description-only change is drift. | MCP-HOST M2 |
| PTC-33 | Tool Host | Drift fails closed: the tool drops to uncallable, surfaces loudly exactly once, and is never auto-re-admitted; only a fresh human ceremony resolves a quarantine (latching the quarantine: not yet implemented, #24). | MCP-HOST M5, M6 |
| PTC-34 | Tool Host | Admission and re-vetting require two verified credentials that a single operator cannot satisfy both of (maker ≠ checker); identity equality is a floor, and where an identity embeds unstable components the comparison must also reject pairs sharing a ceremony role. Both append an append-only, signed admission record binding admitter + tool + hash. | MCP-HOST M7, M8 |
| PTC-35 | Tool Host | A discovered tool's response taints the turn by default under source id `connector:{server_id}.{tool_name}`; per-tool endorsement is honored only for structured-output tools and refused at configuration load otherwise; an implementation that provides a response screen ships it OFF, and the screen can only refuse or pass. | MCP-HOST M9, M10, M11, M12 |
| PTC-36 | Tool Host | Every reconnect is a new discovery: full re-evaluation, fresh integrity reads, fresh credentials, no carried verdicts; death of a server or its transport is a typed, attributed failure, never an indistinct timeout. | MCP-HOST M17, M18 |
| PTC-37 | Tool Host | Configuration that names code to run or an endpoint to reach is honored only from the image-baked declaration, structurally out of reach of any runtime-mutable store; cleartext transports to non-loopback hosts are refused at load. | MCP-HOST M14, M21 |
| PTC-38 | Observer | The observer never gates: it is pure over its typed inputs, holds no on-path seam, and cannot refuse, delay, or throttle a request. | WATCHDOG W1 |
| PTC-39 | Observer | Attribution at recorded strength: signer attribution only for signature-bound AND dedupe-capped reasons; forgery-class reasons attribute to transport only, never the claimed signer; pre-dedupe reasons cap at transport-token regardless of verification. | WATCHDOG W2 |
| PTC-40 | Observer | Unattributable events are terminal: reported, never throttle-counted, never downgraded into a weaker attribution. | WATCHDOG W3 |
| PTC-41 | Observer | Observer outputs are PII-safe (digests, codes, counts, opaque refs; never raw identity or content); the remediation vocabulary is closed, construction-validated, deterministically mapped from attribution basis, and contains no shed action. | WATCHDOG W4, W5, W6 |
| PTC-42 | Receiver (gate) | Escalation is not treated as resolution: a deployment whose safe-default polarity is positive-safe-action operates a liveness contract over a declared expected output and deadline. The monitor is deterministic: it observes that the output did not occur, never why; no model sits on the path, and the sign of life is an enforcement-point-written audit or ledger artifact, never an agent self-report (wiring the liveness predicate to such an artifact: not yet implemented, #26). | friction-doctrine (availability) |
| PTC-43 | Receiver (gate) | Approval-queue amplification is de-amplified, never shed: identical pending approvals SHOULD coalesce (a re-submission can coalesce onto a hold that has already expired: not yet implemented, #41); a depth threshold may alarm but the deployment still holds every intent; no default path denies, drops, or auto-resolves queued approvals under load. | friction-doctrine (availability) |
| PTC-44 | Receiver | A refusal at the airlock discloses no gate, entry, path, signature or evaluated chain state beyond what the receiver publishes. Toward a sender the airlock authenticated and mapped before it evaluated, a refusal of authority is distinguishable as permanent or as transient and as nothing more (the classification: not yet implemented, #174); toward every other sender every delivered envelope SHOULD receive the same response; a screen refusal is indistinguishable from acceptance toward every sender. | TRUST-MAPPING; threat model |
| PTC-45 | Receiver | An envelope is addressed to one receiver: an `audience` other than the receiver's own zone id, compared exactly, is an `audience_mismatch` drop, checked before chain verification and before deduplication and whether or not verification is enabled; the drop names no signer. | SIGNING S9; SCHEMAS C9 |
| PTC-46 | Receiver | A verification key is scoped in the receiver's own configuration to one zone id and a set of sender identities; a signature from a known key outside its scope is rejected as invalid; an unscoped or twice-enrolled key is refused at configuration load. | SIGNING S8 |
| PTC-47 | All | The envelope has one wire form (ASCII JSON that parses back to an equal envelope) and a bounded size (196,608 bytes as sent, 262,144 as forwarded, at most 8 signatures); an envelope outside either is refused at schema validation, before any gate records it as seen. | SCHEMAS (wire form) |
| PTC-48 | Receiver | A verification key carries a custody record in the receiver's own configuration: whether the signer holds the private key in agent-separated custody (stored and used only across a boundary the signer's agent cannot cross), and the evidence class of that record, from the closed vocabulary `declared` / `attested`. A key enrolled without a custody record, and a record of class `attested`, are refused at configuration load. A verified chain is at tier 3 (§7) only when every signature on it was made by a key recorded in agent-separated custody, and the receiver's evidence of that names the class; a verified chain signed by any other key is at tier 2. A `declared` record is never recorded, reported or described as verification of the peer. | SIGNING S10 |
| PTC-49 | Receiver (gate) | Where the evaluation instant is compared with a time asserted under another party's clock, a declared skew bound is applied in the direction that confers less authority: an expiry or an authority-lowering record is effective up to the bound earlier, an authority-raising record or a not-before time up to the bound later (not yet implemented, #175). | §6.13; threat model |
| PTC-50 | Receiver (gate) | For each fetched input a decision depended on, the decision point records the instant as of which it established that input, and no input is used past a maximum age the deployment declares for it; where a source states the time of its answer, the as-of instant is the earlier of that time and the time of the fetch; the decision point records the earliest instant at which any input a decision depended on passes its maximum age, and a call held on the decision or carried on from it is evaluated again past that instant (not yet implemented, #176). | §6.13; threat model |
| PTC-51 | Receiver (gate) | Delegation does not launder: a context an agent causes to exist outside its zone, or to run later, begins with a turn at least as tainted as the turn that caused it (not yet implemented, #177). | TAINT; the subagent-identity doctrine |
| PTC-52 | Receiver (gate) | An implementation provides a control under which, on a tainted turn, a call whose destination the agent composed is treated as an external write and escalated; the destination argument is declared in reviewed configuration; a deployment MAY leave the control off (not yet implemented, #178). | TAINT §5, §6; threat model |
| PTC-53 | Receiver (gate) | A held approval MAY expire, and its expiry period does not shorten as the queue grows; an approval that expires without a human decision is recorded as expired, is distinguishable on the audit record from a human denial, and counts as no human decision in any evidence the grant lifecycle reads; while a queue-depth alarm is raised each such expiry is reported to the approver, by count at least (not yet implemented, #163). | friction-doctrine (availability) |

### 8.3 Origin-mapping coverage

| Origin clause set | Mapped by | Coverage notes |
|---|---|---|
| EventTrigger C1–C9 and wire form | PTC-1..6, 14, 16, 20, 45, 47 | full |
| PUBLISH P1–P8 | PTC-7..11 (P5 "no shared store" folded into §6.1 item 1; P6 "receiver re-gates everything" folded into PTC-17..20; P7 into PTC-4) | full |
| SIGNING S1–S10 | PTC-11..13, 21..23, 45, 46, 48 (S6 "signing ≠ correctness" carried in §9; S7 tiering is packaging doctrine, non-normative here) | full |
| TRUST-MAPPING (one-way rule + consequences) | PTC-15..19, 29 | full |
| SCREENING | PTC-29 (+ §3.6, §6.9) | full (verdict-sink observability valve is implementation-tier, unmapped) |
| TAINT §1–§6 | PTC-26..28 (+§6.3–§6.4); §6 read-side knobs (budgets, egress bounds) are implementation-tier knobs, unmapped | full for floor; knobs partial by design |
| deterministic-gate | PTC-24, 25 | full |
| friction-doctrine (availability / forced abstention) | PTC-42, 43 (+§6.12, §9.6) | full for the floor; the polarity→default derivation is per-deployment and deliberately unmapped |
| WATCHDOG W1–W8 | PTC-38..41, 23 (W7 meta-alarm is an operational standard for scheduled runners, out of scope here) | W1–W6, W8 full; W7 unmapped |
| MCP-HOST M1–M21 | PTC-30..37 | M1–M11, M13, M14, M17, M18, M21 mapped; M12 folded into PTC-35; M15/M16/M19/M20 (construction refusals, env composition, respawn policy, shutdown reaping) are implementation-lifecycle clauses, deliberately unmapped in this version |

Existing conformance suites keyed to origin clause IDs remain traceable through this table.

### 8.4 Implementation Conformance Statement

The clause table of §8.2 constitutes an **implementation conformance statement proforma** in the
sense of ISO/IEC 9646-7. The companion GAL specification defines the convention in full in its
§7.6; this section states it for a reader of PTC alone.

An implementation claiming conformance SHALL publish, for each role of §8.1 it claims, a support
answer against every clause bound to that role, drawn from `Y` (supported), `N` (not supported,
and the claim for that role fails), or `N-A` (not applicable; permitted **only** where a clause's
conditional predicate is unmet, never merely because a capability was not built).

**Every clause is mandatory within its role.** This specification defines no optional clauses.
Where a clause admits a deployment choice (PTC-22's allowance that verification MAY be off), the
permission sits inside a mandatory clause whose obligation (stating the consequence rather than
degrading silently) remains binding. A **PTC Relay** completes the statement for Producer and
Receiver both.

The reference implementation's statement is **derived from the §2 implementation-status markers**
rather than hand-written: unmarked answers `Y`, NOT YET IMPLEMENTED answers `N`. Derivation is what
keeps the statement, the markers and the clause list from drifting apart.

**Traceability is bidirectional.** Every clause SHALL trace downward to a conformance test or an
explicit recorded gap, and every conformance test SHALL trace upward to the clause id it exercises.
The upward direction is the one that detects a suite accumulating confidence about something this
specification never asked for. The objective is borrowed from DO-178C §5.5 and §6.5: **the
objective only**, not that standard's process, assurance levels, coverage criteria, or tool
qualification, and conformance here is not a claim about airborne-software certification.

## 9. Security considerations and open problems

1. **Declassification is the classic IFC hard problem.** Deciding *when* tainted data may
   influence a privileged action is where the real bugs live: over-restrictive lattices break
   usability; over-permissive ones reintroduce injection. PTC's answer is procedural: endorsement
   is configuration-declared, per-source, audited, raise-only (PTC-28), and it remains a knob a
   human can misconfigure.
2. **Propagation-through-transform is not solved by signing.** A signature proves *who asserted a
   hop*, never that taint was *correctly propagated* through a model transform. Lineage carriage
   (PTC-9) bounds the exposure, but whether a model faithfully carried taint through
   summarization or synthesis is open, for PTC and for every signing standard.
3. **The mesh is only as trustworthy as its nodes' brokers.** Cross-zone taint is *correct* only
   if the sending zone also runs a PTC-enforcing broker; against an arbitrary agent, the
   receiver's own trust map and re-derivation (PTC-19) are the floor. Like TLS, PTC's value is
   bilateral and compounds with adoption.
4. **Verification OFF is a stated degradation, not a silent one.** With verification off, a
   transport-credentialed sender can fragment observer attribution by claiming a fresh sender
   identity per attempt: an under-counting gap, never a mis-attribution one; enabling
   verification is the shipped mitigation. Relatedly, attribution granularity is the **sending
   broker**, not an upstream agent behind it: per-envelope signing (§1.3).
5. **The screen is defense in depth, never a dependency.** Every taint property holds with the
   screen absent; an injection that fools the screen gains only what it already had (PTC-29).
6. **Availability is a first-class property, and this specification's own integrity controls are
   the lever against it.** Every floor here answers an input it cannot clear by escalating or
   refusing, and it does so whatever put the input there: an attacker's injection, a revocable
   instruction the agent absorbed as a standing rule, a source that went stale. Forced abstention
   is the failure regardless of cause, and it needs no attacker.
   For a deployment whose polarity is positive-safe-action, that answer *is* the harm (§6.12), so
   an implementation scored only on integrity outcomes will record a successful denial of service
   as a successful defense. PTC-42 exists because the obvious alternative, a model judging whether
   a given abstention was legitimate, puts a probabilistic classifier back on the safety path. The
   residual risk is real and stated: a deterministic monitor *detects* silence, it does not prevent
   it, and the exposure window is bounded by the deadline a deployment chooses, not by this
   specification. A monitor a deployment has not switched on bounds nothing.
7. **An instruction to the model is an instruction to the component under attack.** Guidance that
   phrases a control as a duty of the agent ("the agent should weigh the trust level of its
   sources before acting") misassigns the decision: a persuasive injection reads as trustworthy
   exactly when it is most dangerous. PTC's answer is structural rather than exhortative (PTC-24,
   PTC-26, PTC-28): trust is a deterministic function of source identity, resolved before the
   decision, in a lookup the agent cannot influence. The general rule, which also governs PTC-29
   and PTC-42, is that no probabilistic classifier may occupy the safety path: a validator that
   can be argued into approving is worse than none, because it manufactures confidence.
8. **A zone id is a name the deployment chooses, and the audience check is only as strong as its
   distinctness.** Binding an envelope to its receiver (PTC-45) closes delivery of a signed
   envelope to a receiver it was not addressed to. It closes that only between receivers whose
   zone ids differ. Two deployments configured with the same id, for instance the development
   and production floors of one agent left on a default, are one audience as far as any
   receiver can tell, and no receiver can detect the collision from its own configuration.
   The same holds for key scope (PTC-46): it limits what an enrolled key may speak for, and it
   is exactly as accurate as the receiver's own enrolment record. Both are configuration a
   human can get wrong, and a conformance suite run against one deployment cannot find the
   error.
9. **Signing a named list of fields fails quietly.** Earlier drafts of this specification, and
   the reference implementation with them, signed an enumerated set of envelope fields. The
   set was found short three times: the sender identity, then the raw-original reference, then
   the creation time and sender evidence. Each omission let a signed envelope be altered and
   still verify, and none was visible in a passing test suite, because the tests enumerated the
   same fields the statement did. The rule of §6.6 (sign the envelope minus a named list) is the
   correction, and the general lesson is that a signature's coverage should be stated as what
   it leaves out.
10. **Tier 3 rests on a record the receiver cannot check.** In this version the custody record
    of §7.1 is `declared`: an operator's statement about a peer, written into the receiver's
    configuration. A peer that misdescribes its arrangement, or one that changes it after
    enrolment, is recorded as agent-separated all the same, and nothing a receiver computes
    detects either. What the record buys is narrower and still worth having: the assumption is
    explicit, it is per key, a gate reads it, and a peer known to keep its key within its
    agent's reach is held to tier 2 instead of passing as tier 3 on the strength of a valid
    signature. Checkable evidence is future work (§1.3). Separately, custody says nothing about
    a compromised broker. The broker holds the key by design, so a signer whose broker is
    subverted signs whatever the attacker wants; item 3 applies, and no tier addresses it.

## 10. Extensibility

**Open by design** (vendors MAY extend): `sender.channel_type` and provenance `source` schemes
(open, namespaced vocabularies, PTC-4); screen refuse codes and observer remediation codes beyond
the reserved sets, extended in the implementation's declared configuration or source, **never
minted at runtime**; carriage bindings, signature key substrates, transparency-log anchoring.

**Closed** (extension is a versioned change to this specification, not a vendor option):

- **The sender-class set.** Exactly `owner` / `peer-agent` / `external`. Unmapped is an absence,
  not a class; adding classes changes the one-way rule's label function and is a spec change.
- **The verb alphabet.** Exactly the five verbs of §3.3.
- **The signed envelope set** (PTC-12). The two fields outside the signature are named in §6.6;
  adding a third is a versioned change, and so is any field the statement binds in a form other
  than its wire value.
- **The signed definition set** for tool admission (PTC-31). Narrowing is forbidden. Additive
  growth of the *definition schema* is deliberately **not** a spec change: because the basis is
  every advertised field with non-advertised fields excluded from the preimage, a newly-specified
  field moves no hash until a server advertises it. What is a versioned change: altering the
  canonicalization, or excluding an advertised field from the basis.

**MUST NOT be extended away** (violating any of these forfeits conformance in all roles):

- Non-strippability of taint and the append-only chain (PTC-1, PTC-2, PTC-26).
- The one-way rule and label-floor semantics (PTC-18, PTC-19).
- The no-write-up floor (PTC-27).
- The model-free gate (PTC-24, PTC-25) and refuse-or-pass-never-bless (PTC-29).
- Broker-only stamping and signing (PTC-7, PTC-11).

## 11. Version history

| Version | Date | Notes |
|---|---|---|
| 0.1.0-draft | 2026-07-24 | First consolidated normative draft, unifying the reference implementation's contract-tier documents (envelope, trust-mapping, signing, publish, screening, watchdog, taint floor, deterministic gate, MCP host posture) for Linux Foundation agent-standards discussion. |
| 0.2.0-draft | 2026-07-25 | Adds §6.12 (availability: forced abstention and approval-queue amplification) with PTC-42/43, promoting material previously confined to reference-implementation doctrine; states the gate-language expressiveness floor (value-producing verb, quantification over provenance) in §6.8 and §1.2 with the differential-test record; states the message-layer-versus-mutual-TLS choice in §1.2; adds §9.6–§9.7. Corrects PTC-31 and §6.10: the admitted definition hash covers every advertised field with non-advertised fields excluded from the preimage, not a fixed four, aligning the spec with the shipped contract (MCP-HOST M1). |
| 0.2.1-draft | 2026-07-29 | Resolves the signing-identifier question left open in §6.6, splitting it: the DSSE `payloadType` and statement `_type` are PINNED to in-toto's own registered identifiers (adopted, not minted), while `predicateType` is pinned by **shape** (absolute, explicitly versioned, one URI per statement kind), with the namespace left for the adopting standards body, since a vendor domain in a normative identifier makes conformance depend on one organization's DNS. Also states the document's discussion-only IPR posture and replaces version-relative scope language with "this version" / "this specification". |
| 0.2.2-draft | 2026-08-06 | Resolved a contradiction in which PTC-25, §3.3's preamble and §6.8 required `transform` to produce an operation **plus** clamped arguments while §3.3's own verb table stated an "and/or" form, by softening all four to "and/or". Superseded within the day by 0.2.3-draft, which resolves the same contradiction in the other direction; recorded rather than removed, because the LF received 0.2.1-draft and the intervening revision is part of the record. |
| 0.2.3-draft | 2026-08-06 | Resolves the `transform` contradiction 0.2.2-draft resolved the wrong way. The requirement is the **plus** form in all four places, including §3.3's verb table, and the argument-clamping half now carries the §3-style implementation-status marker (tracking #358) in each place a reader meets it. The earlier softening treated the specification as a description of the reference implementation; it is a description of the design, and the honest way to state a settled requirement the code has not reached is to mark it, not to weaken it. This also restores §6.8's polarity-seam argument, which turns on `transform` emitting *arguments* specifically and was materially weakened by the "and/or" form. Adopting the marker here makes PTC's convention identical to GAL's, where two clauses have carried it since 0.2.1-draft. |
| 0.2.4-draft | 2026-09-19 | States why §6.6 binds `principal` into the signed statement: a receiver without the sender's log cannot recover the on-behalf-of principal from the chain (RFC 8693's `sub`/`act` distinction). Prompted by an implementer finding against the IETF WIMSE cross-org delegation draft. Rationale only; no clause, schema, or wire change. |
| 0.2.5-draft | 2026-09-20 | Rewrites the Reference implementation entry to name the public implementation, [ptc-gal-reference](https://github.com/wjatx/ptc-gal-reference), which was not previously named anywhere in this document: the entry pointed at a private repository a reader cannot open. Replaces "running in production since July 2026" with what a reader can actually check (conformance suites runnable from a checkout with no cloud account) and relabels the forged-signer drill as recorded history rather than reproducible evidence. Non-normative entry; no clause, schema, or wire change. |

| 0.2.6-draft | 2026-09-21 | Repoints §1.3's cross-relay per-signer attribution entry at [ptc-gal-reference#13](https://github.com/wjatx/ptc-gal-reference/issues/13). It previously cited "reference-implementation issue #180", a number that resolves in a PRIVATE repository and in no public one, so a reader of this specification could not reach the tracking issue it was directed to. The entry's substance is unchanged: cross-relay attribution stays a future extension requiring nested per-hop attestations, and the linked issue additionally records the proof-of-possession prerequisite, since an attestation portable across a re-packaging relay is liftable unless the presenter can prove control of the key it binds. Non-normative; no clause, schema, or wire change. |

| 0.2.7-draft | 2026-09-21 | Repoints every implementation-status marker at an issue in the PUBLIC reference implementation. The markers previously cited the private working tracker, so a reader of a marked clause could not reach the work it pointed to. Fourteen PTC markers were renumbered against newly filed [ptc-gal-reference](https://github.com/wjatx/ptc-gal-reference) issues; historical changelog entries keep their original numbers, because they record what was true when written. No clause text, schema, or wire change. |
| 0.2.8-draft | 2026-09-29 | States what a refused sender learns. §6.1 and new clause **PTC-44** generalize PTC-17's silence-toward-the-sender from unmapped identities to every airlock refusal: the response discloses no gate, entry, path, signature or evaluated chain state beyond what the receiver publishes, and toward senders the airlock cannot authenticate before evaluation, every delivered envelope SHOULD receive the same response. This records as a deliberate choice the reference airlock's uniform success response, and its divergence from the permanent-refusal signal `draft-jackson-wimse-evaluation-02` §5 recommends toward an authenticated caller. Satisfied by the reference implementation as it stands; no schema or wire change. |
| 0.3.0-draft | 2026-10-02 | **Wire change.** Corrects four defects in the signing and receiving contract, each found by attacking the reference implementation and each a gap in this specification as well as in the code. (1) §6.6 and PTC-12 enumerated the signed fields, and the list was short: the statement now binds the whole envelope except two named fields, with the raw-original digest and reference bound as a second subject. (2) §5.5 and PTC-13 allowed a signature over a chain prefix, which let a party with no key truncate a chain; every signature now covers the full chain. (3) §6.7 let any enrolled key sign for any zone and sender; new clause **PTC-46** scopes each verification key, in the receiver's configuration, to one zone and a set of sender identities. (4) Nothing named the receiver an envelope was for; new required field `audience`, new drop reason `audience_mismatch` and new clause **PTC-45** bind an envelope to one receiver's zone id, checked before verification. Also adds §5.7 and **PTC-47** (one wire form, bounded size, refused before acceptance), pins the spelling of `sig`, `payload_type` and `payload_digest`, requires a receiver to discard an inbound `sender_class` before any gate reads it, requires distinct zone ids and unshared signing keys, and adds §9.8 and §9.9. An envelope signed under 0.2.x does not verify under this version. Editorial, with no change of requirement: the body text no longer uses dashes as punctuation, and the inline marker's short form changes with that, separating the issue number with a comma (`(not yet implemented, #NNN)`) where it used a dash; §2 now says what `#NNN` stands for; one sentence in §7 is reworded. |
| 0.3.1-draft | 2026-10-02 | Removes the NOT YET IMPLEMENTED marker from **PTC-35** and states its condition explicitly. The clause constrains a response screen where an implementation provides one, and requires none: §1 puts screening itself out of scope, and the airlock's screen in §6.9 is likewise optional. The marker reported an optional component as an unbuilt requirement. Building a response screen in the reference implementation remains tracked, without a marker, at [ptc-gal-reference#25](https://github.com/wjatx/ptc-gal-reference/issues/25). No requirement or wire change. |
| 0.4.0-draft | 2026-10-03 | Redefines tier 3 of the provenance-maturity ladder (§7), which said "the receiver cannot be lied to" and was defined only by what the chain carries. A signature excludes whoever lacks the key, and where a signer's key is within reach of its own agent the agent can forge a chain that verifies, so a receiver verifying it can be lied to by the party the signature exists to exclude. New §7.1 and clause **PTC-48**: a receiver records, per verification key, whether the signer holds the key in agent-separated custody and the evidence class of that record (`declared` or `attested`); a verified chain reaches tier 3 only when every signing key is so recorded, a deployment's tier is the lowest among the chains it accepts, and the class travels with the tier. `declared` is a recorded assumption and verifies nothing; `attested` is reserved and refused at configuration load, and §1.3 names key-custody attestation at enrolment and remote attestation of the signer's deployment as the future extensions that would produce it. §6.6 and §6.7 point at the new section, PTC-22's marker is widened to cover deriving the tier from the custody records, and §9 gains item 10. A receiver's verification-key configuration gains a required record; no wire change. |
| 0.5.0-draft | 2026-10-07 | Five clauses are added and one is rewritten, each stated with the attack it prevents and each marked NOT YET IMPLEMENTED. (1) §6.1 and **PTC-44** are rewritten. Earlier drafts made every airlock refusal silent and said this "departs deliberately" from the refusal signal that verifier-side guidance gives an authenticated caller. The two are reconciled by who is asking and what was refused: a sender the airlock authenticated and mapped learns whether a refusal of authority is permanent or transient and nothing more; every other sender gets a uniform response; a screen refusal reads as acceptance to everyone. A stranger gets no map of the trust configuration, and a compromised peer gets no feedback on content. (2) New §6.13 with **PTC-49** and **PTC-50**: clock skew is applied toward less authority, never symmetrically, and every fetched input a decision depended on has a recorded as-of instant and a declared maximum age. (3) **PTC-51** (§6.3): delegation does not launder. A context an agent causes to exist outside its zone, or to run later, starts at least as tainted as the turn that caused it. (4) **PTC-52** (§6.4): on a tainted turn a call whose destination the agent composed is escalated as an external write, as a control a deployment may leave off. §6.4 says what this does not stop, request forgery inside a tool server, and that containment closes it. (5) **PTC-53** (§6.12): an approval that expires undecided is recorded as expired, is told apart from a human denial, counts as no human decision, and is reported to the approver while a flood alarm is raised. Also: **PTC-43** says identical approvals SHOULD coalesce, as §6.12 always did, and is marked for a re-submission that can join an expired hold; **PTC-11** and **PTC-29** gain the markers their siblings already carried; and §9 item 6 no longer frames forced abstention as a consequence of injection only, since it needs no attacker. No wire change. |
| 0.5.1-draft | 2026-10-07 | §6.13 and **PTC-50** gain the decision-level form of the input-age rule: a decision is good no longer than its shortest-lived input, the decision point records the earliest instant at which any input it depended on passes its maximum age, and a call held on the decision or carried on from it is evaluated again past that instant. Without it a record can show a permit whose every input was in date when read while the decision as a whole has outlived the window it was computed from. Raised in list review of the companion verifier draft. Marked NOT YET IMPLEMENTED with the rest of the clause. |

## 12. References

**Adopted standards and primitives**

- MCP (Model Context Protocol, agent↔tool) and A2A (Agent2Agent protocol, Linux Foundation,
  agent↔agent): the adopted connectivity layers; A2A's extension mechanism and MCP's `_meta`
  field are natural PTC carriages.
- DSSE (Dead Simple Signing Envelope) and the in-toto Attestation Framework: the signature
  envelope / PAE and the Statement/Predicate shape of the signed chain statement.
- Ed25519 (RFC 8032): the signature algorithm; Sigstore/Rekor: optional transparency-log
  anchoring.
- SPIFFE / IETF WIMSE: workload-identity substrate for signing keys (DID as fallback).
- Biba, K. J., *Integrity Considerations for Secure Computer Systems* (1977): the integrity
  lattice and no-write-up rule.
- RFC 2119 / RFC 8174: conformance keywords.
- OAuth 2.0 Token Exchange (RFC 8693): its `sub`/`act` split between the on-behalf-of subject
  and the acting party is the precedent for binding `principal` into the signed statement (§6.6).
- SD-JWT-VC: reserved for the future selective-disclosure content drill-down (§1.3).
- Remote ATtestation procedureS (RATS) Architecture (RFC 9334) and the Entity Attestation Token
  (RFC 9711): the architecture and claims format the reserved attestation extensions of §1.3
  would build on.

**Design inputs**

- *The Connectivity-Orthogonal Trust-Context Standard for AI Agents: Landscape Survey and
  Reference Architecture* (`docs/references/trust-context-landscape.md`): the survey this
  specification is built against; it finds no existing general standard at this seam (closest
  domain-specific precedent: Google AP2, payments-only) and identifies CaMeL (arXiv:2503.18813)
  as the in-process design cousin (PTC = that pattern + a wire format + non-repudiation + a
  production broker).
- GAL-SPEC.md: the companion Grant & Autonomy Lifecycle specification (the autonomy seam this
  specification's maturity ladder joins to).

**Reference implementation**

- **[ptc-gal-reference](https://github.com/wjatx/ptc-gal-reference)**: per-clause conformance
  suites for every origin document in §8.3, runnable from a checkout against these specifications
  with no cloud account and no credential. The differential corpus from the policy-language
  evaluation (§1.2) is retained as the seed of the gate conformance suite. The forged-signer
  attribution drill behind PTC-39 was run against deployed infrastructure; that run is recorded
  history rather than something a reader can reproduce.
