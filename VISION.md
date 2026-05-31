# Decentralized Marketplace on Ergo

*Vision and overview. For the full technical specification, see [DESIGN.md](DESIGN.md).*

You hold your own keys, you own your trading history, and if you want to switch to a different front-end you can leave any time and take that history with you. The idea is neutral public infrastructure for trading directly with another person: two people swapping an item, a service, or a token, with no platform sitting in the middle that can ban them, lock them out by country or bank, or skim a cut it didn't earn.

A one-time seller and a full-time merchant use the exact same rails. There's no business tier, no KYC threshold, no account to apply for, nobody to ask.

The protocol stays thin on purpose: no central operator, no protocol fee, no governance token. It doesn't curate, vet, rank, or moderate anything. That work, and all the money, lives in higher layers where different teams can compete with different opinions. The protocol underneath is a public utility.

This doesn't pretend to abolish concentration. Holding discretion over disputed funds is a position of trust, and trust tends to pool around a few players. The claim is narrower than that. Because reputation, listings, and discovery all live as portable public record, no single service can trap anyone. Leaving is always free, and whoever leaves takes their whole history along, so a dominant service only stays dominant for as long as it's the best one. Low switching cost is what holds that together, and it's a property to defend rather than something guaranteed.

Three commitments shape the design:

- **Neutrality.** The protocol knows nothing about quality, reputation, brands, or categories. It settles trades and does nothing else.
- **Modularity.** Lots of small, specialized contracts instead of one giant one. Each listing is its own instance of whichever contract the seller picked. Anyone can deploy new contracts whenever they like, and tooling picks them up automatically once they speak the shared register interface.
- **Per-transaction trust.** The mediator, registry, and front-end are chosen for each trade. Nothing comes baked in as a blessed default.

# Why this exists

The space where two people used to just trade directly has been hollowing out. Classifieds, local boards, neighborhood listings, chat OTC: most of it got pulled into a handful of platforms, or into unprotected DMs where people get scammed. The platforms (Amazon, eBay, Stripe, PayPal, and their regional equivalents) each take 10–30% of the transaction, can deplatform anyone whenever they like, and decide who's allowed in based on geography and banking. The same consolidation that grinds down merchants' margins also kills the spot where an ordinary person could sell a used phone to a stranger with a bit of safety, no application required, without having to blindly trust whoever they're chatting with.

Ergo's eUTXO model makes a neutral settlement layer practical in a way other chains haven't managed, and the community already has working production code for every key pattern this design leans on. So this is an engineering problem to build, not a research problem to solve ([Why Ergo](#why-ergo)).

This matters most to people who have no good option right now: someone selling one item who wants to deal with a stranger without a middleman; traders in countries with capital controls or thin banking; merchants who've been deplatformed or were never let on in the first place; small repair shops getting bled by chargeback fraud under buyer-favorable platform rules; tight-knit communities that want their own credit and trade rails; high-end specialist markets where authentication ought to cost 1% instead of 10%; and the early agent economies that are just now forming. For everyone else, meaning anyone happily using Amazon or eBay, it's a competitor on price and trust rather than some revolution. It has to actually be better on those terms to win them over, and for plenty of use cases it won't be. That's fine. Not every market needs saving.

It doesn't replace consumer-protection law. It doesn't make physical trade fully private, since shipping addresses leak. It can't promise curation won't re-concentrate up at the application layer, and it won't stop bad actors from using neutral rails for bad ends. What it does offer is a flatter playing field: the rent and gatekeeping of platform-run trade swapped out for open, permissionless infrastructure anyone can build on. To me that's worth building.

---

# How it works

## The core distinction: trustless vs mediated trade

There are two structurally different cases here, and they get different contracts, different user flows, and different risk models.

**Trustless trade** covers exchanges where both sides already exist on-chain: fungible tokens, NFTs, other tokenized assets. The seller's box locks the tokens, and the buyer's spend transaction sends payment to the seller and the asset to the buyer in one atomic step. Nothing ships, nothing has to be checked afterward, and there's no mediator, escrow window, or dispute path. The transaction either goes through with both sides getting what they wanted, or it doesn't happen at all. This case is basically a solved problem, and Ergo already runs production infrastructure for it.

**Mediated trade** covers exchanges where at least one side has to do something off-chain: a physical good ships, a service gets performed, an account changes hands away from the chain, or fiat is swapped for crypto. The chain can't see whether that off-chain delivery happened, so a mediator sits between the match and the settlement, able to redistribute the escrowed funds if someone complains. Buyer and seller agree on who that mediator is at the time of purchase. Most of the protocol's complexity, and nearly all of its hard problems, live in this case.

*Here's the shape of it.* Alice sells Bob a used camera. Bob's payment, plus an optional bond from each side, sits locked in an escrow box. Alice ships the camera and marks it sent on-chain; Bob confirms it arrived as described, and the contract releases the payment to Alice. If it never shows up, or turns up broken, either of them can open a dispute, and the mediator they both agreed on decides how the escrow gets split. Everything else about mediated trade is machinery built around that one basic flow.

[The on-chain layer](DESIGN.md#the-on-chain-layer-what-the-chain-enforces) in the design doc covers what the chain can enforce with cryptographic certainty. [The trust layer](DESIGN.md#the-trust-layer-what-the-chain-cannot-enforce) covers everything it can't, the parts humans and competing services have to supply: dispute mediation, reputation, curation, front-end design.

Trustless trade reuses what Ergo already has (Spectrum Finance, the NFT marketplaces). The new work is the mediated stack for physical goods and services, where the chain can't verify delivery and the protocol has to coordinate trust between people who don't know each other.

**The part that has to ship first is small.** The full system has a lot of parts (they're all in the [design doc](DESIGN.md)), but the critical path is just one escrow contract, a named mediator, a shareable link, and a wallet that shows escrow state. That by itself is already a usable product: trust-minimized escrow for two people who've already agreed on a deal (the [escrow tool](#the-escrow-tool-one-click-otc-escrow)). Everything else only earns its place once that core has proven itself on real trades. Think of the rest as things that grow on top of the escrow tool rather than things needed before it. The [build order](#build-order) maps each part to a phase.

---

## How it works at a glance

The chain settles trades. Everything else (discovery, curation, reputation, dispute mediation, how it all looks) sits in competing layers above it, where teams can build new things without needing protocol-wide agreement. The core is kept small on purpose.

```
   ┌───────────────────────────────────────────────────┐
   │  USERS                                            │
   └───────────────────────────────────────────────────┘
   ┌───────────────────────────────────────────────────┐
   │  FRONT-ENDS        marketplaces, niche UIs,       │
   │                    deal bots, embeds              │
   └───────────────────────────────────────────────────┘
   ┌───────────────────────────────────────────────────┐
   │  OFF-CHAIN         indexers · mediator dirs ·     │
   │  SERVICES          reputation services            │
   └───────────────────────────────────────────────────┘
   ┌───────────────────────────────────────────────────┐
   │  REGISTRIES        federated contract-hash        │
   │  (federated)       directories                    │
   └───────────────────────────────────────────────────┘
   ┌───────────────────────────────────────────────────┐
   │  BOTS / SOLVERS    match · batch · cross-DEX ·    │
   │                    cleanup · time-keeping         │
   └───────────────────────────────────────────────────┘
   ┌───────────────────────────────────────────────────┐
   │  CORE CONTRACTS    atomic swap · escrow ·         │
   │  (immutable)       auctions · sidecars            │
   └───────────────────────────────────────────────────┘
   ┌───────────────────────────────────────────────────┐
   │  ERGO BLOCKCHAIN   settlement of all boxes        │
   └───────────────────────────────────────────────────┘
```

**The trust surface at a glance.** One row per role: what each role is relied on for, the worst it can do if it turns, and what limits the damage. Notice that none of these roles is the protocol itself. Since the protocol runs on no servers, a light client small enough to fit on a phone can check every row directly against the chain.

| Role | Trusted for | Worst it can do | Bounded by |
|------|-------------|-----------------|------------|
| **Contracts / chain** | Atomic settlement; funds exit only through allowed paths; listing terms unbreakable | A permanent bug strands or misroutes funds | Pre-deployment testing; a bug is fixed only by migrating to a new contract, never retroactively |
| **Wallet** | Showing the true box contents, verifying the contract hash, encrypting the address locally, and surfacing the named mediator for the user to check | Sign anything on the user's behalf | Nothing structural; it is the root of trust, so open-source and reproducible builds |
| **Mediator** | An honest price split and honest bad-faith calls on the bonds in a dispute | A corrupt-but-legal ruling shifts the split within the cap | Hard-capped outputs (buyer / fixed seller-side / burn only), mutual handshake, appeal tier, reputation |
| **Counterparty** | Off-chain delivery (seller) or honest receipt (buyer) | Scam: never ship, or fake a defect | Bond, mediator, reputation, timed exits |
| **Front-end** | Discovery, curation, presentation, and recommending the mediator and trade parameters (bonds, timers) for each trade | Bias the user's selection, hide listings, or steer toward a weak mediator or bad terms | Competing front-ends; the wallet (not the front-end) verifies what actually gets signed |
| **Indexer** | Completeness and freshness of listings shown | Omit or stale a listing | Running an independent indexer or a light client; on-chain verification before signing |
| **Reputation service** | Accuracy of metrics | Steer a user to a bad trade with a wrong metric | Per-trade escrow and bond protect the buyer regardless of metric; use several services |
| **Registry** | Which contract hashes are worth scanning | List a malicious contract | Subscribe to several; the contract enforces its own output constraints no matter how it was discovered |

**The escrow, in brief.** When one side of a trade happens off-chain, the chain can't tell whether delivery actually happened, so the payment sits in an escrow box between the match and the settlement. The whole thing pivots on the single event the chain *can* see: the seller claiming they've shipped. Before that claim, silence favors the buyer, who can reclaim their money after a deadline. After it, silence favors the seller, and the payment releases once the inspection window runs out. The escrow can only leave through three doors: success (the buyer confirms, or the inspection window passes), refund (the seller hands the money back), or dispute (a mediator decides). Either side can also post a bond, which comes back on honest completion and burns only on a bad-faith ruling, to discourage shipping garbage or filing a nonsense complaint. The full state machine, the parameters, and the contract family are in [DESIGN.md](DESIGN.md#mediated-escrow).

**Mediators are chosen per trade.** The protocol never assigns one. The seller lists a set of mediators they'll accept, the buyer picks one of them at purchase, and so both sides have signed off on that specific person, and the buyer's front-end can vet that person against reputation services beforehand. A mediator's power is capped hard: in a dispute they decide how the disputed price splits between buyer and seller, and whether each bond comes back or burns. That's the whole list. A corrupt mediator can move the split around within those limits, but can never send funds to some address of their own. Selection, fees, deadlines, and the full range of implementations (single agents, multisig teams, random-assignment pools, appeal panels) are in [DESIGN.md](DESIGN.md#mediators).

**Reputation, identity, and discovery all travel with the user.** Every settled trade is permanent and public, so a seller's entire history (volume, dispute rate, resolution time) can be read straight off the chain. The chain keeps the record; competing services turn it into metrics; and anyone who leaves one front-end brings their full history along to the next. An optional Identity NFT lets a seller or mediator build standing on a token instead of a single key, so they can rotate keys without losing everything. Discovery is federated: anyone can publish a registry of contract addresses worth scanning, and front-ends subscribe to whichever registries they trust, which puts the censorship decision out at the edges instead of in the protocol. The mechanisms are in the design doc ([reputation](DESIGN.md#reputation), [identity](DESIGN.md#identity), [discovery](DESIGN.md#discovery-federated-registries)).

**Privacy is built in.** Sellers can stay pseudonymous, wallets use a fresh address for each purchase, and Ergo's native tools handle the rest: stealth addresses, ErgoMixer for sensitive payouts, and ring-signature *anonymous verified reviews* that prove someone made a real purchase without revealing which one it was. The one leak physical trade can't seal shut is the shipping address (the third [hard part](#the-hard-parts)). The design treats that as a safety issue and points buyers toward address-neutral delivery like lockers and PO boxes, rather than pretending the problem isn't there.

**The goal is to be actually unstoppable, not decentralized in name only.** Since the protocol leans on no servers, a light client can read listings, build trades, and check escrow status with no backend whatsoever. If every centralized front-end went dark tonight, trading would still work tomorrow. Most people will take the faster indexer-backed path, but the trustless one stays open as a matter of design.

The approach that falls out of this: be *generous* at the protocol layer by giving people the tools they need, and *ruthless* at the application layer by curating hard, banning bad actors, and surfacing accurate information. The incentive gaps in between those two layers are real design work this project owns, not something to wave off onto someone else.

## Why Ergo

Ergo uses an eUTXO model, where every listing is its own on-chain box evaluated on its own. Separate trades settle in parallel with no shared mutable state, and a complex transaction costs well under 0.01 ERG. More importantly, Ergo already ships the cryptographic tools this design needs: covenants that constrain how a box can be spent, Sigma zero-knowledge proofs, authenticated AVL+ trees, native oracle pools, and NiPoPoW light-client proofs. The community also has working production examples of every key pattern the design relies on. The full list, and what each piece enables, is in [DESIGN.md](DESIGN.md#why-ergo).

## The hard parts

Three problems are unsolved, and for trade between strangers they may be unsolvable. The design doesn't pretend otherwise.

1. **Off-chain delivery can't be verified, and AI is eroding the evidence.** The chain can't see a package, so condition disputes ("it arrived, but not as described") come down to evidence, and AI-generated photos, video, and tracking screenshots get cheaper to forge every year. Bonds and specialist mediators raise the cost of fooling a ruling. Nothing shuts the gap completely.
2. **Mediator trust concentrates, and the incentives can point the wrong way.** Discretion over disputed funds is a position of trust, so it drifts toward a few incumbents, and for a long-trusted mediator the single biggest payday on offer might be to cash in all that accumulated trust in one coordinated raid. Capped power, the mutual handshake, pools, and appeal tiers limit the damage. They don't remove the temptation.
3. **The shipping address has to become plaintext for whoever ships it.** There's no way to make physical delivery cryptographically private: a real address has to reach a counterparty who might be malicious. That's a safety risk, well past a simple privacy leak. The mitigations are warning the buyer and using address-neutral delivery. There's no crypto fix for it.

Wherever a later section runs into one of these, it says so and links back here.

Below those sits the deepest open question, which is commercial rather than technical: whether stranger-to-stranger decentralized physical-goods trade has real transactional demand at this level of friction. The most serious prior attempt never reached the volume to find out (see [Related work](#related-work)), so this design inherits its per-trade mediation model without proof it works at scale. That's the central thing to *test*, not assume, and the reason the [build order](#build-order) starts small.

## The economic model

There's no protocol fee and no governance token. Anyone can deploy a competing contract that charges nothing, and the moment a rival front-end shows the same listings for less, buyers move. That pushes fees down toward the marginal cost of running the front-end and indexer infrastructure. The money instead gets made by services that have real costs or real scarcity: curation and seller verification, dispute mediation, fast and well-curated indexers, reputation analysis, premium merchant tools (tax export, inventory, shipping, storefronts), and authentication for high-value categories.

The shared base those businesses sit on (the audited contracts, the published standard, an open indexer, wallet support) gets funded in a different way from the businesses themselves. The key difference from OpenBazaar's collapse is that a serverless protocol turns a *recurring* running cost into a *one-time* build cost, and donations and grants fund a one-time build pretty well even though they fund perpetual burn badly. So the base can run on community donations and ecosystem grants, with money set aside specifically for one professional audit of the immutable money-handling contracts, because a permanent bug in those is exactly the line item that goodwill and ideology cover least reliably. The protocol layer stays free of fees and tokens; the services above it are normal businesses that fund themselves however they want. The full revenue breakdown is in [DESIGN.md](DESIGN.md#monetization).

# Getting it off the ground

## The escrow tool: one-click OTC escrow

Two people who've already agreed want to finish the trade safely, and they don't need a whole marketplace to do it. Most peer-to-peer trade today happens in chat platforms (Telegram, Discord, Reddit) and gets scammed all the time. A page that spits out a **one-click escrow link for two people who already agreed on terms** is useful right away: nobody has to learn a marketplace, they just paste a link. Sellers bring their own buyers and come looking for safety. The same link can carry a seller-set allowlist that restricts the listing to that one buyer, so a pre-agreed OTC or Discord trade holds the item for them while the payment comes together.

Be clear about what this actually seeds, though. The escrow tool builds **users and trust in the escrow mechanism**. What it doesn't build, on its own, is **browsable liquidity**, which is a separate two-sided problem. A million people who've all used an escrow link still leave a marketplace with zero inventory to browse. So the marketplace doesn't simply grow out of escrow-link volume on its own. It grows out of the deliberate community-seeding work described below, which the escrow tool's earned trust makes easier to pitch but never automatic. The two efforts share a contract and a reputation base, and not much else.

## Bootstrapping strategies

- **Capture existing P2P trade that has no trust layer.** The escrow-link tool serves loose OTC trade in chat platforms, and the same rails embedded as a widget serve established niche communities that already trade among themselves: vintage watches, retro electronics, mining hardware, particular game communities. It's the same move in both cases: hand people who already trade and already get scammed an escrow-and-dispute layer, without making them leave where they already are. The operator can take an originator fee.
- **Demand-first off-chain boards.** A simple "wanted" board where buyers post what they're after (item, max price, region) and sellers answer by minting normal listings tagged to the request. It lives off-chain, with no on-chain commitment from the buyer until a real listing exists. A professional seller who sees "30 people want iPhone 12s under $400" is getting a useful signal, and no buyer had to lock up funds to send it (see [why there is no RFQ contract](DESIGN.md#register-interface--the-contract-family)).

## Target communities

The protocol's neutrality is worth the most where the existing options are worst:

- **Person-to-person trade of used goods.** This is the layer classifieds, local boards, and chat OTC used to serve, now mostly swallowed by platforms that take a cut and deplatform whenever they like, or by unprotected DMs that get scammed. A one-time seller just wants one trade to be safe and has no use for a storefront. Low-value used-goods trades will realistically run with a small bond or none, leaning on reputation and the low absolute stakes (the [bond-sizing](DESIGN.md#bond-sizing) reasoning is in the design doc).
- **Right-to-repair and refurbished hardware.** eBay's buyer-favorable dispute resolution kills small repair shops through chargeback fraud. Well-bonded escrow with a specialist mediator changes that dynamic.
- **High-end collectibles.** StockX-style platforms charge 10–15% for authentication. A specialist mediator with the same expertise can charge a fraction of that.
- **Hardware miners.** Constant GPU trading, high scam rates, a crypto-native audience.
- **Gaming virtual-asset traders.** Plagued by chargeback scams on PayPal-based platforms.
- **Peer-to-peer fiat on and off-ramps.** Swapping cash or a bank transfer for crypto is mediated trade with the fiat leg as the unverifiable side, so the same escrow and dispute machinery covers Bisq- or LocalBitcoins-style trading.
- **Privacy, cypherpunk, and unbanked traders.** The Monero crowd, people living under capital controls, sellers who don't want a PayPal account tied to their legal name.

The pitch should always be "here's a tool that fixes your specific problem," never "come join a new marketplace."

## Build order

Two things have to come first, in order. After that, everything else stands alone: each block is useful by itself and gets built when demand calls for it.

- **v0, the escrow tool.** The **escrow contract** (Fixed-Price+Escrow with the two-phase claim/inspection timers and optional bonds), plus the OTC link and a named mediator. This is a usable product on its own, with no other layer required.
- **v1, discovery.** The single gate from "shareable link" to "discoverable marketplace": the listing register published as an EIP, plus one open-source indexer so any front-end can read listings.
- **v2 and beyond, the parallel menu.** No fixed order here: a front-end UI, the auction contract, comparable-listings and price-history aggregation, reputation services, mediator directories and pools, and then the connective and application layers (privacy, agent economies, light clients), each as demand calls for it.

The trustless-trade contracts (atomic swap, auction) are well-established Ergo patterns to reuse rather than build from scratch.

---

# Context and tradeoffs

What's been tried before, what this changes out in the wider world for better and for worse, and where it stands legally. The protocol-enforced guarantees and the full failure-mode register live in [DESIGN.md](DESIGN.md#risk-register).

## Related work

Decentralized commerce isn't a new idea, and being honest about what's been tried sharpens the picture of what this does differently.

**OpenBazaar (2014–2021)** was the most serious attempt: a P2P network with multisig escrow and per-trade third-party moderators, conceptually close to this design's mutual-handshake mediation. It was installed more than 250,000 times and still shut down in January 2021, and the lesson lives in that gap. Installs proved an *appetite to install*, which is not the same thing as an *appetite to transact*. A 14-month academic crawl (the "Open Market or Ghost Town?" study) found the economic activity modest at best. The mechanical cause of death was funding: OB1 burned through roughly $9M of venture capital and couldn't keep the seed nodes, the hosted wallet, and the search API running on donations. But the funding only ran out because almost nobody was paying for anything.

The adoption problems are clear in hindsight. It made users run a node (or a hosted one, which quietly recentralized things); search was slow enough that the team fell back to a federated index; and it was Bitcoin-only with no stablecoin until late, which left prices fully exposed to volatility. This design tries to answer all of those: don't make users run infrastructure (light clients are optional), keep discovery in a competitive front-end layer, support stable units from day one, and lean on no servers. The most important lesson is the one OB never got to run, though. It never reached the volume to actually *test* per-trade moderation at scale, so this design inherits that mediation model without proof that it works. That's the demand question, and it's why the build order puts the escrow tool first, as a cheap way to answer it before betting a whole marketplace on the answer.

**Particl** is still going as a privacy-focused marketplace on its own chain with a cross-chain DEX, real and still under active development. But it relies on two-party double-deposit (MAD) escrow, the model this design [rejects](DESIGN.md#mad-double-deposit-escrow-rejected) for everyday trade, and running its own chain and client piles the work into the protocol itself instead of a competitive layer sitting above it.

**Boson Protocol** is the most sophisticated active design for physical goods. A purchase mints a redeemable NFT (a forward contract for an item) backed by escrowed funds, with optimistic fair-exchange incentives, and it's recently been repositioned around agentic commerce. Boson is token-governed and charges a protocol fee, where this design does neither. It lives in the account-based, higher-fee EVM world, where each rNFT is global state rather than a local box. And its dispute resolution leans on designated resolvers inside one governed protocol, where this uses an open set of per-trade mediators with a hard-capped, burn-or-split payout and no token at all. Boson's token also funds ongoing development that this design has to find money for some other way. What this design gives in exchange is neutrality, a smaller trust surface, and single-purpose contracts swapped per use case instead of one governed protocol that has to evolve as a whole.

**Centralized incumbents** (eBay, Amazon, StockX, Vinted, Facebook Marketplace) are the real competition for users. They win on liquidity, polish, and familiar buyer protection. They lose on fees, deplatforming risk, banking gatekeeping, and reputation lock-in. The realistic near-term targets are exactly the niches they serve badly or won't serve at all.

There's a pattern across the decentralized attempts. Most of them tried to be the entire stack at once (protocol, discovery, identity, client, sometimes their own chain) and then either reintroduced centralization to stay usable or stayed too clunky for anyone to adopt. This design goes the other way: a minimal neutral protocol on a general-purpose chain that already exists, with discovery, curation, reputation, communication, and UX all competed out in a layer above it, and no protocol-level token or fee anywhere.

## Externalities

Trade infrastructure is rarely neutral in its effects.

**Positive:**
- **A more efficient market.** Platforms take 10–30% today; on-chain settlement squeezes fees down toward marginal cost. The open record also clears out the information asymmetry: no platform is sitting on proprietary transaction data it can use to undercut its own sellers (the dynamic where Amazon mines its own sellers' data structurally can't arise here), and that same history becomes an input for lending, insurance, and agent-economy credit, none of which can see commerce data today.
- **Open access.** Anyone with a wallet can buy or sell anywhere. No banking-corridor exclusion, no policy-based deplatforming, no remittance fees, and an on-ramp for people who are unbanked or living under capital controls. It's currency-agnostic and cross-border out of the box.
- **Durable, portable records.** Sellers own their own history, so a single platform ban no longer erases years of standing, and reputation follows them across front-ends. The chain proves who signed what and when (not that an item is genuine, but that a specific attester committed to a claim at a specific time), so provenance is auditable through attestation chains, and there's no CEO to subpoena and no central server to take down.

**Negative:**
- **Consumer protection erodes.** Lemon laws depend on central enforcement, and with no corporate counterparty it's unclear what recourse a buyer has for a harmful product. Mediation handles ordinary disputes; it doesn't do product-safety regulation or class-action remedies for systemic harm.
- **The tax base decays.** P2P trade is harder to assess for sales tax than centralized activity, so public revenue collection can take a hit.
- **Dark-market spillover.** The same rails that help a farmer sell grain can move illicit goods just as easily.
- **The loss of choke points for collective norms.** Centralized marketplaces are places where governments and civil society can lean on someone to enforce soft norms: labor practices, embargoed-country trade, age restrictions, dual-use materials, ethical sourcing. Take them away and that leverage goes too, because the protocol can't enforce norms it has no way of knowing about. For most norms this is roughly a wash, since a lot of that enforcement is symbolic anyway. For a few (dual-use scientific equipment, anonymous fundraising for harmful collective action, unsupervised autonomous-agent commerce) the lost leverage matters and could feed tail risks. Front-end curation can filter some of it; the protocol level can't. This is a basic property of neutral infrastructure, the very same one that resists deplatforming and rent extraction, and the trade-off is lopsided: the upside is spread broadly, while the downside concentrates in the tails.

These costs are real. Because the protocol is neutral, it can't do anything about them itself. That work sits above the protocol, in curation and front-ends, or, where it's beyond what curation can reach, nowhere the protocol can touch at all.

## Legal

The protocol layer is open-source code. The strongest legal position follows *Risley v. Uniswap*: we wrote code, and we don't control who uses it. Deploying anonymously or as an unincorporated collective makes that position stronger.

The **front-end layer** is where regulation actually lands. A front-end running a curated marketplace, especially one that connects professional traders to consumers, generally takes on obligations under marketplace and consumer-protection law in most places: seller KYC (enforced more strictly than buyer KYC almost everywhere), tax reporting on seller income, liability for what it displays (illegal items, scam listings), and possibly registration as a virtual-asset service provider. Keeping a clean legal line between the protocol entity (anonymous, no fees) and any front-end entity (a normal LLC that KYCs its sellers and complies locally) helps protect the protocol-level defense. Staying non-custodial, so funds flow buyer → contract → seller and never touch a front-end operator's wallet, keeps the picture simpler in most jurisdictions, though the details vary.

The **mediator** is the most legally exposed person in the system, because they exercise discretion over other people's money. Two characterizations could bite, both depending on jurisdiction: their rulings might count as **arbitration** (which raises due-process and award-enforceability questions), and directing escrowed funds might look like **escrow-agent** or **money-transmitter** activity, which is licensed in plenty of places. Three things soften this. The mediator is **non-custodial**: they never hold the funds, they only sign a transaction that picks among outputs the contract already constrains, which is a much weaker position than an agent who takes possession. Their power is **hard-capped** to a few fractions, never an arbitrary address. And mediation is **chosen by consent** for each trade, so a published rule set and scope can frame the role as agreed dispute resolution. None of that settles the question. "Non-custodial but directs funds" is legally untested for this exact pattern, and a high-volume **mediator pool** running for profit looks far more like a regulated business than a neighbor settling one Discord trade. Exposure rises with formality and volume: a casual mediator carries little risk, while a compliant pool should assume it might need to register somewhere, get counsel, and put jurisdictional limits on which trades it'll take.

The sharp edge is that this pressure **sorts the wrong way.** Legal exposure is heaviest exactly where the stakes are highest, so the most accountable, legally-cautious mediators steer clear of the highest-value or most-regulated categories. Those trades then drift toward anonymous or lax-jurisdiction operators, which are the very actors the reputation layer can't bootstrap and front-ends are told to turn away. So regulation doesn't thin out mediator supply evenly. It pushes the highest-stakes trades toward the lowest-trust mediation. This is part of the second [core open problem](#the-hard-parts), not a separable compliance footnote.

---

# Going deeper

This document covers the why and the overview. The full technical specification, the contract family, the escrow state machine, the parameter and bond economics, the mediator and reputation mechanisms, the connective and application layers, the complete risk register, and worked transaction examples all live in [DESIGN.md](DESIGN.md).

The aim under all of it is a plain one: let two people trade directly, on their own keys, without asking anyone for permission, and keep the base that lets them do it neutral and owned by nobody.
