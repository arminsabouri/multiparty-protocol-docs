broadcast channel with a shared secret is a pubsub topic, grow only unordered
set of messages (of known types/sizes? fixed size?)

clients can connect via OHTTP streams to the directory, receiving round robin,
fixed data rate

submit for broadcast (multihop?)

does round robin IBLT-ish dissemination of data

OHTTP stream


ok so here's what i'm thinking:

directory implements some kind of broadcast channel, which is basically an append only mailbox that people can subscribe to via streaming ohttp, except it's an unordered grow only set of bytestrings, not a sequence of bytes, and the streaming over ohttp uses rateless set reconciliation to stream this
27m

clients could then connect to multiple directories at once, and post best effort whatever messages are missing, possibly anonymously, to whichever directory, and clients sync from there
27m

if there's flooding issues, the directory can issue rate limiting tokens to constrain the total ingress bandwidth to at most some limit, and each ohttp stream is some fraction of that
26m

then clients can connect over n streams to m <= n different directories on maillbox H(directory, topic_secret)  where everyone who knows topic_secret, as they would for multiparty payjoin can subscribe to a stream of that data
25m

the more people post the same data to multiple directories the better the resilience of this, it scales horizontally, and they can use more than one mailbox with this too
24m

then when p2p gossip is introduced, it does symmetric, bft set reconciliation with peers, and directories are just a fast path
24m

either the bytestrings are variable length, or fixed in which case max, or we figure out some chunking approach, but regardless this can then be transmitted through IBLTs constructed so that each stream sees a random sample of the data, and towards the end of its life (each stream should only some fraction of the data) the size diminishes slightly more quickly than in the rateless IBLT paper to ensure that something is decodable, allowing progress to be made WRT other parties' IBLTs
21m

in semi-honest setting short keys can be used, and we can assume no collisions, which is way more efficient and means that IBLTs can be subtracted between different directories (this assumes directories are honest, and peers aren't necessarily but the set of all messages including byzantine messages is bounded due to rate limiting, which if peers are also semi honest they would only post valid messages)
18m

and mobile clients can just talk to a few directories and post only to them
18m

and global broadcast is per directory and again just rate limited, using rate limiting tokens tied to BIP 322 proofs (either anonymous credential, or better yet the directory gives an accumulator for utxo set and client proves membership in it with key image for time epoch? this way the directory can restrict like min coin age, etc etc and privacy is stronger)
15m

so submit your bip-322s via (multihop) ohttp, directories wish to share them, it's like an auto expiring pgp key server basically
14m

tied to UTXOs
14m

then directories can use this to authenticate clients with privacy, and for good measure they can allow authentication by all p2tr keys too using curve tree stuff
14m

so the anonymity set is much larger than just the bip 322 certified online keys at least wrt directories
14m

and then against such proofs you get a rate limiting credential good for some data rate valid for say 0.9* min coin age in the ring selected by the directory
12m

so you GET to query its ring conditions to see if you qualify, and then you ask for a credential, and you can do public broadcast on that directory up to that data rate
12m

and i guess topics are salted hash of an accumulator and you prove membership in the accumulator to the directory when you post on them, key image is combination of one your utxo rate limiting keys and one of these keys, used to generate a per topic nullifier that gives you some total rate limit for non public broadcast independent of that


