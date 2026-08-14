# Broker to airlock — agent-to-agent without a mesh authority

> **Status: doctrine synthesis (2026-08-13).** Written for the LF working-group discussion of
> agent-to-agent communication, where the recurring observation is that no widely adopted agent
> mesh protocol exists and A2A keeps being named without being proposed as the answer. This
> document composes what the channels contracts already ship (`channels/PUBLISH.md`,
> `channels/TRUST-MAPPING.md`, `channels/SCREENING.md`, `channels/SIGNING.md`) into the
> agent-to-agent story. It introduces no new mechanism. The public copy lives in
> `wjatx/ptc-gal-standards`; the diagram page is
> [mesh.html](https://wjatx.github.io/trust-bricks/mesh.html).

## The question the mesh conversation keeps asking

When people ask for an agent mesh protocol, they usually mean three things at once: how agents
find and address each other, how bytes move between them, and why a receiving agent should act on
what arrives. The first two are solved plumbing. Discovery and task exchange have A2A; transport
has MCP, HTTP, message queues, and every other wire the industry already runs. Our standards
landscape adopts these deliberately.

The third question is the unsolved one, and it is not a transport question. Why should Agent B do
anything because Agent A said so? What did A's message actually carry, who vouches for that, and
what is B allowed to do about it? That is a trust seam and an authority seam, and no amount of
transport standardization answers it.

## The shape: every edge is broker to airlock

Agent A never talks to Agent B. A's broker talks to B's airlock, and B's broker talks to A's
airlock.

On the sending side, the agent can only ask. Its broker holds the peer credential and the peer
address the agent never sees, decides the publish like any other brokered call, stamps the
outbound provenance from the sending turn's actual state, signs the envelope, and sends. A fully
compromised sender can still only ask its broker to publish; it cannot forge the transport
authenticity the peer will verify.

On the receiving side, a peer agent is an ordinary sender at the airlock. The airlock re-validates
and re-gates the envelope exactly as it would any inbound signal: authenticity check, screening,
dedup, and taint derivation under the receiver's own trust map. Nothing about being an agent earns
a privileged path. And acting on the message still crosses the receiver's own broker, under the
receiver's own grants.

The edge is symmetric, so a bidirectional relationship is just the same contract twice. A mesh is
the set of such edges, and nothing in the middle has to exist.

## Authenticity is not content trust

The airlock verifying that a message really came from Agent A grants nothing. Three consequences,
each a conformance test in the reference implementation:

1. **Authenticity never cleans.** A verified sender whose chain contains an untrusted origin stays
   tainted; derived taint is any-untrusted over the whole chain.
2. **Sender-asserted labels are a floor, never a grant.** Chain labels arrive from the sending
   zone and could be forged by a compromised peer, so the receiver's own trust map is
   authoritative for the receiving turn. A peer that stamps `trusted` on its origin launders
   nothing; a peer that honestly stamps `untrusted` taints the turn even for sources the receiver
   would otherwise trust.
3. **Screens refuse or pass, never bless.** The one model-judged gate may refuse a message or let
   it through; a passing screen changes nothing about taint. Model judgment tightens only.

This is the line that separates the pattern from mTLS-and-done thinking. Mutual authentication
answers "who is speaking." It says nothing about whether the content deserves to influence an
action, and conflating the two is how a compromised-but-authentic peer becomes an injection
vector.

## Provenance is broker-stamped, never agent-authored

What crosses the seam is a signed envelope whose sending-zone provenance entry, and therefore the
taint the receiver will derive, is set by the sending broker from the turn's ingested taint state.
The agent supplies the message; it does not get to write its own history.

Chains are signed per envelope. A relay re-packages and signs its own hop; upstream hops ride as
lineage rather than as carried signatures. Receiver-side signature verification is an envelope
knob, and honesty requires saying it ships OFF by default in the reference implementation; the
acting autonomy rungs are what require the signed-lineage tier.

## No shared store

The two zones share exactly two things: the envelope on the wire and one audit-join key. No
database, no queue, no shared memory. The receiver trusts nothing but what the envelope carries
and what its own gates re-verify. This is what makes the trust story auditable per edge, and it is
also what makes the pattern deployable across organizational boundaries, where a shared store was
never going to be on offer.

## Taint rides through

A peer's message enters the receiver's turn at the `AGENT` level of the five-level integrity
lattice (`SYSTEM > USER > AGENT > TOOL_OUTPUT > UNTRUSTED_WEB`). It can inform the receiver. The
standing tainted-external-write cut still governs anything the receiver does about it: content
below the required integrity floor cannot influence an external write without an audited human
endorsement. Agent-to-agent communication needs no special rule, because the general mechanism
already covers it, and a rule that existed only for peers would be a rule a non-peer path could
bypass.

## Why this composes to a mesh

- **No center.** There is no mesh authority to stand up, govern, or compromise. Each receiver is
  sovereign: its trust map prices every sender, edge by edge.
- **Unilateral adoption.** An agent joins by deploying an airlock and letting peers' brokers hold
  its address. No existing member reconfigures; no registry admits it. Adoption of the pattern
  does not require the counterparty's cooperation, which is the adoption property most proposed
  mesh schemes lack.
- **Transport indifference.** The envelope contract does not care what carried it. A2A, HTTP, a
  queue, or git-as-mailbox are all admissible wires; the seam is what rides the wire and what
  happens at each end.
- **Blast-radius bounds come free.** Publishing crosses the sender's broker, so per-principal
  budgets and caps apply to outbound traffic like any other op. One noisy agent exhausts its own
  envelope, and only its own.

## What this is, and what it is not

This is a composition of shipped parts: the channels contracts and their conformance suites in the
reference implementation, the PTC envelope as the proposed standard for what crosses the seam, and
GAL as the proposed standard for what the receiving principal may do at all. The transports are
adopted standards.

It is deliberately not presented as an agent mesh protocol. A protocol claim would need things
this document does not attempt: a normative wire format beyond our own `EventTrigger`, a discovery
and addressing story, versioning and capability negotiation, and multiple independent
implementations. Naming it prematurely would spend credibility the specs need.

What can be said precisely: if an agent mesh protocol emerges and is worth adopting, the
broker-to-airlock seam is the trust and authority layer it will need to carry, and PTC and GAL are
our proposals for exactly that layer. The pattern is available to be built on now, one edge at a
time, without waiting for the mesh question to settle.
