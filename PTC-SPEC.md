# PTC — Provenance & Trust Context, Specification

**Version:** 0.2.3-draft
**Date:** 2026-08-06
**Status:** Draft for Linux Foundation agent-standards discussion. Wire schemas may change before
1.0; see Open Problems and Future Extensions.
**Working group:** LF Edge + Agentic AI Foundation (AAIF)
**Author:** Wes Jackson (Red Hat)
**Copyright:** © 2026 Red Hat, Inc.
**License:** Community Specification License 1.0 · `SPDX-License-Identifier: Community-Spec-1.0`
See `LICENSE.md` for terms and `NOTICE.md` for attribution, acceptance, and patent exclusions.

## 1. Scope and non-goals

PTC (Provenance & Trust Context) specifies a **connectivity-orthogonal trust layer** for agentic
systems: a signed trust-context object — sender class, append-only provenance chain, and taint
labels on a Biba integrity lattice — that travels **with data across every agent and tool
boundary** and is consumed by a **deterministic, model-free gate** at each. MCP is how agents
reach tools; A2A is how agents reach agents; PTC is how trust travels across both.

### 1.1 In scope

- The **trust-context object** — envelope schema, provenance-chain entry, taint derivation, and
  signing envelope (§5) — and its **carriage rule**: the object rides orthogonally to transport
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
  authentication**: PTC deliberately places non-repudiation at the *message* layer — a DSSE-signed
  chain (§6.6) — rather than at the session layer (mutual TLS). The two are not equivalent and the
  choice is not a claim that transport authentication is unnecessary. A signature survives the
  relay hops that terminate a TLS session, which is the property a multi-zone mesh needs; mutual
  TLS in turn authenticates a peer that PTC's chain does not otherwise identify. A deployment
  SHOULD do both. This specification constrains only the former.
- **Policy language.** PTC requires the gate to be a pure, deterministic function with a specific
  verb alphabet and a stated expressiveness floor (§6.8); it does not mandate a policy language.
  Cedar and OPA/Rego were evaluated by differential test against the reference gate over its full
  reachable input space — all 73,728 reachable fact combinations — and declined with a published
  record: Cedar cannot checkably express first-match ordering, cannot express the value-producing
  `transform` verb, and **has no quantifier**, so existential reasoning over provenance records and
  over chain hops — this specification's own subject matter — is inexpressible in it. Rego
  reproduces the gate exactly but forfeits the purity and analyzability that motivated the question.
  The requirement is the behavioral contract, not the engine.
- **Model-screening internals.** PTC constrains what a content screen may *do* (refuse or pass,
  never bless — §6.9); the classifier inside a screen is out of scope.

### 1.3 Explicitly out of scope for this version (future extensions)

- **Cross-relay per-signer attribution.** In this version signing is **per-envelope**: a relay
  re-packages the envelope, so upstream signatures are not carried forward; upstream hops still
  ride as lineage. Attributing each intermediate signer *across* a relay requires nested per-hop
  attestations (tracked as reference-implementation issue #180).
- **Selective content disclosure.** The provenance chain is a contentless index; drill-down to
  upstream source content with least-disclosure semantics (SD-JWT-VC) is reserved.

## 2. Terminology and conformance language

The key words MUST, MUST NOT, REQUIRED, SHALL, SHALL NOT, SHOULD, SHOULD NOT, RECOMMENDED, MAY, and
OPTIONAL are to be interpreted as described in RFC 2119 as clarified by RFC 8174, when and only
when they appear in all capitals.

**Implementation status markers.** This specification is derived from a running reference
implementation rather than drafted ahead of one, and its normative text is overwhelmingly a
description of mechanism that exists and has been exercised. Where a clause is deliberately
normative *ahead* of that implementation — because the design question is settled and the
specification tier is the honest place to settle it while the code catches up — it is marked
wherever a reader can meet it, with a line of exactly this form:

> **Implementation status:** NORMATIVE, NOT YET IMPLEMENTED in the reference implementation (tracking: #NNN).

A table row carries the short inline form `(not yet implemented — #NNN)`, which means the same
thing. A marked clause is fully normative: an implementation claiming conformance MUST satisfy
it, and the marker is simultaneously a statement that the reference implementation does not yet.
**Absence of the marker means the clause is implemented in the reference implementation** — which
is what makes its presence worth anything — and no conformance claim, by this project or any
other, may cite a marked clause as a shipped control. A specification clause is a requirement,
never evidence that anything enforces it. The convention is shared with the companion GAL
specification, which defines it in its §3.

| Term | Definition |
|---|---|
| **principal** | The identity under which an agent acts and against which authority, budgets, and audit records are scoped. An envelope addresses exactly one target principal. |
| **turn** | The unit of taint accumulation: a bounded span of agent activity whose identity is minted and owned by the broker, never declared by the agent. Taint accumulates within a turn and is cleared only by broker- or harness-owned rollover. |
| **broker** | The deterministic enforcement point that mediates every tool call an agent makes. The agent holds no credentials; its only egress is the broker. The broker holds the signing key, stamps provenance, and writes audit records under an identity the agent cannot reach. |
| **gate** | A deterministic decision function at an agent or tool boundary, evaluating a call or message against pre-resolved facts. The **airlock** — the receiving-side gate pipeline that validates, verifies, trust-maps, deduplicates, screens, and stamps an inbound envelope — and the broker's policy decision point are both gates. |
| **sender class** | The receiver-derived classification of who addressed the agent: `owner`, `peer-agent`, or `external`. A floor on treatment, never a grant of content trust. |
| **provenance chain** | The ordered, append-only, non-strippable sequence of hop entries recording where content came from and what each zone judged about its own source. A replayable index, not a content log. |
| **taint** | The derived integrity property of content or of a turn, fixed by the trust of its sources through deterministic lookup — never by model judgment of content. Non-strippable within a turn. |
| **endorsement / declassification** | The explicit, audited, configuration-declared act of raising a source's integrity above the floor, exempting its reads from taint. The only mechanism by which lower-integrity data may influence a higher-integrity action. |
| **trust zone** | One broker's span of control: the agent(s), airlock, and stores governed by a single enforcing broker. Provenance hops and signatures are per-zone. |
| **envelope** | The typed trust-context object (§5) that any inbound or outbound signal becomes; it crosses zone boundaries. |

## 3. Enumerations (closed vocabularies)

Unless a table below states otherwise, these vocabularies are **closed**: an implementation MUST
NOT extend them (see §10).

### 3.1 Sender classes

Closed set of three. "Unmapped" is deliberately **not** a fourth class — it is the absence of a
mapping, and it short-circuits the pipeline as a drop.

| Class | Who | What verified authenticity raises | Receiver hop label |
|---|---|---|---|
| `owner` | the target principal's accountable human | the full command surface: commands, questions, approval responses | `trusted` |
| `peer-agent` | an authenticated peer agent the receiver has pre-declared | may wake the worker and deliver signals addressed to the mapped principal | `trusted` — the *hop* is authentic; chain-inherited taint persists untouched |
| `external` | a verified-but-external sender | the minimum: delivery to the mapped principal, subject to every downstream gate | `untrusted` — always taints |

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
| `transform` | execute a safer variant: a substituted operation plus clamped arguments (argument clamping: not yet implemented — #358) |
| `require_approval` | hold the call for out-of-band human ratification before execution |
| `abstain` | decline to decide; surface to a human without executing |

> **Implementation status:** NORMATIVE, NOT YET IMPLEMENTED in the reference implementation
> (tracking: #358). Applies to the **argument-clamping half of `transform` only**: the reference
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
| `malformed` | schema validation |
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
binding; no field of the object may close over a wire technology — the open vocabularies
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
| `sender` | SenderIdentity | yes | Who, at the transport layer, plus verification evidence (§5.2). |
| `payload` | object | yes | The parsed, normalized payload — size-bounded; never the raw original. |
| `payload_digest` | string | no | `"sha256:<hex>"` over the raw original artifact. REQUIRED when `payload_ref` is set. |
| `payload_ref` | string | no | Opaque pointer to the stored raw original; fetched only via a brokered (taint-tracked) read. |
| `provenance` | ProvenanceEntry[] | yes | Append-only chain, minimum length 1. Taint derives from this (§5.4). |
| `chain_signatures` | ChainSignature[] | yes | Per-hop signatures over the chain prefix each broker committed to (§5.5); `[]` on an unsigned chain. |
| `sender_class` | enum §3.1 | no | RECEIVER-owned. Absent (null) on the wire; set only by the receiver's trust-map gate, unconditionally overwriting any inbound value. |
| `ts` | string | yes | ISO-8601 UTC creation time. |
| `expiry` | string | yes | Hard TTL. Expired envelopes MUST drop before any budget-spending gate. |

### 5.2 SenderIdentity

| Field | Type | Required | Description |
|---|---|---|---|
| `channel_type` | string | yes | Open, namespaced vocabulary (`"telegram"`, `"peer-agent"`, …) — never a closed transport enum. |
| `channel_identity` | string | yes | Transport-level identity (chat id, From-domain, publisher id) — opaque; normalized by the receiving adapter. When the chain is signed, its canonicalized value is bound into the signature (§6.6). |
| `evidence` | string[] | yes | `"{scheme}:{result}"` checks stamped by the adapter that PERFORMED the verification (e.g. `"dkim:pass"`). A receiver MUST NOT treat sender-asserted evidence as its own. |

### 5.3 ProvenanceEntry (one hop)

| Field | Type | Required | Description |
|---|---|---|---|
| `zone` | string | yes | The agent/airlock zone that appended this entry. |
| `source` | string | yes | Namespaced source id, `{scheme}:{value}` (e.g. `"email:robinhood.com"`, `"channel:telegram"`, `"peer:email-agent"`, `"connector:{tool}.{op}"`). |
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
alongside the turn's ingested-source record (`session.ingestedSources`) — populated by the
broker, never model-supplied.

### 5.5 ChainSignature

| Field | Type | Required | Description |
|---|---|---|---|
| `key_id` | string | yes | The signing broker's workload-identity key id; a receiver resolves it to a public key to verify. |
| `zone` | string | yes | The zone that signed (attribution). MUST equal the top hop the signature covers. |
| `covers` | integer | yes | Prefix length this signature commits to: `provenance[:covers]`. |
| `payload_type` | string | yes | The DSSE `payloadType` bound into the signed pre-authentication encoding. |
| `sig` | string | yes | Base64 Ed25519 signature over the DSSE PAE of the in-toto-style statement (§6.6). |

### 5.6 Example

A signed trade-signal envelope as received, before the receiver's own hop is stamped:

```json
{
  "schema_version": 1,
  "event_id": "conf-8842-a1",
  "principal": "trading-agent",
  "sender": { "channel_type": "webhook", "channel_identity": "peer:email-agent",
              "evidence": ["hmac:pass"] },
  "payload": { "symbol": "RBH", "signal": "sell", "confidence": 0.72 },
  "payload_digest": "sha256:9f2c…",
  "payload_ref": "artifact://email-agent/raw/8842",
  "provenance": [
    { "zone": "email-agent", "source": "email:robinhood.com", "evidence": ["dkim:pass"],
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

The envelope is tainted (an `untrusted` entry exists) and non-strippably so — nothing the sender
did after reading the email removed the entry; `sender_class` is null because only the receiver
may set it.

## 6. Normative behavior

### 6.1 Ingestion stamping

On accepting an inbound envelope, the receiving airlock MUST:

1. Re-validate the envelope against the schema (§5). The receiver trusts nothing but what the
   envelope carries and what its own gates re-verify; no store is shared between zones.
2. Resolve `(sender.channel_type, sender.channel_identity)` through its own trust map — a
   deterministic, configuration-derived, exact-match lookup; never a model call, never content
   inspection. An unmapped identity, however well-authenticated, is a **drop**: no envelope, no
   screening spend, no reply toward the sender — always recorded, PII-safely (identity as digest).
3. Append **exactly one** receiver provenance entry, whose `label` is the fixed function of the
   resolved sender class (§3.1), and set `sender_class`, overwriting any inbound value — exactly
   one stamped envelope per deduplicated message (dedupe key `(sender.channel_identity,
   event_id)`); replays are silent no-ops.
4. Feed every provenance `source` into the broker-held receiving turn before the worker acts.

Taint derivation at ingestion is **deterministic and never model-judged**: a source taints the
receiving turn if *either* its chain label is `untrusted` *or* the receiver's own input trust map
does not trust it.

### 6.2 The one-way rule

> **No output of trust-mapping — or of any gate — may lower the taint derived from the provenance
> chain.** A gate may only *add* the receiver's own entry and set the receiver-owned
> `sender_class`. No resolution result renders a tainted envelope clean.

Three consequences, each a conformance clause (§8):

1. **Authenticity never cleans.** An `owner` or `peer-agent` resolution over a chain containing an
   `untrusted` origin appends a `trusted` hop — and the envelope stays tainted, because derived
   taint is any-untrusted over the whole chain. Hop authenticity is orthogonal to content trust.
2. **Sender-asserted labels are a floor, never a grant.** An `untrusted` entry taints the
   receiving turn even where the receiver's map would trust that source; a `trusted` entry earns
   nothing the receiver's map does not grant — a compromised peer stamping `trusted` launders nothing.
3. **Screens refuse or pass — never bless** (§6.9).

### 6.3 Non-strippability and the append-only chain

- The provenance chain is **append-only**: a hop appends via a frozen additive path; no operation
  edits or removes prior entries.
- Turn taint is **non-strippable**: once a turn is tainted it cannot be un-tainted within its
  lifetime. The flag has no setter; the gate derives every call's taint from the broker-held turn,
  ignoring any value the model supplies. Turn identity is broker-owned — the agent supplies
  neither the turn id nor a rollover signal, so it cannot launder taint by declaring a fresh turn.
- Taint is cleared **only** by broker/harness-owned turn rollover or session expiry. There is
  deliberately no taint-clearing API reachable by the agent; model output and later clean reads
  in the same turn MUST NOT clear taint.

### 6.4 No-write-up and audited declassification

The Biba rule: data at integrity level *L* MUST NOT flow into an action requiring a level *> L*
without an explicit, **audited endorsement**. In the two-level projection, the structural cut:

> **A tainted turn's external write escalates.** When the turn is tainted and the requested
> operation is an external write, the gate MUST route the call through the deployment's polarity
> seam to `require_approval` where a human is reachable, `deny` where none is. The cut is
> **grant-independent** — no autonomy rung routes around it — floor, never tunable off; and
> `transform` MUST NOT silently downgrade a tainted write.

Tainted **reads** are allowed and audited — they are what taints. Internal (non-boundary-crossing)
writes are not taint-gated by this floor.

**Endorsement** is the consumer's trusted-sources declaration: naming a source (e.g.
`connector:{tool}.{op}`) as trusted asserts a **declassification** — "raise this source's ingest
above the integrity floor for this agent." Endorsement MUST be declared in reviewed
configuration, never by the model or agent at runtime; it endorses a *source*, never individual
content; it can only ever *raise* an ingest's level; every exemption lands on the audit surface (§9).

### 6.5 Outbound stamping

The agent supplies the *intent* of an outbound publish; the broker constructs the *envelope*:

- The agent authors `event_id`, target `principal`, and `payload` — and nothing that could launder
  taint or forge provenance. The broker sets `sender`, `provenance`, `ts`, and `expiry`.
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
hop* — never a taint mechanism, never a correctness proof, never a substitute for the receiver's
own trust map.

- **Shape.** The signature is an **Ed25519** signature over a **DSSE pre-authentication encoding**
  (`DSSEv1 <len> <payloadType> <len> <payload>`) of a canonical-JSON (sorted keys, no whitespace,
  ASCII) **in-toto-style statement** binding, together: `subject` — a canonical hash of the
  actual inline `payload` (a swapped payload breaks the signature); the ordered provenance hops;
  the signature's own `key_id`/`zone` (attribution is non-malleable); and the anti-replay
  envelope identity `event_id` / `principal` / `expiry` / canonicalized
  `sender.channel_identity`. Binding both halves of the dedupe key means
  an exact replay dedupes, and ANY mutation — to `event_id` or the sender identity — breaks the
  signature and classifies as forgery, attributed to transport only, never the impersonated
  signer.
- **Identifiers.** The DSSE `payloadType` MUST be `application/vnd.in-toto+json`, and the
  statement's `_type` MUST be `https://in-toto.io/Statement/v1`. These are in-toto's own
  registered identifiers [IN-TOTO], adopted rather than minted: a PTC statement is an in-toto
  Statement, and a verifier that already understands the framework MUST NOT have to learn a
  second envelope spelling to parse one.

  The `predicateType` is a different matter, because the predicate is PTC's own. This
  specification pins its **shape** and not its value: a `predicateType` MUST be an absolute
  URI, MUST carry an explicit version segment, and MUST identify exactly one statement kind —
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
  image and never a caller assertion: **the agent cannot sign.**
- **Per-envelope.** The sending broker signs the full chain as it leaves
  (`covers = len(provenance)`), including preserved upstream hops; the verifier requires a
  signature's `zone` to equal the top hop it covers; inbound signatures are not carried across a
  relay (§1.3). Optional transparency-log anchoring of high-value chains (Sigstore/Rekor-style)
  is a deployment knob, OFF by default.

### 6.7 Receiver verification

Verification is a receiver-side knob that **MAY be off**; the consequence is expressed by the
maturity ladder (§7), not hidden: unsigned traffic caps the mesh at the unsigned-lineage tier,
and observer attribution degrades to transport-token strength.

When verification is enabled, it MUST fail closed:

- A chain with no signatures, an unknown signer, a `covers` outside `[1, len(provenance)]`, an
  invalid signature, or no signature covering the full chain MUST be rejected — dropped and
  loudly quarantined, before any budget-spending gate (trust map, dedupe, screen). Every
  *present* signature must verify; a valid full-cover signature does not excuse a forged prefix
  signature riding alongside it.
- Misconfigured verification (a configured but unusable key source) MUST fail closed loudly,
  never silently degrade to no verification.
- A successful verification is recorded as evidence of a check **performed** (e.g. `sig:pass` in
  the receiver's hop evidence) only when the gate actually ran — never asserted for an unverified
  chain. Downstream consumers (§6.11) MUST NOT infer, recompute, or override these fields.
- Verification authenticates lineage; it does **not** clean taint. The receiver re-derives taint
  from the chain regardless of signature status.

### 6.8 The deterministic gate

The gate contract is the invariant that makes everything above enforceable:

> **The model may only SURFACE a concern. It never sets a decision verb, never suppresses taint,
> and never modifies its own grant.** Every security-relevant fact the gate keys on comes from
> code and configuration — never from what a model said.

- The decision function MUST be **pure and deterministic**: no I/O, no model call, no side
  effects; the same inputs always produce the same decision. All environmental facts are resolved
  *before* the decision into a closed, pre-computed fact set; no fact may be model-derived.
- The verb alphabet is exactly the five verbs of §3.3. Rule evaluation is **first-match** over an
  ordered rule set; the matched rule's identity is recorded on the audit surface. A write no rule
  permits falls through to **default-deny**.
- The one model-authored input, the call's arguments, MUST be opaque to the verb: varying the
  arguments — including overt "this is pre-approved" injection text — never changes the verb.
- A model MAY surface findings, request a stricter outcome (`abstain` with escalation), or raise
  suspicion in a screen; it MUST NOT authorize, loosen, or decide. The model may tighten, never
  loosen — it is never the judge in the dangerous direction.

The safe-default polarity — whether an unresolvable case resolves to abstention or to a positive
safe action — is deliberately NOT fixed by this specification: it is re-derived per deployment at
a single, identifiable seam. Baking one polarity into the base is a latent safety bug for
deployments with the opposite harm profile. The seam is not merely a configuration convenience:
because a forced abstention is itself an attacker objective for one of the two polarities, the
gate's escalation is a *consequence to be monitored*, not an outcome to be trusted (§6.12).

This has a consequence for any policy language a deployment might substitute for the reference
gate. A language whose outcome alphabet is two-valued (permit / forbid) cannot express `transform`,
which must emit a substituted operation plus clamped arguments
(argument clamping: not yet implemented — #358) — and for the same reason it cannot host the
polarity seam, since a positive safe action is a positive action *with arguments*. The
value-producing verb and the caller-owned polarity seam are one requirement seen twice. A
conforming gate language MUST therefore support a value-producing outcome, and MUST support
existential quantification over provenance records and over chain hops — the Biba direction of
§6.4 and the chain reasoning of §5.3 and §6.7 are both quantified shapes, and a language without a
quantifier cannot state this specification's own subject matter.

### 6.9 The content screen (the one model-judged gate)

A deployment MAY run a content screen (injection classifier) at the receiving airlock. It is the
**only** gate that may consult a model, and it is confined to the suspicion-only direction:

- **A screen refuses or passes — it never blesses.** A pass changes nothing: not the envelope,
  not the chain, not the derived taint, not the sender class. A pass is **contentless** (no
  reason, no score, nothing downstream could cache as a safety assertion).
- A refusal's reason is a **closed-vocabulary machine code** (§3.6), never free text: model
  free-text derived from an untrusted message is itself untrusted content, and writing it to the
  trusted drop log would hand an injected payload a path into the audit trail.
- The seam MUST fail closed: an exception escaping the screen resolves as a `screen_error`
  refusal, recorded and dedupe-marked. An implementation may choose fail-open *inside* the
  screen; what is forbidden is the silent version, where infrastructure failure is
  indistinguishable from a clean verdict.
- The screen ships OFF; taint and every floor property hold identically with or without it —
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
  `(server_id, tool_name)` namespace with the host-assigned effect classification the gate reads
  — never the server's self-description; and (2) an **integrity-protected registry activation**
  recording the definition-hash a human admitted. The mutable layer can select and tighten, never
  mint a callable; missing either key fails toward uncallable.
- **The signed definition set.** The admitted hash MUST cover **every advertised field** of the
  definition, as canonical-JSON `sha256`: the always-present core `(server_id, tool_name,
  input_schema, description)` plus each metadata field the server actually advertised. A field the
  server does not advertise MUST be **excluded from the preimage**, never serialized as a null —
  the host cannot distinguish an absent field from one advertised as null, so neither may the hash.
  A field appearing, changing, or vanishing IS drift. Two properties follow from the exclusion, and
  both are load-bearing: a definition advertising no metadata hashes identically to the core-only
  basis, so no deployment is forced through an **empty-delta re-vet** — a drift alarm with nothing
  to show teaches operators to dismiss the alarm that matters; and additive growth of the
  definition schema moves no hash until a server actually advertises the new field, so tracking an
  evolving tool-description format is not a fleet-wide re-admission event. `description` is
  deliberately in the set: it is the model-facing injection surface, and a description-only change
  IS drift — as is a flipped annotation hint or a rewritten output schema.
- **Drift quarantines.** At every connect the host re-hashes the live advertised definition. A
  mismatch drops the tool to uncallable, surfaces loudly exactly once, and only a fresh human
  ceremony re-admits. An undeclared or unadmitted discovered tool is uncallable and surfaces as a
  quarantine-class finding.
- **Admission is a ceremony**: two distinct verified credential identities (maker ≠ checker;
  self-admission structurally impossible) and an append-only, signed admission record binding who
  admitted which tool at which hash.
- **Responses taint** by default under source id `connector:{server_id}.{tool_name}` (§6.4).
  Per-tool endorsement is legal **only** for tools declaring structured (schema-bounded) output,
  refused at configuration load otherwise — free text is exactly the injection surface
  endorsement must not wave through.
- **Reconnect is a new discovery.** A restarted or reconnected server is a new server: discovery
  evaluation re-runs in full before any call; no verdict or finding is carried across a death,
  which surfaces as a typed, attributed failure, never an indistinct timeout.
- **Capability-naming configuration is image-baked only.** Anything that names code to run or an
  endpoint to reach (spawn commands, server URLs) is honored only from the reviewed, image-baked
  declaration, structurally out of reach of any runtime-mutable store.

*Reference implementation note (non-normative):* the reference host's operator tooling renders
description drift verbatim, in full — the countermeasure is against reviewer complacency, not invisibility.

### 6.11 The off-path observer (campaign correlation)

A deployment SHOULD run an off-path observer correlating the airlock's recorded exhaust (drops,
screen refusals, approval-queue flood signals) into attributed campaigns for human review. It:

- MUST NOT gate: it holds no on-path seam and cannot refuse, delay, or throttle a request. Its
  output is a report and a suggested remediation from the closed vocabulary (§3.8); applying a
  remediation is a human ceremony.
- MUST attribute at recorded strength (§3.7). Signer (`signed-chain`) attribution is legal only
  for reasons **both signature-bound and dedupe-capped** (in this version, exactly
  `screen_refused`); forgery-class reasons attribute to the transport identity **only — never the
  claimed signer** — or the observer becomes a reflected-denial-of-service weapon against the
  impersonated party; and pre-dedupe reasons cap at transport-token strength regardless of
  verification, or a captured genuinely-signed envelope could be replayed to accrue an unbounded
  attributed count against an innocent signer.
- MUST treat unattributable events as terminal (reported, never throttle-counted); MUST be
  PII-safe (digests, key ids, machine codes, counts, timestamps, opaque refs — never raw identity
  or content); and MUST NOT shed — the remediation vocabulary contains no shed/deny/drop action.

### 6.12 Availability: forced abstention and queue amplification

Every control above is an **integrity** control: it prevents an agent from being made to do the
wrong thing. An adversary who cannot achieve that can still achieve something valuable — making the
agent do **nothing**. Poison an input, and the no-write-up floor (PTC-27) escalates the tainted turn
exactly as specified; the agent falls silent. The escalation is correct. Whether the resulting
silence is *safe* is not a property this specification can decide, and a conforming deployment MUST
NOT treat "escalated" as "handled".

**Polarity decides the harm.** Under **abstain-is-safe** polarity, a forced abstention is the
attacker's whole objective: the agent's job was to act on demand, and silencing it is the win. Under
**positive-safe-action** polarity, a forced abstention causes the exact harm the agent exists to
prevent — a silenced monitor does not raise the alarm it was deployed to raise. The safe response to
a poisoned input is escalation *to a human*, not abstention; those are the same thing only when a
human is actually reached. Two requirements follow.

**Liveness — the forced-abstention half.**

- A deployment whose safe-default polarity is positive-safe-action MUST operate a **liveness
  contract**: a declared expected output and a deadline, monitored independently of the agent.
- The monitor MUST be **deterministic**. It observes *that* an expected output did not occur by its
  deadline; it MUST NOT judge *why*. No model may sit on this path. The temptation is a smarter
  monitor that decides whether a given abstention was legitimate — that reintroduces a probabilistic
  classifier onto the safety path (§6.8) and hands the attacker a second thing to fool. A timestamp
  comparison cannot be prompt-injected.
- The sign of life MUST be an artifact the agent can neither forge nor suppress without actually
  failing to act — the enforcement point's own audit or ledger append of the declared operation. A
  self-report from the agent is not a sign of life.
- A deployment whose polarity is abstain-is-safe MAY omit the monitor. The polarity derivation is
  per-deployment; an implementation MUST NOT fix it on the deployment's behalf (§6.8).

**Queue de-amplification — the escalation half.**

- Escalating to a human is the safe response to a poisoned input. An adversary able to trigger many
  calls converts one injection into a **flood of approval requests**: a denial of service against
  human attention, and — worse — cover under which the one approval that matters goes unread.
- A conforming deployment MUST de-amplify and MUST NOT shed. Identical pending approvals (same
  principal, operation, and arguments) SHOULD coalesce onto one; coalescing is safe under both
  polarities because it never hides a genuinely distinct approval. A queue-depth threshold MAY raise
  an alarm, and when it does the deployment MUST still hold every intent.
- Shedding under flood — denying, dropping, or auto-resolving queued approvals — re-enters the
  forced-abstention harm above through the mitigation itself, and MUST NOT be a default. This is the
  same rule the off-path observer obeys at its own layer (PTC-41), and it generalizes: **the
  remediation vocabulary of a control that fires under attack must not contain the attacker's
  objective.**

## 7. The provenance-maturity ladder

A capability's autonomy is bounded by what the mesh can currently *prove* about its premise. The
ladder is normative:

| Tier | Provenance maturity | What it enables |
|---|---|---|
| 1 | **taint bit** propagates | correct gating *if you trust the sender*: low-blast autonomy against simulated or sandboxed effects only |
| 2 | full **lineage** in the chain (real origin sources carried, §6.5) | the receiver derives its own taint; a human sees the true origin: real-world action *with human approval* |
| 3 | **signed** lineage, receiver verification ON (§6.6–§6.7) | the receiver cannot be lied to: autonomous cross-mesh high-blast action becomes eligible |

**The join to GAL.** A capability's autonomy rung MUST NOT exceed the mesh's current provenance
maturity. The companion GAL specification (Grant & Autonomy Lifecycle, drafted in parallel as
this document's sibling) defines the grant levels, the evidence-licensed promotion ceremony, and
the deterministic demotion triggers; its promotion predicate MUST enforce this ceiling — the
acting rungs require the signed-lineage tier for high-blast external action — rather than leaving
it to reviewer judgment (see GAL-SPEC.md, the promotion-predicate and rung-ceiling sections).
This is not a limitation; it is the correct expression of what the trust layer can currently
prove. Signing moves the rung — but only the provenance ceiling, and that ceiling is not the only
bound. What the trust layer proves is *premise*, not *response time*: no provenance maturity
shortens the window between a high-blast act and its detection. So tier 3 makes high-blast
autonomous action *eligible* — it lifts the provenance ceiling — while GAL's independent
requirement that every high-blast promotion be human-ratified (GAL-SPEC.md §6.4.4) still holds:
after-the-fact controls bound a compromise's *duration*, not a single act's blast, and a control's
response time must be matched to the blast rate of the authority it bounds. Signing raises the
ceiling; it does not place a human inside a high-blast window, and detection MUST NOT substitute
for one.

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
| PTC-2 | All | The provenance chain is append-only: hops append via a frozen additive path; no operation edits or removes prior entries. | SCHEMAS C3 |
| PTC-3 | All | The raw original is never embedded: `payload` is parsed and bounded; the raw original sits behind `payload_ref` + `payload_digest` and is fetched only via a brokered, taint-tracked read. | SCHEMAS C2 |
| PTC-4 | All | Transport-neutral: no field closes over a wire technology; `channel_type` and `source` are open, namespaced vocabularies. | SCHEMAS C8 |
| PTC-5 | All | `expiry` is a hard TTL, checked deterministically before any budget-spending gate. | SCHEMAS C6 |
| PTC-6 | All | `event_id` joins the audit surfaces of both zones for end-to-end replay. | SCHEMAS C7 |
| PTC-7 | Producer | Outbound provenance is broker-stamped from the sending turn's taint state; the agent supplies no provenance and no label; a tainted turn yields a non-strippable `untrusted` entry; relaying appends and never edits. | PUBLISH P3; TRUST-MAPPING (sender side) |
| PTC-8 | Producer | The agent authors intent, not identity: it supplies `event_id`, target `principal`, `payload`; the broker sets `sender`, `provenance`, `ts`, `expiry`; `sender_class` is absent on the wire. | PUBLISH P4 |
| PTC-9 | Producer | Lineage, not a collapsed bit: a fresh origination carries the turn's actual ingested taint sources as origin hops; a relay does not duplicate sources already in the chain. | PUBLISH P8 |
| PTC-10 | Producer | A cross-zone publish is an external write; on a tainted turn it escalates through the standing no-write-up cut; no publish-specific rule exists. | PUBLISH P2; TAINT §5 |
| PTC-11 | Producer | The agent holds no transport credential and no signing key; both are broker-held and broker-resolved. | PUBLISH P1; SIGNING S3 |
| PTC-12 | Producer | The signature is Ed25519 over the DSSE PAE of a canonical in-toto-style statement binding the payload hash, the ordered hops, the signer's own `key_id`/`zone`, and the anti-replay set `event_id`/`principal`/`expiry`/canonicalized `sender.channel_identity`. The DSSE `payloadType` is `application/vnd.in-toto+json` and the statement `_type` is `https://in-toto.io/Statement/v1`; a `predicateType` is an absolute, explicitly versioned URI identifying exactly one statement kind. | SIGNING S1, S1b |
| PTC-13 | Producer | Signing is per-envelope with full-chain cover (`covers = len(provenance)`); attribution is non-malleable (a signature's `zone` must equal the top hop it covers); inbound signatures are not carried across a relay. | SIGNING S2 |
| PTC-14 | Receiver | Exactly one stamped envelope per deduplicated message, dedupe key `(sender.channel_identity, event_id)`; replays are silent no-ops. | SCHEMAS C1 |
| PTC-15 | Receiver | A `principal` the receiver does not serve is a drop, never a re-route. | SCHEMAS (principal); TRUST-MAPPING |
| PTC-16 | Receiver | `sender_class` is receiver-derived only: absent on the wire, set solely by the receiver's trust-map gate, unconditionally overwriting any inbound value. | SCHEMAS C4 |
| PTC-17 | Receiver | Trust mapping is a deterministic, configuration-derived, exact-match lookup; unmapped identities drop — silently toward the sender, always recorded, identity as digest never raw. | TRUST-MAPPING |
| PTC-18 | Receiver | The one-way rule: no gate output lowers chain-derived taint; authenticity never cleans; an `external` hop always taints. | TRUST-MAPPING (one-way rule) |
| PTC-19 | Receiver | Sender-asserted chain labels are a floor, never a grant: a source taints the receiving turn if its chain label is `untrusted` OR the receiver's own input trust map does not trust it. | TRUST-MAPPING (consequence 2) |
| PTC-20 | Receiver | Every provenance source is fed into the broker-held receiving turn before the worker acts. | SCHEMAS C5 |
| PTC-21 | Receiver | Enabled verification fails closed: missing/unknown-signer/invalid/short-cover chains are rejected and loudly quarantined before any budget-spending gate; every present signature must verify; misconfigured verification fails closed loudly. | SIGNING S4, S5 |
| PTC-22 | Receiver | Verification MAY be off; with it off, unsigned traffic passes and the deployment's maturity tier (§7) and observer attribution strength degrade accordingly — stated consequences, never silent ones. | SIGNING S5 |
| PTC-23 | Receiver | Evidence-of-check is stamped only by the gate that performed the check, only when it ran; downstream consumers never fabricate, infer, or override it. | SIGNING (gate evidence); WATCHDOG W8 |
| PTC-24 | Receiver (gate) | The decision function is pure, deterministic, and model-free; facts are pre-resolved into a closed fact set with no model-derived field; evaluation is first-match over an ordered rule set; the matched rule is recorded; unmatched writes default-deny. | deterministic-gate |
| PTC-25 | Receiver (gate) | The verb alphabet is exactly `allow` / `deny` / `transform` / `require_approval` / `abstain`; `transform` produces a substituted operation plus clamped arguments (argument clamping: not yet implemented — #358); model-authored arguments never change the verb. | deterministic-gate |
| PTC-26 | Receiver (gate) | Turn taint is source-based and non-strippable; turn identity is broker-owned; taint clears only by broker/harness-owned rollover; there is no agent-reachable clearing path. | TAINT §1, §3, §4 |
| PTC-27 | Receiver (gate) | No-write-up is floor: a tainted turn's external write escalates through the polarity seam (`require_approval` where a human is reachable, `deny` otherwise), grant-independently, with no silent `transform` downgrade; the cut is never tunable off. | TAINT §5, §1.1 |
| PTC-28 | Receiver (gate) | Endorsement (declassification) is configuration-declared, per-source, audited, and raise-only; never model- or agent-declared, never per-content, never a lowering operator. | TAINT §1.1, §2 |
| PTC-29 | Receiver (gate) | A content screen refuses or passes, never blesses: a pass is contentless and changes nothing; a refusal's reason is a closed-vocabulary machine code; an escaped screen exception fails closed as `screen_error`; ordering keeps expired/unmapped/replayed traffic away from the screen. | SCREENING; TRUST-MAPPING (consequence 3) |
| PTC-30 | Tool Host | Discovery is untrusted input: a tool is callable only via two-key admission — an image-baked declaration of the `(server_id, tool_name)` namespace + host-assigned effect classification, AND an integrity-protected activation at a matching admitted hash; the mutable layer can select and tighten, never mint; missing either key is uncallable. | MCP-HOST doctrine, M3, M4, M13 |
| PTC-31 | Tool Host | The admitted hash covers every advertised definition field as canonical-JSON `sha256` — the core `(server_id, tool_name, input_schema, description)` plus each metadata field the server advertised; a non-advertised field is excluded from the preimage, never serialized as null, so a metadata-less definition hashes identically to the core-only basis. The set is not narrowable. | MCP-HOST M1 |
| PTC-32 | Tool Host | A description-only change is drift. | MCP-HOST M2 |
| PTC-33 | Tool Host | Drift fails closed: the tool drops to uncallable, surfaces loudly exactly once, and is never auto-re-admitted; only a fresh human ceremony resolves a quarantine. | MCP-HOST M5, M6 |
| PTC-34 | Tool Host | Admission and re-vetting require two distinct verified credential identities (maker ≠ checker) and append an append-only, signed admission record binding admitter + tool + hash. | MCP-HOST M7, M8 |
| PTC-35 | Tool Host | A discovered tool's response taints the turn by default under source id `connector:{server_id}.{tool_name}`; per-tool endorsement is honored only for structured-output tools and refused at configuration load otherwise; any response screen ships OFF and can only refuse-or-pass. | MCP-HOST M9, M10, M11, M12 |
| PTC-36 | Tool Host | Every reconnect is a new discovery — full re-evaluation, fresh integrity reads, fresh credentials, no carried verdicts; death of a server or its transport is a typed, attributed failure, never an indistinct timeout. | MCP-HOST M17, M18 |
| PTC-37 | Tool Host | Configuration that names code to run or an endpoint to reach is honored only from the image-baked declaration, structurally out of reach of any runtime-mutable store; cleartext transports to non-loopback hosts are refused at load. | MCP-HOST M14, M21 |
| PTC-38 | Observer | The observer never gates: it is pure over its typed inputs, holds no on-path seam, and cannot refuse, delay, or throttle a request. | WATCHDOG W1 |
| PTC-39 | Observer | Attribution at recorded strength: signer attribution only for signature-bound AND dedupe-capped reasons; forgery-class reasons attribute to transport only, never the claimed signer; pre-dedupe reasons cap at transport-token regardless of verification. | WATCHDOG W2 |
| PTC-40 | Observer | Unattributable events are terminal: reported, never throttle-counted, never downgraded into a weaker attribution. | WATCHDOG W3 |
| PTC-41 | Observer | Observer outputs are PII-safe (digests, codes, counts, opaque refs — never raw identity or content); the remediation vocabulary is closed, construction-validated, deterministically mapped from attribution basis, and contains no shed action. | WATCHDOG W4, W5, W6 |
| PTC-42 | Receiver (gate) | Escalation is not treated as resolution: a deployment whose safe-default polarity is positive-safe-action operates a liveness contract over a declared expected output and deadline. The monitor is deterministic — it observes that the output did not occur, never why — no model sits on the path, and the sign of life is an enforcement-point-written audit or ledger artifact, never an agent self-report. | friction-doctrine (availability) |
| PTC-43 | Receiver (gate) | Approval-queue amplification is de-amplified, never shed: identical pending approvals coalesce; a depth threshold may alarm but the deployment still holds every intent; no default path denies, drops, or auto-resolves queued approvals under load. | friction-doctrine (availability) |

### 8.3 Origin-mapping coverage

| Origin clause set | Mapped by | Coverage notes |
|---|---|---|
| EventTrigger C1–C8 | PTC-1..6, 14, 16, 20 | full |
| PUBLISH P1–P8 | PTC-7..11 (P5 "no shared store" folded into §6.1 item 1; P6 "receiver re-gates everything" folded into PTC-17..20; P7 into PTC-4) | full |
| SIGNING S1–S7 | PTC-11..13, 21..23 (S6 "signing ≠ correctness" carried in §9; S7 tiering is packaging doctrine, non-normative here) | full |
| TRUST-MAPPING (one-way rule + consequences) | PTC-15..19, 29 | full |
| SCREENING | PTC-29 (+ §3.6, §6.9) | full (verdict-sink observability valve is implementation-tier, unmapped) |
| TAINT §1–§6 | PTC-26..28 (+§6.3–§6.4); §6 read-side knobs (budgets, egress bounds) are implementation-tier knobs, unmapped | full for floor; knobs partial by design |
| deterministic-gate | PTC-24, 25 | full |
| friction-doctrine (availability / forced abstention) | PTC-42, 43 (+§6.12, §9.6) | full for the floor; the polarity→default derivation is per-deployment and deliberately unmapped |
| WATCHDOG W1–W8 | PTC-38..41, 23 (W7 meta-alarm is an operational standard for scheduled runners, out of scope here) | W1–W6, W8 full; W7 unmapped |
| MCP-HOST M1–M21 | PTC-30..37 | M1–M11, M13, M14, M17, M18, M21 mapped; M12 folded into PTC-35; M15/M16/M19/M20 (construction refusals, env composition, respawn policy, shutdown reaping) are implementation-lifecycle clauses, deliberately unmapped in this version |

Existing conformance suites keyed to origin clause IDs remain traceable through this table.

## 9. Security considerations and open problems

1. **Declassification is the classic IFC hard problem.** Deciding *when* tainted data may
   influence a privileged action is where the real bugs live: over-restrictive lattices break
   usability; over-permissive ones reintroduce injection. PTC's answer is procedural — endorsement
   is configuration-declared, per-source, audited, raise-only (PTC-28) — and it remains a knob a
   human can misconfigure.
2. **Propagation-through-transform is not solved by signing.** A signature proves *who asserted a
   hop*, never that taint was *correctly propagated* through a model transform. Lineage carriage
   (PTC-9) bounds the exposure, but whether a model faithfully carried taint through
   summarization or synthesis is open — for PTC and for every signing standard.
3. **The mesh is only as trustworthy as its nodes' brokers.** Cross-zone taint is *correct* only
   if the sending zone also runs a PTC-enforcing broker; against an arbitrary agent, the
   receiver's own trust map and re-derivation (PTC-19) are the floor. Like TLS, PTC's value is
   bilateral and compounds with adoption.
4. **Verification OFF is a stated degradation, not a silent one.** With verification off, a
   transport-credentialed sender can fragment observer attribution by claiming a fresh sender
   identity per attempt — an under-counting gap, never a mis-attribution one; enabling
   verification is the shipped mitigation. Relatedly, attribution granularity is the **sending
   broker**, not an upstream agent behind it — per-envelope signing (§1.3).
5. **The screen is defense in depth, never a dependency.** Every taint property holds with the
   screen absent; an injection that fools the screen gains only what it already had (PTC-29).
6. **Availability is a first-class property, and this specification's own integrity controls are
   the lever against it.** Every floor here answers a poisoned input by escalating or refusing.
   For a deployment whose polarity is positive-safe-action, that answer *is* the harm (§6.12) — so
   an implementation scored only on integrity outcomes will record a successful denial of service
   as a successful defense. PTC-42 exists because the obvious alternative, a model judging whether
   a given abstention was legitimate, puts a probabilistic classifier back on the safety path. The
   residual risk is real and stated: a deterministic monitor *detects* silence, it does not prevent
   it, and the exposure window is bounded by the deadline a deployment chooses, not by this
   specification.
7. **An instruction to the model is an instruction to the component under attack.** Guidance that
   phrases a control as a duty of the agent — "the agent should weigh the trust level of its
   sources before acting" — misassigns the decision: a persuasive injection reads as trustworthy
   exactly when it is most dangerous. PTC's answer is structural rather than exhortative (PTC-24,
   PTC-26, PTC-28): trust is a deterministic function of source identity, resolved before the
   decision, in a lookup the agent cannot influence. The general rule, which also governs PTC-29
   and PTC-42, is that no probabilistic classifier may occupy the safety path — a validator that
   can be argued into approving is worse than none, because it manufactures confidence.

## 10. Extensibility

**Open by design** (vendors MAY extend): `sender.channel_type` and provenance `source` schemes —
open, namespaced vocabularies (PTC-4); screen refuse codes and observer remediation codes beyond
the reserved sets — extended in the implementation's declared configuration or source, **never
minted at runtime**; carriage bindings, signature key substrates, transparency-log anchoring.

**Closed** (extension is a versioned change to this specification, not a vendor option):

- **The sender-class set.** Exactly `owner` / `peer-agent` / `external`. Unmapped is an absence,
  not a class; adding classes changes the one-way rule's label function and is a spec change.
- **The verb alphabet.** Exactly the five verbs of §3.3.
- **The signed definition set** for tool admission (PTC-31). Narrowing is forbidden. Additive
  growth of the *definition schema* is deliberately **not** a spec change — because the basis is
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
| 0.2.0-draft | 2026-07-25 | Adds §6.12 (availability: forced abstention and approval-queue amplification) with PTC-42/43, promoting material previously confined to reference-implementation doctrine; states the gate-language expressiveness floor (value-producing verb, quantification over provenance) in §6.8 and §1.2 with the differential-test record; states the message-layer-versus-mutual-TLS choice in §1.2; adds §9.6–§9.7. Corrects PTC-31 and §6.10: the admitted definition hash covers every advertised field with non-advertised fields excluded from the preimage, not a fixed four — aligning the spec with the shipped contract (MCP-HOST M1). |
| 0.2.1-draft | 2026-07-29 | Resolves the signing-identifier question left open in §6.6, splitting it: the DSSE `payloadType` and statement `_type` are PINNED to in-toto's own registered identifiers (adopted, not minted), while `predicateType` is pinned by **shape** — absolute, explicitly versioned, one URI per statement kind — with the namespace left for the adopting standards body, since a vendor domain in a normative identifier makes conformance depend on one organization's DNS. Also states the document's discussion-only IPR posture and replaces version-relative scope language with "this version" / "this specification". |
| 0.2.2-draft | 2026-08-06 | Resolved a contradiction in which PTC-25, §3.3's preamble and §6.8 required `transform` to produce an operation **plus** clamped arguments while §3.3's own verb table stated an "and/or" form, by softening all four to "and/or". Superseded within the day by 0.2.3-draft, which resolves the same contradiction in the other direction; recorded rather than removed, because the LF received 0.2.1-draft and the intervening revision is part of the record. |
| 0.2.3-draft | 2026-08-06 | Resolves the `transform` contradiction 0.2.2-draft resolved the wrong way. The requirement is the **plus** form in all four places, including §3.3's verb table, and the argument-clamping half now carries the §3-style implementation-status marker (tracking #358) in each place a reader meets it. The earlier softening treated the specification as a description of the reference implementation; it is a description of the design, and the honest way to state a settled requirement the code has not reached is to mark it, not to weaken it. This also restores §6.8's polarity-seam argument, which turns on `transform` emitting *arguments* specifically and was materially weakened by the "and/or" form. Adopting the marker here makes PTC's convention identical to GAL's, where two clauses have carried it since 0.2.1-draft. |

## 12. References

**Adopted standards and primitives**

- MCP — Model Context Protocol (agent↔tool) and A2A — Agent2Agent protocol, Linux Foundation
  (agent↔agent): the adopted connectivity layers; A2A's extension mechanism and MCP's `_meta`
  field are natural PTC carriages.
- DSSE (Dead Simple Signing Envelope) and the in-toto Attestation Framework — the signature
  envelope / PAE and the Statement/Predicate shape of the signed chain statement.
- Ed25519 (RFC 8032) — the signature algorithm; Sigstore/Rekor — optional transparency-log
  anchoring.
- SPIFFE / IETF WIMSE — workload-identity substrate for signing keys (DID as fallback).
- Biba, K. J., *Integrity Considerations for Secure Computer Systems* (1977) — the integrity
  lattice and no-write-up rule.
- RFC 2119 / RFC 8174 — conformance keywords.
- SD-JWT-VC — reserved for the future selective-disclosure content drill-down (§1.3).

**Design inputs**

- *The Connectivity-Orthogonal Trust-Context Standard for AI Agents: Landscape Survey and
  Reference Architecture* (`docs/references/trust-context-landscape.md`) — the survey this
  specification is built against; it finds no existing general standard at this seam (closest
  domain-specific precedent: Google AP2, payments-only) and identifies CaMeL (arXiv:2503.18813)
  as the in-process design cousin (PTC = that pattern + a wire format + non-repudiation + a
  production broker).
- GAL-SPEC.md — the companion Grant & Autonomy Lifecycle specification (the autonomy seam this
  specification's maturity ladder joins to).

**Reference implementation**

- **safe-agents** — running in production since July 2026, with per-clause conformance suites for
  every origin document in §8.3, adversarially drilled on live infrastructure (including the
  forged-signer attribution drill behind PTC-39). The differential corpus from the
  policy-language evaluation (§1.2) is retained as the seed of the gate conformance suite.
