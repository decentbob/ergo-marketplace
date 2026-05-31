# Decentralized Marketplace on Ergo: Design

*The technical specification. For motivation, the economic model, the go-to-market, and the honest reckoning, see [VISION.md](VISION.md).*

This document works outward from what the chain guarantees to what it can't. It builds on the framing in [VISION.md](VISION.md): a neutral settlement protocol with competing services stacked above it, split into **trustless trade** (both sides on-chain, settled atomically, basically a solved problem reused from existing Ergo infrastructure) and **mediated trade** (at least one side off-chain, where a per-trade mediator sits between match and settlement, and where almost all of the design's complexity lives). The three [core open problems](VISION.md#the-hard-parts) the trust layer can't close are laid out in the vision doc, and the sections here point back to them.

# Why Ergo

Ergo uses an eUTXO model and ships with a set of tools this design leans on:

- **Local state.** Every listing is its own box, evaluated on its own. No shared mutable state, no re-entrancy.
- **Parallelism.** Different items trade in the same block without conflicting.
- **Atomic trades.** A swap either completes in full or fails in full.
- **Covenants.** Boxes can enforce whatever output structure a contract needs (escrow constraints, fee outputs, multi-recipient splits), and the spender can't get around it for that specific box.
- **Sigma protocols.** Native zero-knowledge primitives that make sealed-bid auctions, ring signatures (for anonymous verified reviews), and other privacy-preserving proofs possible.
- **AVL+ trees.** A native authenticated data structure that lives in registers, with membership proofs verifiable on-chain. Useful for mediator acceptance sets, affiliate whitelists, and registry membership, where the set might be large but only one proof has to fit in a transaction.
- **Data inputs.** Read-only references to other boxes inside a transaction. They let a listing pull configuration (fees, shipping tables, oracle prices) from a separate Config Box without consuming it. This is the mechanism behind the sidecar pattern.
- **Storage rent.** Native on-chain garbage collection: boxes nobody touches for about 4 years can be reclaimed by miners, so abandoned listings don't bloat the state forever. The minimum ERG every box has to lock also puts a small capital cost on each listing.
- **Cheap.** A complex transaction runs well under 0.01 ERG.
- **Native oracle pools** for dynamic pricing. A listing can be priced against a live feed (say, a gram of gold sold at its dollar equivalent plus a margin) and settled at the current rate, with no oracle infrastructure to build.
- **NiPoPoWs** make trustless light clients practical (see [Light clients](#light-clients)).

# Architecture overview

The chain settles. Everything else sits above it in competing layers (front-ends, off-chain services, federated registries, bots, and the immutable core contracts), where teams can build without needing protocol-wide consensus. The [layer diagram and trust table](VISION.md#how-it-works-at-a-glance) are in the vision doc; this document specifies the layers themselves. The [risk register](#risk-register) is the failure-mode counterpart to that trust table.

One detail the diagram leaves out: buyer and seller exchange contact details over an off-chain channel (Signal, Telegram), bootstrapped by an on-chain handshake (see [Communication](#communication)).

---

# The on-chain layer: what the chain enforces

This layer provides strong cryptographic guarantees: settlement is atomic, listings can't be spent against their own terms, and escrowed funds can only leave through the paths their contract allows.

## Trustless trade

These contracts already exist on Ergo in production form. The protocol just adds them to the shared register interface so they live in the same ecosystem as the mediated contracts.

**Atomic swap.** The seller's box locks a token (or some quantity of fungible tokens) along with a spec for the asset they want. Anyone can spend the box as long as they send the wanted asset to the seller in the same transaction. Partial fills work through box self-replication: a sell box for 100 units re-creates itself as a 60-unit box when someone buys 40. This handles fungible token swaps, NFT sales, multi-unit fungible inventory, and most of the simple on-chain trade patterns.

*What the contract guarantees:* the swap settles in full on the agreed terms or doesn't happen at all. There's no partial settlement of a single unit, and a buyer can't pay less than the listed price, because the box can only be spent the way its script allows.

## Mediated escrow

Mediated trade for physical goods and services isn't a solved problem on Ergo today. This is where the design earns its keep.

**Flow.** A seller lists a physical item with metadata, a price, how that price gets distributed on a successful sale, an acceptable mediator (or set of them), and optionally a seller bond and a buyer bond. The buyer purchases by signing a transaction that consumes the listing box and creates an **Escrow Box** holding *price + optional seller bond + optional buyer bond*, naming one specific mediator from the seller's accepted set. The seller ships off-chain and claims shipment on-chain; the buyer confirms receipt, or the inspection window expires. How the escrow behaves across that lifecycle is governed by the state machine below.

**Price distribution.** The seller doesn't have to be the only recipient. A listing declares a set of `(address, share)` outputs: the seller, plus any front-end originator, affiliate, or drop-ship supplier, all of them parties whose payment depends on the sale completing. The buyer pays one total and the contract splits it on settlement (front-ends shouldn't bolt many recipients onto tiny sales, since sub-dust outputs fail). Recipients get paid **only to the extent the sale settles in the seller's favor**. A scammed buyer's refund is never trimmed to pay them, and on a dispute every recipient takes the same proportional haircut the seller does. Sale-contingent parties earn on a completed sale and not on a refunded one, so this is the right treatment. Any party whose payment should *survive* a buyer refund (a shipping carrier, a supplier doing independent work) doesn't belong in the pool at all; the seller pays them separately as a cost of doing business. The human-readable labels live off-chain in metadata, and the contract only ever sees addresses and shares.

### Two phases

The escrow has two normal phases, `Active` and `Claimed`, plus a `Disputed` state for when the two sides can't agree. The chain can't watch a package move. It works around that by tracking the one thing it *can* see, namely whether the seller has actively claimed shipment, and flipping the default accordingly. The escrow holds one of three states and a single reused `deadline` field whose meaning depends on which state it's in:

| State | `deadline` means | Silence favors | Who can act, and how |
|-------|------------------|----------------|----------------------|
| `Active` (pre-claim) | `claim_by` long-stop | Buyer | Buyer triggers timed refund after it passes; seller may claim or refund anytime |
| `Claimed` (post-claim) | inspection-window expiry | Seller | Seller or keeper triggers timed release after it passes; buyer may confirm or dispute anytime |
| `Disputed` | `mediation_deadline` | (mediator rules; timeout favors buyer) | Mediator decides; fallback fires on timeout |

- **`Active` (pre-claim).** The buyer has paid, and the seller hasn't made any on-chain claim of shipment yet. The buyer waits out a seller-set `claim_by` window, and once it passes they can trigger a **timed refund** that returns the price and both bonds. This is a *race* rather than a forced outcome: the seller can still claim or refund right up until the buyer actually reclaims. Nothing fires by itself.
- **`Claimed` (post-claim).** The seller has claimed shipment, which sets `deadline = claim_height + inspection_window`. The claim also states an expected shipping time, which is context and isn't enforced; the buyer should normally wait for that to pass before disputing non-arrival, though they're allowed to dispute earlier. The window has to be generous enough to cover shipping, inspection, and some unforeseen delay on top. The buyer can confirm receipt at any time to release the funds immediately; otherwise, once the window closes, the seller can trigger a **timed release**. The buyer's tool against a package that never arrives or arrives wrong is the *dispute*, which is available throughout this phase and pauses the clock.
- **`Disputed`.** A mediator decides. The clock gets replaced by a `mediation_deadline` (see [Mediators](#mediators)).

Two principles run the system, and they're mirror images of each other:

1. **Normal flow.** In each phase, whichever party took that phase's affirmative action can force settlement right away, and the other side's *silence* only resolves against them once that phase's timer runs out. The two phases mirror each other, refund versus release with the roles swapped, which reflects that the seller holds the leverage before shipping and the buyer holds it after.
2. **Breakdown flow.** When the mediation machinery itself fails (the mediator never rules and every named mediator is unavailable), the fallback resolves toward the **buyer**, who carried the loss-of-funds risk during shipping. This is the one place the normal-flow direction gets overridden.

Nothing in eUTXO happens automatically. Once the timer passes, the favorable direction simply becomes *spendable*. The **timed refund** can be triggered by the buyer (which fits the pre-claim race), and the **timed release** by the seller or by a keeper bot (see [Bots, solvers & keepers](#bots-solvers--keepers)), so the seller doesn't have to sit and watch the clock.

The buyer's side has no keeper and doesn't need one, because **filing a dispute pauses the timer.** The buyer's only liveness obligation is to *notice once and file once* inside a window measured in weeks. After that the clock runs on the mediator instead of the buyer, and the wallet just has to surface the timer and send reminders. That's a real regression against a custodial platform's "complain whenever you want from your phone," but a mild one, and it's the price of the funds never sitting in a platform's wallet in the first place.

**Tracking is how the seller stays accountable, and it's optional.** The claim transition carries an optional tracking commitment, and the recommended seller flow is to **claim at dispatch with tracking attached.** Tracking proves *something* was sent, but not *what* was sent, so it defends against the seller who never shipped, not the seller who shipped a rock. A claim with no tracking is still valid, but it's flagged as such to the buyer and counts for less in any dispute. A future carrier-API oracle could confirm delivery, which would allow a tighter, better-defined shipping and inspection window than a self-asserted claim does (see [Evidence & delivery verification](#evidence--delivery-verification)).

> *Failure mode, inattentiveness.* Since nothing fires automatically, a forgetful party can lose money. A seller who ships but forgets to claim before `claim_by` can get refunded by the buyer even though a real shipment is in transit. A buyer who never comes back to confirm or dispute lets the timed release pay out on a package that didn't arrive or arrived wrong. The mitigation lives at the wallet and front-end layer: actively remind both parties as each timer gets close to expiring.

### The three doors

Whatever phase it's in, the escrow can only ever release funds through three terminal paths. Up to three pools settle independently: the **price** (between the buyer and the seller-side recipients), the **seller bond** (returned to the seller or burned), and the **buyer bond** (returned to the buyer or burned). The mediator fee never comes out of these pools; it's paid in the buyer's purchase transaction or the filer's dispute transaction (see [Fees](#mediators)).

- **Success.** The buyer signs "received," or the inspection window expires in `Claimed`. The full price gets distributed across the seller-side recipients in their declared proportions, any seller bond returns to the seller, and any buyer bond returns to the buyer.
- **Refund.** The seller signs "refund" (available in `Active` or `Claimed`, not once a dispute is open), or the buyer triggers a timed refund after `claim_by` in `Active`. The full price goes back to the buyer, no seller-side recipient is paid, and both bonds return to their posters.
- **Dispute.** Either party flips the state to `Disputed`, the clock pauses, and the mediator decides each pool:
  - **Price split:** what share of the price the buyer gets back. The rest goes to the seller-side recipients in their declared proportions. The mediator moves only this one boundary; they can't shuffle money between seller-side recipients or zero one of them out. If the buyer is made whole, no seller-side recipient gets paid.
  - **Seller bond:** returned to the seller, or burned, depending on whether the seller acted in bad faith. It never flows to the buyer.
  - **Buyer bond:** returned to the buyer, or burned, depending on whether the dispute was filed in bad faith. It never flows to the seller.

So the mediator's allowed outputs are fixed: from the price pool, the buyer and the declared seller-side recipients (in fixed proportion); from each bond, its poster or the burn address. Nothing beyond that. *This bakes the prevention of outright theft into the contract: funds can never reach some arbitrary address the mediator picks.*

**Bonds: the reasoning.** Both bonds are deterrents rather than compensation, and each one is binary: it returns to its poster, or it burns on a bad-faith ruling. The seller bond discourages shipping nothing or shipping garbage. The buyer bond discourages a frivolous dispute, above all a condition complaint ("arrived damaged" or "not as described") backed by forged evidence, which is the one class of dispute that evidence can't cleanly settle. Each bond is the poster's only at-risk capital, so it has to be big enough that acting in bad faith is negative-EV against the named mediator (the sizing math for both sides is in [Appendix B](#appendix-b-escrow-economics)). A burned bond pays nobody, because paying the wronged party instead would hand every bond-less counterparty a positive return just for filing. Routing burned bonds to a public-goods wallet instead of destroying them still leaves a faint gravitational pull toward disputes resolving a particular way, so a clean burn is the more neutral deterrent. Bonds are **seller-set and optional**, never protocol-mandated, and sized to the trade.

**The mutual-close predicate** handles all the in-between cases. If both parties sign, the contract accepts any distribution at all: a partial refund, a deadline extension (shipping got delayed, a return window left open), forwarding to a next milestone, or recreating the escrow with amended parameters. Recreation can even *add* value: if the buyer co-signs an extra funding input, the recreated box can hold more than the consumed escrow did (a scope upgrade or a budget increase mid-job). Otherwise the recreation path is guarded so it can only produce another valid escrow box of the same contract or a terminal settlement, never some arbitrary output. This is the general form of a rule that governs the whole escrow: **any action that strictly favors the other party can be done by one side alone** (seller refund, buyer confirm-received, the two timed exits), while **anything that moves the terms against a party needs both signatures.** The dispute door only comes into play when the two sides can't agree.

```
                        SELLER
                          │  lists
                          ▼
                   ┌─────────────┐
                   │   Listing   │  (atomic-swap OR escrow-bound,
                   │     Box     │   per chosen contract)
                   └──────┬──────┘
                          │ spent by BUYER
              ┌───────────┴───────────┐
              ▼                       ▼
       (on-chain asset)        (physical / off-chain)
              │                       │
              ▼                       ▼
        atomic swap            ┌─────────────┐
        tokens move            │ Escrow Box  │  price + bonds + mediator ref
        directly               │  ACTIVE     │  silence → BUYER (timed refund
                               └──────┬──────┘  after claim_by)
                                      │ seller claims shipment (+tracking)
                                      ▼
                               ┌─────────────┐
                               │   CLAIMED   │  silence → SELLER (timed release
                               └──────┬──────┘  after inspection_window)
                                      │ either party may dispute (pauses clock)
              ┌───────────────────────┼───────────────────────┐
              ▼                       ▼                       ▼
            DOOR 1                  DOOR 2                  DOOR 3
           Success                 Refund                  Dispute
     (buyer-sign, or         (seller-sign, or        (mediator decides,
      inspection timeout)     buyer timed refund)     else timeout → buyer)
              │                       │                       │
              ▼                       ▼                       ▼
        price → seller-side     price → buyer           price: buyer vs
        s.bond → seller         both bonds return       seller-side (graduated)
        b.bond → buyer                                  each bond: return or burn
              │
              ▼  optionally
        Receipt NFT → buyer  (reviews, warranty, tax records)
```

A multi-stage state-machine pattern like this one already runs in **Bountiful** (StabilityNexus' bounty platform on Ergo): a creation → contribution → submission → judgment → withdrawal/refund flow, with judge multisig and a dispute period, all driven by State-NFT-guarded register transitions. It's worth studying as a reference implementation.

## Choosing parameters

The protocol hard-codes none of the numbers a trade depends on. Every one of them is a listing term: the seller chooses it, the buyer sees and accepts it at purchase, and a front-end suggests sensible defaults based on category, value, and the two parties' histories. Each number buys a protection at some cost, and the useful way to think about each is to ask what it defends against and what it charges for that defense.

| Parameter | Too short / small | Too long / large | How to size |
|-----------|-------------------|------------------|-------------|
| **`claim_by`** (pre-claim window) | Honest seller can't ship and claim before the buyer reclaims a real shipment | Buyer's funds sit recoverable-but-stuck longer when a seller vanishes | Seller's realistic dispatch time, with margin |
| **`inspection_window`** (post-claim window) | Buyer can't receive and inspect before timed release pays the seller | Seller's capital locked well past delivery | Transit + inspection, with margin; the buyer's dispute right runs the whole window, so it needn't predict exact arrival |
| **`listing_deadline`** | A still-valid offer gets cleaned up before it sells | Dead listings linger; seller's capital and rent buffer stay locked | By category; renewable. See [Listing lifecycle](#register-interface--the-contract-family) |
| **Seller bond** | Weakly deters shipping nothing or garbage | Locks seller capital; raises the bar to listing at all | Against the condition-dispute axis ([Appendix B](#appendix-b-escrow-economics)); a front-end can lower or waive it for a strong reputation or known identity |
| **Buyer bond** | Frivolous condition disputes become profitable | Deters honest buyers (it is real capital they put at risk to buy) | To the trade ([Appendix B](#appendix-b-escrow-economics)); recommended, never mandatory |
| **Mediator fee** | Mediator won't serve | Seller picks a cheaper acceptable mediator | Set by the mediator; the seller picks which mediators to accept, the buyer selects one at purchase; paid by the buyer, never from escrow (see [Fees](#mediators)) |

Both windows trade one party's headroom against the other's locked capital or time-to-exit, and both bonds trade deterrent strength against the poster's committed capital and willingness to trade at all.

**Sanity guardrails.** The contract enforces generous outer bounds on the economic parameters it reads. It rejects only the pathological cases (a zero `claim_by` that would refund before any seller could ship, a multi-year `inspection_window`) and never polices values that are merely suboptimal. The bounds are immutable and set wide enough that no honest trade ever hits them. A front-end should still recommend sane values inside them.

**Timers are block-denominated.** Ergo has no native wall clock, so every window is a count of blocks, and real block time drifts with hashrate. Size the windows by dividing the target wall-clock duration by a *conservative-slow* block-time assumption, so drift can only ever make the real window longer than intended, never shorter than transit. That puts the failure on the recoverable side ("seller's capital locked a little longer") instead of the unrecoverable side ("buyer paid before the package could possibly arrive"). The rule that the dispute right runs the whole window is the backstop if a seller still sets transit too tight.

> *Failure mode, buggy vs malicious interface.* The guardrails protect a buyer against a *buggy* front-end that skips validation. They do nothing against a *malicious* one that quietly routes the user to a different contract with the bounds stripped out, behind an identical-looking UI. The defense there is a wallet that checks the contract's ErgoTree hash against a known-good value before it signs.

## Listing options

On top of price and currency, a listing carries a handful of seller-set options the contract understands.

**Multi-unit.** *(Build, post-v0; not part of the v0 core.)* A seller with 50 identical units doesn't need 50 listing boxes. The Fixed-Price+Escrow contract self-replicates on a partial fill: when a buyer purchases 1 from a 50-unit listing, the box re-creates itself as a 49-unit listing and opens an independent Escrow Box for that single unit. Each sale is its own escrow, with its own mediator, deadlines, and dispute path, and nothing about one touches the others. The bond can be per-listing (one bond covering the batch, drawn down as sales happen) or per-unit (recommended for higher-value items). Per-unit drawdown across several concurrently live escrows spawned from one self-replicating box is a real design task the contract has to enforce, and it's fiddlier in eUTXO than the single-escrow case, so per-unit bonding sidesteps the problem.

**Bulk discount.** A tiered-pricing variant (Fixed-Price-Tiered+Escrow) reads a short list of `(min_quantity, price_per_unit)` tuples from R8 and works out the buyer's total from the matching tier. It's a different contract hash that shares the same register interface, so indexers handle it like any other variant. When a lot of listings share one schedule (a seller's whole catalog on the same breakpoints), the schedule lives in a Config Box and the listings reference it as a data input.

**Shipping cost** is usually baked into the listed price so the buyer sees a single number. For sellers juggling many shipping zones (per-region surcharges, customs), the listing references a shipping-rate Config Box, and the front-end resolves the buyer's region and shows the all-in price before purchase.

**Concurrency and contention.** A box is spent by exactly one transaction, so two buyers going after the same listing in one block can't both win. This is per-listing rather than global, though: separate listings settle in parallel, so there's no throughput ceiling. For fungible inventory it never bites, because a bot composes one transaction that settles both buyers and re-creates the remainder, or the losing buyer just resubmits against the re-created box one block later. Only *unique* single-unit listings race, and there a losing Ergo transaction never lands at all, so the cost is a wasted wallet round-trip rather than gas or locked funds. Front-ends grey the listing out once they see a pending spend in the mempool.

**Purchase allowlist.** The base listing carries an optional `allowed_buyers` field. Empty means anyone can buy; non-empty restricts the purchase to those addresses. It's one field with two main uses. It can **reserve** a listing for a specific buyer by setting it to their address, for a pre-agreed or OTC sale, or **pause** a listing without delisting it by setting it to the seller's own address or a burn address, so nobody can buy while the seller is briefly unable to ship. The seller edits it whenever they want by spending and re-creating their own listing box, with no timer and no separate state involved. Because the seller controls it, it's revocable and doesn't *guarantee* any buyer the item. A single buyer or a self-pause is stored inline; a large set (a community's members, say) is stored as an AVL+ root, with the buyer proving membership at purchase.

## Services & milestones

Services trade through the same mechanisms as physical goods, with two differences.

**Milestones.** A 5,000-unit coding contract paid in one lump is risky for both sides: the buyer holds all the leverage at the end, and the seller has done all the work upfront. Splitting it cuts that risk: 20% on spec approval, 30% on first delivery, 30% on second iteration, 20% on final acceptance. This needs no new mechanism. It's just the mutual-close predicate applied over and over: at each stage both sides sign a transaction that releases the tranche and re-creates the escrow holding the remainder, with the dispute door waiting as a fallback if they disagree. The one real choice is the funding cadence, which is a question of who carries the risk. Fund the whole job up front and the seller knows the money exists while the buyer eats the lockup; fund each stage as it begins and the buyer has no lockup while the seller carries the risk of an unfunded next stage. It's the same contract either way.

**Revisions and partial outcomes.** A designer delivers a logo and the buyer wants two color tweaks. There's no clean "shipped" event here. The mutual-close predicate does the work: at each iteration both parties jointly sign whatever state they want, extending the deadline, releasing part of the funds, moving to the next milestone, or refunding a portion. For the length of the job the escrow is effectively a state channel, settled on-chain only when something has to be enforced or the work is finished.

Mediators for services need domain expertise (was the spec ambiguous? does the code actually run?), so the realistic answer is category-specialized mediators.

## Register interface & the contract family

Listing-type boxes share a register convention so any compatible tool can read any listing. Here's the minimum useful subset:

| Reg | Tuple | Purpose |
|-----|-------|---------|
| R4 | Owner_PK | The seller. (For purely on-chain assets a bid is symmetric, a buyer's box is just a seller of currency, so the same field serves; physical listings are seller-only, see note.) |
| R5 | (Offered_TokenID, Offered_Amount, Optional_Min_Fill) | What the box locks. Min-fill prevents dust-sized partial fills. |
| R6 | (Wanted_TokenID, Wanted_Amount) | What the owner wants. For physical listings the wanted-token is the currency. |
| R7 | Metadata_Blob | A single `Coll[Byte]` the contract never reads: a leading `schema_version` byte followed by self-describing tag-length-value entries (title, short description, condition grade, category, GTIN/ISBN/identifier, brand, …). Readers skip unknown tags; a listing carries only the fields it has. The tag registry lives in the EIP, not the contract, so new fields and new categories need no contract or register-layout change. |
| R8 | Mode_Specific | For escrow listings: `claim_by` and `inspection_window` durations, seller-bond and buyer-bond requirements. For auctions: auction parameters, oracle references. Shipping zones where used. Depends on the contract. |
| R9 | (Contract_Extras, Price_Distribution, Optional_IPFS_Hash) | Mediator spec for escrow flows, encrypted payloads, and the price-distribution list of `(address, share)` pairs (empty or single-entry means seller takes all). The IPFS hash references extended metadata (images, long-form description, spec sheets, PDFs). |

That's the *listing* convention. After purchase the **escrow box** is a different box that reuses the same registers in its own way, per its contract:

| Reg | Listing box | Escrow box |
|-----|-------------|------------|
| R4 | Owner_PK (seller) | (seller_pk, buyer_pk) |
| R5 | (Offered_TokenID, Amount, Min_Fill) | (chosen_mediator_pk, fallback_mediator_pk) |
| R6 | (Wanted_TokenID, Wanted_Amount) | (state, deadline) |
| R7 | Metadata_Blob (TLV) | (currency_id, price, seller_bond, buyer_bond) |
| R8 | Mode-specific (timers, bonds, zones) | (inspection_window, price_distribution) |
| R9 | (Extras, Price_Distribution, IPFS_hash) | (listing_ref, ECIES_encrypted_buyer_handle) |

Since the one `deadline` field gets reused across states, a reader checks `state` to know how to read it: in `Active` it's the `claim_by` long-stop, in `Claimed` it's the inspection-window expiry, and in `Disputed` it's the mediation deadline. Full layouts are in [Appendix A](#appendix-a-full-transaction-examples).

Putting the searchable text on-chain in the R7 blob, instead of behind an IPFS hash, keeps indexers cheap and resilient. The storage rent on a few hundred extra bytes is negligible and well worth the indexing benefit. The contract treats R7 as opaque bytes; only indexers and front-ends parse it, following the EIP's tag registry, which should build on the established Schema.org Product vocabulary instead of reinventing one. The R9 hash is optional and meant for bulk content (images, long-form description, spec sheets, PDFs), and listings without any leave it empty.

Tooling that only displays or indexes a listing reads R4–R7 from any conforming box. R8–R9 are interpreted per contract; the contract is identified by its ErgoTree hash (the box's guard script), which is what the box's address derives from. That keeps the universal part small and stable while letting each contract use the remaining registers however it needs.

*Symmetry, and where it breaks down:* for purely on-chain assets a buyer's bid uses the same registers (currency in R5, desired token in R6/R7), and any seller can fill it on their own because the offer is fungible (an NFT works too, as long as any holder of the wanted asset can complete the trade). For physical items this falls apart, because no two seller offers are interchangeable: they differ in reputation, shipping, bond, and mediator. **So there's no RFQ contract for physicals.** A buyer's "wanted-box" couldn't be consumed unilaterally, since the buyer still has to sign to accept one specific offer, so the funds would just sit idle during review and the whole thing would amount to an off-chain forum post with extra capital locked up. Buyer-initiated demand for physicals belongs off-chain, on a "wanted" board or a chat post, until a specific seller turns it into a normal listing.

**Listing lifecycle and cleanup.** A listing carries a seller-set `listing_deadline` (R8). The seller can delist or renew at any time by spending the box back to themselves. Once the deadline passes, anyone can spend the box through a constrained **cleanup spend** that returns its full contents (the locked ERG and any bonded tokens) to the seller, minus a small capped keeper reward for whoever submits it. This is the "graceful expiry" pattern seen across Ergo order and auction contracts: the keeper bounty keeps cleanup reliable, and the seller's capital comes back rather than getting confiscated. Storage rent is the slow backstop underneath all this (a box nobody touches eventually becomes reclaimable by a miner), but the contract deadline is the fast, intended path.

One protocol fact this has to respect: **storage rent is paid only in ERG, and tokens can't pay it.** A listing or escrow box holding a *token* bond (a stablecoin, say) with only minimal ERG can, once the rent period is up, be consumed by a miner, which takes the bonded tokens along with it. So any token-bonded box needs an ERG buffer sized to its intended lifetime (a few ERG covers decades). Front-ends set this, and the cleanup spend returns it together with the bond. Boxes that hold ERG bonds protect themselves.

**Many contract versions, one interface.** A deployed contract's code is immutable, and each listing is its own box guarded by that contract's logic, with no EVM-style "deployment" event anywhere. A contract is identified by its ErgoTree hash, and two boxes with the same ErgoTree share an address. So the "protocol" is really a growing family of contracts identified by hash: an improved version gets a new hash and address, existing boxes keep working until they're spent, and tooling handles both as long as they share the register interface. Nothing is welded into a monolithic core, because there's no core to weld it into. The discipline that keeps this working: keep contracts narrow and well-tested, deploy new versions freely, and maintain the register interface across versions.

> *Failure mode, immutable bugs.* A bug in a deployed contract is permanent for anyone using it. Testnet stress-testing reduces the odds but can't eliminate them, and the only remedy is voluntary migration to a fixed version, which some users won't do. Old listings on a buggy contract can't be repaired after the fact.

**Relation to EIP-4, and publishing this as an EIP.** Ergo's asset standard (EIP-4) defines register conventions for token *issuance*. This listing layout is a parallel convention for tradeable *offers*, and it should be published as its own EIP so that wallets, indexers, and tooling treat it as a stable interop target, and so the standard outlives the original team. NFTs being traded still follow EIP-4. **EIP-24** (the artwork standard) provides royalty enforcement, collection grouping, and trait metadata for NFT listings, so adopt it rather than reinventing it.

A minimal starter set that's enough to launch: atomic swap with partial fill (reuse Ergo's existing versions), Fixed-Price+Escrow with the three-doors flow, and one auction (English via EIP-22, for instance). Everything else can come later without disturbing existing listings.

The family at a glance, with what each one reuses:

| Contract | Status | Purpose | Ergo prior art |
|----------|--------|---------|----------------|
| Atomic swap (+ partial fill) | reuse existing | On-chain token/NFT trade | Spectrum, NFT marketplaces |
| Fixed-Price+Escrow | build, v0 | Mediated physical/service trade | Bountiful state-machine pattern |
| Fixed-Price-Multi-unit+Escrow | build, post-v0 | Multi-unit inventory, one escrow per unit | shares escrow interface |
| Fixed-Price-Tiered+Escrow | proposed | Bulk-discount variant | shares escrow interface |
| English auction | reuse existing | Price discovery | EIP-22 |
| Dutch auction | proposed | Simpler price discovery | order-book patterns |
| Sealed-bid (Vickrey) | proposed | Private price discovery | Sigma commit-reveal |

They all share the register interface, so one indexer reads every one of them, and new hashes can be added to a registry at any time without disturbing live boxes.

### Auctions

Auctions are a price-discovery mechanism rather than a trade type of their own. An auction can settle a purely on-chain asset atomically, or it can set the price that then opens a mediated escrow for a physical good. Each style is its own contract at its own address. For an on-chain asset the win is enforced atomically; for an off-chain item the auction only discovers a price and opens an escrow, and nothing on-chain forces the seller to go through with it, so the seller's bond and reputation are the only pressure to honor the result.

- **English.** Bids replace each other, the previous bidder is refunded atomically, and the deadline extends on last-minute bids. **EIP-22** already specifies a complete contract (any-token currency, buy-it-now, auto-extension, minimum bid step, royalties), so adopt it.
- **Dutch.** The price drops linearly with block height, and the first to accept wins. Simpler than English.
- **Sealed-bid (Vickrey).** A two-phase commit-reveal via Sigma protocols, where the highest bidder pays the second-highest price. More complex, with a reveal phase and slashing for non-revealers, so deploy it once there's demand.

## Sidecars / config boxes

For shared, updatable state, a Config Box sits on-chain holding parameters, and listings reference it as a **data input** (read but not consumed). It's used for shipping-rate tables, tax routing, affiliate whitelists, oracle-fed price feeds, and master price lists.

- **Singleton Token / State NFT (anti-spoof).** The contract trusts only a config box that holds a specific NFT minted at deploy time; a fake config lacks the NFT and gets rejected. This is needed any time a contract reads from a config box. (Spectrum pools, oracle pools, and most multi-stage contracts use the pattern.)
- **Governance guardrail (optional cap).** A contract can also cap the values it accepts: `actual_rate = min(config_rate, hardcoded_max)`. Handy when the buyer can't directly verify a parameter that gets consumed later in a multi-step contract, and unnecessary in most cases because the front-end shows the executed price before signing.

> *Failure mode, oracle manipulation.* Wherever dynamic pricing or other oracle-fed values are in play, an attack on the oracle is an attack on the listing. Use the established Ergo oracle pools where possible; a new oracle needs its own security argument.

## Bots, solvers & keepers

The eUTXO model needs bots for two reasons. First, when several users want to spend the same UTXO (an AMM pool, a fungible-token sell box) they collide and only one transaction succeeds, so bots batch the competing intents into one valid transaction. Second, real-world matching is rarely 1:1 (a 1000-unit sell against ten 100-unit buys, three sellers for one large order, or a cross-DEX arbitrage that bridges this protocol's order with a Spectrum pool), so bots compose the multi-input transaction that makes those atomic.

What bots do: **concurrency batching** (combine intents against shared state), **multi-party matching** (stitch sell boxes, buy boxes, and external pools into one settlement, including non-monetary barter rings), **cross-DEX liquidity** (source fills from Spectrum and other AMMs and take the spread), **cleanup** (submit the cleanup spend on expired listings to claim the keeper reward), and **time-keeping** (trigger a timed release or timed refund once an escrow's deadline passes, and settle auctions at end-block). The chain has no autonomous clock on its own, so a keeper triggering the spendable-after-timer outcome is exactly what lets neither party have to babysit the trade.

**How bots get paid:** for matching and batching, a listing specifies a minimum acceptable output to the owner and the bot keeps whatever surplus is left (the bid/ask spread, or the fees saved by batching). Time-keeping and cleanup produce no surplus, so they're paid by a small keeper bounty baked into the box (claimable dust-ERG), or run for free by a marketplace as table stakes for its own listings. Competition drives margins down toward transaction fee plus overhead, and the edge is latency, completeness, and specialization, so no operator holds onto large rents while running a bot stays profitable even at modest volume. Reference implementations should be open source to keep the entry threshold low. **Spectrum Finance** is one production precedent: anyone can run an executor bot, off-chain matchers build the multi-input transactions, and execution fees spread across miners, UI providers, executors, and LPs, all of it permissionless.

---

# The trust layer: what the chain cannot enforce

Everything hard lives here. The chain can guarantee settlement, but it can't verify that a package arrived, that an item matched its description, that a mediator ruled honestly, or that a seller is who their reputation says they are. Those guarantees come from an off-chain, competitive, human-and-service layer, and that layer carries far more of the system's real-world trustworthiness than the contracts do. It's also, by design, the layer the protocol can't fix from the inside. The most it can do is supply the tools and make behavior legible. The real risk is that the *incentives* in this layer don't all point the right way, even with perfect contracts sitting underneath them.

Three problems here are unsolved, and for trade between strangers they may be unsolvable, so the sections below treat everything else as best-effort mitigation around them. They're laid out in full as the [core open problems](VISION.md#the-hard-parts) in the vision doc: off-chain delivery that can't be verified (made worse by AI-forged evidence), mediator trust that concentrates with misaligned incentives, and the shipping address that has to become plaintext for whoever ships.

## Mediators

**Selection: mutual handshake.** The seller's listing specifies a set of mediators it'll accept (a direct key, an AVL+ root with a buyer-supplied membership proof, or a small inline "any of these" set), and the buyer's purchase names one specific mediator from that set, so both parties have effectively consented. This is the structural defense against the **pocket mediator** attack, where the seller names a colluding mediator the buyer doesn't realize is colluding. The buyer's front-end can and should check the named mediator against reputation services before signing, and warn or refuse if it's unknown, fresh, or poorly rated. The protocol can't prevent collusion, but it can force the buyer to actively consent to a specific identity.

**Who files.** The dispute door is bilateral by contract: either signature can flip the escrow to `Disputed`. In goods trade the *seller* has little reason to file, because time already favors them (the timed release pays out on buyer silence), so they usually wind up in front of a mediator only because the *buyer* filed, whether for a real grievance or a frivolous one. Two cases still make seller-side filing necessary. First, services and milestones: there's no shipped object and no automatic timed release, so a buyer who simply won't sign off on delivered work leaves the funds sitting indefinitely, and the seller's only recourse is to escalate. Second, defensive filing: a buyer threatening a frivolous dispute or a retaliatory review to extort a partial refund can be pre-empted by the seller putting the matter to the mediator first, on the record.

**Evidence and contact.** The chain records only the ruling. The evidence (tracking, photos, unboxing video, the salted-hash reveals from [Evidence & delivery verification](#evidence--delivery-verification)) gets exchanged off-chain. To reach both parties the mediator uses the same encrypted-handshake mechanism the buyer and seller use at purchase, extended to the named mediator, so no party's plaintext contact ever touches the chain (see [Communication](#communication)).

**Implementations** (the protocol doesn't care who or what the mediator is):
- A single human or AI agent for low-value claims.
- A multisig customer-support team for marketplace-managed trades.
- A **curated mediator pool**: a service that vets applicants, admits them, and assigns a mediator at random for each dispute. It watches its members' track records and pays them per action or by revenue split, taking a small cut.
- A 3-of-N panel for high-value items, ruling by median outcome. ErgoScript's native `atLeast(k, Coll(...))` threshold sigma-proof handles the k-of-n signing directly and hides which k of the n actually signed, so panelists can rule without the on-chain record exposing them one by one.
- A Kleros-style decentralized court with random juror selection.

**Multi-tier appeal** (primary → panel → pool) composes above the core, so the escrow only needs to support one mediator plus one fallback. Putting more than that in the core would drag every trade through complexity most of them don't need. A diverse appeal tier, or equivalently an n-of-m panel drawn from distinct operators and reputation clusters, takes the edge off most single-mediator trust risk: a corrupt first ruling can be escalated, so collusion now has to capture *several* independent parties instead of one. The diversity is the part doing the work, since appealing to the colluder's friend buys nothing. It's cheap to build but costs more per dispute, so it's strongly recommended for high-value items. It only addresses the *independence* axis, though: a diverse panel staring at the same forged unboxing video gets fooled exactly as well as one mediator would. Mediator bonding is an implementation choice rather than a protocol feature in the same way: a pool can bond its members, scale the bond with volume, and slash it through a higher-tier panel on proven corruption.

**Operating practices** (not protocol-enforced, but a trust signal front-ends should surface or require): mediators should publish their rule set, their scope of work (which categories they handle and which they explicitly refuse), any jurisdictional or legal constraints, and whether they work only for specific front-ends or are available to any trade that names them.

**Fees.** The mediator fee is paid by the buyer and never taken from escrow, so neither the listing box nor the escrow box needs to know about it. There are two structures, and the mediator publishes which one they use while the seller accepts it at listing time:

| Structure | How it works | Strength | Weakness |
|-----------|--------------|----------|----------|
| **Standby fee** | Small fee on every trade, paid at purchase, whether or not a dispute happens | Mediator earns from the first clean trade; newcomers can bootstrap; no per-dispute reverse incentive | Buyer pays a little on trades that never dispute |
| **Initiator-pays** | Only the party who files a dispute pays, at filing | Nothing paid on the common no-dispute path | Mediator earns only on disputes, so a record is hard to build |

The mediator publishes their fee structure and size; the seller picks which mediators to accept, and the buyer selects one at purchase. Nobody bargains the number down, since it's the mediator's floor for serving at all. This is worth being clear about: it's a *different and lighter product* than centralized authentication. A standby mediator charging, say, 1% only adjudicates the disputed minority of trades, while a StockX-style 10% authenticates *every* item before it ships. The 1% isn't the same assurance at a discount. It's recourse after the fact rather than a guarantee before purchase, which makes it the right tool when the buyer doesn't need universal authentication and the wrong one when they do.

**Mediation deadline.** A dispute can't lock funds forever, since that would itself become an extortion lever ("settle off-chain or I'll just sit on it"). Filing sets `mediation_deadline = dispute_height + mediation_window`. If the mediator rules inside that window, their decision settles the escrow. If they don't, the dispute drops through to the next named mediator, and if none is left, to the **breakdown fallback**, which refunds the buyer as the party who carried the funds-at-risk while the machinery failed (see [Two phases](#two-phases)). The mediator can extend the deadline for genuinely complex cases that need more evidence, but only up to a hard ceiling, past which the fallback fires no matter what.

*Reverse-incentive note:* since a mediator timeout favors the buyer, a buyer could file hoping for an absent or slow mediator. Two things bound that: the initiator-pays fee, and where it exists the buyer bond (burnable if a fallback mediator or the eventual ruling finds bad faith). The window should also be long enough that any working mediator rules in time, so a timeout signals real mediator failure instead of a viable buyer strategy.

The soft spot: in the low-value used-goods case, the standby fee removes the per-dispute cost and the bond is often waived, so all that stands between the buyer and a timeout gamble is the window length and the mediator's reliability. That's an accepted trade for not double-charging capital on a cheap trade, and it isn't an argument against the standby default.

**Risks the protocol limits but can't kill off.** The most serious threats are on the mediator's side. The protocol blocks outright theft, because the output constraints are hard-coded and the mediator only chooses the price split and a binary call on each bond. Corrupt rulings *within* the power they're allowed are a different matter, and only reputation services and front-end curation can address those:

- **Collusion (one-off):** the mediator is secretly aligned with one party for this trade, whether it's a seller's pre-arranged "pocket" mediator or a buyer who controls the named mediator and disputes in order to rule for themselves. The scope is limited per trade, but for a long-running mediator it's a powerful exit, a chance to cash out years of trust in one coordinated raid. The mutual handshake is the defense, forcing both parties to consent to a specific identity the front-end can vet and refuse.
- **Sustained collusion (pattern):** the same pairing across many disputes, with outcomes that lean the same way every time. It can only be caught in aggregate, with reputation services watching the dispute graph, and it's hard to spot from any single trade.
- **Bribery during a dispute:** a long-running mediator has public reputation on the line, which raises the price of bribery without removing it. Open rule sets and published records make the outliers stand out.
- **Stolen mediator key:** the damage is capped, since the attacker can only redistribute between buyer and seller, never to an arbitrary address. The fallback mediator, the mediation deadline, and the inspection-window timed release all cap the exposure further.
- **Coordinated mediator unavailability:** if every chosen mediator goes offline (takedown, key loss, mass abandonment), or none of them rules before the mediation deadline, the trade drops to the breakdown fallback and refunds the buyer, who carried the loss-of-funds risk during shipping. The fallback can't favor both sides at once, and favoring the buyer matches the pre-claim default. *There's no clean fix here: it trades a stuck seller's payment for a buyer's recovery when the mediation layer fails.*

Detecting the collusion cases comes down to cluster analysis: shared funding sources, simultaneous activity, identical IPFS pinning. Curated pools and high-value panels (below) are the main structural mitigations. None of it is perfectly preventable, which is a basic limit of any mediated system.

**Economics and bootstrapping.** Under initiator-pays, a mediator only earns on disputes, yet their trustworthiness depends on a long record of well-resolved ones. A newcomer with no record can't attract the flow that would build one, so the market drifts toward a few powerful incumbents, which is concentration at the one layer with discretion over funds. The standby-fee default (above) is the first part of the answer, paying a mediator from the first clean trade instead of only on disputes. Curated pools are the second: a newcomer gets flow right away and builds a record without having to be individually trusted first, and random assignment from a diverse pool also blunts collusion, since neither party gets to pick the mediator. Diverse appeal tiers (above) sit on top for the highest-value trades. All of this tilts the structure toward contestability without eliminating concentration: a degraded mediator or pool can be abandoned by new or updated offers, though trades already in flight don't get moved retroactively.

## MAD double-deposit escrow (rejected)

The obvious question, after all that mediator machinery, is whether the mediator can be removed entirely. The best-known attempt is **MAD** (mutually assured destruction): both buyer and seller post collateral, and on a dispute both deposits and the escrowed price burn, with no mediator involved. The idea is to make scamming negative-EV. It's worth working through because it sounds clever and the question keeps coming back, and it was used historically by BitBay and Particl.

**The real advantage** is that MAD needs no third party, and no trade details, identities, or dispute facts shared with anyone outside the two parties. For trades where both sides want maximum anonymity and minimum disclosure, that's a real benefit, and it's why people keep trying MAD.

**The structural problem** is that the honest outcome isn't a stable equilibrium. The game is sequential (ship → accept-or-not → refund-or-not), and backward induction turns up two symmetric traps: after shipment the buyer can keep the item and stay silent, and before shipment the seller can simply decline to ship. In each case the wronged party's own MAD threat isn't credible, so they rationally swallow the loss instead of firing it. That leaves security resting on at least one side acting against their own financial interest, which is psychology rather than logic. The full payoff matrix and the scam-profitability threshold are worked through in [Appendix B](#appendix-b-escrow-economics).

Three things follow, and together they make MAD both weaker and more capital-hungry than mediated escrow. First, **MAD has no recovery path**: the deposit *is* the whole mechanism, so a wronged honest party just eats the loss, where mediated escrow can adjudicate the disputed funds back to them. Second, and because of that, **MAD needs larger bonds to deter at all.** With no recovery, the deposit alone has to make a scam negative-EV, which (per the threshold math) takes a deposit on the order of the item's full value; a mediated bond only has to tip the EV given that adjudication already claws most of the value back, so it can be a fraction of the price. So MAD locks up more capital to buy less protection. Third, **MAD destroys value with positive probability even between non-malicious parties**, because its honest equilibrium is unstable, where escrow bonds come back almost every time and burn only on an adjudicated bad-faith ruling, never on a plain coordination failure.

**Where it might still fit:** very low-value trades where a mediator fee would cost more than the item; repeated trades inside communities, where the reputation cost amplifies the deposit's deterrent; or as a configurable fallback for when chosen mediators go silent. Anyone deploying a MAD-style contract should say plainly that its security is psychological rather than mathematical. The protocol doesn't stop such a contract from being deployed and registered. It just shouldn't be the default for trade between strangers.

## Evidence & delivery verification

The mediator is the bridge between on-chain commitment and off-chain reality, and the quality of their rulings can't exceed the quality of the evidence they get. This is the biggest unresolved threat to mediated physical-goods trade, and it's getting worse over time.

**Anchoring facts at purchase time** is one possible defense, and the friction is usually only worth it on high-value trades. The shipping address, tracking number, and agreed conditions get salted and hashed on-chain, with the salt shared off-chain. In a dispute the aggrieved party reveals only the salts for the facts at issue, and the mediator checks that the hash matches. The chain then holds a tamper-proof commitment to what was agreed, which defends against *forging agreed-upon facts after the fact*. For a casual low-value buy it's overkill and can be skipped.

**It does nothing against forged evidence about what happened after purchase.** AI keeps making believable fake photos, delivery videos, tracking screenshots, and unboxing footage cheaper and harder to tell apart every year. Hash-anchoring can't touch that. Mediators will increasingly need harder-to-fake evidence chains (live unboxing video calls, tamper-evident shipping seals, third-party delivery confirmation through carrier APIs like USPS Web Tools, FedEx, or DHL, and eventually IoT-tagged packages with continuous custody), or else accept that a dispute comes down to whose story is more plausible.

For a *disputed* trade the mediator already acts as the delivery attester, querying the carrier against the revealed tracking number as part of the ruling, so no standing oracle is needed at launch. A general carrier-delivery oracle is possible future work, but its only gain is narrow: gating the timed release on confirmed delivery, so silence can't pay a seller for a package that never arrived. That protects the *absent* buyer against non-delivery and does nothing for the condition dispute, which is the hard one (see the [Risk register](#risk-register) for the residual it leaves open).

> *Failure mode, information asymmetry on used and custom items.* A buyer can't fully inspect before buying, and even verified-seller reviews say nothing about what's wrong with *this particular* unit. Category-specialist mediators and standardized condition grading close part of the gap. Perfect information isn't possible.

## Receipts: the proof-of-purchase primitive

The settlement transaction can optionally mint a small NFT to the buyer (a Receipt NFT, around 0.001 ERG). The protocol does nothing else with it; its value is whatever other systems decide to do with it. It answers one question: **how does a future contract, service, or person verify that this wallet really bought this item?**

Using the issuer-box pattern, minting the receipt in the same transaction that closes a specific escrow ties the receipt to *that trade*, and through it back to the originating listing. So the receipt says more than "buyer of this seller." It says "buyer in *this specific trade*," and it says it cryptographically.

Downstream uses: **reviews** limited to receipt-holders (see [Reviews](#reviews)); **warranties and returns**, which in practice are *policy promises* by the seller, since locking seller funds for a multi-month warranty window would tie up too much capital, where the buyer presents the receipt through the seller's channel or a mediated process and reputation enforces compliance; and **tax, accounting, and expense documentation**, a permanent timestamped signed record that front-ends can render as a printable invoice. Receipts are opt-in at purchase. They're worth it for valuable trades and after-sale relationships and overkill for cheap fungibles, so front-ends can default the toggle by category.

## Reputation

The protocol doesn't compute reputation. It just emits the raw record. Reputation here is mainly about the two roles running ongoing businesses, sellers and mediators. Every settled escrow is permanent, so lifetime volume, dispute rate, average resolution time, and fees paid can all be read off the chain. Reputation services are what turn that history into metrics.

Reputation is a competitive layer. In practice the natural shape is **reputation aggregators**: a few services combine on-chain history, off-chain attestations, and KYC tier into composite scores, and front-ends consume one or more of them. Users rarely curate their own trust lists. They pick a front-end and inherit its choices, which makes the front-end the accountable unit, since it aggregates several services plus its own metric and, unlike a pure rater, has users and a business to lose when trust breaks.

Reputation is only the first filter. The economic backstops (bond, escrow, capped mediator power) are the per-trade protection that limits the damage when reputation fails, so a reputation miss means a few more bad trades slip through rather than catastrophic loss, as long as reputation doesn't fully waive the bond on high-value trades (see *Bond sizing*).

The layer stays healthy as long as front-ends and users are the ones paying for *accuracy*, because a rater is only valuable when it predicts well. The corrupt case is sellers paying for their own scores, which costs the service its trust. The on-chain record is already public, so what services actually sell is the *analysis* (entity-linking, labels, freshness, coverage, API) rather than the data. Services should also accept *user flags*, but a flag is a candidate for human review and not a verdict: the flagger usually has no proof and may just have lost a dispute. So the real work is being done by "weighted by source and manually reviewed for serious cases," and that brings back a trusted reviewer rather than establishing any ground truth.

The deeper limit is that the chain records only outcomes, never whether a ruling was actually justified by the evidence, so a mediator who quietly tilts things within plausible bounds is nearly undetectable. Two residuals fall out of that: a dominant front-end with a bad metric becomes the weak link, and a patient scammer can build up clean history and then cash out. Wash trading lives here too; services weight by fees paid, account age, unique-counterparty count, and funding-cluster analysis, but perfect detection isn't possible.

## Identity

Reputation lives at the address level by default, which makes key rotation painful, since a compromised key means starting over with a fresh identity. For sellers and mediators, both plausibly running ongoing businesses, an **Identity NFT** fixes this: mint it once, reference the NFT instead of a raw key, and treat the current holder as the signing authority. Reputation then accumulates on the NFT, rotation becomes a delegation update, and a compromised key can be swapped out before an attacker misuses the identity.

A worse variant is theft. If one key both signs and re-delegates, a stolen key lets the thief re-point the Identity NFT to themselves and capture the *entire* accumulated reputation. The seller case is the dangerous one, since the thief can immediately list scams under the stolen standing. The fix for high-value identities is to split the keys: a **hot signing key** for trades and rulings, and a separate **cold delegation key** that alone can re-point the NFT. Stealing the hot key then limits the damage to individual adjudicable trades and can't capture the identity itself. It's recommended for high-value sellers and mediators rather than mandated, so a casual seller's setup stays simple.

> *Failure mode, selling the identity as an exit.* Free transferability means a reputable identity can be sold to a scammer planning one big extraction, and the protocol has no way to treat ownership changes as meaningful. The defense sits entirely at the reputation layer: services drop trust after a long-running identity changes hands until the new holder builds history; front-ends show ownership history next to the metric; and high-stakes interactions can require an off-chain identity check that doesn't move with the NFT.

## Discovery: federated registries

A registry box is a singleton holding a list of valid contract hashes, basically a phone book for the protocol. Bots and front-ends subscribe to whichever registries they trust and scan those addresses for listings.

**Anyone can publish a registry.** A conservative one lists only audited contracts, an experimental one lists betas, a niche one lists category-specific contracts, and a community one lists whatever gets submitted. That pushes the censorship decision out to the edges. Upgrading a contract just means deploying the new version and adding its hash to a registry; subscribers pick it up, and old listings stay on the old contract.

> *Failure mode, spam.* Curation filters what a front-end *displays*, but spam boxes still exist and still burden every indexer that has to ingest and exclude them. The per-box minimum ERG is the creation deterrent, though it's small; listing-deadline cleanup and storage rent then clear the boxes out over time. A one-time flood imposes a parse-and-filter cost until cleanup catches up, and marketplaces add reputation filters and minimum-bond-to-list rules on top (their policy, not the protocol's). Spam is made costly to create and finite in lifetime, not free to ignore.

## Reviews

Reviews are how buyers filter for quality and service, how they decide what to buy and not only whom to trust. Receipt-NFT-backed reviews block the cheapest and most common fraud, which is review-bombing or review-boosting by people who never bought anything. They don't stop a seller manufacturing positive reviews through low-cost circular purchases, but that takes real on-chain trades and leaves patterns reputation services can pick up. Verified-buyer reviews don't make reviews honest. What they do is make dishonest reviews easier to spot.

**Anonymous verified reviews.** A buyer should be able to prove "I hold a receipt from this seller" without revealing *which* receipt, which makes honest negative reviews possible without the risk of retaliation. A ring signature over the seller's receipt-NFT set, using Ergo's native Sigma protocols, does this today, and larger anonymity sets and richer predicates open up as the privacy stack matures.

Multiple review systems run in parallel. Each front-end picks which ones to display or builds its own, and different weighting schemes (recency, value-weighted, sentiment) will show up.

**Griefing and extortion.** A griefing buyer can't extract anything by waiting, since post-claim silence pays the seller. They have to either dispute, where the buyer bond bites, or threaten a credible negative review. The bond puts a price on the first. Review threats get blunted by annotating reviews with dispute outcomes and seller responses, and they shrink as a seller builds up history. Review-extortion is bounded rather than closed, and it bites new sellers the hardest.

## The trust layer in summary

The chain's guarantees stop at settlement. Everything in this layer depends on off-chain reality, humans with discretion, or market dynamics, so it's best-effort by nature, and the three [core open problems](VISION.md#the-hard-parts) never fully close. The generous-protocol, ruthless-application stance that follows from all this is laid out in [VISION.md](VISION.md), and the incentive gaps in between are real design work this project owns rather than someone else's problem. The [honest accounting](#honest-accounting) section adds up where that leaves the design overall.

---

# Connective systems

The plumbing that links on-chain settlement to the trust layer and to the wider world.

## Currencies

The protocol supports any currency a token can represent. The notable options: **ERG**; **USE** and **SigUSD** (Ergo-native stablecoins, and the better default for listings); **bridged stablecoins** (via Rosen Bridge); and **bridged crypto** (rsETH, rsADA, rsBTC). A seller lists a single offer in their preferred currency and a buyer can pay in another, because bots route the buyer's currency through DEX pools into the seller's receive currency in one transaction, so there's no need for duplicate listings. Routing carries pool-rate risk, so the buyer should see a max-slippage bound before signing.

**Babel fees (EIP-31)** let users pay transaction fees in any token they already hold. Providers supply the ERG at a small spread, so a buyer holding only USE doesn't have to go acquire ERG just to make a purchase. CyberVerse is one production reference, where players pay for everything in CYPX and touch zero ERG.

**ChainCash** is a note-based monetary protocol on Ergo. Notes can be issued on trust, fully reserve-backed, or anywhere in between, and they circulate off-chain through a signature custody chain, with on-chain redemption against any prior signer's reserve. For the marketplace, a community can use its own notes as the listing currency and only settle on-chain at redemption. The payoff is the liquidity of mutual credit, which is useful for B2B trade where counterparties extend each other credit instead of pre-funding every transaction.

> *Failure mode, bridge risk.* Bridged assets carry the security assumptions of the bridge underneath them, so an exploit puts those tokens at risk. The protocol itself is unaffected; the holders of the bridged tokens are the ones exposed.

## Communication

For most trades, the established external apps are the right call (Signal, Telegram) for the rich media, threading, push, and group chat that on-chain messaging can't match right now. The protocol just provides the handshake that bootstraps any such channel:

- **ECIES handshake.** At purchase the buyer encrypts their contact handle to the seller's public key and writes the ciphertext into the escrow box's R9, and the seller's wallet decrypts it and starts contact. There's no need for the reverse direction: a buyer who wants to reach the seller first can use the public listing channel, and the seller verifies the buyer's identity once contact is made.
- **Identity verification (EIP-28, `ergo.auth()`).** A nonce-based sign-message flow (supported by Nautilus) confirms that the chat counterparty owns the on-chain address, which closes the impersonation gap that plagues OTC chats.

This isn't only buyer↔seller. The same handshake extends to a mediator during a dispute (see [Mediators](#mediators)).

**On-chain encrypted communication** is also possible, since both wallets are keypairs anyway, and it brings properties chat apps don't have (tamper-evident timestamps, censorship resistance, no third-party server), at the cost of being slower. **Ephemeral Messenger** (qx) is a working reference: on-chain encrypted messages with a configurable lifetime, where any third party can burn an expired message-box for the locked ERG. Off-chain apps are the right call for now and for most trade, but on-chain messaging is plausibly the cleaner long-term integration, since the same key that trades can carry the conversation with no third-party server in the path. It's worth standardizing wallet support for at least one channel so that "buy" → "open chat" is a single click.

## Privacy

Privacy in physical trade is lopsided: sellers can stay pseudonymous, while buyers leak more, because a package has to arrive somewhere.

**Default practices.** Wallets should generate a fresh derived address for each purchase by default, which is the cheapest defense against cross-trade linkability. Sensitive on-chain data is salted and hashed, with the salt shared only off-chain, and disclosure to a mediator is selective. Receipts can be kept in a wallet separate from the one funding purchases.

**Ergo-native primitives that are live today.** **Stealth addresses** let a recipient receive funds without their public address showing up on chain. **ErgoMixer** (non-interactive Sigma-protocol coin mixing) handles high-sensitivity payouts. Stronger ZK tools are coming but aren't shipping at the application layer yet; see [Future surface](#future-surface).

**The shipping address.** This is the third [core open problem](VISION.md#the-hard-parts). Physical delivery needs the buyer's real-world address to reach whoever ships, so it can't be made private, and the exposure is a safety risk that goes beyond a privacy leak. The single most important mitigation is **telling the buyer about the risk before they share an address.**

For most commercial trade the risk is small, but it bites in non-commercial P2P trade and with the occasional bad actor. The clean mitigation is **home-address-neutral delivery**: parcel lockers, PO boxes, poste restante, or mail-forwarding services, where the seller sees a pickup point and the party being trusted is a neutral commercial operator instead of the counterparty. It isn't always feasible (availability, pickup friction), but where it is, it's a real solution.

> *Failure mode, privacy fingerprinting.* Chain analysis can de-anonymize wallets over time by accumulating patterns. For on-chain trade and seller-side payment, stealth addresses, ErgoMixer, fresh derived addresses, and the emerging ZK primitives close most of the gap. The shipping-address leak above is the one exception that can't be closed for physical trade.

## Composability with other protocols

Composability is one of the structural advantages of building on eUTXO instead of an account-based chain, and here it's close to free. A listing or escrow is just a UTXO with typed registers, so any other on-chain application can read it as a **data input** (referenced without being consumed) or spend it in a shared transaction, with no permission, no protocol-level hook, and no integration the marketplace has to ship for it. On an account-based chain the equivalent state is global and mutable, so a third party composing on top has to reason about contention and call into the marketplace's own contracts. Here a box is local and immutable once created, so reading or referencing it can't interfere with anything else in flight.

That turns the marketplace's public record into something other protocols can build on directly: lending against on-chain price history used as an oracle, trustless options on lockable on-chain assets, crowd-assurance or threshold orders for pre-orders and group buys, prediction markets hedging on delivery, streaming-payment contracts for subscriptions, insurance pricing risk off observed dispute rates. More broadly, any third-party financial service (lending, hedging, factoring, escrow insurance) can attach itself to live trades without the protocol ever knowing it exists. None of this is on the critical path. It's upside the local-box model gives for free, and one reason the neutral base is worth more than the sum of its contracts.

---

# The application layer

Explicitly not part of the core. The protocol imposes none of it, and different front-ends make different choices.

## Front-ends & curation

**What a front-end provides:** wallet connection; browse and search through an indexer; item pages with images, specs, and history; a sell flow (barcode scan → API auto-fill → mint listing); a buy flow with escrow-status visualization; a dispute UI; order history and receipt rendering; receipt-backed reviews; reputation visualization pulled from one or more services; and comparable-listings views with price history for items that have stable identifiers.

**Curation** is where the same on-chain listings get presented in different ways. A premium front-end shows only verified sellers and runs an in-house mediator team; a general one shows most listings with warning labels on the unverified ones; a raw one shows everything from every registry; a niche one shows a single category; a local one shows local-pickup listings in one city. Each defines "verified" however it likes, and none of them has any privileged protocol access. A front-end should label a low-bond or no-bond listing as such, so the buyer can weigh a lightly-bonded listing from an established seller against a well-bonded one from a fresh account, which is the real choice in front of them.

> *Failure modes, application layer.* **Curation monopoly:** network effects push users toward whichever front-end has the best curation and the largest selection, and the protocol preserves the *option* of competing front-ends but can't stop one from dominating. **Front-end manipulation:** a dominant front-end can bias search toward sellers who pay for placement, or bury listings that make it look bad; only competing front-ends correct that, and only if users bother to switch. The claim here is conditional. Because listings and reputation are portable shared record, the capture risk is *lower* than on a traditional platform, but only as long as the wallet (below) holds transaction integrity. If users sign whatever a web app hands them, a captured front-end is just as dangerous here as anywhere else. With the wallet doing its job, capture shrinks from *what the user signs* (wallet-guaranteed) down to *which listings they see* (discovery bias), and at that point the cheap switch to a competitor showing the same listings is the real check.

## Wallet: the trusted computing base

Trust should sit in the wallet rather than the front-end, and most of the front-end-as-adversary surface closes once the wallet does its job. It's the single most important non-contract component in the system. A supporting wallet should:

- **Show the real box, not the front-end's version of it.** Parse the transaction about to be signed and display the true price, bonds, timers, named mediator, and recipients read straight from the box, so a front-end can't show one thing and build another.
- **Verify the contract.** Check the ErgoTree hash against a known-good registry before signing, and warn on a mismatch or an unrecognized contract.
- **Encrypt the shipping address locally.** Do the ECIES encryption of the address to the seller's key inside the wallet, so the front-end never touches plaintext. That's the difference between a front-end that *sees* every address and one that never sees a single one.
- **Check the named mediator** against reputation services before signing, and warn if it's unknown, fresh, or poorly rated.
- **Surface escrow state and timers.** Show open escrows with their state (`Active` / `Claimed` / `Disputed`), the time left on the current timer, the counterparty, and the mediator; decrypt incoming shipping handles; sign confirm-received, shipping-claim (with optional tracking), refund, and dispute transactions; and remind both parties as each timer gets close to expiring.
- **Handle the currency plumbing.** Show multi-currency balances, generate a fresh derived address per purchase by default, and handle babel fees behind the scenes.

## Monetization

There's no protocol fee and no governance token. The value gets captured at higher layers instead.

**Why no fee survives bare matching.** In eUTXO anyone can deploy an alternative contract at a different address. A fee hardcoded into a listing contract is enforceable *for that listing*, but a competing fee-less contract can be deployed for free, and sellers minting fresh listings will prefer it. The same logic hits front-end-collected fees: the moment a competing front-end shows the same listings with a smaller surcharge, buyers move. The end state is **fees pushed down toward the marginal cost of running the front-end and indexer infrastructure.** Past that point, value has to come from bundled services with real cost or scarcity:

- **Verification services:** KYC the sellers and issue badges; sellers pay a subscription for exposure, and buyers trust the badges.
- **Mediation:** per-dispute or standby fees.
- **Indexer and API access:** a fast, well-curated indexer is real infrastructure others will subscribe to.
- **Reputation services:** the on-chain record is free, so the product is the analysis (entity-linking, labels, freshness, API); front-ends subscribe, and the big ones build it in-house.
- **Premium merchant tools:** tax export, inventory management, shipping labels, advertising, or a full Shopify-like storefront service; SaaS for power sellers.
- **Specialized appraisal and authentication** for high-value categories.
- **White-label widgets and embeds** licensed to niche communities.

The **price-distribution mechanism** ([defined above](#mediated-escrow)) is one way these get *collected*: an "originator fee" is just the front-end that minted the listing taking a declared share at settlement. It creates no moat, since a fee-less competitor can always be deployed, so the share is best understood as collection *for bundled services the seller actually wants* rather than rent on bare matching, and a flat subscription with no fee at all works just as well.

A free protocol doesn't mean a free *quality* marketplace. Curation (vetting, support, moderation) is funded by the same open-base, unforkable-service logic that funds reputation: a fee-less fork gets the code but not the live vetting pipeline, the brand, the support org, or the domain expertise, and revenue scales with the volume good curation pulls in, so a cheap, thin front-end stays a niche player rather than a market-killer.

**Funding the commons.** Value capture funds the *businesses*. How the shared base beneath them (audited contracts, the EIP, an open indexer, wallet support) gets funded is a positioning question covered in [VISION.md](VISION.md#the-economic-model). One build-time note belongs here: wallet support should ride on an existing wallet rather than fund a brand-new one, which folds the one genuinely ongoing cost into infrastructure that already exists.

## Light clients

The architecture assumes most users go through a front-end talking to an indexer, which is the pragmatic path: fast, familiar, browser-friendly. It also concentrates trust, though: if the indexer lies or the front-end gets taken down, users with no alternative are stuck. (This is a later-phase upgrade rather than part of the core, but the protocol should emit clean events early so it stays possible.)

**NiPoPoWs** (Non-Interactive Proofs of Proof-of-Work) make a stronger model possible: a client small enough to run on a phone can cryptographically verify any claim about the chain without downloading it and without trusting a server. Once the protocol emits clean, standardized events from key transactions (listing created, escrow opened, escrow resolved), such a client can verify on-device that a listing exists at a given price, that a seller has completed N escrows with M disputes, or that an escrow is currently in dispute. The UI needs no trusted backend, and if every centralized front-end goes dark overnight, a light-client app can still read listings, build trades, and verify status. Takedown resistance then rests on the chain and the client's own verification instead of any operator staying online.

The trade-off is UX. Light clients are slower than indexer-backed front-ends and can only verify what's on-chain, so they can't show off-chain images, reputation scores, or curated lists without trusting some source. A reasonable design lets the user choose which off-chain services to trust (an indexer for search, a reputation service for ranking, a mediator directory for vetting) while all the transaction logic stays verified on-device. **Citadel** (arkadia) and **TrufflΣ** (Flying Pig) are working precedents: desktop clients that talk directly to a local Ergo node, with no third-party services in the path.

## Future surface

Speculative or research-stage items, listed here so the design's edges are visible. None of them is load-bearing, and none is a launch dependency.

- **Agent economies and on-chain drop-shipping.** Agents with wallets can act as first-class traders: a drop-shipping agent lists items it doesn't hold, takes orders, and triggers upstream purchases, with multi-stage escrow chains handling its margin and keeping it from getting stuck mid-chain. The rails exist; the demand doesn't yet.
- **Advanced ZK privacy.** **Curve Trees** (feasible through Ergo 6.0's `UnsignedBigInt`) and **EIP-0045** (proposed native STARK verification, trustless-setup and post-quantum) would open up larger anonymity sets and arbitrary ZK statements. Worth tracking, though neither ships at the application layer yet.
- **Carrier-delivery oracle.** A signed delivery-fact service that would let the release gate on confirmed delivery without a dispute. Narrow unique value and a real new trust surface; see [Evidence & delivery verification](#evidence--delivery-verification).

---

# Honest accounting

The failure modes are documented inline next to the designs they threaten (look for the *failure mode* notes throughout). This section is the cross-referenced register, and the broader context (related work, externalities, legal) is in [VISION.md](VISION.md#context-and-tradeoffs).

## Risk register

**Prevented by the protocol** (cryptographically enforced):
- Outright theft by the mediator → [Mediated escrow](#mediated-escrow) (output constraints hard-coded).
- Atomic-swap failure modes and listing tampering → [Trustless trade](#trustless-trade).

**Limited but not eliminated** (handled by reputation services plus front-end curation):
- Mediator corruption within allowed power; pocket or colluding mediator; double role; sustained collusion; bribery; stolen key; coordinated unavailability → [Mediators](#mediators).
- Mediator consolidation and the bootstrapping barrier → [Mediators](#mediators) (curated pools, high-value panels).
- Wash trading; selling an identity to exit; reputation failure (a dominant front-end with a bad metric, or a patient long-game scammer) → [Reputation](#reputation), [Identity](#identity).
- Buyer-side griefing: dispute-extortion → [Mediated escrow](#mediated-escrow) (buyer bond); review-extortion → [Reviews](#reviews) (bounded, not closed).
- Seller claims shipment with fake tracking or none (an existence-dispute the buyer has to catch and contest inside the inspection window) → [Mediated escrow](#mediated-escrow) (mediator backstop plus seller bond; the tracking commitment makes the claim accountable).
- Absent buyer plus non-delivery plus a faked shipment claim: bounded by tracking-flagging and the buyer's dispute right, but a buyer who's away the whole window is unprotected, and it's only fully closed by a future delivery oracle → [Evidence & delivery verification](#evidence--delivery-verification), [Future surface](#future-surface).
- Ship-claim gap: a buyer reclaims in the window between the seller getting tracking and submitting the claim, walking off with an in-transit item → [Mediated escrow](#mediated-escrow).
- Mediator holds a dispute hostage → bounded by the mediation deadline and capped extensions, with a buyer-favorable fallback on timeout → [Mediators](#mediators).
- Spam listings → [Discovery](#discovery-federated-registries).
- Hot-item contention on unique listings (a bounded UX cost, not a scaling flaw) → [Listing options](#listing-options) (concurrency and contention); auctions for contested items.
- Privacy fingerprinting → [Privacy](#privacy).

**Fundamental, must be designed around at higher layers:**
- The three [core open problems](VISION.md#the-hard-parts): off-chain delivery that can't be verified, with AI making evidence forgery worse; mediator trust concentration and incentive misalignment; and shipping-address exposure to a pseudonymous counterparty.
- Immutable contracts mean immutable bugs → [contract family](#register-interface--the-contract-family).
- Block-denominated timers drift with block time → sized against worst-case-slow blocks so drift only ever lengthens the real window → [Choosing parameters](#choosing-parameters).
- IPFS pinning isn't guaranteed → it degrades gracefully, since searchable text and core metadata stay on-chain in the R7 blob, and marketplaces re-pin the content they care about.
- Bridge risk and oracle manipulation → [Currencies](#currencies), [Sidecars](#sidecars--config-boxes).
- Curation-layer monopoly and front-end manipulation → [Front-ends & curation](#front-ends--curation), bounded by the [wallet](#wallet-the-trusted-computing-base) holding transaction integrity.
- Information asymmetry on used or custom items → [Evidence & delivery verification](#evidence--delivery-verification).
- Curation funding under fee competition, and funding the one-time commons → [Monetization](#monetization).
- Mediator legal exposure (arbitration, escrow-agent, or money-transmitter characterization; it scales with formality and volume, and it sorts the highest-stakes trades toward the least accountable mediators) → [Legal](VISION.md#legal).

The pattern across the whole register: a hard floor of cryptographic guarantees, and above it a best-effort layer where most of the design work is concentrated. The [trust-layer summary](#the-trust-layer-in-summary) lays out the approach that follows from that.

---

# Appendix A: Full transaction examples

These are illustrative only, and the specific register layouts depend on the chosen contract. The format is pseudo-transaction, with fee outputs left out. The escrow box uses a single `state` enum (`Active` / `Claimed` / `Disputed`) and one reused `deadline` whose meaning depends on the state.

**Listing a physical item for a fixed price.** Inputs: the seller's funding box (ERG for fees, dust, and a rent buffer) plus the seller's bond box (currency for the bond). Output: a Listing Box at the Fixed-Price+Escrow contract address, holding the bond plus an ERG buffer (the bond here is a token, so the box needs enough ERG to cover storage rent for its lifetime), with:
- R4 = seller_pk
- R5 = (placeholder, 1, no_min_fill) *(the offered side is symbolic for physicals)*
- R6 = (USE_token_id, 100) *(wants 100 USE)*
- R7 = metadata_blob: schema_v1 { title:"iPhone 12 Pro 256GB Pacific Blue, lightly used, original box", grade:4, category:263, GTIN:194252029999, short_desc:"Unlocked, all accessories included, minor screen scratches" } *(TLV-encoded, contract-opaque)*
- R8 = (listing_deadline_height, claim_by_blocks, inspection_window_blocks, seller_bond_amount, buyer_bond_amount)
- R9 = (mediator_acceptance_root, price_distribution=[(seller_pk, 980), (frontend_addr, 20)], optional_ipfs_hash_of_images) *(seller 98%, front-end originator 2%)*

**Purchase and escrow open.** Inputs: the Listing Box plus the buyer's funding box (100 USE plus the buyer bond if any plus the network fee, all payable in USE through babel fees). Output: an Escrow Box holding 100 USE plus the seller's bond plus the buyer bond:
- R4 = (seller_pk, buyer_pk)
- R5 = (chosen_mediator_pk, fallback_mediator_pk) *(chosen from the acceptance root with a membership proof)*
- R6 = (state = Active, deadline = current_height + claim_by) *(in Active, deadline is the claim long-stop)*
- R7 = (USE_token_id, 100, seller_bond_amount, buyer_bond_amount) *(bonds are 0 if none)*
- R8 = (inspection_window_blocks, price_distribution=[(seller_pk, 980), (frontend_addr, 20)]) *(carried over from the listing; recipients paid only on settlement)*
- R9 = (reference_to_listing_box, ECIES_encrypted_buyer_shipping_handle)

**Seller claims shipment (Active → Claimed).** Inputs: the Escrow Box (seller's signature; the contract checks state = Active). Output: an updated Escrow Box with `state → Claimed`, `deadline = current_height + inspection_window`, and an optional `tracking_commitment` (a hash, or empty and flagged when absent) written to a register. No funds move.

**Success close, buyer signs received.** Inputs: the Escrow Box (buyer's signature; valid in Active or Claimed, since the buyer can release early at any point). Outputs: 98 USE → seller_pk, 2 USE → frontend_addr (per R8); seller bond → seller_pk; buyer bond → buyer_pk. Optionally a Receipt NFT minted to buyer_pk.

**Success close, inspection timeout (Claimed).** Inputs: the Escrow Box (seller or keeper signs). The contract releases if `state = Claimed` and `current_height > deadline`. Outputs: the same as the buyer-signed close. The contract doesn't care who triggered it, only that the inspection window has passed.

**Refund, seller signs.** Inputs: the Escrow Box (seller's signature; valid in `Active` or `Claimed`, not once a dispute is open). Outputs: 100 USE → buyer_pk (no distribution recipients get paid); seller bond → seller_pk; buyer bond → buyer_pk.

**Timed refund, buyer reclaims (Active).** Inputs: the Escrow Box (buyer or keeper signs). The contract refunds if `state = Active` and `current_height > deadline` (the `claim_by` long-stop passed with no seller claim). Outputs: 100 USE → buyer_pk; seller bond → seller_pk (returned rather than burned, since failing to claim isn't an adjudicated fault); buyer bond → buyer_pk.

**Mutual close, joint signature for any distribution.** Inputs: the Escrow Box (both signatures), optionally plus a buyer funding box for a top-up. Outputs: whatever split the two agree on, such as a partial refund, a deadline extension or parameter change (recreate the box with amended R6/R8), a milestone tranche (pay the seller and re-create the escrow with the remainder), a top-up (re-create the box holding the added value), or a full release. The contract limits the recreation output to a valid escrow box of the same contract or a terminal settlement.

**Dispute, initiator files.** Inputs: the Escrow Box plus the initiator's funding box, which under the initiator-pays structure carries the mediator fee as a direct output to the mediator (the escrow itself holds no fee). Output: an updated Escrow Box with `state → Disputed` and `deadline = current_height + mediation_window`.

**Dispute resolution, mediator rules.** Inputs: the disputed Escrow Box (mediator's signature, before `deadline`). The mediator supplies one graduated value and two binary calls (no fee comes from the box, since it was paid separately): the **price split**, where `buyer_fraction × price` → buyer_pk and the rest goes across the R8 recipients in fixed proportion; the **seller-bond call**, return to seller_pk or send to burn_address; and, if a buyer bond was posted, the **buyer-bond call**, return to buyer_pk or burn. The contract enforces that no other output addresses appear and that the proportions hold. Example (no buyer bond): a 30% buyer refund with the seller bond burned, on a 100 USE / 20 USE-bond escrow, gives 30 → buyer; of the remaining 70, 68.6 → seller and 1.4 → front-end (the 98/2 split); and 20 → burn. A baseless dispute with a 10 USE buyer bond would instead burn that bond: 0 to the buyer, 10 → burn.

**Mediation timeout (Disputed).** Inputs: the Escrow Box (any party or keeper signs). If `state = Disputed` and `current_height > deadline` with no ruling, control passes to the fallback mediator if one is named and still in window; otherwise the breakdown fallback fires: 100 USE → buyer_pk, with both bonds returned to their posters. The mediator can instead extend `deadline` for a complex case, up to a hard ceiling.

---

# Appendix B: Escrow economics

## Bond sizing

Both bonds are deterrents rather than compensation: on a bad-faith ruling the bond burns, and otherwise it comes back. They guard one axis in particular. *Existence* disputes ("never shipped" or "never arrived") are mostly settled by tracking, so the rate at which a party can fool a mediator there is low and the bond does little work. *Condition* disputes ("shipped or arrived, but worthless or not as described") turn on forgeable evidence, and that's the axis each bond actually deters: the seller bond against a seller shipping garbage, the buyer bond against a buyer fabricating a defect.

Take a party weighing whether to act in bad faith, with the price normalized to 1 and a bond `b` (as a fraction of price). Let `f` be the fraction of the price they capture if they win the resulting dispute (for a frivolous buyer, the refund extracted while keeping the item; for a seller, the price kept on successfully defended garbage), and let `q` be their probability of fooling the mediator. The marginal expected value of acting in bad faith over acting honestly is

```
EV  =  q·f  −  (1 − q)·b
```

Bad faith pays when `q·f > (1 − q)·b`, which is to say when the success rate clears the **break-even threshold**

```
q*  =  b / (f + b)
```

The capture fraction `f` matters more than it looks at first, because it splits into two very different cases. **Full extraction** (`f = 1`: the cheat keeps the item *and* wins a complete refund, or holds onto the whole price on defended garbage) is the rare, high-scrutiny case. There a token 10% bond gives `q* = 0.1/1.1 ≈ 9%`, which means bad faith turns a profit if it works barely one time in eleven, far too low a bar against cheap forged evidence, so this case calls for a much larger bond. But most real condition fraud is **graduated**: "30% not as described," a partial refund extracted on an item that's basically fine. With `f ≈ 0.3`, even a 10% bond gives `q* = 0.1/0.4 = 25%`, a much more demanding bar for the same bond. So the headline 9% is the worst case rather than the typical one.

The sizing rule that follows is loose on purpose, because the bond is only one of three terms that set the deterrent together: the **bond**, the mediator's **evidentiary strength** (which sets the achievable `q`), and the poster's **reputation stake**. Set the bond so that `q*` is higher than the success rate a bad-faith party could realistically hit against *the named mediator* on condition complaints in *that category*, and let the other two terms carry their share. A strong, reputation-staked mediator who demands live unboxing video drives the achievable `q` down and lets the bond stay modest; a generalist mediator working off still photos, or an easily-faked category, pushes the required bond well above 10% and toward the larger end for trades prone to full extraction. The cost is capital lockup: a large buyer bond roughly doubles the buyer's committed capital (price plus bond), which is why the bond is sized to the trade instead of maximized, and why low-value trades may rationally carry none at all. The takeaway isn't a fixed percentage but a direction: against forgeable condition evidence, 10% is usually too low, and the right number rises with `f` and falls as the mediator's evidence standard rises.

Each side's bond is sized on its own, because the seller holds the leverage before shipping and the buyer holds it after, so each bond is calibrated to the fraud its poster could commit in the phase where they have the upper hand.

**The model takes `q` as an input, and that input is on the move.** Every threshold above assumes some achievable success rate `q` against the mediator, set by evidentiary strength. The first [core open problem](VISION.md#the-hard-parts) is exactly that `q` is climbing over time as AI makes forged condition evidence cheaper and harder to detect. So the bond isn't a stable guarantee. It's a floor that erodes as `q` climbs: a bond sized correctly today is undersized against the same category and mediator a few years from now, unless the mediator's evidence standard rises to match. This is why the deterrent leans on all three terms instead of the bond alone, why reputation stake and category-specialist mediators carry weight the bond mathematically can't, and why this axis is mitigated rather than closed.

## MAD escrow payoff math

Take an item value of 1 and an equal deposit `D` from each side, with payoffs `(seller, buyer)` shown as the net change from before the trade. MAD (burning both deposits and the escrowed price) fires only when **both** parties withhold: the buyer doesn't accept *and* the seller doesn't refund, usually resolved by a timeout. Any other combination returns funds normally.

|                         | Buyer accepts | Buyer doesn't accept, seller doesn't refund | Buyer doesn't accept, seller refunds |
|-------------------------|---------------|---------------------------------------------|--------------------------------------|
| **Seller ships**        | (0, 0) honest trade | (−1−D, −D) MAD fires | (−1, +1) seller refunds |
| **Seller doesn't ship** | (+1, −1) seller scam | (−D, −1−D) MAD fires | (0, 0) refund, no trade |

This isn't a simultaneous-move game. It's sequential (ship, then accept-or-not, then refund-or-not), so the honest `(0, 0)` outcome has to be tested by backward induction rather than by dominance over the grid. Two traps come out of that:

- **After shipment, the buyer keeps the item.** Once it ships, the buyer holds value 1. If they just don't accept, the seller's remaining choice is to refund (−1, +1) or fire MAD (−1−D, −D), and since −1 > −1−D, the seller rationally refunds to recover their own deposit. Seeing that coming, the buyer keeps the item *and* gets refunded by staying quiet.
- **Before shipment, the seller scams.** The mirror image: a seller who never ships and never refunds forces the buyer to choose between eating the loss (−1) and firing MAD (−1−D), and the buyer rationally eats it, so the seller keeps the price (+1, −1) and walks away.

**Scam-profitability threshold.** Since the deposits are symmetric, so is the threshold. A party deciding whether to scam, facing a counterparty who fires MAD with probability `q`, has expected payoff `1 − q(1+D)`. Scamming beats honesty whenever `q < 1/(1+D)`, or equivalently, since the scam pays off in the `1 − q` case where the counterparty does *not* fire MAD, whenever that case happens more than `D/(1+D)` of the time. With D = 1 (100%) the threshold is 50%; with D = 2 it's two-thirds. Counterparty rationality (above) pushes the actual success rate above either threshold. Letting the parties mutually settle for a partial loss instead of firing MAD doesn't save it: the same backward-induction logic applies to the settlement amounts, so a scammer can credibly threaten the full MAD outcome to extract any partial settlement short of it. Security therefore rests on at least one side acting against their own financial interest, which is psychology rather than logic. Real humans do this sometimes, especially in repeated games with a reputation cost, but the protocol can't count on it for trades between strangers.

---

# Appendix C: Glossary

- **eUTXO.** Extended UTXO. Ergo's state model.
- **Box.** A UTXO with value, tokens, registers R4–R9, and a guard script.
- **Atomic swap.** On-chain trade that fully completes or fully fails.
- **Escrow box.** Programmable holding box between match and settlement, carrying a `state` (`Active` / `Claimed` / `Disputed`) and one reused `deadline`; funds exit only through the three doors.
- **Three doors.** Success / refund / dispute: the escrow contract's three terminal exit paths.
- **Active / Claimed phases.** The two pre-settlement states. In `Active` (before the shipment claim) silence favors the buyer; in `Claimed` (after) it favors the seller.
- **`claim_by`.** Seller-set window in `Active` after which the buyer may trigger a timed refund if the seller has not claimed shipment. Sized to dispatch time.
- **Inspection window.** Seller-set window in `Claimed`, starting at the shipment claim, after which the seller may trigger a timed release. Sized transit + inspection; the buyer's dispute right runs throughout it.
- **Timed refund / timed release.** After a phase timer passes, the favorable outcome becomes spendable: refund to the buyer in `Active` (buyer-triggered), release to the seller in `Claimed` (seller- or keeper-triggered). Nothing fires automatically.
- **Shipment claim.** The seller's transition from `Active` to `Claimed`, optionally committing a tracking number; flips the silence default and starts the inspection window. Tracking is recommended, not required, and flagged when absent.
- **Mediation deadline.** Bounded clock on a `Disputed` escrow; if the mediator doesn't rule within it (extensions capped by a protocol constant), control falls to the fallback mediator, then to a buyer-favorable breakdown fallback.
- **Seller bond.** Optional, seller-set deposit; returned on honest completion, burned on a bad-faith ruling. Deters shipping nothing or garbage.
- **Buyer bond.** Optional, seller-set deposit posted by the buyer; returned on honest completion, burned on a bad-faith dispute. The mirror of the seller bond, deterring frivolous disputes rather than bad shipments.
- **Mutual handshake.** Both buyer and seller agree on the specific mediator at purchase time.
- **Standby fee.** Mediator fee paid up front by the buyer for availability rather than adjudication; the recommended default (see [Fees](#mediators)).
- **Initiator-pays fee.** The privacy-minimal alternative: the party filing a dispute commits the mediator fee at filing time.
- **Price distribution.** A seller-declared list of `(address, share)` outputs splitting the sale price among sale-contingent parties (seller, originator, affiliate, supplier), paid at settlement in proportion to how much settles in the seller's favor.
- **Originator fee.** The price-distribution case where one recipient is the front-end that minted the listing. Not a separate mechanism.
- **Sidecar / Config Box.** Read-only data-input box referenced by listings for shared state.
- **Data inputs.** Ergo's term for read-only box references in a transaction; the mechanism behind sidecars.
- **Singleton Token / State NFT.** An NFT minted once and held in a config or state box; the consuming contract checks it is present to reject spoofed copies.
- **Guardrail.** Hard-coded cap in an immutable contract limiting what a mutable config box can specify.
- **Sigma protocol.** Ergo's zero-knowledge proof primitive.
- **`atLeast(k, …)`.** Native ErgoScript threshold primitive proving a k-of-n signing condition was met while concealing *which* k signed; the basis for panelist anti-retaliation.
- **AVL+ tree.** Native authenticated data structure in Ergo, with membership proofs verifiable on-chain.
- **NiPoPoW.** Non-interactive proof of proof-of-work; lets light clients verify the chain without downloading it.
- **MAD escrow.** Mutually Assured Destruction: double-deposit escrow with no mediator and no recovery path. Its honest outcome is not a stable equilibrium, so it is unsuitable as a default for trade between strangers.
- **GTIN.** Global Trade Item Number; umbrella for UPC, EAN, etc.
- **Receipt NFT.** Optional NFT minted on settlement, proving purchase; bound to the spent escrow box via the issuer-box pattern for trade-level provenance.
- **Identity NFT.** An NFT a seller or mediator mints so reputation lives on the NFT rather than a key, enabling key rotation; freely transferable (see [Identity](#identity) for the hot/cold key split high-value holders should use).
- **Mutual close.** Predicate allowing any output distribution if both parties sign; enables partial refunds, deadline extensions, milestone tranches, top-ups, and state channels.
- **Babel fees (EIP-31).** Ergo mechanism letting users pay transaction fees in any token they hold, via providers who supply ERG in exchange.
- **Pocket mediator.** A mediator secretly controlled by one of the trade parties.
- **Stealth addresses.** Live Ergo feature letting a recipient receive funds without their public address appearing on chain.
- **Curve Trees.** ZK membership proofs without trusted setup, feasible on Ergo via 6.0's `UnsignedBigInt`; still research-stage.
- **EIP.** Ergo Improvement Proposal. Referenced: EIP-4 (asset standard), EIP-22 (auction), EIP-23 (oracle pool v2), EIP-24 (artwork/royalty), EIP-28 (wallet auth challenge), EIP-31 (babel fees), proposed EIP-0045 (native STARK verification opcode).
- **Schelling point.** A natural coordination focal point, shared use of GTIN with no central enforcement is one.
