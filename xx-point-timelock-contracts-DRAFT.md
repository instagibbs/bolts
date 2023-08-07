# Extension BOLT XX: Point Timelock Contracts (MuSig)

## Abstract

Description here is for a non-stuckless, simplified update, MuSig2-based construction.

## Motivation

## Mechanism Overview

Point Timelock Contracts differ from HTLCs in that the opening of a commitment is not the revelation
of a hash preimage, but rather the revelation of a discrete logarithm.

The mechanism is best illustrated by an example:

![](./figma_stuckable_ptlcs_01.png)

In the above diagram, let's drill down into the communication between Carol and Dave. In an HTLC
scenario, Carol would send Dave an `update_add_htlc` message containing various fee information,
as well as an onion packet containing the information sent by Alice.

In the PTLC scenario, however, Carol must instead prepare a commitment that would be opened by
a discrete log, i. e. a signature verification tied to a specific public key.

Further, because Dave completing the joint MuSig2 signature with Carol must necessarily reveal to Carol how to open the commitment that
she, in turn, received from Bob, we must work with Adaptor signatures, which means that we need
one signature set up beforehand, which does not open the commitment, and one signature afterwards
that does. The delta between those signatures must be fixed beforehand.

`P_c` would be Carol's PTLC pubkey, could be the `htlc_basepoint` or similar.
`P_d` etc. derived similarly.

In such a scheme, Carol could send Dave the following:

1. Carol sends Dave the following:
   - `Z + A + B + C`: the aggregated PTLC point that Dave will need to open
   - `R_c`: a random MuSig2 nonce
   - Alice's onion containing sender-secret `d`, which Dave can use to create his subsequent PTLC with Emily
2. Dave responds to Carol with the following:
   - `R_d`: a random MuSig2 nonce
   - a MuSig2 partial signature: `s_d = h(R_c + R_d + A+B+C+Z || P_c + P_d || m) * x_d + r_d`
3. Finally, Carol responds to Dave:
   - complementary MuSig2 partial signature: `s_c = h(R_c + R_d + A+B+C+Z || P_c + P_d || m) * x_c`
4. Now Dave can perform that same Dance with Emily.

Carol and Dave now both have the complete signature where the following holds:

- `s' = s_c + s_d = h(R_c + R_d + A+B+C+Z || P_c + P_d || m) * (x_c + x_d) + r_c + r_d`
- `s'` is not a valid signature
- A valid signature would be `s = h(R_c + R_d + A+B+C+Z || P_c + P_d || m) * (x_c + x_d) + r_c + r_d + z+a+b+c`
- `s - s' = z + a + b + c`
- Carol cannot unilaterally produce a valid signature
- Dave can produce a valid signature if he learns `z+a+b+c`, but only by modifying `s'`
  - This guarantees that if ever a valid signature is produced, it is necessarily one from which
    Carol can infer `z+a+b`.

To "bind" this PTLC, the updated commitment transaction should require an `OP_CHECKSIGVERIFY` where
the public keys `P_c` and `P_d` are combined via MuSig2.

## Messaging Changes

As can be gleamed from the description above, the PTLC counterpart to `update_add_htlc` cannot
be a single message, because we now require additional roundtrips in the protocol.

I propose we split up the PTLC initialization into three messages: `update_offer_ptlc`,
`update_accept_ptlcs`, `commitment_signed`, and `countersign_ptlcs`.

        +-------+                                 +-------+
        |       |--(1)---- update_offer_ptlc ---->|       |
        |       |--(2)---- update_offer_ptlc ---->|       |
        |       |--(3)--- commitment_signed ----->|       | Bob knows all MuSig nonces for Alice, commit tx sig
        |       |                                 |       |
        |   A   |<-(4)---- revoke_and_ack --------|   B   | Implicit addition of PTLCs to Alice's commit tx
        |       |<-(5)---- update_accept_ptlcs -- |       | Alice knows Bob's nonces + psigs for both commit txs PTLCs
        |       |                                 |       |
        |       |--(6)--- countersign_ptlcs ----->|       | Bob knows psigs for for both commit tx's PTLCs
        |       |                                 |       |
        |       |<-(7)--- commitment_signed ------|       | Alice now knows Bob's commit tx sigs
        |       |                                 |       |
        |       |--(8)---- revoke_and_ack ------->|       |
        |       |                                 |       | -> Can now forward
        +-------+                                 +-------+

FIXME can we move stuff around? Let's decouple messages a bit...

        +-------+                                 +-------+ Alice's turn
        |       |--(1)---- update_offer_ptlc ---->|       | new amounts, lock info
        |       |--(2)---- update_offer_ptlc ---->|       | new amounts, lock info
        |       |--(3)--- commitment_signed ----->|       | Bob knows full local commit tx sig for Alice
        |       |--(4)--- ptlc_nonces ----------->|       | Bob knows all MuSig PTLC nonces for Alice, remote and local, offered and not
        |       |                                 |       |
        |   A   |<-(5)---- revoke_and_ack --------|   B   | Implicit addition of PTLCs to Alice's commit tx
        |       |<-(6)--- ptlc_nonces ------------|       | Alice knows all MuSig nonces for Alice
        |       |<-(7)---- sign_ptlcs ------------|       | Alice knows Bob's psigs for both commit txs PTLCs
        |       |<-(8)--- commitment_signed ------|       | Alice now knows Bob's commit tx sigs
        |       |                                 |       | Bob wasn't allowed to add his own PTLCs so Alice can finish up
        |       |--(9)--- countersign_ptlcs ----->|       | Bob knows psigs for for both commit tx's PTLCs
        |       |                                 |       |
        |       |--(10)--- revoke_and_ack ------->|       | Alice is now committed, Bob can now safely forward
        |       |                                 |       |
        +-------+                                 +-------+

1.5 RTT? What am I missing? Is this actually secure? What happens if Alice/Bob have commit tx sigs but not PTLCs? Async may be problematic?

Assume there was already a live PTLC from Bob to Alice.

(3) With `commitment_signed` Alice updating Bob's tx invalidates Bob's received PTLCs and Alice's offered PTLCs
Bob can go to chain with commit tx, can't retrieve incoming
Alice can trigger timeouts
Seems ok.

(5) If Bob revokes, he can't go to chain with latest commit tx with existing PTLC-Success txs!
Alice is no longer "committed" :( Bob needs complete psigs(9???) from Alice before revoking

Sync I Think Works(TM):
        +-------+                                 +-------+ Alice's turn
        |       |--(1)---- update_offer_ptlc ---->|       | new amounts, lock info
        |       |--(2)---- update_offer_ptlc ---->|       | new amounts, lock info
        |       |--(3)--- commitment_signed ----->|       | Bob knows full local commit tx sig for Alice
        |       |--(4)--- local_nonce ----------->|       | Bob's local PTLC txs nonces from Alice
        |       |--(5)--- remote_nonce----------->|       | Bob's remote PTLC txs nonces from Alice
        |       |                                 |       |
        |       |<-(6)--- local_nonce-------------|       | Bob's remote *tx* nonce from Bob
        |       |<-(7)--- remote_nonce------------|       | Bob's local tx nonce from Bob
        |       |<-(8)--- local_psig--------------|       | Bob's remote *tx* psig from Bob
        |       |<-(9)--- remote_psig-------------|       | Bob's local tx psig from Bob
        |       |                                 |       |
        |       |--(10)-- local_psig ------------>|       | Bob's local tx psig from Alice (Bob can now safely go to chain)
        |       |--(11)-- remote_psig ----------->|       | Bob's remote tx psig from Alice (Bob can now spend received PTLCS on either tx)
        |       |                                 |       |
        |   A   |<-(12)--- revoke_and_ack --------|   B   | Bob can broadcast latest state safely, revoke older (ACK ignored?)
        |       |<-(13)-- commitment_signed ------|       | Alice now knows Bob's sigs for Alice's commit tx
        |       |                                 |       | Bob wasn't allowed to add his own PTLCs so Alice can finish up
        |       |--(14)--- revoke_and_ack ------->|       | Alice is now committed, Bob can now safely forward
        |       |                                 |       |
        +-------+                                 +-------+

2.RTT, matches expectation. Async case I think adds another RTT since you can't predict your local transactions' txid/sighash.

Async
        +-------+                                 +-------+
        |       |<-(0)---- update_offer_ptlc -----|       | new amounts, lock info
        |       |--(1)---- update_offer_ptlc ---->|       | new amounts, lock info
        |       |--(2)---- update_offer_ptlc ---->|       | new amounts, lock info
        |       |--(3)--- commitment_signed ----->|       | Bob knows full local commit tx sig for Alice
        |       |--(4)--- local_nonce ----------->|       | Bob's local PTLC txs nonces from Alice
        |       |--(5)--- remote_nonce----------->|       | Bob's remote PTLC txs nonces from Alice
        |       |                                 |       |
        |       |<-(6)--- local_nonce-------------|       | Bob's remote *tx* nonce from Bob
        |       |<-(7)--- remote_nonce------------|       | Bob's local tx nonce from Bob
        |       |<-(9)--- remote_psig-------------|       | Bob's local tx psig from Bob
        |       |                                 |       |
        |       |--(10)-- local_psig ------------>|       | Bob's local tx psig from Alice (Bob can now safely go to chain)
        |       |                                 |       |
        |   A   |<-(12)--- revoke_and_ack --------|   B   | Bob can broadcast latest state safely, revoke older
        |       |<-(13)-- commitment_signed ------|       | Alice now knows Bob's sigs for Alice's commit tx
        |       |<-(8)--- local_psig--------------|       | Bob's remote *tx* psig from Bob, now that remote tx is fixed
        |       |                                 |       |
        |       |--(11)-- remote_psig ----------->|       | Bob's remote tx psig from Alice (Bob can now spend received PTLCS on either tx)
        |       |--(14)--- revoke_and_ack ------->|       | Alice is now committed, Bob can now safely forward
        |       |                                 |       |
        +-------+                                 +-------+

Also 2.5RTT. Is it safe? *receiver* of the PTLC needs to get psig from counter-party first
*{local, remote}_psigs sending pattern of receiver-first is complicated*

TODO quickly sketch out the single-sig adaptor case

Here are output labels because I get confused so often what things in BOLTs mean:
(1) Alice-offered "offered PTLC" in Alice's tx
(2) Alice-offered "received PTLC" in Bob's tx
(3) Bob-offered "received PTLC" in Alice's tx
(4) Bob-offered "offered PTLC" in Bob's tx

Single-sig Adaptor, sync
        +-------+                                 +-------+
        |       |                                 |       |
        |       |--(1)---- update_offer_ptlc ---->|       | new amounts, lock info
        |       |--(2)---- update_offer_ptlc ---->|       | new amounts, lock info
        |       |--(3)--- commitment_signed ----->|       | Alice's B_commit sig, Alice's presignatures for (1), (2), and full sig for (4)
        |       |                                 |       |
        |   A   |<-(4)--- revoke_and_ack ---------|   B   | Bob can broadcast latest state safely, revoke older
        |       |<-(5)-- commitment_signed -------|       | Bob's A_commit sig, Bob's presignatures for (3), (4), and full sig for (1)
        |       |                                 |       |
        |       |--(6)--- revoke_and_ack -------->|       | Alice is now committed, Bob can now safely forward
        |       |                                 |       |
        +-------+                                 +-------+

Pretty simple, pretty sure safe? I don't think Bob can send his own commitment until Alice has provided(1)'s presignature?
If Bob sends commit tx sigs etc, Alice can take newest commit tx to chain, with no no success-path to receive an already existing
PTLC Bob's forwarded.

A TLV extension of `commitment_signed` to include the "self adaptor sigs" seems most straight forward here, no new message
patterns to speak of.

Single-sig Adaptor, *async*
        +-------+                                 +-------+
        |       |<-(0)---- update_offer_ptlc -----|       | new amounts, lock info
        |       |                                 |       |
        |       |--(1)---- update_offer_ptlc ---->|       | new amounts, lock info
        |       |--(2)---- update_offer_ptlc ---->|       | new amounts, lock info
        |       |--(3)--- commitment_signed ----->|       | Alice's B_commit sig, Alice's presignatures for (2) and full sig for (4)
        |       |                                 |       |
        |   A   |<-(4)--- revoke_and_ack ---------|   B   | Bob can broadcast local latest state safely, revoke older
        |       |<-(5)--- self_ptlc_plz ----------|       | Bob says he's done, please give your local PTLC presignatures
        |       |                                 |       |
        |       |--(6)--- self_ptlc_signed ------>|       | Alice gives Bob her local presignatures, Bob can now give commit sig...
        |       |                                 |       |
        |       |<-(7)-- commitment_signed -------|       | Bob's A_commit sig, Bob's presignatures for (3) and full sig for (1)
        |       |                                 |       |
        |       |--(8)--- revoke_and_ack -------->|       | Alice is now committed, Bob can now safely forward
        |       |                                 |       |
        +-------+                                 +-------+

2.5 RTT? Bob can't safely send `commitment_signed` until he has presignatures for all existing PTLCs, but structure
needs to be fixed, so Bob needs to signal Alice that he's done updating. Or for 1.5RTT  Alice has to start shooting off presingatures
and Bob sends `commitment_signed` once he gets the thing he wants. O(n^2) adaptor sigs seems uhhh not great.

APO fixes this.

Let's try MuSig again...

sync with MuSig:

        +-------+                                 +-------+ Alice's turn
        |       |--(1)---- update_offer_ptlc ---->|       | new amounts, lock info
        |       |--(2)---- update_offer_ptlc ---->|       | new amounts, lock info
        |       |--(3)--- commitment_signed ----->|       | Bob knows full local commit tx sig for Alice
        |       |--(4)--- a_o_btx_nonce --------->|       | For all Alice-offered on Bob's tx (2)
        |       |--(5)--- a_o_atx_nonce---------->|       | For all Alice-offered on Alice's tx (1)
        |       |--(X)--- b_o_btx_nonce --------->|       | For all Bob-offered on Bob's tx (4)
        |       |--(X)--- b_o_atx_nonce---------->|       | For all Bob-offered on Alice's tx (3)
        |       |                                 |       |
        |       |<-(6)--- a_o_atx_nonce-----------|       | For all Alice-offered on Alice's tx (1)
        |       |<-(7)--- a_o_btx_nonce-----------|       | For all Alice-offered on Bob's tx (2)
        |       |<-(X)--- b_o_atx_nonce-----------|       | For all Bob-offered on Alice's tx (3)
        |       |<-(X)--- b_o_btx_nonce-----------|       | For all Bob-offered on Bob's tx (4)
        |       |<-(8)--- a_o_atx_psig------------|       | Bob psigning Alice-offered PTLC in Alice tx (1)
        |       |<-(9)--- a_o_btx_psig------------|       | Bob psigning Alice-offered PTLC in Bob tx (2)
        |       |                                 |       |
        |       |--(10)-- a_o_atx_psig----------->|       | Alice psigning Alice-offered PTLC in Alice tx (1)
        |       |--(11)-- a_o_btx_psig----------->|       | Alice psigning Alice-offered PTLC in Bob tx (2)
        |       |--(XX)-- b_o_atx_psig----------->|       | Alice psigning Bob-offered PTLC in Alice tx (3) 
        |       |--(XX)-- b_o_btx_psig----------->|       | Alice psigning Bob-offered PTLC in Bob tx (3) 
        |       |                                 |       |
        |       |<-(XX)--- b_o_atx_psig-----------|       | Bob psigning Bob-offered PTLC in Alice tx (3)
        |       |<-(XX)--- b_o_btx_psig-----------|       | Bob psigning Bob-offered PTLC in Bob tx (4)
        |   A   |<-(12)--- revoke_and_ack --------|   B   | All Alice-offered PTLCs locked in, new tx safe for Bob
        |       |<-(13)-- commitment_signed ------|       | Alice now knows Bob's sigs for Alice's commit tx
        |       |                                 |       | Bob wasn't allowed to add his own PTLCs so Alice can finish up
        |       |--(14)--- revoke_and_ack ------->|       | Alice is now committed, Bob can now safely forward
        |       |                                 |       |
        +-------+                                 +-------+

2.5RTT, let's play some ordering games...

*handwave nonce presharing by sending a new nonce along with each psig*
        +-------+                                 +-------+ Alice's turn
        |       |--(1)---- update_offer_ptlc ---->|       | new amounts, lock info
        |       |--(2)---- update_offer_ptlc ---->|       | new amounts, lock info
        |       |--(3)--- commitment_signed ----->|       | Bob knows full local commit tx sig for Alice
        |       |                                 |       |
        |       |<-(8)--- a_o_atx_psig------------|       | Bob psigning Alice-offered PTLC in Alice tx (1)
        |       |<-(9)--- a_o_btx_psig------------|       | Bob psigning Alice-offered PTLC in Bob tx (2)
        |       |                                 |       |
        |       |--(10)-- a_o_atx_psig----------->|       | Alice psigning Alice-offered PTLC in Alice tx (1)
        |       |--(11)-- a_o_btx_psig----------->|       | Alice psigning Alice-offered PTLC in Bob tx (2)
        |       |--(XX)-- b_o_atx_psig----------->|       | Alice psigning Bob-offered PTLC in Alice tx (3) 
        |       |--(XX)-- b_o_btx_psig----------->|       | Alice psigning Bob-offered PTLC in Bob tx (3) 
        |       |                                 |       |
        |       |<-(XX)--- b_o_atx_psig-----------|       | Bob psigning Bob-offered PTLC in Alice tx (3)
        |       |<-(XX)--- b_o_btx_psig-----------|       | Bob psigning Bob-offered PTLC in Bob tx (4)
        |   A   |<-(12)--- revoke_and_ack --------|   B   | All Alice-offered PTLCs locked in, new tx safe for Bob
        |       |<-(13)-- commitment_signed ------|       | Alice now knows Bob's sigs for Alice's commit tx
        |       |                                 |       | Bob wasn't allowed to add his own PTLCs so Alice can finish up
        |       |--(14)--- revoke_and_ack ------->|       | Alice is now committed, Bob can now safely forward
        |       |                                 |       |
        +-------+                                 +-------+

can we get rid of a RTT by pushing psigs around?
        +-------+                                 +-------+ Alice's turn
        |       |--(1)---- update_offer_ptlc ---->|       | new amounts, lock info
        |       |--(2)---- update_offer_ptlc ---->|       | new amounts, lock info
        |       |--(3)--- commitment_signed ----->|       | Bob knows full local commit tx sig for Alice
        |       |--(XX)-- b_o_atx_psig----------->|       | Alice psigning Bob-offered PTLC in Alice tx (3) 
        |       |--(XX)-- b_o_btx_psig----------->|       | Alice psigning Bob-offered PTLC in Bob tx (3) 
        |       |                                 |       |
        |       |<-(8)--- a_o_atx_psig------------|       | Bob psigning Alice-offered PTLC in Alice tx (1)
        |       |<-(9)--- a_o_btx_psig------------|       | Bob psigning Alice-offered PTLC in Bob tx (2)
        |       |                                 |       |
        |       |<-(XX)--- b_o_atx_psig-----------|       | Bob psigning Bob-offered PTLC in Alice tx (3)
        |       |<-(XX)--- b_o_btx_psig-----------|       | Bob psigning Bob-offered PTLC in Bob tx (4)
        |   A   |<-(12)--- revoke_and_ack --------|   B   | All Alice-offered PTLCs locked in, new tx safe for Bob
        |       |<-(13)-- commitment_signed ------|       | Alice now knows Bob's sigs for Alice's commit tx
        |       |  ^^^ nope                       |       | Bob wasn't allowed to add his own PTLCs so Alice can finish up
        |       |--(10)-- a_o_atx_psig----------->|       | Alice psigning Alice-offered PTLC in Alice tx (1)
        |       |--(11)-- a_o_btx_psig----------->|       | Alice psigning Alice-offered PTLC in Bob tx (2)
        |       |--(14)--- revoke_and_ack ------->|       | Alice is now committed, Bob can now safely forward
        |       |                                 |       |
        +-------+                                 +-------+

Again, Bob can't send `commitment_signed` until he has *all* Alice-offered PTLCs on Alice tx, otherwise
Alice can go to chain with her tx, invalidating his incoming PTLCs he's already forwarded!

Think MuSig/sync is stuck with 2.5RTT fundamentally.

Let's try *async* MuSig, handwave away nonces like before

        +-------+                                 +-------+ Alice's turn
        |       |<-(0)---- update_offer_ptlc -----|       | new amounts, lock info
        |       |                                 |       |
        |       |--(1)---- update_offer_ptlc ---->|       | new amounts, lock info
        |       |--(2)---- update_offer_ptlc ---->|       | new amounts, lock info
        |       |--(3)--- commitment_signed ----->|       | Bob knows full local commit tx sig for Alice
        |       |                                 |       |
        |       |<-(9)--- a_o_btx_psig------------|       | Bob psigning Alice-offered PTLC in Bob tx (2)
        |       |                                 |       |
        |       |--(11)-- a_o_btx_psig----------->|       | Alice psigning Alice-offered PTLC in Bob tx (2)
        |       |--(XX)-- b_o_btx_psig----------->|       | Alice psigning Bob-offered PTLC in Bob tx (3) 
        |       |                                 |       |
        |       |<-(XX)--- b_o_btx_psig-----------|       | Bob psigning Bob-offered PTLC in Bob tx (4)
        |   A   |<-(12)--- revoke_and_ack --------|   B   | All Alice-offered PTLCs locked in, new tx safe for Bob
        |       |<-(XX)--- done ------------------|       | Bob is done, atx is fixed, psig time
        |       |<-(8)--- a_o_atx_psig------------|       | Bob psigning Alice-offered PTLC in Alice tx (1)
        |       |                                 |       |
        |       |--(10)-- a_o_atx_psig----------->|       | Alice psigning Alice-offered PTLC in Alice tx (1)
        |       |--(XX)-- b_o_atx_psig----------->|       | Alice psigning Bob-offered PTLC in Alice tx (3) 
        |       |                                 |       |
        |       |<-(XX)--- b_o_atx_psig-----------|       | Bob psigning Bob-offered PTLC in Alice tx (3)
        |       |<-(13)-- commitment_signed ------|       | Alice now knows Bob's sigs for Alice's commit tx
        |       |                                 |       | Bob wasn't allowed to add his own PTLCs so Alice can finish up
        |       |--(14)--- revoke_and_ack ------->|       | Alice is now committed, Bob can now safely forward
        |       |                                 |       |
        +-------+                                 +-------+

3.5RTT, and I don't see a way to remove a RTT since each offering side in atx needs to "go first" once

It seems MuSig adds 1 RTT, and async-PTLCs 1 RTT.

### `update_offer_ptlc`

1. type: 128.1 (`update_offer_ptlc`)
2. data:
   - `channel_id`: `channel_id`
   - `u64`: `id`
   - `u64`: `amount_msat`
   - `u32`: `cltv_expiry`
   - `pubkey`: PTLC commitment pubkey (e. g. `A+B+C+Z`)
   - `1366*byte`: `onion_routing_packet`

This message is sent for every new PTLC to be added.

### `commitment_signed` Extensions

Pubnonces are pre-sent by offerer for each live and untrimmed PTLC

FIXME should be a list for all live (untrimmed?) PTLCs
1. `tlv_stream`: `commitment_signed_tlvs`
2. types:
   1. type: 10 (`ptlc_remote_pubnonces`)
   2. data:
      - `66*byte`: MuSig2 pubnonce (for remote commit tx, this update)
   1. type: 10 (`ptlc_local_pubnonces`)
   2. data:
      - `66*byte`: MuSig2 pubnonce (for local commit tx, this update)
XXX   1. type: XX (`ptlc_partial_sig`)
XXX   2. data:
XXX      - `32*byte`: MuSig2 partial signature (for local commit tx, this update)

Rationale: Public nonces are sent only at signature time since we'll
need MuSig pubnonces for every live PTLC in every commitment transaction.
We can't send partial signatures in this step, because we need to have
the recipient's partial sig for safe recovery in an on-chain event.

Since we're doing simplified updates, both offerer and receiver know
what the complete state of each commitment transaction is by this point, allowing
partial signatures to be sent after this.

`revoke_and_ack` sent here by recipient for implicit addition of PTLCs?

If there are live PTLCs for recipient, then respond with:

FIXME should just be a list of this sent all at once?
1. type: 128.2 (`update_accept_ptlcs`)
2. data:
   - `channel_id`: `channel_id
   - `u64`: `id`
   - `66*byte`: MuSig2 pubnonce (for this local update)
   - `66*byte`: MuSig2 pubnonce (for this remote update)
   - `32*byte`: MuSig2 partial signature for local(receiver) commit tx
   - `32*byte`: MuSig2 partial signature for remote(offerer) commit tx

Rationale: Recipient cannot send a partial signature until the entire commitment transaction
structure is known. `update_accept_ptlc` is sent immediately in response to `commitment_signed`
for any live, untrimmed PTLC in the corresponding commitment transaction.

If ANYPREVOUT was active, we could send the requisite PTLC partial signatures just in time,
and only once for the lifetime of the PTLC.

Now that offerer has all necessary information for the hop lock, they send:

FIXME should be list
1. type: 128.2 (`countersign_ptlcs`)
2. data:
   - `channel_id`: `channel_id
   - `u64`: `id`
   - `66*byte`: MuSig2 pubnonce (for this local update)
   - `66*byte`: MuSig2 pubnonce (for this remote update)
   - `32*byte`: MuSig2 partial signature for local(receiver) commit tx
   - `32*byte`: MuSig2 partial signature for remote(offerer) commit tx

Then the receiver does `commitment_signed`, receives `revoke_and_ack`, offerer is now committed!

2.5 RTT

Non-async means you have to break apart `update_accept_ptlc` and `commitment_ptlc` into messages that
are sent only when each commitment transaction has been determined from the other side's updates. This
potentially adds another RTT if both sides are offering PTLCs, coming to 3.5 total.


## Sphinx Changes

### Hop Data Extensions

1. `tlv_stream`: `tlv_payload`
2. types:
   1. type 10 (`ptlc_secret`)
   2. data:
      - `32*byte`: `ptlc_secret` (e. g. the `d` sent from Alice to David)

## Script Changes

The only script changes are to the `htlc_success`, now `ptlc_success` branch of the Taptree.

### Offered PTLCs

`ptlc_success`:

```
<ptlc_musig_pubkey> OP_CHECKSIGVERIFY // SIGHASH_SINGLE | SIGHASH_ANYONECANPAY
<remote_ptlc_pubkey> OP_CHECKSIG // SIGHASH_ALL
OP_CHECKSEQUENCEVERIFY
```

#### Rationale

We need the receiver's key to be added for anti-DoS/fee reasons. If pinning were not a problem, this
wouldn't be required.

### Accepted PTLCs

`ptlc_success`:

```
<ptlc_musig_pubkey> OP_CHECKSIGVERIFY // SIGHASH_SINGLE | SIGHASH_ANYONECANPAY
<local_ptlc_pubkey> OP_CHECKSIG // SIGHASH_ALL
```

Notes: Seems like a bad tradeoff if we already have two keys/sigs in tapscript if we can just
do two single-sigs.

MuSig and single-sig adapters with tapscript `SIGHASH_SINGLE|ACP`:
33+32=55WU control block
~34x2=68WU pubkeys revealed in tapscript
~65x2=130WU signatures
receiver can inflate PTLC-Success tx (A -> B -> C. C pins B->C timeout, A executes timeout for A->B, C does PTLC-Success OOB RBF)

vs MuSig keyspend `SIGHASH_SINGLE|ACP`
65WU witness data total
188WU cheaper
*anyone* can inflate PTLC-Success tx

vs MuSig keyspend `SIGHASH_ALL` with ephemeral anchor
(8+1+1)x4=40WU anchor output creation
(36+1+4)x4=164WU
10x4=40WU transaction overhead for CPFP
== 244WU in CPFP overhead
no pinning

#### Rationale

We need the receiver's key to be added for anti-DoS/fee reasons. If pinning were not a problem, this
wouldn't be required.
