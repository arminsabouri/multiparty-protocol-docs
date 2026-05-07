### meta/TODO

The immediate goal of this document is just to be a minimal list of fields that
are necessary and safe to share.

Ideally this document should comprehensively categorize fields by privacy
implications wrt different kinds of peers, for all BIP 174 & 370 roles.

---

There should be clear diagrams showing how the different roles interact around
the privacy boundary. For example, Creator role can already add a bunch of
private information in single party setting (`createfundedpsbt`) and this is
appropriate. (nested) Multisig may involve multiple privacy boundaries, and the
udpater role may interact in complicated ways with this.

---

## Abstract

PSBT creation, construction, updating, and signing may all add privacy
sensitive information to PSBTs.

When exchanging (modifiable) PSBTs during construction (BIP 370 or otherwise)
or signing with semi-trusted or untrusted counterparties, only some fields are
necessary to share, and privacy can be harmed significantly if certain fields
that are required for a signing device for example are leaked to an untrusted
party.

## Specification

### Input fields

All data that eventually winds up in the chain is fine:

- construction
- prev txid and spent output index
- nsequence
- combiner (signing)
- witness or non witness utxo
- finalized witnesses/scriptSigs

Non-finalized signature data:: this is a bit more naunced and needs evaluation
of the other BIPs, in particular tapscript stuff, because not all data comitted
in the script needs to make it to the chain, depending on the spend paths.

All other fields should only be shared with trusted parties (e.g. own devices,
within an organization, etc)

### Output fields

Consensus fields:

- value
- script
- depending on the situation bip 375 silent payments info field may or may not be appropriate to share

Derivation paths etc definitely need to be scrubbed.

### Global fields

Most BIP 174 or 370 global fields are fine but anything related to BIP 32 paths
or keys, multisig, miniscript etc should not be included.


