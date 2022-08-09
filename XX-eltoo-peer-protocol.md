# BOLT #??: Peer Protocol for Eltoo Channel Management

The peer eltoo channel protocol has three phases: establishment, normal
operation, and closing.

# Table of Contents

  * [Channel](#channel)
    * [Definition of `channel_id`](#definition-of-channel_id)
    * [Channel Establishment](#channel-establishment)
      * [The `open_channel_eltoo` Message](#the-open_channel_eltoo-message)
      * [The `accept_channel_eltoo` Message](#the-accept_channel_eltoo-message)
      * [The `funding_created_eltoo` Message](#the-funding_created_eltoo-message)
      * [The `funding_signed_eltoo` Message](#the-funding_signed_eltoo-message)
      * [The `funding_locked_eltoo` Message](#the-funding_locked_eltoo-message)
    * [Channel Close](#channel-close)
      * [Closing Initiation: `shutdown_eltoo`](#closing-initiation-shutdown)
      * [Closing Negotiation: `closing_signed_eltoo`](#closing-negotiation-closing_signed)
    * [Eltoo Normal Operation](#eltoo-operation)
      * [Forwarding HTLCs](#forwarding-htlcs)
      * [`cltv_expiry_delta` Selection](#cltv_expiry_delta-selection)
      * [Adding an HTLC: `update_add_htlc`](#adding-an-htlc-update_add_htlc)
      * [Removing an HTLC: `update_fulfill_htlc`, `update_fail_htlc`, and `update_fail_malformed_htlc`](#removing-an-htlc-update_fulfill_htlc-update_fail_htlc-and-update_fail_malformed_htlc)
      * [Committing Updates So Far: `update_signed`](#committing-updates-so-far-update_signed)
      * [Completing the Transition to the Updated State: `update_signed_ack`](#completing-the-transition-to-the-updated-state-update_signed_ack)
      * [Updating Fees: `update_fee`](#updating-fees-update_fee)
    * [Message Retransmission: `channel_reestablish` message](#message-retransmission)
  * [Authors](#authors)

# Channel

## Definition of `channel_id`

Same as BOLT02 definition

## Channel Establishment

### The `open_channel_eltoo` Message

When `option_eltoo` is negotiated, this message contains information about
a node and indicates its desire to set up a new eltoo channel. This is the first
step toward creating the funding transaction and the initial update transaction. 

1. type: 32778 (`open_channel_eltoo`)
2. data:
   * [`chain_hash`:`chain_hash`]
   * [`32*byte`:`temporary_channel_id`]
   * [`u64`:`funding_satoshis`]
   * [`u64`:`push_msat`]
   * [`u64`:`dust_limit_satoshis`]
   * [`u64`:`max_htlc_value_in_flight_msat`]
   * [`u64`:`htlc_minimum_msat`]
   * [`u16`:`shared_delay`]
   * [`u16`:`max_accepted_htlcs`]
   * [`point`:`funding_pubkey`]
   * [`point`:`settlement_pubkey`]
   * [`byte`:`channel_flags`]
   * [`nonce`:`next_nonce`]
   * [`open_channel_eltoo_tlvs`:`tlvs`]

1. `tlv_stream`: `open_channel_eltoo_tlvs`
2. types:
    1. type: 0 (`upfront_shutdown_script`)
    2. data:
        * [`...*byte`:`shutdown_scriptpubkey`]
    1. type: 1 (`channel_type`)
    2. data:
        * [`...*byte`:`type`]

#### Requirements

Changed fields from `open_channel`:
  - `to_self_delay` is replaced with a symmetrical `shared_delay` which must be agreed upon by nodes. This is currently set by the opener.
  - `dust_limit_satoshis` must be shared, and is currently set by the opener. A negotiation protocol can be added in future versions.
  - there is no `revocation_basepoint` as the security of the eltoo design does not rely on penalty transactions
  - there is no `delayed_payment_basepoint`, as there are no second-stage HTLC transactions to be pre-signed
  - `payment_basepoint` is replaced with a static `settlement_pubkey`
  - no `feerate_per_kw` as there is no up-front negotiated fee for update or settlement transactions
  - `first_per_commitment_point` is removed
  - there is no `channel_reserve_satoshis`
  - `next_nonce`

Sending node:
  - SHOULD set `shared_delay` to a reasonable number

A receiving node:
  - MUST either accept the `shared_delay` given by the sender, of fail the channel
  - if `next_nonce` is not a valid BIP-musig2 nonce
    - MUST send a `warning` and close the connection, or send an
      `error` and fail the channel.
  - otherwise MUST store and apply `next_nonce` to the associated `funding_created` transaction,
      unless channel reestablishment occurs, in which the nonces must be discarded

#### Rationale

The symmetrical transaction state among all peers means that we can simplify some aspects while
requiring additional range negotiation in others.

Reserve requirements are removed as there are no penalty transactions.

#### Defined Eltoo Channel Types

The currently defined types are:
  - FIXME option_scid_alias and option_zeroconf now exist, support them
  - no features (no bits set)

### The `accept_channel_eltoo` Message

This message contains information about a node and indicates its
acceptance of the new eltoo channel initiated by `open_channel_eltoo`. This
is the second step toward creating the funding, update, and settlement
transactions.

1. type: 32769 (`accept_channel_eltoo`)
2. data:
   * [`32*byte`:`temporary_channel_id`]
   * [`u64`:`dust_limit_satoshis`]
   * [`u64`:`max_htlc_value_in_flight_msat`]
   * [`u64`:`htlc_minimum_msat`]
   * [`u32`:`minimum_depth`]
   * [`u16`:`shared_delay`]
   * [`u16`:`max_accepted_htlcs`]
   * [`point`:`funding_pubkey`]
   * [`point`:`settlement_pubkey`]
   * [`nonce`:`next_nonce`]
   * [`accept_channel_eltoo_tlvs`:`tlvs`]

1. `tlv_stream`: `accept_channel_eltoo_tlvs`
2. types:
    1. type: 0 (`upfront_shutdown_script`)
    2. data:
        * [`...*byte`:`shutdown_scriptpubkey`]
    1. type: 1 (`channel_type`)
    2. data:
        * [`...*byte`:`type`]

#### Requirements

The same requirements as `accept_channel`, except a few redundant fields removed.

Sending node:
  - MUST set `dust_limit_satoshis` to the same value as set in `open_channel`
  - MUST set `shared_delay` to the same value as set in `open_channel`

  - if `next_nonce` is not a valid BIP-musig2 nonce
    - MUST send a `warning` and close the connection, or send an
      `error` and fail the channel.
  - otherwise MUST store and apply `next_nonce` to the associated `funding_signed` transaction,
      unless channel reestablishment occurs, in which the nonces must be discarded


#### Rationale

Symmetrical state means fewer parameters are required compared to `accept_channel` channel type.

### The `funding_created_eltoo` Message

This message describes the outpoint which the funder has created for
the initial update and settlement transactions. After receiving the peer's
signature, via `funding_signed_eltoo`, it will broadcast the funding transaction.

1. type: 32770 (`funding_created_eltoo`)
2. data:
    * [`32*byte`:`temporary_channel_id`]
    * [`sha256`:`funding_txid`]
    * [`u16`:`funding_output_index`]
    * [`partial_sig`:`update_psig`]
    * [`nonce`:`next_nonce`]

#### Requirements

Requirements are identical to `funding_created` except:

There is no `signature` field sent, or required.

The sender MUST set:
  - `update_psig` to the valid partial signature using its `funding_pubkey` for the initial update transaction and the previously sent `nonce`s
    from `open_channel_eltoo` and `accept_channel_eltoo` messages, as defined in [BOLT #3](03-transactions.md#update-transaction).
  - `next_nonce` to a valid BIP-musig2 nonce to be used for the next channel update during the same connection

The recipient:
  - if `update_psig` is incorrect:
    - MUST send a `warning` and close the connection, or send an
      `error` and fail the channel.
  - if `next_nonce` is not a valid BIP-musig2 nonce
    - MUST send a `warning` and close the connection, or send an
      `error` and fail the channel.
  - otherwise MUST store and apply `next_nonce` to the next channel update,
      unless channel reestablishment occurs, in which the nonces must be discarded

#### Rationale

Eltoo style channels require two pre-signed transactions for backing out value, rather than one.

### The `funding_signed_eltoo` Message

This message gives the funder the signature it needs for the first
update transaction, so it can broadcast the transaction knowing that funds
can be redeemed, if need be. It is sent in response to `funding_created_eltoo`.

This message introduces the `channel_id` to identify the channel. It's derived from the funding transaction by combining the `funding_txid` and the `funding_output_index`, using big-endian exclusive-OR (i.e. `funding_output_index` alters the last 2 bytes).


1. type: 32771 (`funding_signed_eltoo`)
2. data:
    * [`channel_id`:`channel_id`]
    * [`partial_sig`:`update_psig`]
    * [`nonce`:`next_nonce`]

#### Requirements

Both peers:
  - if `channel_type` was present in both `open_channel` and `accept_channel` and is not empty(no bits set):
    - MUST send a `warning` and close the connection, or send an
      `error` and fail the channel.
  - otherwise:
    - the `channel_type` is empty
  - MUST use that `channel_type` for all transactions.

The sender MUST set:
  - `channel_id` by exclusive-OR of the `funding_txid` and the `funding_output_index` from the `funding_created_eltoo` message.
  - `update_psig` to the valid BIP-musig2 partial signature, using its `update_pubkey` for the initial update transaction, and the previously sent `nonce`s
    from `open_channel_eltoo` and `accept_channel_eltoo` messages as defined in [BOLT #3](03-transactions.md#update-transaction).
  - `next_nonce` to a valid BIP-musig2 nonce to be used for the next channel update during the same connection

The recipient:
  - if `update_psig` is incorrect:
    - MUST send a `warning` and close the connection, or send an
      `error` and fail the channel.
  - MUST NOT broadcast the funding transaction before receipt of a valid `funding_signed_eltoo`.
  - on receipt of a valid `funding_signed_eltoo`:
    - SHOULD broadcast the funding transaction.
  - if `next_nonce` is not a valid BIP-musig2 nonce
    - MUST send a `warning` and close the connection, or send an
      `error` and fail the channel.
  - otherwise MUST store and apply `next_nonce` to the next channel update,
      unless channel reestablishment occurs, in which the nonces must be discarded


#### Rationale

Eltoo style channels require two pre-signed transactions for backing out value, rather than one.

### The `funding_locked_eltoo` Message

This message indicates that the funding transaction has reached the `minimum_depth` asked for in `accept_channel_eltoo`. Once both nodes have sent this, the channel enters normal operating mode.

1. type: 32772 (`funding_locked_eltoo`)
2. data:
    * [`channel_id`:`channel_id`]

#### Requirements

The sender MUST:
  - NOT send `funding_locked_eltoo` unless outpoint of given by `funding_txid` and
   `funding_output_index` in the `funding_created_eltoo` message pays exactly `funding_satoshis` to the scriptpubkey specified in [BOLT #3](03-transactions.md#funding-transaction-output).
  - wait until the funding transaction has reached `minimum_depth` before
  sending this message.

A non-funding node (fundee):
  - SHOULD forget the channel if it does not see the correct funding
  transaction after a timeout of 2016 blocks.

From the point of waiting for `funding_locked_eltoo` onward, either node MAY
send an `error` and fail the channel if it does not receive a required response from the
other node after a reasonable timeout.

#### Rationale

Same rationale as ln-penalty.

## Eltoo Channel Close

Channel closes are nearly identical to BOLT02 channel closes, with the main
difference being the `closing_signed_eltoo` requiring 2 rounds for nonce
sharing to make a MuSig2 signature.

Closing happens in two stages:
1. one side indicates it wants to clear the channel (and thus will accept no new HTLCs)
2. once all HTLCs are resolved, the final channel close negotiation begins.

        +-------+                                    +-------+
        |       |--(1)-----  shutdown_eltoo -------->|       |
        |       |<-(2)-----  shutdown_eltoo ---------|       |
        |       |                                    |       |
        |       | <complete all pending HTLCs>       |       |
        |   A   |                 ...                |   B   |
        |       |                                    |       |
        |       |--(3)-- closing_signed_eltoo  F1--->|       |
        |       |<-(4)-- closing_signed_eltoo  F2----|       |
        |       |              ...                   |       |
        |       |--(?)-- closing_signed_eltoo  Fn--->|       |
        |       |<-(?)-- closing_signed_eltoo  Fn----|       |
        +-------+                                    +-------+

### Closing Initiation: `shutdown_eltoo`

Either node (or both) can send a `shutdown_eltoo` message to initiate closing,
along with the `scriptpubkey` it wants to be paid to.

1. type: 32773 (`shutdown_eltoo`)
2. data:
   * [`channel_id`:`channel_id`]
   * [`u16`:`len`]
   * [`len*byte`:`scriptpubkey`]
   * [`nonce`:`nonce`]

#### Requirements

A sending node:

 - MUST set the `nonce` to a valid bip-musig2 public nonce.

A receiving node:

 - MUST verify that the `nonce` value is a valid bip-musig2 public nonce.

#### Rationale

Nonces are pre-shared here to allow closing transaction partial
signatures to be sent immediately via `closing_signed_eltoo`,
as described below.

### Closing Negotiation: `closing_signed_eltoo`

The eltoo-variant of `closing_signed` which is sent in the same situation,
but for eltoo channels.

Fee negotiation is done in an identical manner as `closing_signed`.

1. type: 32774 (`closing_signed_eltoo`)
2. data:
   * [`channel_id`:`channel_id`]
   * [`u64`:`fee_satoshis`]
   * [`closing_signed_eltoo_tlvs`:`tlvs`]

1. `tlv_stream`: `closing_signed_eltoo_tlvs`
2. types:
    1. type: 1 (`fee_range`)
    2. data:
        * [`u64`:`min_fee_satoshis`]
        * [`u64`:`max_fee_satoshis`]
    1. type: 2 (`nonces`)
    2. data:
        * [`nonce`:`nonce`]
    1. type: 3 (`partial_sig`)
    2. data:
        * [`partial_sig`:`partial_sig`]

#### Requirements

A sending node:
  - MUST include `closing_signed_eltoo_tlvs`.
  - MUST include a tlv for `nonces`.
  - MUST set `nonce` to valid [MuSig2](https://github.com/jonasnick/bips/blob/musig2/bip-musig2.mediawiki) nonce with secure randomness, to be used for the *next* `closing_signed_eltoo` round of fee negotiation.
  - MUST include a tlv for `partial_sig`:
    - The partial signature must be generated by using the combined public nonces exchanged last by `shutdown` messages or `closing_signed_eltoo` messages, and the private nonce generated
        by the signing node, as per BIPMuSig2.

A node receiving the first messaging round:
  - If message does not include `closing_signed_eltoo_tlvs`:
    - MUST fail the channel.
  - If a tlv for `nonces` is not included or is not a valid MuSig2 public nonce as per BIPMuSig2:
    - MUST fail the channel.
  - Otherwise, the `nonce` will be used to generated the aggregated nonce for the next closing transaction signing, if any.
  - If a tlv for `partial_sig` is not included, not valid, or when combined through BIPMuSig2 does not result in a valid BIP340 signature for the closing
    transaction:
    - MUST send a `warning` and close the connection, or send an
      `error` and fail the channel.

Fee negotiation is followed as required in parallel.

#### Rationale

We use MuSig2 multisignature algorithm to close eltoo channels. This allows a healthy
operating channel to appear to be a single pubkey output even after being spent,
reducing fees in the common case.

It's critically important that nonces are never re-used, giving the recommendation
that the nonces be wiped in between sessions.

### Eltoo Simplified Operation

If `open_channel_eltoo` was used to initiate the channel, `option_simplified_update` rules are implied,
with modifications.

        +-------+                                     +-------+
        |       |--(1)---- update_add_htlc ---------->|       |
        |       |--(2)---- update_add_htlc ---------->|       |
        |       |--(3)--- update_signed ------------->|       |
        |   A   |<--(4)--- update_signed_ack ---------|   B   |
        |       |                                     |       |
        |       |<-(5)---- update_add_htlc -----------|       |
        |       |<-(6)--- update_signed --------------|       |
        |       |--(7)-- update_signed_ack ========-->|       |
        |       |                                     |       |
        +-------+                                     +-------+

The flow is similar except for the symmetrical state. This means there is no
`revoke_and_ack` message, meaning all updates are immediately applied to the
pending update and settlement transactions and signed with `update_signed`.

Note that once the recipient of an HTLC offer receives a
`update_signed` message, the new offers may be forwarded immediately
as the update and settlement transactions can be signed locally and broadcasted at any point.

#### Requirements

Same requirements as `option_simplified_update` except:

A node:
  - At any time:
    - if it receives a `update_signed` or `revoke_and_ack` message
      - SHOULD send an `error` to the sending peer (if connected).
      - MUST fail the channel.
  - During this node's turn:
    - if it receives an update message or `update_signed`:
      - if it has sent its own update or `update_signed`:
        - MUST ignore the message
      - otherwise:
        - MUST reply with `yield` and process the message.
  - During the other node's turn:
    - if it has not received an update message or `update_signed`:
      - MAY send one or more update message or `update_signed`:
        - MUST NOT include those changes if it receives a later update message or `update_signed`.
        - MUST include those changes if it receives a `yield` in reply.

and channel reestablishment, defined by `channel_reestablish_eltoo`

#### Rationale

Update numbers will stay synchronized after the successful end of each turn. On reconnection this allows
a trivial comparison to determine if there was an unfinished turn. Note that this means upgrading of penalty
channels to eltoo channels is made more difficult if done in the future as we can not assume the update
numbers are synchronized.

### Forwarding HTLCs

HTLC forwarding logic has been adapted to the symmetrical transaction state
case of eltoo but uses the same messages as BOLT02 aside from `update_signed`
and `update_signed_ack`:.

The respective **addition/removal** of an HTLC is considered *irrevocably committed* when:

1. The settlement transaction **with/without** it is committed to by the offering node
2. The settlement transaction **with/without** it has been irreversibly committed to
the blockchain.

#### Requirements

A node:
  - until an incoming HTLC has been irrevocably committed:
    - MUST NOT offer the corresponding outgoing HTLC (`update_add_htlc`) in response to that incoming HTLC.
  - until the removal of an outgoing HTLC is irrevocably committed, OR until the outgoing on-chain HTLC output has been spent (with sufficient depth):
    - MUST NOT fail the incoming HTLC (`update_fail_htlc`) that corresponds
to that outgoing HTLC.
  - once the `cltv_expiry` of an incoming HTLC has been reached, OR if `cltv_expiry` minus `current_height` is less than `cltv_expiry_delta` for the corresponding outgoing HTLC:
    - MUST fail that incoming HTLC (`update_fail_htlc`).
  - if an incoming HTLC's `cltv_expiry` is unreasonably far in the future:
    - SHOULD fail that incoming HTLC (`update_fail_htlc`).
  - upon receiving an `update_fulfill_htlc` for an outgoing HTLC, OR upon discovering the `payment_preimage` from an on-chain HTLC spend:
    - MUST fulfill the incoming HTLC that corresponds to that outgoing HTLC.

#### Rationale

Same rationale as BOLT02 except we do not have HTLC-X second stage transactions

### `cltv_expiry_delta` Selection

FIXME eltoo reasoning

#### Requirements

Same requirements as BOLT02

### Adding an HTLC: `update_add_htlc`

Same requirements as BOLT02

### Removing an HTLC: `update_fulfill_htlc`, `update_fail_htlc`, and `update_fail_malformed_htlc`

ame requirements as BOLT02

### Committing Updates So Far: `update_signed`

When a node has changes for the shared update for an eltoo channel, it can apply them,
sign the resulting transaction (as defined in [BOLT #3](03-transactions.md)), and send a
`update_signed` message.

Once the recipient of `update_signed` checks the signatures and knows
it has a valid new update transaction, it replies with its own `update_signed_ack`
message over the same transactions to ACK the updates and finalize it.

1. type: 32775 (`update_signed`)
2. data:
   * [`channel_id`:`channel_id`]
   * [`partial_sig`:`update_psig`]
   * [`nonce`:`next_nonce`]

Separating `update_signed` from `update_signed_ack` allows for
slightly simpler logic and disambiguation of message intent.

#### Requirements

A sending node:
  - during their turn(or when attempting to cause the counter-party to yield):
    - MUST NOT send a `update_signed` message that
      does not include any updates.
    - MAY send a `update_signed` message that only
      alters the fee.
  - otherwise:
    - MUST NOT include any changes

A receiving node:
  - during another's turn:
    - once all pending updates are applied:
      - if `update_psig` is not valid for update transaction:
        - MUST send a `warning` and close the connection, or send an
          `error` and fail the channel.
      - if `next_nonce` is not a valid BIP-musig2 nonce
        - MUST MUST send a `warning` and close the connection, or send an
          `error` and fail the channel.
    - otherwise MUST respond with a `update_signed_ack` message of their own.
    - MUST consider the transaction as final

#### Rationale

HTLCs outputs do not require signatures by the offerer, which is why only a single signature
for update transactions is required at this stage.

### Finalizing the update: `update_signed_ack`

1. type: 32776 (`update_signed_ack`)
2. data:
   * [`channel_id`:`channel_id`]
   * [`partial_sig`:`update_psig`]
   * [`nonce`:`next_nonce`]

#### Requirements

A sending node:
  - MUST set `update_psig` to a valid partial signature for the update transaction as defined
    in [BOLT #3](03-transactions.md).
  - MUST set `next_nonce` to a valid BIP-musig2 nonce.

A receiving node:
  - during another's turn:
    - if `update_psig` is not valid for update transaction:
      - MUST send a `warning` and close the connection, or send an
        `error` and fail the channel.
    - if `next_nonce` is not a valid BIP-musig2 nonce
      - MUST MUST send a `warning` and close the connection, or send an
        `error` and fail the channel.

### Updating Fees: `update_fee`

No fees are attached to update or settlement transactions, so this message is
unused.

### Requirements

A receiving node:
  - MUST send a `warning` and close the connection, or send an
    `error` and fail the channel.

## Message Retransmission for eltoo

1. type: 32777 (`channel_reestablish_eltoo`)
2. data:
   * [`channel_id`:`channel_id`]
   * [`u64`:`last_update_number`]
   * [`partial_sig`:`update_psig`]
   * [`nonce`:`next_nonce`]

#### Requirements

Reestablishment of eltoo channels is handled similarly
to the penalty-based simplified update scheme, with
the extraneous fields removed, and using the `_eltoo`
based messages respectively.

All nodes MUST discard any prior `nonce` fields on loss of connection.

A sending node:
  - MUST set `last_update_number` to the value of the channel state number of the last
    pair of update and settlement transactions the node has sent signatures of to its peer.
  - If it has sent `update_signed` on the other peer's turn without receiving `yield`:
    - MUST NOT consider that `update_signed` sent when setting `last_update_number`.
  - MUST set `update_psig` to the corresponding channel state local
    update transaction partial signature from the `last_update_number`.

A receiving node:

Upon reconnection when `channel_reestablish_eltoo` is exchanged by all channel peers:
  - If both local and remote `last_update_number`s are identical:
    - partial signature from the non-turn-taker can be applied to the turn-taker's
      transaction if not previously received before disconnect
      - If this final signature does not validate, MUST fail the channel
    - a new turn then starts with the peer with the lesser
      SEC1-encoded node_id.
  - If the local and remote node's `last_update_number` is exactly one different:
      - The turn is aborted. All pending updates are removed, next state number is one greater
        than the largest reported state number, and a new turn starts with the peer with
        the lesser SEC1-encoded node_id.
      - Offering nodes MUST be able to handle the aborted turns' update transaction, settlement transaction,
        and resolved HTLCs on-chain.
  - otherwise:
    - ??? FIXME must be something we can do here to be nice to the peer that forgot stuff.
      Ahead node can just convince the peer of latest state and share the entire tx?

#### Rationale

Due to symmetry and lack of penalty transactions, we only need to
communicate to the remote node which state we are on, which should match
the number they send as well. If they don't match, this indicates an unfinished turn
or error.

This assumes that HTLCs are not fast-forwarded until after a `update_signed_ack`
has been sent back to the offerer(and persisted). This allows for more reliable forwarding
during reconnects without requiring extensive retransmission.

# Authors

[ FIXME: Insert Author List ]

![Creative Commons License](https://i.creativecommons.org/l/by/4.0/88x31.png "License CC-BY")
<br>
This work is licensed under a [Creative Commons Attribution 4.0 International License](http://creativecommons.org/licenses/by/4.0/).
