# BOLT #2: Peer Protocol for Eltoo Channel Management

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
      * [Committing Updates So Far: `commitment_signed`](#committing-updates-so-far-commitment_signed)
      * [Completing the Transition to the Updated State: `commitment_signed_ack`](#completing-the-transition-to-the-updated-state-commitment_signed_ack)
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
step toward creating the funding transaction and both versions of the
commitment transaction.

1. type: 32768 (`open_channel_eltoo`)

2. data:
   * [`chain_hash`:`chain_hash`]
   * [`32*byte`:`temporary_channel_id`]
   * [`u64`:`funding_satoshis`]
   * [`u64`:`push_msat`]
   * [`u64`:`dust_limit_satoshis`]
   * [`u64`:`max_htlc_value_in_flight_msat`]
   * [`u64`:`channel_reserve_satoshis`]
   * [`u64`:`htlc_minimum_msat`]
   * [`u16`:`shared_delay`]
   * [`u16`:`max_accepted_htlcs`]
   * [`point`:`funding_pubkey`]
   * [`point`:`settlement_pubkey`]
   * [`point`:`htlc_pubkey`]
   * [`byte`:`channel_flags`]
   * [`open_channel_tlvs`:`tlvs`]

1. `tlv_stream`: `open_channel_tlvs`
2. types:
    1. type: 0 (`upfront_shutdown_script`)
    2. data:
        * [`...*byte`:`shutdown_scriptpubkey`]
    1. type: 1 (`channel_type`)
    2. data:
        * [`...*byte`:`type`]

#### Requirements

FIXME do we want to "mix" the version offset in these opening messages somehow?

FIXME Going from `to_self_delay` to `shared_delay` means both sides have to agree on a single number.
This likely requires a new round of interaction on channel open like closing fee negotiation.
FIXME Also probably required for `dust_limit_satoshis`

Changed fields from `open_channel`:
  - `to_self_delay` is replaced with a symmetrical `shared_delay` which must be agreed upon by nodes
  - `dust_limit_satoshis` must be agree upon by all parties
  - there is no `revocation_basepoint` as the security of the eltoo design does not rely on penalty transactions
  - there is no `delayed_payment_basepoint`, as there are no second-stage HTLC transactions to be pre-signed
  - `payment_basepoint` is replaced with a static `settlement_pubkey`
  - no `feerate_per_kw` as there is no up-front negotiated fee for update or settlement transactions
  - `htlc_basepoint` is replaced by `htlc_pubkey`
  - `first_per_commitment_point` is removed

#### Rationale

The symmetrical transaction state among all peers means that we can simplify some aspects while
requiring additional range negotiation in others.

#### Defined Eltoo Channel Types

Eltoo commitment transactions(also known as update transactions) need no anchor outputs and have
symmetrical state, which nullifies the requirement for the currently defined `channel_types` for
ln-penalty-based channels.

The currently defined types are:
  - no features (no bits set)

### The `accept_channel_eltoo` Message

This message contains information about a node and indicates its
acceptance of the new eltoo channel initiated by `open_channel_eltoo`. This
is the second step toward creating the funding transaction and both
versions of the commitment transaction.

1. type: 32769 (`accept_channel_eltoo`)
2. data:
   * [`32*byte`:`temporary_channel_id`]
   * [`u64`:`dust_limit_satoshis`]
   * [`u64`:`max_htlc_value_in_flight_msat`]
   * [`u64`:`channel_reserve_satoshis`]
   * [`u64`:`htlc_minimum_msat`]
   * [`u32`:`minimum_depth`]
   * [`u16`:`shared_delay`]
   * [`u16`:`max_accepted_htlcs`]
   * [`point`:`funding_pubkey`]
   * [`point`:`settlement_pubkey`]
   * [`point`:`htlc_pubkey`]
   * [`accept_channel_tlvs`:`tlvs`]

1. `tlv_stream`: `accept_channel_tlvs`
2. types:
    1. type: 0 (`upfront_shutdown_script`)
    2. data:
        * [`...*byte`:`shutdown_scriptpubkey`]
    1. type: 1 (`channel_type`)
    2. data:
        * [`...*byte`:`type`]

#### Requirements

The same requirements as `accept_channel`, except a few redundant fields removed.

#### Rationale

Symmetrical state means fewer parameters are required compared to `accept_channel` channel type.

### The `funding_created_eltoo` Message

This message describes the outpoint which the funder has created for
the initial commitment transactions. After receiving the peer's
signature, via `funding_signed`, it will broadcast the funding transaction.

1. type: 32770 (`funding_created_eltoo`)
2. data:
    * [`32*byte`:`temporary_channel_id`]
    * [`sha256`:`funding_txid`]
    * [`u16`:`funding_output_index`]
    * [`signature`:`update_signature`]
    * [`signature`:`settlement_signature`]

#### Requirements

Requirements are identical to `funding_created` except:

There is no `signature` field sent, or required.

The sender MUST set:
  - `update_signature` to the valid signature using its `funding_pubkey` for the initial update transaction, as defined in [BOLT #3](03-transactions.md#update-transaction).
  - `settlement_signature` to the valid signature using its `update_pubkey` for the initial settlement transaction, as defined in [BOLT #3](03-transactions.md#settlement-transaction).

The recipient:
  - if `update_signature` is incorrect:
    - MUST send a `warning` and close the connection, or send an
      `error` and fail the channel.
  - if `settlement_signature` is incorrect:
    - MUST send a `warning` and close the connection, or send an
      `error` and fail the channel.

#### Rationale

Eltoo style channels require two pre-signed transactions for backing out value, rather than one.

### The `funding_signed_eltoo` Message

This message gives the funder the signature it needs for the first
commitment transaction, so it can broadcast the transaction knowing that funds
can be redeemed, if need be. It is sent in response to `funding_created_eltoo`.

This message introduces the `channel_id` to identify the channel. It's derived from the funding transaction by combining the `funding_txid` and the `funding_output_index`, using big-endian exclusive-OR (i.e. `funding_output_index` alters the last 2 bytes).


1. type: 32771 (`funding_signed_eltoo`)
2. data:
    * [`channel_id`:`channel_id`]
    * [`signature`:`update_signature`]
    * [`signature`:`settlement_signature`]

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
  - `update_signature` to the valid signature, using its `update_pubkey` for the initial update transaction, as defined in [BOLT #3](03-transactions.md#update-transaction).
  - `settlement_signature` to the valid signature, using its `settlement_pubkey` for the initial settlement transaction, as defined in [BOLT #3](03-transactions.md#settlement-transaction).

The recipient:
  - if `update_signature` is incorrect:
    - MUST send a `warning` and close the connection, or send an
      `error` and fail the channel.
  - if `settlement_signature` is incorrect:
    - MUST send a `warning` and close the connection, or send an
      `error` and fail the channel.
  - MUST NOT broadcast the funding transaction before receipt of a valid `funding_signed_eltoo`.
  - on receipt of a valid `funding_signed_eltoo`:
    - SHOULD broadcast the funding transaction.

#### Rationale

Eltoo style channels require two pre-signed transactions for backing out value, rather than one.

### The `funding_locked_eltoo` Message

This message indicates that the funding transaction has reached the `minimum_depth` asked for in `accept_channel_eltoo`. Once both nodes have sent this, the channel enters normal operating mode.

1. type: 36 (`funding_locked_eltoo`)
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

1. type: 38 (`shutdown_eltoo`)
2. data:
   * [`channel_id`:`channel_id`]
   * [`u16`:`len`]
   * [`len*byte`:`scriptpubkey`]

#### Requirements

Same for BOLT03 (for now)

#### Rationale

Same for BOLT03 (for now)

### Closing Negotiation: `closing_signed_eltoo`

The eltoo-variant of `closing_signed` which is sent in the same situation,
but for eltoo channels. This is a two-round communication due to
MuSig2 signatures being completed without nonce pre-generation.

Fee negotiation is done in an identical manner as `closing_signed`.

1. type: 39 (`closing_signed_eltoo`)
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

A node sending the first messaging round:
  - MUST include `closing_signed_eltoo_tlvs`.
  - MUST NOT include a tlv for `partial_sig`.
  - MUST include a tlv for `nonces`.
  - MUST set `nonce` to valid [MuSig2](https://github.com/jonasnick/bips/blob/musig2/bip-musig2.mediawiki) nonce with secure randomness
    - If insufficient randomness is available, MUST immediately fail the channel

A node receiving the first messaging round:
  - If message does not include `closing_signed_eltoo_tlvs`:
    - MUST fail the channel.
  - If a tlv for `partial_sig` is included:
    - MUST fail the channel.
  - If a tlv for `nonces` is not included or is not a valid MuSig2 public nonce as per BIPMuSig2:
    - MUST fail the channel.

Once `nonces` are exchanged and the final unsigned transaction can be constructed a node:
  - MUST send a `closing_signed_eltoo` message:
    - MUST NOT include a tlv for `nonces`
    - MUST include a tlv for `partial_sig`:
      - The partial signature must be generated by using the combined public nonces exchanged, and the private nonce generated
        by the signing node, as per BIPMuSig2.

A node receiving the `partial_sigs` messaging round:
  - MUST wipe private nonce data
  - If message does not include `closing_signed_eltoo_tlvs`:
    - MUST fail the channel.
  - If a tlv for `nonces` is included:
    - MUST fail the channel.
  - If a tlv for `partial_sig` is not included, not valid, or when combined through BIPMuSig2 does not result in a valid BIP340 signature for the closing
    transaction:
    - MUST send a `warning` and close the connection, or send an
      `error` and fail the channel.

Fee negotiation is followed as required in parallel.

#### Rationale

We use MuSig2 multisignature algorithm to close eltoo channels. This allows a healthy
operating channel to appear to be a single pubkey output even after being spent,
reducing fees in the common case.

First round is parties sending each other nonces and completing fee negotiation,
the second round results in partial signatures being passed to all parties, resulting
in a valid signature for cooperative closing transaction.

It's critically important that nonces are never re-used, giving the recommendation
that the nonces be wiped in between sessions.

### Eltoo Simplified Operation

If `open_channel_eltoo` was used to initiate the channel, `option_simplified_update` rules are implied,
with modifications.

        +-------+                                     +-------+
        |       |--(1)---- update_add_htlc ---------->|       |
        |       |--(2)---- update_add_htlc ---------->|       |
        |       |--(3)--- commitment_signed_eltoo --->|       |
        |   A   |<--(4)--- commitment_signed_eltoo_ack|   B   |
        |       |                                     |       |
        |       |<-(5)---- update_add_htlc -----------|       |
        |       |<-(6)--- commitment_signed_eltoo ----|       |
        |       |--(7)-- commitment_signed_eltoo_ack->|       |
        |       |                                     |       |
        +-------+                                     +-------+

The flow is similar except for the symmetrical state. This means there is no
`revoke_and_ack` message, meaning all updates are immediately applied to the
pending commitment transaction and signed with `commitment_signed_eltoo`.

Note that once the recipient of an HTLC offer receives a
`commitment_signed_eltoo` message, the new offers may be forwarded immediately
as the update and settlement transactions can be signed locally and broadcasted at any point.

#### Requirements

Same requirements as `option_simplified_update` except:

A node:
  - At any time:
    - if it receives a `commitment_signed` or `revoke_and_ack` message
      - SHOULD send an `error` to the sending peer (if connected).
      - MUST fail the channel.
  - During this node's turn:
    - if it receives an update message or `commitment_signed_eltoo`:
      - if it has sent its own update or `commitment_signed_eltoo`:
        - MUST ignore the message
      - otherwise:
        - MUST reply with `yield` and process the message.
  - During the other node's turn:
    - if it has not received an update message or `commitment_signed_eltoo`:
      - MAY send one or more update message or `commitment_signed_eltoo`:
        - MUST NOT include those changes if it receives a later update message or `commitment_signed_eltoo`.
        - MUST include those changes if it receives a `yield` in reply.

and channel reestablishment, defined by `channel_reestablish_eltoo`

#### Rationale

Commitment numbers will stay synchronized after the successful end of each turn. On reconnection this allows
a trivial comparison to determine if there was an unfinished turn. Note that this means upgrading of penalty
channels to eltoo channels is made more difficult if done in the future as we can not assume the commitment
numbers are synchronized.

### Forwarding HTLCs

HTLC forwarding logic has been adapted to the symmetrical transaction state
case of eltoo but uses the same messages as BOLT02 aside from `commitment_signed_eltoo`
and `commitment_signed_eltoo_ack`:.

The respective **addition/removal** of an HTLC is considered *irrevocably committed* when:

1. The commitment transaction **with/without** it is committed to by the offering node
2. The commitment transaction **with/without** it has been irreversibly committed to
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

### Committing Updates So Far: `commitment_signed_eltoo`

When a node has changes for the shared commitment for an eltoo channel, it can apply them,
sign the resulting transaction (as defined in [BOLT #3](03-transactions.md)), and send a
`commitment_signed_eltoo` message.

Once the recipient of `commitment_signed_eltoo` checks the signatures and knows
it has a valid new commitment transaction, it replies with its own `commitment_signed_eltoo_ack`
message over the same transactions to ACK the updates and finalize it.

1. type: 32772 (`commitment_signed_eltoo`)
2. data:
   * [`channel_id`:`channel_id`]
   * [`signature`:`update_signature`]
   * [`signature`:`settlement_signature`]

Separating `commitment_signed_eltoo` from `commitment_signed_eltoo_ack` allows for
slightly simpler logic and disambiguation of message intent.

#### Requirements

A sending node:
  - during their turn(or when attempting to cause the counter-party to yield):
    - MUST NOT send a `commitment_signed_eltoo` message that
      does not include any updates.
    - MAY send a `commitment_signed_eltoo` message that only
      alters the fee.
  - otherwise:
    - MUST NOT include any changes

A receiving node:
  - during another's turn:
    - once all pending updates are applied:
      - if `update_signature` is not valid for its local commitment transaction:
        - MUST send a `warning` and close the connection, or send an
          `error` and fail the channel.
      - if `settlement_signature` is not valid for its local commitment transaction:
        - MUST send a `warning` and close the connection, or send an
          `error` and fail the channel.
    - MUST respond with a `commitment_signed_eltoo_ack` message of their own.
    - MUST consider the transaction as final

#### Rationale

HTLCs outputs do not require signatures by the offerer, which is why only the two signatures
for update and settlement transactions are required at this stage.

### Finalizing the update: `commitment_signed_eltoo_ack`

1. type: 32773 (`commitment_signed_eltoo_ack`)
2. data:
   * [`channel_id`:`channel_id`]
   * [`signature`:`update_signature`]
   * [`signature`:`settlement_signature`]

#### Requirements

A receiving node:
  - if `update_signature` is not valid for its update transaction:
      - MUST send a `warning` and close the connection, or send an
        `error` and fail the channel.
  - if `update_signature` is not valid for its update transaction:
      - MUST send a `warning` and close the connection, or send an
        `error` and fail the channel.
  - otherwise:
    - MUST consider the transaction as final

### Updating Fees: `update_fee`

No fees are attached to update or settlement transactions, so this message is
unused.

### Requirements

A receiving node:
  - MUST send a `warning` and close the connection, or send an
    `error` and fail the channel.

## Message Retransmission for eltoo

1. type: 32773 (`channel_reestablish_eltoo`)
2. data:
   * [`channel_id`:`channel_id`]
   * [`u64`:`next_commitment_number`]

#### Requirements

Reestablishment of eltoo channels is handled similarly
to the penalty-based simplified update scheme, with
the extraneous fields removed, and using the `_eltoo`
based messages respectively.

Upon reconnection when `channel_reestablish_eltoo` is exchanged:
  - If it has sent `commitment_signed_eltoo` on the other peer's turn without receiving `yield`:
    - MUST NOT consider that `commitment_signed_eltoo` sent when setting `channel_reestablish` fields.
  - If a node's sent `next_commitment_number` is one less than its received
    `next_commitment_number`, that node's turn is unfinished.
    - due to symmetrical state of eltoo and simplified operation only,
      there can only be one unfinished side
  - else if both sent and received `next_commitment_number`s are identical:
    - the turn starts with the peer with the lesser
      SEC1-encoded node_id.
  - otherwise:
    - ??? FIXME We can just send them the latest state if we're being nice

#### Rationale

Due to symmetry and lack of penalty transactions, we only need to
communicate to the remote node which state we are on, which should match
the number they send as well.

# Authors

[ FIXME: Insert Author List ]

![Creative Commons License](https://i.creativecommons.org/l/by/4.0/88x31.png "License CC-BY")
<br>
This work is licensed under a [Creative Commons Attribution 4.0 International License](http://creativecommons.org/licenses/by/4.0/).
