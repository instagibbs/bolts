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
          * [Pay-To-Anchor (P2A) Output](#pay-to-anchor-p2a-output)
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

The consensus assumptions are:

1. These transactions assume [BIP118](https://github.com/bitcoin/bips/blob/master/bip-0118.mediawiki) (SIGHASH_ANYPREVOUT) in its current written form.

The policy assumptions are:

1. [TRUC Transactions (BIP431)](https://github.com/bitcoin/bips/blob/master/bip-0431.mediawiki): All update, settlement, and closing transactions use `nVersion=3` to signal TRUC (Topologically Restricted Until Confirmation). TRUC transactions enforce:
   - A cluster limit of 2 (1 unconfirmed parent, 1 unconfirmed child)
   - Maximum transaction size of 10,000 vB
   - Maximum child transaction size of 1,000 vB
   - Implicit replaceability signaling (no BIP125 flag required)
   - Zero-fee parent transactions are permitted when packaged with a fee-paying child
1. [Package Relay](https://github.com/bitcoin/bitcoin/blob/master/doc/policy/packages.md): 1-parent-1-child (1p1c) package relay is implemented and deployed, allowing zero-fee TRUC parent transactions to propagate with a fee-paying child.
1. [Package RBF](https://github.com/bitcoin/bitcoin/pull/25038): Package-level replace-by-fee is adopted for transaction pinning mitigation.
1. [Pay-To-Anchor (P2A)](https://github.com/bitcoin/bitcoin/pull/30352): Ephemeral anchor outputs use the P2A output type (`OP_1 <0x4e73>`), which is a keyless, anyone-can-spend witness v1 output. P2A outputs:
   - Require no witness data to spend (empty witness stack)
   - Have a dust threshold of 240 satoshis
   - Are standard as of Bitcoin Core 28.0
1. [Ephemeral Dust](https://github.com/bitcoin/bitcoin/pull/30239): Zero-value P2A outputs are permitted in TRUC transactions, provided the output is spent by a child transaction in the same package. This allows trimmed output value to flow to miner fees via CPFP.
1. Mitigation for ACP-style pinning on APO (preventing 0-value junk inputs from being added to APO transactions).

# Transactions

## Transaction Symmetry

The biggest difference between this BOLT and [BOLT03](03-transactions.md) is the fact that channel updates now have symmetrical state.

Each party in the relevant channel has an identical view of the transactions being collaboratively created. This means that we no longer
rely on revocation and punishment transactions to enforce correctness, but on-chain claim and response periods via two types of transactions:

1. update transactions to make a "claim" of latest channel state
1. settlement transactions to expand out payments once challenge phase ends.

## Transaction Output Ordering

The same rules for output ordering from [BOLT03](03-transactions.md#transaction-output-ordering) applies to update and settlement transactions at time of signing.

## Rationale

## Use of Taproot

All outputs except P2A anchor outputs are pay-to-taproot<sup>[BIP341](https://github.com/bitcoin/bips/blob/master/bip-0341.mediawiki#constructing-and-spending-taproot-outputs)</sup> (P2TR) outputs. We are ommitting all non-script related witness stack items for brevity such as control block, inner pubkeys, et al.

A `<>` designates an empty vector as required for compliance with [BIP342](https://github.com/bitcoin/bips/blob/master/bip-0342.mediawiki)

All taproot leaf versions are 0xC0 unless stated otherwise.

## Use of miniscript

[Miniscript](https://bitcoin.sipa.be/miniscript/) compatibille scripts are used whenever possible to
avoid ambiguity in notation, and allow for further analysis.

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
     * control block: 0xC0 marker for tapscript, internel public key, merkle proof unless spending funding tx
     * tapscript: EXPR_UPDATE(m+1)
     * `signature_for_inner_pubkey`
* txout count: 3
   * `txout[0]` amount: 0
   * `txout[0]` script: P2A (`OP_1 <0x4e73>`, witness v1 program, `bc1pfeessrawgf`)
   * `txout[1]` amount: 0
   * `txout[1]` script: `OP_RETURN <hash_TapLeaf(0xC0 || compact_size(size of EXPR_SETTLE(m)) || EXPR_SETTLE(m))>`
   * `txout[2]` amount: the channel capacity
   * `txout[2]` script: `tr(aggregated_key, {EXPR_UPDATE(n+1), EXPR_SETTLE(n)})`

and where EXPR_SETTLE(n) =

`<CovSig(TL(n))> <1_G> OP_CHECKSIG`

where `CovSig(x)` is the SIGHASH_ALL|ANYPREVOUTANYSCRIPT signature of the corresponding settlement transaction with a
locktime of `x`, and `1_G` the 33-byte BIP118 public key matching the secp256k1 generator `G`, using the [BIP340](https://github.com/bitcoin/bips/blob/master/bip-0340.mediawiki#Default_Signing) "Default Signing" nonce derivation function, with no auxiliary input.

and where `signature_for_inner_pubkey uses SIGHASH_ALL|ANYPREVOUTANYSCRIPT.

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

The `OP_RETURN` output is required to allow recovery of the control block to spend an invalidated update
transaction with the latest update transaction, without requiring participants to hold all invalidated
settlement states forever. This is due to the signature that is included in the script, which cannot
be otherwise deterministically regenerated without access to the full settlement state.
Alternatively, we could add another round-trip to HTLC additions and forwards,
as well as further complicate the protocol.

The "state-masking offset" is used to hide the total number of updates in the channel from
blockchain observers. Future versions of this spec can introduce a randomized negotiation
of this value.

The P2A anchor and OP_RETURN outputs will always be of value 0 (and the state output non-0), so output sorting will result in
the P2A anchor output being first, followed by the OP_RETURN output, and finally the state output.

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

Ideally, we could directly use a raw script to allow maximal flexibility, but we do not want to ensure
that all output scripts given are mempool-standard. We do not need one block relative timelock in scripts
since we can no longer be pinned by package sizes (BIP125 rule #3) or package limits thanks to TRUC
transactions (BIP431) and P2A anchors.

#### HTLC Outputs

The output encumbers funds to the receipient of the HTLC offer with a primage, or back to the offerer upon
timeout. Unlike BOLT03, these require no second stage transactions, and can be signed at any point.

`tr(aggregated_key, {EXPR_SUCCESS, EXPR_TIMEOUT})`

where EXPR_SUCCESS =

`<htlc_pubkey> OP_CHECKSIGVERIFY OP_SIZE <20> OP_EQUALVERIFY OP_HASH160 <H>
OP_EQUAL`

with a policy of `and(pk(htlc_pubkey),hash160(H))`

where `H` is the payment hash and `htlc_pubkey` the *recipient* pubkey

and EXPR_TIMEOUT =

`<htlc_pubkey> OP_CHECKSIGVERIFY N OP_CHECKLOCKTIMEVERIFY`

with policy of `and(after(N),pk(htlc_pubkey))`

where `N` is the HTLC expiry blockheight, and `htlc_pubkey` is the *offerer's* pubkey.

 The key-spend path is currently unused.

The recipient node can redeem the HTLC with the witness:

    <payment_preimage> <recipient_htlc_pubkey_signature>

And the offerer via:

    <offerer_htlc_pubkey_signature>

on an otherwise valid transaction.

#### Pay-To-Anchor (P2A) Output

A single shared anchor output is attached to the settlement transaction. When the settlement
transaction has zero fees (the typical case), this output MUST be spent by a child transaction
in the same relay package to satisfy TRUC rules and be considered for block inclusion.

This output uses the P2A (Pay-To-Anchor) output type:

* scriptPubKey: `OP_1 <0x4e73>` (hex: `51024e73`)
* Witness program: `0x4e73` (2 bytes, witness version 1)
* Address: `bc1pfeessrawgf` (bech32m)
* Output value: the sum of all trimmed output values, minimum 0 satoshis

The P2A output is spent with an empty witness stack (no witness data required), making it
a "keyless" anyone-can-spend output. This allows:

1. Either channel participant to provide CPFP fees
2. Third parties (e.g., watchtowers) to bump fees without requiring key material
3. Minimal on-chain footprint (~13 vB to create, ~41 vB to spend)

The 2-byte witness program `0x4e73` provides no cryptographic security, making it unsuitable
for any use case requiring authentication. As a witness v1 program, the spending input's
scriptSig MUST be empty per BIP141. Witness stuffing is prevented by policy rules that
reject non-empty witness stacks for P2A inputs.

### Trimmed Outputs

Outputs with value below `dust_limit_satoshis` should not be produced;
these outputs that are not produced are termed "trimmed". A trimmed output is
considered too small to be worth creating and is instead added
to the P2A anchor output's value to be used as CPFP fees.

#### Requirements

Fees are handled by spending outputs from the outputs of the settlement transaction,
as the settlement transaction includes no fee itself. This relies on package
relay and package CPFP.

The settlement transaction:
  - for every `to_node` output:
    - if the amount of any of the settlement transaction `to_node` outputs would be
less than `dust_limit_satoshis` set by the channel negotiation:
      - MUST NOT contain that output.
    - otherwise:
      - MUST be generated as specified in [`to_node` Output](#to_node-output).
  - for every HTLC:
    - if the HTLC amount would be less than
    `dust_limit_satoshis`:
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

Both update and settlement transactions are constructed with zero fees, relying on
CPFP (Child-Pays-For-Parent) via the P2A anchor output for propagation and timely block inclusion.

Under TRUC rules (BIP431):
- The parent transaction (update or settlement) can have zero fees
- A child transaction spending the P2A output provides fees for the entire package
- The child transaction is limited to 1,000 vB
- Package RBF allows fee-bumping by replacing the child transaction

Transaction replacement of the CPFP child is the primary mechanism to increase the effective
feerate of the package. Since P2A outputs are keyless, either party (or a third-party watchtower)
can create or replace the fee-bumping child transaction.

## Dust Limits

These are statically set at 330 satoshis. 

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
1. Add a single P2A anchor output.
1. Sort the outputs into [BIP 69+CLTV order](#transaction-output-ordering).

# References

# Authors

Greg Sanders <gsanders87@gmail.com>

![Creative Commons License](https://i.creativecommons.org/l/by/4.0/88x31.png "License CC-BY")
<br>
This work is licensed under a [Creative Commons Attribution 4.0 International License](http://creativecommons.org/licenses/by/4.0/).
