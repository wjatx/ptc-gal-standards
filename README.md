# spec/ — normative specification drafts (the LF contribution artifact)

This directory owns the **normative spec tier**: the formal PTC and GAL specifications prepared
for the Linux Foundation agent-standards track (epic #167, terminal issue #178). It is distinct
from the design spines (`docs/PTC.md`, `docs/GAL.md` — positioning, history, joins) and from the
contract-tier docs (`broker/*.md`, `channels/*.md` — the shipped contracts with conformance
suites). **Spec text follows the shipped contracts, never the reverse**: a spec/contract
disagreement is a drafting bug here or a tracked contract change there, not a silent edit.

| File | What it is |
|---|---|
| `PTC-SPEC.md` | PTC — Provenance & Trust Context. The trust seam: signed trust-context envelope + deterministic gate. |
| `GAL-SPEC.md` | GAL — Grant & Autonomy Lifecycle. The autonomy seam: authority as signed, evidence-gated, auto-demotable state. |

Conventions (taken deliberately, 2026-07-24): RFC 2119/8174 normative keywords; enumerations
defined before the structures that use them; every object as a field table plus a concrete
example; numbered conformance clauses (`PTC-n` / `GAL-n`) with a mapping table back to the origin
clause sets (M/W/S/L) so the existing conformance suites stay traceable; an explicit maturity
statement in each header (wire schemas may change before 1.0); implementation-agnostic
normative text, with the reference implementation (safe-agents) confined to clearly-marked
non-normative notes. Both headers carry a **discussion-only IPR statement**: the drafts are
shared for discussion and grant no license or other IP right.

Three drafting rules follow from "spec text follows the shipped contracts". **A clause number is
never reused** — withdrawing a clause marks it withdrawn and reserved in place, never renumbers
the survivors, because renumbering silently breaks traceability to earlier drafts and to review
comments keyed to the old number. **Version-relative scope language names a version above the
current one, or says "this version"** — a document headed `0.2.0-draft` that defers a question
"to 0.2.0" is deferring it to itself. And **a clause that is normative ahead of the reference
implementation says so, in a machine-readable marker**:

```
> **Implementation status:** NORMATIVE, NOT YET IMPLEMENTED in the reference implementation (tracking: #NNN).
```

The convention is defined in GAL-SPEC §3. It repeats at every place a reader can meet the clause
— the §7 conformance clause, the §6 behavior section, and the field-table row — so no route into
the document reaches an unimplemented requirement without the caveat. **Absence of the marker
means the clause is implemented**, which is what makes its presence worth anything. Grep for it
with `grep -rn "Implementation status:" spec/`.

Names PTC and GAL are final (wes, 2026-07-24). 0.1.0-draft was prepared for filing with the LF the
week of 2026-07-27; 0.2.0-draft (2026-07-25) folded in the #223 signed-set widening and the Five
Eyes conformance read (`docs/references/five-eyes-agentic-guidance.md`) and corrected two places
where spec text had fallen behind the shipped contracts (PTC §6.10/PTC-31, GAL §5.1/§6.9).
**`0.2.1-draft` (2026-07-29) superseded both and is the version filed and presented to the LF on
2026-07-30.** The two specs have since diverged by one revision: **GAL is at `0.2.2-draft`
(2026-08-03)**, correcting §8.1's evidence-poisoning mitigation, which was keyed on taint while the
attack it names does not require it (#342). Section numbering is unchanged. PTC remains at
`0.2.1-draft`; the same adversary-keyed framing in its §9 item 6 is #343, deliberately split so the
two specs do not bump on one pass.

**Both Five Eyes gaps that became GAL clauses landed as normative text, and both are marked
unimplemented**: FE-1 is the certification-term / lapse arc (GAL §6.7.6, `certifiedUntil`, the
`lapse` record type, GAL-34; tracking #255) and FE-2 is two-direction principal reconciliation
(GAL §6.11, GAL-35; tracking #256). The third spec-tier answer, FE-6, is a deliberate divergence
stated in PTC §1.2 rather than a requirement, so it carries no marker. Specifying FE-1 and FE-2
ahead of the code is deliberate, and it is not a violation of "spec text
follows the shipped contracts": that rule exists to stop the spec drifting away from what ships
*unnoticed*, and the marker is what keeps it noticed. A settled design question is honestly
settled in the spec tier while the implementation catches up, provided nobody can mistake the
clause for a shipped control — which is exactly the posture recorded in the Five Eyes read. The
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

**Unlike the diagram pages below, these two are copies.** Both are edited in the safe-agents working
tree (`docs/lf-standards-brief.md`, `docs/sci-fi-primer.md`) and copied here, so an edit made in this
repo is one the next upstream copy will overwrite.

---

**Visual resources — THIS REPO IS THE CANONICAL HOME of the diagram pages** (they are not part
of the spec snapshot): [`trust-bricks.md`](trust-bricks.md) renders the safe-agents composition
model as mermaid diagrams inline on GitHub; [`trust-bricks.html`](trust-bricks.html) is the full
interactive version (self-contained, open in any browser). [`data-flow.html`](data-flow.html)
walks a signal across two agents step by step, labeling which of PTC and GAL governs each
element, with the rung ladder, both lifecycle paths, and the maturity-ceiling join (accurate to
the 0.2.1-draft section numbers). [`standards.html`](standards.html) is the standards landscape:
everything adopted (connectivity, signing, identity, models, conventions), the two candidates
declined with a published record, the conceptual ancestry, and the two proposed seams.

The HTML pages are also published (publicly) on GitHub Pages, for presenting without a clone:

- Trust bricks: https://wjatx.github.io/trust-bricks/
- PTC & GAL data flow: https://wjatx.github.io/trust-bricks/data-flow.html
- Standards landscape: https://wjatx.github.io/trust-bricks/standards.html

**Sync discipline.** Every other location is a deploy target or a pointer, never an editing
surface: the Pages repo (`wjatx/trust-bricks`) carries byte-identical copies
(`trust-bricks.html` → `index.html`, `data-flow.html` → `data-flow.html`), and the old claude.ai
artifacts are stubs pointing at the Pages URLs. The file mapping is `trust-bricks.html` →
`index.html`, `data-flow.html` → `data-flow.html`, `standards.html` → `standards.html`. To
update a diagram: edit it HERE, copy into the Pages repo, push both. Never edit a copy in place.

---

**The field tables are now mechanically checked against the shipped schemas**
(`safe_agents/broker/tests/test_spec_contract_drift.py`): a field in a §5 table and not in the
Pydantic model fails unless its row carries the implementation-status marker, and a field in the
model and not in the table always fails. That covers field *names* only — types, required-ness,
enumerations and all normative prose remain unchecked, so the periodic re-derivation against the
contract docs is narrowed, not retired.
