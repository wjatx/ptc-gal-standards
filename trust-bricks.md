# Trust bricks: the safe-agents composition model

A visual companion to the two spec drafts, for readers who think in pictures. The full
interactive version is [`trust-bricks.html`](trust-bricks.html) (open it in any browser; it is
self-contained). The diagrams below are the same model in mermaid, rendered by GitHub inline.

The metaphor: an agent is one studded Lego stack. The **studs** are the contract, which is why
bricks compose at all. **Amber** bricks are per-agent trust cells that are never shared; a fully
compromised agent can still only *ask*, because these are its alone. **Teal** bricks are shared
mechanism: one implementation under every agent, so fifty agents is not fifty platforms.

Mapping to the specs: the studs (signed provenance, trust-map admission, the schema envelope)
are the **PTC** seam; the amber authority state (envelope, grants, rungs, and their lifecycle)
is what **GAL** governs.

## Plate 01: one agent is an atomic stack

```mermaid
flowchart TB
  STUDS["<b>The studs · the contract</b><br/>schemas · EventTrigger · trust_map · signed provenance<br/><i>the interface, not the transport</i>"]

  subgraph TRUST["Per-agent · un-shareable"]
    direction TB
    P["<b>Principal</b><br/>identity = zone · one agent, one principal"]
    T["<b>Turn · taint · budget</b><br/>one turn per principal, broker-owned · taint can't be laundered"]
    TM["<b>Trust map · polarity</b><br/>who may reach it, and what admitted content may do"]
    ENV["<b>Envelope</b><br/>grants · rungs · caps · reversibility"]
    AUD["<b>Audit chain</b><br/>hash-chained, WORM, un-forkable"]
    P --- T --- TM --- ENV --- AUD
  end

  subgraph MECH["Shared · one implementation"]
    direction TB
    BR["<b>Broker</b><br/>deterministic PDP · same code under every agent"]
    AIR["<b>Airlock · drain</b><br/>inbound dispatch + single-principal drain"]
    CON["<b>Connectors</b><br/>the only thing that touches a real API · broker-held creds"]
    INF["<b>KMS · network · audit &amp; ledger buckets</b><br/>shared infra, zero per-agent cost"]
    BR --- AIR --- CON --- INF
  end

  STUDS --- TRUST --- MECH

  classDef contract fill:#f9e2db,stroke:#d9563c,color:#28323f
  classDef trust fill:#f6e5d0,stroke:#c8863a,color:#28323f
  classDef mech fill:#dbeeec,stroke:#2f9c94,color:#28323f
  classDef tier fill:transparent,stroke:#b3c0d0,color:#5c6a7a
  class STUDS contract
  class P,T,TM,ENV,AUD trust
  class BR,AIR,CON,INF mech
  class TRUST,MECH tier
```

## Plate 02: a mesh is bricks snapping stud-to-stud

One agent's emitter is another's receiver: a brokered `peer.publish` lands at the receiver's
airlock as `sender-class=peer-agent`. Same studs, so they just fit. The worked example is the
trading-agent set.

```mermaid
flowchart LR
  subgraph PROD["Producers"]
    R["<b>Research</b><br/>abstain · read-only<br/>market_data.read · news.read · search.query<br/><i>no trade op at all</i>"]
    CS["<b>Chief-of-Staff (email)</b><br/>mesh neighbor · own spec<br/>reads email (untrusted), publishes a confirmation"]
  end

  subgraph CONS["Consumers"]
    PA["<b>Paper-Alpaca trader</b><br/>trade.place · paper<br/>rung sweeps blocked → out-of-loop, config-only"]
    LRH["<b>Live-Robinhood trader</b><br/>trade.place · live<br/>pinned in-loop until signed provenance verifies ON"]
  end

  R -- "peer.publish · signal<br/>broker-stamped provenance" --> PA
  CS -- "peer.publish · trade-confirm<br/>admission ≠ belief" --> LRH

  classDef contract fill:#f9e2db,stroke:#d9563c,color:#28323f
  classDef trust fill:#f6e5d0,stroke:#c8863a,color:#28323f
  classDef tier fill:transparent,stroke:#b3c0d0,color:#5c6a7a
  class R,CS contract
  class PA,LRH trust
  class PROD,CONS tier
```

The two traders run one program; only provider and rung change. A signal is *data*: the trader
re-decides under its own polarity, so an approved peer is never a believed peer.

## Plate 03: grow by widening mechanism, never the boundary

Isolation is the default: identical stacks, one drain and one chain per agent, independent
blast radius. To spend fewer resources you widen a *mechanism* brick under the trust cells.
The shared brick still keeps each principal's chain separate inside it.

```mermaid
flowchart TB
  subgraph A["Agent A"]
    TA["<b>Principal · turn · envelope</b><br/>trust cells stay separate"]
  end
  subgraph B["Agent B"]
    TB2["<b>Principal · turn · envelope</b><br/>trust cells stay separate"]
  end

  subgraph MUX["Multiplexed drain · one wide mechanism brick · routes by principal"]
    CA["<b>Audit chain A</b><br/>still separate"]
    CB["<b>Audit chain B</b><br/>still separate"]
  end

  TA --- MUX
  TB2 --- MUX

  classDef trust fill:#f6e5d0,stroke:#c8863a,color:#28323f
  classDef mech fill:#dbeeec,stroke:#2f9c94,color:#28323f
  classDef tier fill:transparent,stroke:#b3c0d0,color:#5c6a7a
  class TA,TB2,CA,CB trust
  class MUX mech
  class A,B tier
```

The rule: some bricks can never widen (principal, turn/taint, trust map, audit chain).
Everything you gain by multiplexing, you gain under those cells, never through them.
Isolated-vs-multiplexed is a per-deployment risk call, not part of the agent contract.

---

The trust boundary is per-agent and never shared. The mechanism behind it is shared code plus
shared infra plus a small per-agent resource set, and how much of that set is truly per-agent
versus multiplexed is a deployment dial, not part of the agent contract.

*Source diagrams in the safe-agents repo: `docs/scaling-and-mesh.md` (topology),
`docs/canonical-consumer.md` (consumer anatomy).*
