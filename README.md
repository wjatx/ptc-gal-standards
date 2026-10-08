# spec/ — normative specification drafts (the LF contribution artifact)

This directory owns the **normative spec tier**: the formal PTC and GAL specifications prepared
for the Linux Foundation agent-standards track (epic #167, terminal issue #178). It is distinct
from the design spines (`docs/PTC.md`, `docs/GAL.md`: positioning, history, joins) and from the
contract-tier docs (`broker/*.md`, `channels/*.md`: the shipped contracts with conformance
suites). **Spec text follows the shipped contracts, never the reverse**: a spec/contract
disagreement is a drafting bug here or a tracked contract change there, not a silent edit.

| File | What it is |
|---|---|
| `PTC-SPEC.md` | PTC: Provenance & Trust Context. The trust seam: signed trust-context envelope + deterministic gate. |
| `GAL-SPEC.md` | GAL: Grant & Autonomy Lifecycle. The autonomy seam: authority as signed, evidence-gated, auto-demotable state. |

Conventions (taken deliberately, 2026-07-24): RFC 2119/8174 normative keywords; enumerations
defined before the structures that use them; every object as a field table plus a concrete
example; numbered conformance clauses (`PTC-n` / `GAL-n`) with a mapping table back to the origin
clause sets (M/W/S/L) so the existing conformance suites stay traceable; an explicit maturity
statement in each header (wire schemas may change before 1.0); implementation-agnostic
normative text, with the reference implementation
([ptc-gal-reference](https://github.com/wjatx/ptc-gal-reference)) confined to clearly-marked
non-normative notes. Both are licensed under the **Community Specification License 1.0**
(`SPDX-License-Identifier: Community-Spec-1.0`); see `LICENSE.md` for terms and `NOTICE.md`
for attribution, acceptance, and patent exclusions.

Three drafting rules follow from "spec text follows the shipped contracts". **A clause number is
never reused**: withdrawing a clause marks it withdrawn and reserved in place, never renumbers
the survivors, because renumbering silently breaks traceability to earlier drafts and to review
comments keyed to the old number. **Version-relative scope language names a version above the
current one, or says "this version"**: a document headed `0.2.0-draft` that defers a question
"to 0.2.0" is deferring it to itself. And **a clause that is normative ahead of the reference
implementation says so, in a machine-readable marker**:

```
> **Implementation status:** NORMATIVE, NOT YET IMPLEMENTED in the reference implementation (tracking: #NNN).
```

The convention is defined in GAL-SPEC §3 and restated in PTC-SPEC §2; both specs use it. It repeats
at every place a reader can meet the clause
(the §7 conformance clause, the §6 behavior section, and the field-table row), so no route into
the document reaches an unimplemented requirement without the caveat. **Absence of the marker
means the clause is implemented**, which is what makes its presence worth anything. Grep for it
with `grep -rn "Implementation status:" spec/`.

Names PTC and GAL are final (wes, 2026-07-24). 0.1.0-draft was prepared for filing with the LF the
week of 2026-07-27; 0.2.0-draft (2026-07-25) folded in the #223 signed-set widening and the Five
Eyes conformance read (`docs/references/five-eyes-agentic-guidance.md`) and corrected two places
where spec text had fallen behind the shipped contracts (PTC §6.10/PTC-31, GAL §5.1/§6.9).
**`0.2.1-draft` (2026-07-29) superseded both and is the version filed and presented to the LF on
2026-07-30.** Both specs have since moved past it, separately and for unrelated reasons. **GAL is at
`0.2.2-draft` (2026-08-03)**, correcting §8.1's evidence-poisoning mitigation, which was keyed on
taint while the attack it names does not require it (#342). **PTC is at `0.2.3-draft` (2026-08-06)**,
correcting PTC-25, §3.3 and §6.8, which required the `transform` verb to produce an operation *plus*
clamped arguments while §3.3's own verb table said "and/or". The requirement is the "plus" form, and
the argument-clamping half now carries the §3-style implementation-status marker rather than being
weakened to match the reference implementation; PTC states that marker convention in its §2. Section
numbering is unchanged in both. The same adversary-keyed framing #342 fixed in GAL is still live in
PTC §9 item 6.

**Both specifications are now at `0.4.0-draft` (2026-10-03).** PTC redefines tier 3 of its
provenance-maturity ladder, which said "the receiver cannot be lied to" and was defined only by
what the chain carries. Where a signer's key is within reach of its own agent, that agent can forge
a chain that verifies. A receiver now records, for each verification key, whether the signer holds
the key where its agent cannot reach it, and the evidence class of that record (PTC §7.1, PTC-48).
A verified chain reaches tier 3 only when every signing key is so recorded. In this version the
class is always `declared`: an operator's statement that the receiver has not checked. `attested`
is reserved, and §1.3 names the attestation work that would produce it. GAL's ceiling inherits the
class: a promotion into an acting rung states its maturity and that maturity's evidence class, and
the ratifier is shown both (GAL-20, marked NOT YET IMPLEMENTED).

**PTC is now at `0.5.1-draft` and GAL at `0.6.1-draft` (2026-10-07).** Three changes since the
revision below. A decision is good no longer than its shortest-lived input: the decision point
records the earliest instant at which any input it depended on passes its maximum age, and a call
held on the decision or carried on from it is evaluated again past that instant (PTC §6.13,
PTC-50; GAL §6.15, GAL-43). A new audit clause holds a lapse record's term to the one its latest
promotion ratified, so a planted lapse record cannot misstate how long a grant was certified
(GAL §6.11, GAL-44). And the reference implementation built the lapse record's term field and the
audit rule that holds a grant's `ts` to its latest ledger record, so their markers come off; GAL
§6.11 now says an existing ledger is brought into line with that rule by signed acknowledgment
and by nothing else. 42 of 97 clauses are marked.

**PTC `0.5.0-draft` and GAL `0.6.0-draft` (2026-10-07).** Both add rules ahead of the
reference implementation, each marked NOT YET IMPLEMENTED and each stated with the attack it
prevents. The class they share is the confused deputy: a component that holds authority is induced
to use it for a party that holds none.

- **What a refused sender learns** (PTC §6.1, PTC-44). A sender the receiver authenticated and
  mapped learns whether a refusal of authority is permanent or transient, and nothing more. Every
  other sender gets a uniform response, and a refusal on content reads as acceptance to everyone.
  A stranger gets no map of the receiver's trust configuration, and a compromised peer gets no
  feedback on which payload got through.
- **Delegation does not launder** (PTC §6.3, PTC-51). A context an agent spawns outside its zone,
  or schedules for later, starts at least as tainted as the turn that caused it. Otherwise an agent
  that has read planted content hands the task to a worker whose turn is clean.
- **A destination the agent composed** (PTC §6.4, PTC-52). On a tainted turn, a call whose
  destination the agent composed is escalated like an external write, as a control a deployment may
  leave off. This narrows request forgery steered by injected content. It does not stop request
  forgery inside a tool server, which only containment closes, and the specification says so.
- **Time** (PTC §6.13, PTC-49 and PTC-50; GAL §6.7.6 and §6.15, GAL-42 and GAL-43). Clock skew is
  applied toward less authority and never symmetrically, and every fetched input a decision
  depended on has a recorded as-of instant and a declared maximum age.
- **An approval that expires undecided** (PTC §6.12, PTC-53) is recorded as expired, is told apart
  from a human denial, and is reported to the approver while a flood alarm is raised.
- **Issuer standing** (GAL §6.10, GAL-41). A grant whose issuer's key has been withdrawn is
  enforced at its last safe level; a verifier in another domain, which holds no such level, treats
  the path as conferring nothing. Rotation is not withdrawal.

GAL also states that its lifecycle is proven by test and has no proven-in-use evidence (§3), and
GAL-18 and three PTC clauses gain markers an audit found they needed. Across both specifications 42 of 96 clauses were marked at that revision.

**GAL `0.5.1-draft` (2026-10-07).** The `reattestation`
record is built in the reference implementation, so its NOT YET IMPLEMENTED markers come off.
Building it found that GAL-15 forbade any rewrite of a grant at an unchanged level, which
contradicted the demotion that fires on a grant already at its floor. The clause now states the
rule that was meant: no write to a grant goes unrecorded. A re-attestation is also always signed
under the issuer role and is refused when no issuer signing key is configured, so that a
quarantined grant cannot be returned to acting authority with nobody accountable for it (GAL §6.6,
GAL-15).

**`0.5.0-draft` (2026-10-04).** The revision answers
[#2](https://github.com/wjatx/ptc-gal-standards/issues/2), which asked what a ledger record's `ts`
denotes. A record's `ts` is the instant its writer appended it, for every record type, and it does
not establish the time of any event that was not separately retained (GAL §5.2, GAL-40). The
ledger now journals every write to a grant: re-attestation after an envelope change appends a
record of a sixth type, `reattestation`, where earlier drafts said it appended none (GAL §6.6,
GAL-15). A `lapse` record may carry the term that expired, and an auditor reports a grant whose
`ts` its latest ledger record does not match (GAL §6.11). The new record type, the lapse field and
the audit rule were marked NOT YET IMPLEMENTED; the record type's marker came off in `0.5.1-draft`.

**`0.3.1-draft` (2026-10-02).** GAL extends two of 0.3.0's
corrections to the records the issuer signs: a promotion that skips a rung is refused wherever a
record is parsed, not only by the ceremony (GAL-17), and every record must start from the level the
ledger held before it, whichever role signed it (GAL-37). Both are marked NOT YET IMPLEMENTED.
PTC-35 now states its condition: it constrains a response screen where an implementation provides
one and requires none, so its marker is removed.

**`0.3.0-draft` (2026-10-02).** PTC's is a wire change: a signature
covers the whole envelope except two named fields, every signature covers the full chain, each
verification key is scoped to one zone and its sender identities, and an envelope names the one
receiver it is for (`audience`, PTC-45 to PTC-47). GAL's corrects the ledger's integrity contract:
a record the evaluator signs can only lower a level and only from where the ledger stands, no
record is excused from signing by anything it says about itself, and an acknowledgment binds the
stored bytes of the record it excuses. Each was found by attacking the reference implementation,
and each was a gap in the specification as well as in the code. The version history at the end of
each document has the detail. The same revision removes dashes used as punctuation from both
documents, so the inline marker's short form is now `(not yet implemented, #NNN)`.

**Both Five Eyes gaps that became GAL clauses landed as normative text, and both are marked
unimplemented**: FE-1 is the certification-term / lapse arc (GAL §6.7.6, `certifiedUntil`, the
`lapse` record type, GAL-34; tracking #255) and FE-2 is two-direction principal reconciliation
(GAL §6.11, GAL-35; tracking #256). The third spec-tier answer, FE-6, is a deliberate divergence
stated in PTC §1.2 rather than a requirement, so it carries no marker. Specifying FE-1 and FE-2
ahead of the code is deliberate, and it is not a violation of "spec text
follows the shipped contracts": that rule exists to stop the spec drifting away from what ships
*unnoticed*, and the marker is what keeps it noticed. A settled design question is honestly
settled in the spec tier while the implementation catches up, provided nobody can mistake the
clause for a shipped control, which is exactly the posture recorded in the Five Eyes read. The
implementation issues stay open until code closes them; do not close them against spec text.

The 2026-07-29 pre-filing pass also closed the five deferred-decision markers the drafts carried,
added the shipped `PromotionRecord.attestation` field to GAL §5.2, and pinned GAL's canonical JSON
encoding. The remaining feeder is #180 cross-relay attribution, still declared out of scope in PTC
§1.3; LF feedback lands in the pass after.

---

**Companion reading (non-normative).** [`lf-standards-brief.md`](lf-standards-brief.md) is the short
framing: what the two seams are, and why they belong in a standards track rather than in one
vendor's design. [`sci-fi-primer.md`](sci-fi-primer.md) walks the same ground through familiar films,
for a reviewer meeting both seams for the first time. Paragraphs are labeled PTC or GAL, there are no
clause IDs, and it carries its own section on what neither proposal solves. Neither document is
normative and neither should be cited for a requirement.

**Unlike the diagram pages below, these two are copies.** Both are maintained upstream and copied
here, so an edit made in this repo is one the next upstream copy will overwrite. Raise changes to
them against the reference implementation rather than here.

---

**Visual resources: THIS REPO IS THE CANONICAL HOME of the diagram pages** (they are not part
of the spec snapshot): [`trust-bricks.md`](trust-bricks.md) renders the reference
implementation's composition model as mermaid diagrams inline on GitHub; [`trust-bricks.html`](trust-bricks.html) is the full
interactive version (self-contained, open in any browser). [`data-flow.html`](data-flow.html)
walks a signal across two agents step by step, labeling which of PTC and GAL governs each
element, with the rung ladder, both lifecycle paths, and the maturity-ceiling join (accurate to
the 0.2.1-draft section numbers). [`standards.html`](standards.html) is the standards landscape:
everything adopted (connectivity, signing, identity, models, conventions), the two candidates
declined with a published record, the conceptual ancestry, and the two proposed seams.
[`sub-grants.html`](sub-grants.html) shows how a durable agent delegates to ephemeral sub-agents:
broker-computed, strictly-attenuating sub-grants with full-chain attribution. Delegation is out of
scope for GAL 0.2.x (§1.3), so that page documents reference-implementation doctrine and a
candidate seam for a future revision, and says so on the page.
[`mesh.html`](mesh.html) shows the agent-to-agent shape: every edge is the sender's broker talking
to the receiver's airlock, and a mesh is that one edge repeated with nothing in the center. Its
companion prose document is [`broker-airlock-mesh.md`](broker-airlock-mesh.md) (a copy, like the
brief; maintained upstream).

The HTML pages are also published (publicly) on GitHub Pages, for presenting without a clone:

- Trust bricks: https://wjatx.github.io/trust-bricks/
- PTC & GAL data flow: https://wjatx.github.io/trust-bricks/data-flow.html
- Standards landscape: https://wjatx.github.io/trust-bricks/standards.html
- Sub-grant delegation: https://wjatx.github.io/trust-bricks/sub-grants.html
- Broker-to-airlock mesh: https://wjatx.github.io/trust-bricks/mesh.html

**Sync discipline.** Every other location is a deploy target or a pointer, never an editing
surface: the Pages repo (`wjatx/trust-bricks`) carries byte-identical copies
(`trust-bricks.html` → `index.html`, `data-flow.html` → `data-flow.html`), and the old claude.ai
artifacts are stubs pointing at the Pages URLs. The file mapping is `trust-bricks.html` →
`index.html`, `data-flow.html` → `data-flow.html`, `standards.html` → `standards.html`,
`sub-grants.html` → `sub-grants.html`, `mesh.html` → `mesh.html`. To
update a diagram: edit it HERE, copy into the Pages repo, push both. Never edit a copy in place.

---

**The field tables are now mechanically checked against the shipped schemas**
(`safe_agents/broker/tests/test_spec_contract_drift.py`): a field in a §5 table and not in the
Pydantic model fails unless its row carries the implementation-status marker, and a field in the
model and not in the table always fails. That covers field *names* only; types, required-ness,
enumerations and all normative prose remain unchecked, so the periodic re-derivation against the
contract docs is narrowed, not retired.
