# Overview: Multiparty transaction construction in semi-honest peer threat model

The following is a concrete description of the semi-honest multiparty PayJoin protocol. To understand why certain choices design were made, it is recommended to read the [overview document](./00_overview.md) and [honest protocol document](./honest.md) first.

## Motivation

Two-party and honest versions of the multi party Payjoin protocol preserves privacy of input/output ownership against third-party observers, but it does not preserve privacy from the view of the counterparty itself. E.g In a two-party protocol, each participant can trivially attribute all unknown inputs and outputs to the other party. This reveals cluster information and requires counterparty trust.

With n > 2 and metadata privacy, this privileged view is reduced. Payments and change outputs become ambiguous within the participant set. As a result, participants do not need to trust any specific counterparty with their clustering information. Increasing the number of parties therefore reduces counterparty trust and weakens clustering inferences.

## Threat model

This protocol operates in a semi-honest (honest-but-curious) model. All participants are assumed to economically proximate and thus follow the protocol as specified. They are not expected to deviate from the rules or misbehave. However, they may attempt to learn as much as possible from the messages they observe and the final transaction.

Concretely, if any party learns the full plaintext transcript of messages, they should not be able to determine which inputs or outputs belong to which of the other participants.

The protocol does not assume Byzantine robustness, and it does not attempt to detect or punish misbehavior. If a participant fails to follow through, the protocol may fail, but safety is not compromised - i.e participants will only provide witness if their expected outputs are included in the final transaction.

## Roles

The roles defined in the [honest protocol](./honest.md#roles) apply here without modification. The semi-honest model does not change who the Initiator, Responder, SessionCreator, or Participant are, nor how sessions are initiated. The only difference is that participants in this model may attempt to infer ownership links from observed messages and the final transaction. The communication model is therefore strengthened to prevent this.

## Communication model

In the semi-honest model, participants follow protocol rules but are still curious and may try to infer ownership links from any side channel available to them. For that reason, content encryption alone is not sufficient: if transport metadata reveals who posted which message and when, peers can correlate messages into participant-level clusters.

Metadata privacy is therefore a protocol requirement in this setting. The communication layer must hide sender network identity and reduce linkability across messages, so that learning the transcript does not trivially reveal input-output ownership.

<!-- This is why Iroh gossip is not sufficient here as the primary transport. While Iroh provides efficient dissemination, it does not by itself provide the metadata-hiding guarantees this threat model requires. -->

Separately, dissemination and agreement should be distinguished. Gossip is enough to disseminate messages and can still provide eventual convergence when deterministic merge rules are used. A separate agreement mechanism (some specific instantiation of a lattice agreement protocol) is only needed when stronger guarantees are required for intermediate consistency, timely termination, or recovery under communication disruptions.

Given those tradeoffs, using the BIP77 directory for total order is a simpler direction. A shared append-only mailbox gives participants a practical, common message order to process, which reduces the need to deploy and tune a separate distributed agreement layer. In other words, it combines the metadata privacy this model requires with a straightforward coordination primitive that is easier to implement and operate.

### BIP77 Directory as anonymous broadcast channel

Communication is mediated by a PayJoin directory accessed via OHTTP, following the same metadata privacy model as BIP77. For multiparty use, the directory mailbox supports append semantics. Multiple participants write to the same mailbox, and peers poll and retrieve all appended messages. The mailbox functions as an anonymous broadcast log of encrypted payloads.

#### Shared session secret

A session is defined by a single ephemeral shared secret s. Any participant who learns s can join the session. Knowledge of this secret is the only admission control mechanism.
Participants derive mailbox identifiers and initialize their HPKE context with s. The shared secret is distributed via the existing bidirectional channel. Parties who learn s can both read and write to the mailbox. All payloads are encrypted using HPKE, so the directory only handles opaque ciphertext blobs and cannot link messages to participants.

## Protocol phases

For the canonical phase-by-phase flow, see the honest protocol document: [Overview: Honest multiparty PayJoin](./honest.md#protocol-phases).

### Message Timing

To prevent timing correlation between a participant's messages, each message must be assigned a randomized delay before posting.

If the total number of messages is known in advance, sample n uniform random times within the session window and post each message at its assigned time.

Message publication times are assigned in two phases:

1. Input registration, output registration, and RTS declarations are assigned publication times at the start of the session.
2. Witness provision messages must not be assigned publication times until registration closes.

## Silent Payments

Silent Payment recipients blind their scan-key material and publish it as independent messages to prevent linking a static recipient identifier within the participant set. Unlike the honest model, participants must finalize the input set before deriving the final Silent Payment `scriptPubKey`. Because only the recipient can unblind the recipient-specific material, the protocol must enforce consistent input-set finalization (for example, via an agreed checkpoint or equivalent agreement mechanism) before output construction.
