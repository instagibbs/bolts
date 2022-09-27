# BOLT #??: Bitcoin Transaction and Script Formats for Eltoo Channels

This details the exact format of on-chain transactions, which both sides need to agree on to ensure signatures are valid. This consists of the funding transaction output script, the update transactions, and the settlement transactions.

# Table of Contents

  * [Mempool Policy and Consensus Changes Required](#policy-and-consensus)
  * [Transactions](#transactions)
    * [Transaction Output Ordering](#transaction-output-ordering)
    * [Use of Taproot](#use-of-segwit)
    * [Funding Transaction Output](#funding-transaction-output)
    * [Update Transaction](#update-transaction)
    * [Settlement Transaction](#settlement-transaction)
        * [Settlement Transaction Outputs](#settlement-transaction-outputs)
          * [`to_node` Output](#to_node-output)
          * [HTLC Outputs](#htlc-outputs)
          * [Ephemeral Anchor Output](#anchor-output)
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

# Mempool Policy and Consensus Changes Required

The rest of the document presumes a specific set of mempool policy enhancements and
consensus changes.

roughly detailed [here](https://gist.github.com/instagibbs/b3095752d6289ab52166c04df55c1c19).

The consensus assumptions are:

1. These transactions assume [BIP118](https://github.com/bitcoin/bips/blob/master/bip-0118.mediawiki) in its current written form.
1. No SIGHASH_GROUP like functionality, so we either commit via SINGLE or ALL in each situation.
1. We rely on both the "re-binding" functionality as designed in the original eltoo paper, as well
as the signature in output covenant trick that was discovered by other researchers.

The policy assumptions and implications in short are roughly detailed [here](https://gist.github.com/instagibbs/b3095752d6289ab52166c04df55c1c19).
in short:

1. Package relay is implemented and deployed, as per [this link](https://github.com/bitcoin/bips/pull/1324)
1. [This policy](https://lists.linuxfoundation.org/pipermail/bitcoin-dev/2022-September/020937.html) is adopted for rule#3 pinning mitigation.
1. Ephemeral Anchors are implemented. This new output type must be immediately spent in package relay.
   This output is allowed dust-level values, including zero.
1. Annex data is allowed, up to at least 33 bytes worth for our purposes.

Alternative designs not implemented:

1. Don't timelock balance outputs in settlement transactions, and rely on mempool carve-out
in two-party channel setups to avoid mempool limit pinning. HTLC outputs still need to be locked
and this does not generalize to N-party. The benefits would be that funds can go directly to a destination
wallet of arbitrary type and also be allowed to directly pay for fees using CPFP. We instead choose to design a system
that fairly naively scales to N-party channels within the limits of mempool policy development today.
1. Use SIGHASH_GROUP. This would allow more flexible bring your own fees for settlement
transactions, as it allows the creation of additional secure change outputs. The motivation
here is fairly weak, and requires another BIP to be designed, implemented, and accepted
by the community for limited gain.

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

Most transaction outputs used here are pay-to-taproot<sup>[BIP341](https://github.com/bitcoin/bips/blob/master/bip-0341.mediawiki#constructing-and-spending-taproot-outputs)</sup> (P2TR) outputs. We are ommitting all non-script related witness stack items for brevity such as control block, inner pubkeys, et al.

A `<>` designates an empty vector as required for compliance with [BIP342](https://github.com/bitcoin/bips/blob/master/bip-0342.mediawiki)

All taproot leaf versions are 0xC0 unless stated otherwise.

## Use of miniscript

[Miniscript](https://bitcoin.sipa.be/miniscript/) is used whenever possible to avoid ambiguity in notation,
and allow for further analysis.

## Funding Transaction Output

* The value is the channel capacity.

TL(n) = `500000000+o+n`

* Where `o` equals the state-masking offset(always 0 for now)

* Where `n` equals the channel state version, starting at 0

* The funding output script is a P2TR to:

`sorted_pubkey1, sorted_pubkey2 = KeySort(<set of funding_pubkey fields negotiated>)`
`aggregated_key = KeyAgg(sorted_pubkey1, sorted_pubkey2)`
`tr(aggregated_key, EXPR_UPDATE(0))`


* where EXPR_UPDATE(x) =

`<1> OP_CHECKSIGVERIFY <TL(x)> OP_CHECKLOCKTIMEVERIFY` if `x > 0`
with the policy of `and(pk(1),after(TL(x)))`

else

`<1> OP_CHECKSIG`, in the case of `x == 0`
with the policy of `pk(1)`

* Output descriptors as defined by [BIP386](https://github.com/bitcoin/bips/blob/master/bip-0386.mediawiki#tr) and abused by the author.

* Where `KeyAgg` and `KeySort` are defined as per BIP-musig2.

## Update Transaction

* version: 3
* locktime: TL(n), where `n` is the state number for this update transaction
* txin count: 1
   * `txin[0]` outpoint: `txid` and `output_index` from last published state `m` output (can be 0, the funding output)
   * `txin[0]` sequence: 0xFFFFFFFD
   * `txin[0]` script bytes: 0
   * `txin[0]` witness:
     * annex: 0x50 followed by ''hash<sub>TapLeaf</sub>(0xC0 || compact_size(size of EXPR_SETTLE(m)) || EXPR_SETTLE(m))''
     * control block: 0xC0 marker for tapscript, internel public key, merkle proof unless spending funding tx
     * tapscript: EXPR_UPDATE(m+1)
     * `signature_for_inner_pubkey`
* txout count: 1
   * `txout[0]` amount: the channel capacity
   * `txout[0]` script: `tr(aggregated_key, {EXPR_UPDATE(n+1), EXPR_SETTLE(n)})`

and where EXPR_SETTLE(n) =

`<CovSig(TL(n))> <1_G> OP_CHECKSIG`

where `CovSig(x)` is the SIGHASH_ALL|ANYPREVOUTANYSCRIPT signature of the corresponding settlement transaction with a
locktime of `x`, and `1_G` the 33-byte BIP118 public key matching the secp256k1 generator `G`, using the [BIP340](https://github.com/bitcoin/bips/blob/master/bip-0340.mediawiki#Default_Signing) "Default Signing" nonce derivation function, with no auxiliary input.

and where `signature_for_inner_pubkey uses SIGHASH_SINGLE|ANYPREVOUTANYSCRIPT.

Note that the locktime must increase monotonically as it's used as the consensus ratchet for allowing rebinding of updates.

### Rationale

Anyprevout style covenants are used in the update transaction taptree to avoid requiring
additional communication round-trips for forwarding HTLC messages. If we required a
standard MuSig2 signature for the settlement transaction, it would be unsafe to hand
your counterparty both update and settlement signatures in one step, as the counterparty
could ransom your funds by publishing a completed update transaction, and withhold the
final settlement signature.

We use the public generator G as a "well known" point, where privkey is `1`,
so the signature can be recreated by any software, including watchtowers. The script
does not need rotation since the signature is committed to directly in the script,
and the signature directly commits to the settlement transaction, including the nlocktime
which is unique per update in a given channel.

The `annex` is required to allow recovery of the control block to spend an invalidated update
transaction with the latest update transaction, without requiring particpants to hold all invalidated
settlement states forever. This is due to the signature that is included in the script, which cannot
be otherwise deterministically regenerated without access to the full settlement state.
 Alternatively, we could add another round-trip to HTLC additions and forwards,
as well as further complicate the protocol. FIXME we may want to devise an OP_RETURN like marking
that will never be used in a softfork definition.

The "state-masking offset" is used to hide the total number of updates in the channel from
blockchain observers. Future versions of this spec can introduce a randomized negotiation
of this value.

## Settlement Transaction

* version: 3
* locktime: corresponding update transaction's locktime
* txin count: 1
   * `txin[0]` outpoint: `txid` and `output_index` from latest committed state `n` output
   * `txin[0]` sequence: set to `shared_delay`, initially set in channel open negotiation
   * `txin[0]` script bytes: 0
   * `txin[0]` witness:
     * control block: merkle proof
     * tapscript: EXPR_SETTLE(n)
     * (no additional witness data needed)

Note there may be additional attached transaction inputs due to the ANYPREVOUTANYSCRIPT signatures which can be used to attach fees during settlement.

### Settlement Transaction Outputs

Since we are relying on the `shared_delay` timelock to ensure that the final update transaction confirmed on the blockchain
is indeed the final update in the channel, we do not require revocation paths or additional delays to spend the outputs
contained in the settlement transaction. We have no second-stage pre-signed transactions, simply outputs that can be
spent by the authorized parties. For balance outputs, these can be immedietly spent by the owners. For HTLC outputs,
these can be spent immediately once the receiving party obtains the preimage, or can be clawed back by the offerer
once the CLTV of the output expires.

The lack of second-stage transactions means that HTLCs must have timeouts longer than the `shared_delay` timeout to ensure on-chain enforcement.

Alternative proposals such as [layered commitments for eltoo](https://lists.linuxfoundation.org/pipermail/lightning-dev/2020-January/002448.html) have been proposed, but require increased complexity and additional layer violations.

The amounts for each output MUST be rounded down to whole satoshis. If this amount is less than the `dust_limit_satoshis` set by the owner of the output, the output MUST NOT be produced (thus the funds add to fees).

#### `to_node` Output

This output sends funds back to the owner of the satoshi amount. It can be claimed without delay. The output is a P2TR of the form:

`rawtr(settlement_pubkey)`

There are `N` copies of this output, one for each channel participant and their associated `settlement_pubkey` sent during channel negotiation.

##### Rationale

Ideally, we could directly use a `rawtr(settlement_pubkey)` or even a raw script to allow maximal flexibility,
but we have mempool package limit considerations to contend with. This means all settlement outputs must be
timelocked, or we use the carve-out rule in two-party scenarios, but this does not scale to N-party channels.

#### HTLC Outputs

The output encumbers funds to the receipient of the HTLC offer with a primage, or back to the offerer upon
timeout. Unlike BOLT03, these require no second stage transactions, and can be signed at any point.

`tr(aggregated_key, {EXPR_SUCCESS, EXPR_TIMEOUT})`

where EXPR_SUCCESS =

`<settlement_pubkey> OP_CHECKSIGVERIFY OP_SIZE <32> OP_EQUALVERIFY OP_HASH160 <H>
OP_EQUALVERIFY 1 OP_CHECKSEQUENCEVERIFY`

with a policy of `and(pk(settlment_pubkey),and(hash160(H),older(1)))`

where `H` is the payment hash and `settlement_pubkey` the *recipient* pubkey

and EXPR_TIMEOUT =

`<N> OP_CHECKLOCKTIMEVERIFY OP_VERIFY <settlement_pubkey> OP_CHECKSIGVERIFY 1
OP_CHECKSEQUENCEVERIFY`

with policy of `and(after(N),and(pk(settlement_pubkey),older(1)))`

where `N` is the HTLC expiry blockheight, and `settlement_pubkey` is the *offerer's* pubkey.

 The key-spend path is currently unused.

The recipient node can redeem the HTLC with the witness:

    <recipient_settlemet_pubkey_signature> <payment_preimage>

And the offerer via:

    <offerer_settlement_pubkey_signature>

with the proper nlocktime set to include in the next block.

#### Ephemeral Anchor Output

A single shared anchor output, also known as ephemeral dust, is attached to the settlement
transaction. This MUST be spent in a relay package to be considered for block inclusion.

This output contains the scriptpubkey of:

`OP_TRUE`, the output value is the value of all the trimmed output values, summed.

This allows an "empty" spend of the output by anyone, and no known mempool malleabilty vectors.

There is no consensus-level fix for malleability with bare scripts that lack a sighash,
so that is the best we are going to do without committing to an expensive script hash.

### Trimmed Outputs

Peers agree on a `dust_limit_satoshis` below which outputs should
not be produced; these outputs that are not produced are termed "trimmed". A trimmed output is
considered too small to be worth creating and is instead added
to the anchor output.

#### Requirements

Fees are handled by spending outputs from the outputs of the settlement transaction,
as the settlement transaction includes no fee itself. This relies on package
relay and package CPFP.

The update transaction:
  - for every `to_node` output:
    - if the amount of any of the settlement transaction `to_node` outputs would be
less than `dust_limit_satoshis` set by the channel negotiation:
      - MUST NOT contain that output.
    - otherwise:
      - MUST be generated as specified in [`to_node` Output](#to_node-output).
  - for every HTLC:
    - if the HTLC amount would be less than
    `dust_limit_satoshis` set by the negotiation:
      - MUST NOT contain that output.
    - otherwise:
      - MUST be generated as specified in
      [HTLC Outputs](#htlc-outputs).

## Closing Transaction

Note that there are two possible variants for each node.

* version: 3
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
on CPFP of the single anchor output for propogation and timely inclusion into the blockchain.

Transaction replacement of the CPFP can be a way to increase the feerate of the overall
package, even if a counterparty is making CPFP transactions.

FIXME add details about how much weight these transactions
will be with BIP341/342, for CPFP reasons?

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
1. Add a single ephemeral output anchor.
1. Sort the outputs into [BIP 69+CLTV order](#transaction-output-ordering).

# References

# Authors

[ FIXME: ]

![Creative Commons License](https://i.creativecommons.org/l/by/4.0/88x31.png "License CC-BY")
<br>
This work is licensed under a [Creative Commons Attribution 4.0 International License](http://creativecommons.org/licenses/by/4.0/).
