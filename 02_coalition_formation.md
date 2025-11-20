# Market based coalition formation for collaborative transaction construction

## Problem definition

In several existing market based decentralized protocols involving collaborative transactions a single party initiates and typically bears the costs. For example JoinMarket or liquidity ads.

Transactions meeting the demand of more than one "taker" can be more efficient depending on their desired outcomes. CoinJoin privacy in particular is a non-rivalrous positive externality obtained by each participant from the other participants, and as such it is inherently positive sum.

The following protocol aims to generalize the type of order matching that JoinMarket supports, allowing multiple users' intents to be aggregated together.

This is not a transaction construction protocol (e.g. WabiSabi), but a precursor to such a protocol. This protocol bootstraps such a protocol by finding consensus on initial set of UTXOs whose (honest) owners intend to spend in a collaborative transaction. See the last section for a brief discussion of such a protocol.

## Overview of proposed solution

This document describes a permissionless, peer to peer protocol for negotiating collaborative transaction construction. This takes place over two primary phases: bilateral negotiation followed by aggregation. Successful execution of the protocol results in a quorum of UTXO owners unanimously deciding to build a transaction together.

### Bilateral negotiation

A proposer, the owner of a UTXO, selects some other UTXOs and initiates peer to peer bilateral negotiations with the owners of these coins by sending them a message that includes a *co-spend proposal*. This expresses an intent to cooperate to spend this set of coins together in a single Bitcoin transaction.

Co-spend proposals are publicly verifiable, confidential adjustments to the effective values of the specified coins. These adjustments can represent arbitrary incentive structures. Whether a proposal takes effect (the adjustment is applied) is contingent on the constraints specified in the proposal being satisfied by the transaction which spends the specified UTXOs. For example, a proposal may require that non-SegWit inputs be excluded for txid stability, or that the feerate be within an acceptable range, among other constraints.

Co-spend proposals rely on cryptography to keep each coin's adjustment hidden except from the proposer and each coin's owner. The proposer's UTXO is indistinguishable from the other UTXOs. Before a proposal is unanimously accepted, it is not known which UTXO owners have accepted the proposal.

### Aggregation

Co-spend proposals are not exclusive. Many proposals can be aggregated together before collaboratively constructing a transaction which in which the combination of all the aggregated proposals is in effect. The UTXOs specified in proposals that are aggregated together may overlap but don't need to, the main requirement is for the constraints to be compatible.

Fully accepted proposals can be broadcast and then aggregated with other compatible proposals. Aggregation makes it possible for multiple participants to simulatenously optimize their outcome according to their individual preferences. When they value privacy, this is a positive sum interaction and aggregation optimizes the generated surplus.

To aggregate, several proposals are bundled together with a *quorum proposal* that all parties must agree to. Unanimous acceptance of such a quorum proposal signals that everyone is ready to construct a transaction.

Because transactions are only valid if all inputs are signed, multiparty transaction require unanimous agreement. A precondition for denial of service protection in privacy preserving multiparty transaction construction protocols is incentive compatibility for the honest participants. No honest participant should have a reason to withold their signature.

The primary purpose of this protocol is to construct an [imputation](https://en.wikipedia.org/wiki/Imputation_%28game_theory%29), a payoff distribution over the inputs, which aligns incentives for successful transaction construction, while maintaining confidentiality of the individual payoffs.

## Background

### Hedonic games and games with transferrable utility

Coalition formation games output a partition over $N$ peers. Finding stable equilibria in such games is generally hard. Depending on the structures in the preferences of the peers, many special cases described in the hedonic games literature are still hard. In the byzantine setting appropriate for Bitcoin protocols, no such structure can be assumed, which makes finding stable partitions intractable.

Since the participants in the protocol unilaterally control independent UTXOs during the execution of this protocol, we are constrained to the non-cooperative setting, where agreements are non-binding and can't be enforced. Additionally, for privacy the cost functions should remain hidden, so no mechanism can access peers' preferences and globally optimize. Note that due to the nature of revealed preferences the cost function can't be fully hidden.

Fortunately, the introduction of a transferrable utility greatly simplifies finding stable coalition structures even under these assumption. Bitcoin can approximate such a transferrable utility. This makes it possible to simply assign payoffs to the coalitions and allow them to be distributed among the peers through bilateral negotiation. If Alice prefers not to co-spend with Bob, but Carol strongly favors both Alice's and Bob's participation, Carol is able to compensate Alice, so that as long as the coalition generates some surplus all parties will be able to obtain a positive payoff.

### Transaction construction liveness assumptions

For a quorum of $n \leq N$ peers, $f$ of which are adversarial, liveness for transaction construction can be achieved (with privacy) by the honest subset of peers in $O(f)$ time. Any defection requires transaction construction to start over. Due to random network disruptions, necessarily some rate of (apparent) defection must be tolerated.

Because exclusivity of spending attempts by inputs can't be enforced without global consensus, and because transaction replacement is desirable for fee market efficiency, we assume inputs are allowed to participate in concurrent sessions which will result in conflicting transactions. Opt in signalling can be used and obeyed as a courtesy, but this ultimately reduces to the same liveness model as transaction construction, potentially requiring additional attempts to recreate the transaction.

### Cost functions

This protocol is mostly agnostic to the cost function that describes the peers' preferences.

Much like in the problem of coin selection (which this work seeks to generalize), the cost function is denominated in sats, and is the combination of objective terms (i.e. the fee cost) and subjective ones (any positive utility obtained from transacting, which must dominate over the objective costs).

Although these are out of scope for this document, privacy related terms might quantify things like how much cover other peers' coins provide as per the sub-transaction model, perceived Sybil resistance against $n-1$ deanonymization attacks (for instance an adversary using older coins incurs a higher cost to do such an attack).

A precondition for liveness is that any such cost function is monotonically decreasing in new information revealed during transaction construction.

Concretely, whenever the action of some peer is revealed (i.e. when an input or output is added), at worst other peers should be indifferent to this, but they may also obtain positive utility. If this is not the case, parties may rationally choose to defect. The purpose of the constraints in proposals is to allow the honest subset peers to avoid such non-malicious conflicts a priori, since monotonicity is required for liveness a peer that engages in the protocol without adhering to this restriction and refuses to sign is deviating from the protocol, and therefore by definition not a member of the honest subset.

## Simplified example

Note that this example omits many details discussed below.

Alice, Bob, Carol and Dave are owners of UTXOs $A$, $B$, $C$ and $D$ respectively.

Alice wishes to CoinJoin with Bob and Carol. She creates a co-spend proposal with UTXOs $\{ A, B, C \}$. In this proposal she adjusts her UTXO $A$ down by 100 sats, and adjusts $B$ and $C$ up by 50 sats each, providing an incentive for Bob and Carol to accept. She sends this to Bob and Carol, who accept, after which the proposal is fully accepted and broadcast.

For all Bob knows, either Carol or Alice made the proposal, and the same goes for Carol not knowing if it was Alice or Bob. Bob only learns that he was offered 50 sats, he doesn't learn how much Carol (or whomever) was offered. All Bob knows is that the 50 sats, which he would receive if a transaction was constructed with this proposal in effect, would be paid for by one of the UTXOs $\{A, C\}$.

Carol also wishes to CoinJoin, with Alice, and Dave. She creates a co-spend proposal with UTXOs $\{ A, C, D \}$, and adjusts $C$ down by 50 sats, $A$ up by 20, and $D$ up by 30. This too is unanimously accepted and broadcast.

Dave then notices both proposals are compatible, and chooses to aggregate them together. He produces a quorum proposal, which depends on these two proposals. The quorum proposal chooses a specific feerate among other things, combining the constraints of both previous proposals. It names $\{ A, B, C, D \}$ as the UTXOs.

Once all parties accept Dave's proposal, consensus has been bootstrapped. Everyone can derive the same total adjustments: $A - 80$, $B + 50$, $C + 0$, $D + 30$. In general only the owner of each coin knows their final adjustment, but in this example Carol knows that Dave obtained $+30$ because her proposal is the only one that applies to his coin, and likewise Alice knows Bob obtained $+50$.

All peers can then proceed to transaction construction, where each party is entitled to add up to their adjusted value's worth in outputs. They may add additional inputs, and arbitrary outputs so long as for each peer, the sum of the outputs' effective cost does not exceed the sum of the adjusted effective values of that peer's inputs.

## Technical Details

### Setup: BIP 322 gossip, online key enrollment

Every spendable coin could potentially be spent in a multiparty transaction (although not all coins can be co-spent). To facilitate this we require a gossip mechanism for BIP-322 ownership proofs, with a replacement and expiration mechanism for flood protection, committing to a public key referred to as the *online key* for the coin.

The purpose of this online key is to authenticate the owner of the UTXO without requiring them to access the spending keys for communication. Anonymous authentication requires the use of ring signatures (and are therefore less likely to be supported by a HWW than BIP-322 proofs), and also simplify the mapping to just a single public key per UTXO regardless of the complexity the `scriptPubKey` might have (complex multisig or ambiguity from multiple spend paths).

Ownership proofs are also used to estimate input weights. For P2TR outputs this indicates the spend path that is intended to be used in any collaborative transactions. Note however that this is not enforceable.

Each proof's endorsement of the online key has a validity interval specified in terms of Bitcoin block height or MTU (valid-after and valid-until). When multiple proofs by the same UTXO are nominally in effect, the one with the latest expiry time takes precedence. Ownership proofs commit to a block hash whose height or MTU is within some set interval of the valid-after field.

#### Flood protection

Online keys may need to be rotated. Since ownership proofs are tied to UTXOs that provides some degree of Sybil protection (identities aren't costless because UTXOs aren't costless), but that is insufficient for flood protection with regards to gossipped messages, since unlike Bitcoin transaction gossip, such messages do not consume any of the scarce resource.

Note that Sybil resistance for flood protection is considered separately of Sybil resistance in the context of CoinJoin $n-1$ deanonymization attacks on CoinJoins. That assumed to be encoded of the cost function, and therefore out of scope for this protocol.

To be accepted (and propagated through gossip) by a peer, any newly made proof associated with a coin must have a hash value (e.g. wtxid of the BIP 322 `to_sign` virtual transaction) numerically smaller all the other proofs already associated with that output which are known to the peer.

Because ownership proofs may be valid at disjoint time intervals, a peer should store up to $k$ proofs (in total, not per validity time interval) in its gossip set for each candidate UTXO, so long as the hashes of *all* of these are numerically smaller than $c_1 + 2^{(c_2 d k)}$, where $d$ is the total duration of all proofs, and $c_i$ are dynamically set policy values (similar to `minrelayfee`, based on local resource limits).

#### Precedence ordering

In addition to rudimentary flood protection for the ownership proof gossip, this defines a clear precedence order. This is so that with respect to a specific block tip, even without global consensus on the set of all ownership proofs it's possible to efficiently to arrive the unambiguous mapping from a particular set of UTXOs to the same set of online keys by sharing with peers any relevant proofs a they might be missing.

If the online keys are generated by hash chain (similar to LN revocation keys), revealing the key can revoke a previous ownership proof, forcing a linear sequence of key updates and providing a mechanism for efficient revocation. However it is unclear at the time of writing whether revocation is required at all and the proof of work approach to flood protection seems sufficient.

Similarly, rate limiting could be done by restricting the rate of broadcast on the basis of the committed block hash, but on its own this would not establish a precedence order and in the absence of consensus on the set of ownership proofs may result in ambiguities about which online key to use.

### Negotiation: listen announcements and/or gossip of partially accepted proposals

Short lived listen announcements may be signed by "makers" using their online keys to indicate that the owner of a UTXO is soliciting proposals pertaining to its spending over the named communication channel. Peers should only gossip listen announcements for currently active or soon to be active online keys. The hash of the signature is used as a flood protection mechanism similarly to ownership proofs.

"Takers" evaluate listening coins, and can construct proposals (defined in the next section) over the specified communication channels.

Alternatively, negotiation with listening but unaddressible UTXO owners can be made possible using an opt-in gossip layer. On this layer it is permitted to broadcast partially accepted proposals. See below for a discussion of flood protection considerations in this setting.

### Co-Spend Proposals

Co-spend proposals specify a set of outpoints of coins intended to be spent together. Fully accepted proposals take effect contingent on the conditions they specify, such as the range of feerates the offer is valid under, `nLocktime` ranges, or whether or not txid stability is required (i.e. only SegWit inputs). The effect of a proposal is some confidential redistribution of the input funds, e.g. fees negotiated between the transacting parties.

#### Making and ratifying co-spend proposals

A proposal specifying $n$ UTXOs can have $1 \leq m \leq n$ linkable ring signatures by the associated online keys of the named UTXOs.

While $m < $n$, a proposal is only partially accepted. Which of the peers have accepted a partial proposal is kept hidden by the ring signatures. This includes keeping the proposer anonymous within this set of signers; technically the proposer is just the first to accept, and at least one peer needs to accept for anti-Sybil flood protection reasons.

If $m=n$ linkable ring signatures with distinct key images have been collected, then a proposal has been unanimously accepted.

Finally, for a proposal to be fully accepted the $n$ linkable ring signatures are replaced with a single Schnorr signature made by the MuSig2 aggregate key formed from the set of associated online keys of the UTXOs. This reduces the the size as well as the computational cost of verifying proposals. Because all parties have accepted, no single party can deny having signed making a joint multisignature is semantically equivalent to a the $n$ individual linkable ring signatures, the distinction is only meaningful during negotiations before all parties have accepted.

Fully accepted proposals can then be broadcast for aggregation. A fully accepted proposal ndicates that all owners of the specified UTXOs have some interest in this proposal being in effect.

Individual proposals commit to a specific set of BIP 322 ownership proofs so that the signatures on them are verifiable. Proposals that refer to expired ownership proofs are still considered valid, and more generally proposals that commit to different ownership proofs are potentially valid as long there is a non-expired ownership proof associated with the UTXO, which is also used for input weight estimations. See the quorum proposal section below for details.

Co-spend proposals are not updatable or revocable once fully accepted, but do have an expiry (specified in block height, MTU, or UTC time). More than one signature on the same proposal should be considered a flooding attack by all participants, and honest parties should only sign (MuSig2) a proposal once. Again, see below for discussion of flood protection with respect to partially accepted proposals.

Co-spend proposals are however able to explicitly include conflict hints for other proposals, indicating that (at least) the proposer of the conflicting proposal would not accept a quorum proposal that depends on the excluded proposals.

#### Effective value adjustments

Proposals verifiably redistribute sats among the UTXOs by adjusting their effective values. Corresponding to each UTXO, a homomorphic value commitment to an adjustment term is included. This adjustment can implement fee payments between peers, and may be positive or negative for every UTXO.

The sum of all the adjustment value commitments in a proposal is a commitment to the sum of the adjustments. This value, the *surplus* of a proposal, is included in cleartext and the sum commitment is proven to commit to it in a balance proof. This too may be positive or negative.

When communicating a partial proposal, the proposer shares with each recipient the opening of the commitment associated with their UTXO. This payoff is denominated in sats, and can be positive or negative.

Each adjustment value commitment is also covered by a range proof certifying that the adjustment is a small positive or negative integer. Small means at least $\lfloor\frac{v}{log_2(v)}\rfloor$, where $v$ is the minimal effective value among all specified UTXOs. This minimum value is computed by taking the highest acceptable feerate in the proposal conditions, and multiplying that by the input weights estimated from the ownership proofs.

The sum of the adjustment commitments, when tweaked by the effective value of the UTXO according to the final feerate of the UTXO, must be a commitment to a positive number of sats. This is ensured by accounting for the maximum values in the range proofs, and ensuring it doesn't overflow the effective value.

Imposing a minimal range width size imposes a limit on how many proposals can be aggregated together, roughly logarithmic in the total value to be spent. Proposals can use wider range proofs than the minimum, which limits their ability to be aggregated and leaks information about the size of the adjustments.

### Proposal aggregation and condition details

A quorum proposal is a special kind of co-spend proposal that aggregates together a bundle of non-conflicting, fully accepted co-spend proposals. The quorum proposal is made by one of the peers, the aggregator, to *all* other peers implicated in the aggregation. The aggregator must also set specific values for any constrained parameters (e.g. a concrete feerate not just a range).

#### Making and ratifying quorum proposals

Any peer may attempt to construct a quorum proposal by aggregating unanimously accepted co-spend proposals together, so long as it controls at least one of the online keys implicated in the aggregation. For rate limiting, the aggregator's key is made explicit and the hash of this signature is used for flood control when gossipping partially signed quorum proposal, much like the hash of ownership proofs.

Unlike co-spend proposals, quorum proposals are accepted with a regular signature by the online key. This makes it tractable for aggregators to make quorum proposals that revise the payoff or omit peers who did not accept. As discussed above, quorum proposals are not mutually exclusive. If a peer rejects a quorum proposal due to the inclusion of a specific proposal it may broadcast a counter final proposal of its own, so there is no mechanism for explicit rejection.

#### Bootstrapping consensus for transaction construction

Once a quorum proposal is fully accepted, a coalition has been formed complete with a payoff imputation. The online keys are used to initiate a consensus protocol in order to construct the full transaction.

Since the quorum proposal addresses every UTXO, its ownership proof set commitment is authoritative. This determines the canonical online key set for the quorum, allowing a consensus protocol to be initialized. The aggregator should use precedence ordering for symmetry breaking in order to minimize additional gossip and improve the chances of the quorum proposal being unanimously accepted, but there is no consensus state for these data at the time the quorum proposal is being authored. The purpose of quorum proposals is to bootstrap both the consensus protocol and incentive compatibility to make use of it.

#### Quorum proposal structure

The surplus of all included co-spend proposals is added to the quorum proposal's adjustment vector, which covers all UTXOs. The quorum proposal's surplus must be positive and needs to cover the shared transaction fields at the set feerate, but otherwise the aggregated surplus is at the discretion of the aggregator, it may be claimed as a coordination fee, redistributed among all parties arbitrarily or included in the mining fees.

The logical conjunction of co-spend proposals implies the union of their adjusted UTXOs and the intersection of their conditions is in effect. The per-UTXO adjustments are collected for each outpoint. The total adjustment for each UTXO may not overflow its effective value, based on the range proof widths.

Co-spend proposals therefore form a commutative semi-group, unioning the sets of UTXOs and summing the adjustments component wise. It's only a semi-group because proposals can't always be aggregated together. One reason for this that the conditions might be mutually exclusive. Another is consensus rules (e.g. `OP_CLTV` making conflicting assertions).

Conditions are intersected elementwise. The intersection of mutually exclusive conditions, for example a time based vs. a height based nLocktime is empty, precluding the combination of such proposals.

The following are all numerical intervals under intersection:

- Transaction consensus fields
  - version
  - individual flag bits
  - nIns
  - (nOuts constrains cannot be enforced and so can't be included)
  - nSequence (constrains all nSequence fields)
  - nLocktime
- Non consensus parameters
  - feerate
  - vbytes allocation per input (nIns must be set accordingly to pass standardness or consensus limits)

The following are sets under intersection

- Allowed input types

### Flood protection for partially accepted proposals

Partially accepted proposals can be communicated directly only between involved peers, or gossipped to third parties. Gossip of partial proposals enables a single proposer to make many nuisance offers involving different sets of keys. Normally linkable ring signatures will protect their identity, which means that other nodes can only use a statistical approach to detect the culprit and may mistakenly penalize the UTXOs the attacker names, without their owners contributing to this flooding attack. "Normally" means key image generator point is just be a hash to curve of the proposal data itself, allowing practically unlimited number of ring signatures per UTXO (exponential in the size of the candidate UTXO set).

To limit this behavior the linkable ring signature key image base point can be time dependent (e.g. making use of a sufficiently buried block's hash as a randomness beacon) and include a truncated hash of the proposal when choosing the generator (by hashing to the curve). Truncating to 10 bits, for example, would allow up to 1024 unrelated proposals to be accepted or made by a single UTXO equivocation per timestep, allowing flooding to be prevented (note however honest users may be restricted from accepting some honest proposals due to the birthday bound, but honest proposers with knowledge of other proposals can grind the proposal data to avoid collisions, and we of course assume attackers can do this without restriction).

Additionally, it is possible to make use of a verifiable encryption scheme for the openings of the commitments, allowing verification of the proposal during gossip, but this does not prevent the broadcast of nuisance proposals that would not be accepted despite being formally valid.

In contrast, in the semi honest setting, and if acceptors don't mind revealing whether they accept to the proposer, the linkable ring signatures can be short cut entirely and MuSig2 negotiated directly, with the proposer as the coordinator between all named parties. MuSig2 signing should still be initiated concurrently even when linkable ring signatures are used in order to minimize the number of round trip.

### PayJoin like payments in multiparty transactions

Payments requests or intents may be encoded in the co-spend proposals using the value adjustment mechanism. Even if fully accepted, such proposals are merely signalling the intention to pay, because finality is contingent on successful collaborative transaction construction.

The sender and receiver must arrange for proposals to transfer the payment amount. This can be achieved by the receiver extending the sender's proposal with additional UTXOs (potentially of unrelated parties, although at least one UTXO should be the receiver's in order to claim the adjustment, and multiple proposals may be required to represent the full payment amount due to the range proof width restriction). The sender and receiver must collaborate to produce a balance proof, as only one or the other knows the openings of some of the homomorphic value adjustments.

With this approach fallback transactions as in BIPs 79-7 is no longer as prudent, since the receiver's UTXOs are much harder to probe. Neither sender nor receiver needs to disclose any UTXOs to each other, only the payment amount is fully known to both.

Although out of scope for this document, peers also negotiate the payment details out of band and arrange for valid registration of the receiver outputs during transaction construction. In this setting the receiver does not need to contribute a UTXO at all and may still get privacy from the sender since their output(s) may be ambiguous, and there are no limitations arising from the use of width limited range proofs.

### Post quantum security & privacy

Apart from the BIP-322 ownership proofs, which are related to spending keys, the signatures used throughout do not require PQ security for privacy (assuming perfectly hiding commitments) or safety, but only for liveness.

However, for privacy it is important that any proofs of knowledge of openings of the commitments do not compromising the unconditional hiding of commitments in order to avoid "harvest now, decrypt later" attacks on anonymity (although out of the scope of this document, this is also required for transaction construction).

Secondly, in addition to the secp256k1 online key, a PQ key for end to end encrypted communication (e.g. the encrypted openings of commitments included in a proposal) can also be used. HPKE is an obvious choice for secp256k1 based encryption, and hybrid HPKE is specified using ML-KEM which makes it a relatively simpler change, but neither these is verifiable without general purpose ZK, which is often costly for the prover or in proof size. Lattice based schemes (without a symmetric cipher) seem potentially amenable to such verifiable encryption, but I have yet to look into that. Verifiable encryption is necessary for partial proposal gossip, as otherwise there is no guarantee that the proposal can even be accepted, as some parties may not learn the opening of the value commitments associated with their UTXOs.

Finally, PQ privacy must of course also assume that metadata privacy at the transport layer is also preserved.

### Transaction construction protocol

A protocol similar to WabiSabi can provide liveness, although in this setting WabiSabi's reliance on KVACs would require one of the peers to serve as the coordinator, and it would be trusted. With modification, this coordinator could prove all transcript messages it accepted are valid, but may still arbitrarily censor certain operations, causing a UTXO to be unfairly blamed for failing to sign.

Technically the quorum proposal can implement a coordination fee mechanism, for example by making all adjustments be slightly negative apart from that of the coordinator (which may or may not be the aggregator). This provides the coordinator with an incentive to coordinate honestly on average. Unfortunately this does little for any targetted censorship risk so long as the payoff from censorship (of e.g. a specific TxOut) dominates over the expected loss of revenue from sabotaging a single attempt at transaction construction. It is likely that the majority of the revenue is still recoverable in a subsequent blame round, as the other participants have no indication that censorship has occurred.

For a more robust mechanism, coalition formation outputs a set of online keys for a quorum, and among other things these are intended to be used to implement byzantine fault tolerant state machine replication for transaction construction. The initial state, based on the quorum proposal, can be proven valid with respect to each UTXO's adjustment sum commitments, which enforces that the agreed to imputation is in effect.

The state machine collects inputs and outputs of a transaction (or tree of transactions a la CoinJoinXT). Since state messages are self authenticating, authenticating using zero knowledge proofs (anonymous credentials or ring signatures), they can be posted anonymously much like in a permissioned blockchain, where all peers act as validators.
