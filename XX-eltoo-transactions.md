# BOLT #??: Bitcoin Transaction and Script Formats for Eltoo Channels

This details the exact format of on-chain transactions, which both sides need to agree on to ensure signatures are valid. This consists of the funding transaction output script, the update transactions, and the settlement transactions.

# Table of Contents

  * [Transactions](#transactions)
    * [Transaction Output Ordering](#transaction-output-ordering)
    * [Use of Segwit](#use-of-segwit)
    * [Funding Transaction Output](#funding-transaction-output)
    * [Update Transaction](#update-transaction)
    * [Settlement Transaction](#settlement-transaction)
        * [Settlement Transaction Outputs](#settlement-transaction-outputs)
          * [`to_node` Output](#to_node-output)
          * [HTLC Outputs](#htlc-outputs)
        * [Trimmed Outputs](#trimmed-outputs)
	* [Closing Transaction](#closing-transaction)
    * [Fees](#fees)
        * [Fee Calculation](#fee-calculation)
        * [Fee Payment](#fee-payment)
    * [Dust Limits](#dust-limits)
    * [Commitment Transaction Construction](#commitment-transaction-construction)
  * [Keys](#keys)
  * [References](#references)
  * [Authors](#authors)

# Transactions

## Transaction Symmetry

The biggest difference between this BOLT and [BOLT03](03-transactions.md) is the fact that channel updates now have symmetrical state.

Each party in the relevant channel has an identical view of the transactions being collaboratively created. This means that we no longer
rely on revocation and punishment transactions to enforce correctness, but on-chain claim and response periods via two types of transactions:
update transactions to make a "claim" of latest transaction version, and settlement transactions to expand out payments once the delay expires.

## Transaction Output Ordering

The same rules for output ordering from [BOLT03](03-transactions.md#transaction-output-ordering) applies to settlement transactions.

## Rationale

Only the settlement transaction has multiple outputs that are collaboratively built with the channel peer, therefore it can only be
enforced here.

## Use of Taproot

Most transaction outputs used here are pay-to-taproot<sup>[BIP341](https://github.com/bitcoin/bips/blob/master/bip-0341.mediawiki#constructing-and-spending-taproot-outputs)</sup> (P2TR) outputs. To spend such outputs FIXME note what we're omitting from witness

A `<>` designates an empty vector as required for compliance with MINIMALIF-standard rule.<sup>[MINIMALIF](https://lists.linuxfoundation.org/pipermail/bitcoin-dev/2016-August/013014.html)</sup>

## Funding Transaction Output

* The funding output script is a P2TR to:

FIXME: define a sorted MuSig2 ordering macro
`tr(MuSig2(<pubkey1>, <pubkey2>), EXPR)`

where

`EXPR = 0 0<pubkey1> OP_CHECKSIGADD 0<pubkey2> OP_CHECKSIGADD 2 OP_EQUAL`

* As defined by [BIP386](https://github.com/bitcoin/bips/blob/master/bip-0386.mediawiki#tr) and abused by the author.

* Where `pubkey1` is the lexicographically lesser of the two `funding_pubkey` in compressed format, and where `pubkey2` is the lexicographically greater of the two, and each public key has `0` prepended onto the data push to indicate ANYPREVOUT signature.

The key-spend path is used during collaborative channel closes, while for simplicity naive signature aggregation is used for update
messages to reduce the required amount of p2p changes and state. These naive updates can be converted to MuSig2 on a future version.

## Update Transaction

* version: 2
* locktime: 500000000+o+k, where `o` equals the state-masking offset and `k` equals the channel state version
* txin count: 1
   * `txin[0]` outpoint: `txid` and `output_index` from latest state `k` output (can be 0, the funding output)
   * `txin[0]` sequence: 0xFFFFFFFE
   * `txin[0]` script bytes: 0
   * `txin[0]` witness: `<signature_for_pubkey1> <signature_for_pubkey2>`
* txout count: 1
   * `txout[0]` amount: the HTLC amount minus fees (see [Fee Calculation](#fee-calculation))
   * `txout[0]` script: `tr(MuSig2(<pubkey1>, <pubkey2>), EXPR)`

where EXPR =

FIXME: define the key ordering correctly with the sorting
`<locktime+1>` OP_CLTV OP_DROP 0 0<pubkey1> OP_CHECKSIGADD 0<pubkey2> OP_CHECKSIGADD 2 OP_EQUAL`

and where `signature_for_pubkey1 and `signature_for_pubkey1` use SIGHASH_SINGLE|ANYPREVOUTANYSCRIPT.

Note that the locktime must increase monotonically as it's used as the consensus ratchet for allowing rebinding of updates.

## Settlement Transaction

* version: 2
* locktime: latest update transaction's locktime + 1
* txin count: 1
   * `txin[0]` outpoint: `txid` and `output_index` from latest committed state `k` output
   * `txin[0]` sequence: set to `shared_delay`, initially set in channel open negotiation
   * `txin[0]` script bytes: 0
   * `txin[0]` witness: `<signature_for_pubkey1> <signature_for_pubkey2>`

where `signature_for_pubkey1 and `signature_for_pubkey1` use SIGHASH_ALL|ANYPREVOUT.

Note there may be additional attached transaction inputs due to the ANYPREVOUT signatures which can be used to attach fees during settlement.

### Settlement Transaction Outputs

Since we are relying on the `shared_delay` timelock to ensure that the final update transaction confirmed on the blockchain
is indeed the final update in the channel, we do not require revocation paths or additional delays to spend the outputs
contained in the settlement transaction. We have no second-stage pre-signed transactions, simply outputs that can be
spent by the authorized parties. For balance outputs, these can be immediently spent by the owners. For HTLC outputs,
these can be spent immediately once the receiving party obtains the preimage, or can be clawed back by the offerer
once the CLTV of the output expires.

The lack of second-stage transactions means that HTLCs must have timeouts longer than the `shared_delay` timeout to ensure on-chain enforcement.

Alternative proposals such as [layered commitments for eltoo](https://lists.linuxfoundation.org/pipermail/lightning-dev/2020-January/002448.html) have been proposed, but require increased complexity and additional layer violations.

The amounts for each output MUST be rounded down to whole satoshis. If this amount is less than the `dust_limit_satoshis` set by the owner of the output, the output MUST NOT be produced (thus the funds add to fees). FIXME is this another consensus problem, since HTLCs can go either way?

#### `to_node` Output

This output sends funds back to the owner of the satoshi amount. It can be claimed without delay. The output is a P2TR of the form:

`rawtr(<settlement_pubkey>)`

There are `N` copies of this output, one for each channel partner and their associated `settlement_pubkey` sent during channel negotiation.

Since these outputs can be immediately spent, they can be used for CPFP fee bumping as required to achieve confirmation.

#### HTLC Outputs

The output encumbers funds to the receipient of the HTLC offer with a primage, or back to the offerer upon
timeout. Unlike BOLT03, these require no second stage transactions, and can be signed at any point.

`tr(MuSig2(<pubkey1>, <pubkey2>), {SUCCESS, TIMEOUT})`
FIXME leaves need to have defined leaf type? switch to miniscript notation?

where SUCCESS =

    OP_HASH160 <RIPEMD160(payment_hash)> OP_EQUALVERIFY <recipient_funding_pubkey> OP_CHECKSIG

and TIMEOUT =

    <timeout sufficient for safety> OP_CLTV <offerer_funding_pubkey> OP_CHECKSIG

with the same pubkeys and ordering as the funding transaction output. The key-spend path is currently unused.

The recipient node can redeem the HTLC with the witness:

    <remotehtlcsig> <payment_preimage>

And the offerer via:

    <offerer_funding_pubkey_signature>

with the proper nLocktime set

### Trimmed Outputs

FIXME figure out dust limit stuff, consensus problem again?

Peers agree on a `dust_limit_satoshis` below which outputs should
not be produced; these outputs that are not produced are termed "trimmed". A trimmed output is
considered too small to be worth creating and is instead added
to the commitment transaction fee.

#### Requirements

Fees are handled by spending outputs from the outputs of the settlement transaction,
as the settlement transaction includes no fee itself. This relies on package
relay and package CPFP.

The update transaction:
  - for every `to_node` output:
    - if the amount of any of the settlement transaction `to_node` outputs would be
less than `dust_limit_satoshis` set by the transaction owner:
      - MUST NOT contain that output.
    - otherwise:
      - MUST be generated as specified in [`to_noe` Output](#to_node-output).
  - for every HTLC:
    - if the HTLC amount would be less than
    `dust_limit_satoshis` set by the negotiation:
      - MUST NOT contain that output.
    - otherwise:
      - MUST be generated as specified in
      [HTLC Outputs](#htlc-outputs).

## Closing Transaction

Note that there are two possible variants for each node.

* version: 2
* locktime: 0
* txin count: 1
   * `txin[0]` outpoint: `txid` and `output_index` from `funding_created` message
   * `txin[0]` sequence: 0xFFFFFFFF
   * `txin[0]` script bytes: 0
   * `txin[0]` witness: `<signature_for_musig2_pubkey>`
* txout count: 0, 1 or 2
   * `txout` amount: final balance to be paid to one node
   * `txout` script: as specified in that node's `scriptpubkey` in its `shutdown` message

### Requirements

Each node offering a signature:
  - MUST round each output down to whole satoshis.
  - MUST remove any output below its own `dust_limit_satoshis`.
  - MAY eliminate its own output.

### Rationale

Contrary to ln-penalty based chanels, we have a shared `dust_limit_satoshis`
and a symmetrical transaction. Users can still opt to remove their own output
and the signature will indicate to the peer which variant has been used.

There will be at least one output, if the funding amount is greater
than twice `dust_limit_satoshis`.

## Fees

The settlement transaction contains no fees, so implementers must rely
on CPFP of `to_node` and HTLC outputs for propogation and timely inclusion into the blockchain.

FIXME add details about how much weight these transactions
will be with BIP341/342, for CPFP reasons?

FIXME should we CSV delay all HTLC outputs to disallow package limit based pinning? e.g. peer
spends received HTLC with preimage, makes it huge, then also spends their `to_node` amount
as well? If they're CSV locked, then settlement txs can only be settled properly if
user has settled balance already. Maybe an argument for not ommitting your output ala anchors
or switching to APO with no commitment to input value.

## Dust Limits

The `dust_limit_satoshis` parameter is used identically to BOLT03 except:

### Unknown segwit versions

Unknown segwit versions excludes BIP341 outputs defined as:

- 8 bytes for the output amount
- 1 byte for the script length
- 34 bytes for the script (`OP_1` followed by a single push of 32 bytes)

In Bitcoin Core the dust limit is FIXME

## Update and Settlement Transaction Construction

This section ties the previous sections together to detail the
algorithm for constructing the update and settlement transactions for both peers:
given that peer's `dust_limit_satoshis`:

### Update Transaction

This transaction is relatively trivial to set up.

1. Initialize the update transaction input, output, and locktime as specified
   in [Update Transaction](#update-transaction).

### Settlement Transaction

1. Initialize the settlement transaction input and locktime, as specified
   in [Settlement Transaction](#settlement-transaction).
1. Calculate which committed HTLCs and `to_node` outputs need to be trimmed (see [Trimmed Outputs](#trimmed-outputs)).
1. For every HTLC, if it is not trimmed, add an
   [HTLC output](#htlc-outputs).
1. For every `to_node` output, if it is not trimmed,
   add a [`to_node` output](#to_local-output).
1. Sort the outputs into [BIP 69+CLTV order](#transaction-output-ordering).

# References

# Authors

[ FIXME: ]

![Creative Commons License](https://i.creativecommons.org/l/by/4.0/88x31.png "License CC-BY")
<br>
This work is licensed under a [Creative Commons Attribution 4.0 International License](http://creativecommons.org/licenses/by/4.0/).
