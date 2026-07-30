# PTC & GAL — standards brief for the Linux Foundation discussion

> One/two-pager for the LF agent-standards conversation: what we're building, which existing
> standards we adopt, why two seams remain unstandardized, and how these proposals fill them.
> Names are final (2026-07-24); the normative spec drafts are `spec/PTC-SPEC.md` and
> `spec/GAL-SPEC.md` (0.2.0-draft). Internal spines: `docs/PTC.md`, `docs/GAL.md`.

## What we're building

This is a **controls-engineering project** for agentic systems — not another agent framework or
harness. You bring your own harness (the model, the loop, the prompts, the orchestration);
the controls secure it from outside, in ways the harness probably cannot secure itself. A
harness's own guardrails run *inside* the thing being guarded — a system prompt is advice to the
component under attack, and anything the model can be talked out of is not a control. The
controls we are designing are enforced by separate infrastructure holding separate identity,
around one architectural commitment: **the agent holds no credentials, and its only egress is a
deterministic tool broker.** Every tool call crosses a broker that decides per call — allow /
deny / transform / require-approval / abstain — from signed policy state, and writes a
tamper-evident, hash-chained audit under a separate identity the agent cannot reach. A fully
compromised agent can still only *ask*.

**safe-agents is the reference implementation that demonstrates these controls.** It has been
running continuously on live cloud infrastructure since early July 2026 with a real consumer
agent; the two proposals below are the control layers extracted from that build. Neither is a
paper design: both are implemented and adversarially drilled on live infrastructure, and
conformance-tested at the contract tier, with clause-level traceability from the spec drafts back
to those suites. A small number of spec clauses are deliberately normative *ahead* of the
implementation; each is individually marked in the draft, because a spec clause is not a shipped
control.

## What we adopt

Our working rule is **adopt the standard where one exists; build only the seam**:

- **Connectivity:** MCP (agent↔tool) and A2A (agent↔agent). We ride them — MCP `_meta`, A2A
  extensions, or native channels — and do not compete with them.
- **Signing & attestation:** DSSE envelopes over in-toto-style statements, Ed25519 keys; optional
  Sigstore/Rekor-style transparency anchoring; SD-JWT-VC reserved for selective content
  disclosure on audit drill-down.
- **Identity:** SPIFFE/WIMSE workload identity as the signing-key substrate (DID as fallback).
- **Integrity model:** the Biba lattice — low-integrity data may not influence a higher-integrity
  action without an explicit, audited endorsement.
- **Policy languages:** Cedar and OPA/Rego were evaluated seriously, not dismissed — a
  differential spike ran our 14-rule reference gate over all 73,728 reachable inputs in both.
  Both were declined with a published record: Cedar cannot checkably express first-match ordering
  or value-producing decisions (`transform`); Rego reproduces the gate exactly but forfeits the
  formal-analyzability motivation while adding an external binary to a path whose property is
  purity. We cite this because the adoption posture is evidence-based in both directions.
- **Conceptual ancestry for autonomy:** Sheridan–Verplanck and SAE J3016 levels-of-autonomy
  vocabulary; FDA's PCCP (a regulator pre-authorizing change inside a declared envelope); ODD
  (capability claims bounded by a declared operating domain).

## The gap — two seams nobody standardizes

**1. Trust does not travel.** MCP and A2A move calls and data but carry no trust model — no
provenance, no taint, no sender class. MCP tool-poisoning and A2A impersonation are open problems
precisely because of that. The adjacent standards don't close it: DSSE/in-toto prove *who signed*
something, not how trustworthy it is for a given action; Cedar/OPA evaluate policy, but nothing
standardizes the trust context they evaluate *over*. The seam — a signed trust-context object,
carried with the data across every agent and tool boundary and consumed by a deterministic gate —
is unowned. Our landscape survey found no general standard here; the closest precedent (AP2) is
payments-only.

**2. Autonomy is a label, not state.** Forty years of levels-of-autonomy work produced
descriptive taxonomies — a label on a system, not stored state an enforcement point reads per
call. FDA PCCP is the strongest precedent for pre-authorized change, but it is paperwork: no
runtime object, no automatic rollback when the evidence sours. Nothing in the current agent
stack makes an agent's authority something it must *earn* through measured evidence and *loses*
automatically — enforced by infrastructure the agent cannot write past.

## How PTC and GAL fill it

**PTC (Provenance & Trust Context)** is the trust seam: a signed envelope — sender class
(owner / peer-agent / external, a floor never a grant) + an append-only, non-strippable provenance
chain + taint labels on the Biba lattice — riding orthogonally to transport and consumed by a
deterministic, model-free gate at every boundary. Positioning line: *MCP is how agents reach
tools; A2A is how agents reach agents; PTC is how trust travels across both.* Implemented today:
broker-side taint self-ingestion (an untrusted read taints the turn; a later external write
escalates to human approval); Ed25519/DSSE chain signing with receiver-side verification; an MCP
host contract that treats tool discovery as untrusted input (a tool is callable only via two-key
admission over a signed definition set, and a changed description — the injection surface —
quarantines the tool until re-vetted); and a campaign watchdog whose live drill turned six forged
events claiming a real signer's key into cryptographic attribution that placed the claimed signer
nowhere in the forgery.

**GAL (Grant & Autonomy Lifecycle)** is the autonomy seam: authority is a per-(principal,
action-class) grant whose level (in-loop / on-loop / out-of-loop, with "no grant" as the floor)
is stored, signed state — moved **up** only by a maker≠checker ceremony licensed by a
deterministic predicate over measured evidence (no model in the licensing path; high-blast
classes always human-ratified), moved **down** automatically by four deterministic demotion
triggers (stale confidence, corroboration failure, budget breach, owner-flagged false action),
and enforced by the broker per call. Every state change is a signed, append-only ledger record;
an independent audit re-verifies the whole ledger in CI. Implemented and proven end-to-end: a
real principal was promoted on real approval evidence, demoted by an induced trigger, re-climbed,
and then *exercised* — every leg under a distinct least-privilege identity, all four demotion
triggers fired live.

**The join** is the rule that makes the pair more than the parts: *a capability's autonomy rung
is capped by what the mesh can currently prove about its inputs.* A bare taint bit permits at
most trusted-sender autonomy; unsigned lineage means approval-gated; only signed, receiver-
verified lineage makes high-blast autonomous action eligible. PTC supplies the proof; GAL moves
the state; the ceiling is enforced in the promotion predicate, not left to reviewer judgment.

Both proposals compose with — rather than fork — the ecosystem above, and both come with the
thing a standards process needs most: a running reference implementation, conformance suites,
and the scars of live adversarial drills.
