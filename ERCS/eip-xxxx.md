---
title: Usability & Security Extensions for ERC-6909
description: Alternative mechanisms for protocols to work with ERC-6909 tokens addressing shortcomings of allowance and operators.
author: Francisco Giordano (@frangio)
discussions-to: <URL>
status: Draft
type: Standards Track
category: ERC
created: 2024-01-26
requires: 6909, 5267, 712, 165
---

## Abstract

An extension of the ERC-6909 token standard with a combination of temporary approval and approve-by-signature mechanisms, addressing known usability and security issues inherited from prior token standards. Temporary approval allows a token holder to delegate access to their assets for the scope of a function call with predetermined parameters, removing a significant source of overexposure to smart contract risk. Approve-by-signature allows a third-party to act on behalf of the token holder, removing the burden of acquiring and holding the gas token. We refer to this extension as ERC-6909X.

## Motivation

The first Ethereum token standard, ERC-20, introduced `approve` and `transferFrom` as the main way for protocols to work with users' tokens, with subsequent token standards largely following the same or an equivalent pattern. In a sense, it has been successful, but it has also been widely criticized. Many alternatives have been proposed since then, although none have gained wide adoption. This ERC agrees with the criticism, arguing that the pattern has usability and security problems that go hand in hand.

First note that interaction with a protocol under this pattern requires a token holder to send two separate transactions: 1) a transaction invoking the token's `approve` function to allow a protocol contract to spend some of their balance, and 2) a transaction to invoke the desired protocol operation, which may pull the approved tokens using `transferFrom`. Each additional transaction adds multiple costs to the usage of a protocol, such as gas fees, friction, and cognitive overhead.

In an attempt to minimize these costs, it has become common to implement the "infinite approval" pattern, where the token holder is encouraged to approve an effectively or actually infinite amount of tokens to the protocol, so that `approve` transactions are not required beyond a first interaction. While `approve`-related friction is reduced, this is comes with a significant security cost. The protocol contract is granted potentially unrestricted access to the user's assets for an indefinite amount of time, far beyond what is needed for an interaction with the protocol, and as a result users are overly exposed to smart contract risk, i.e., bugs that may later be discovered and exploited.

ERC-6909 has adopted both of these patterns in its core interface, but as a new token standard and potential new token ecosystem it presents an opportunity to improve on the satus quo. The goal of this ERC is to develop an alternative pattern that addresses the friction points without resorting to infinite approval.

## Specification

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in RFC 2119 and RFC 8174.

### Overview

An ERC-6909 token MAY implement the extension described here, known as "ERC-6909X". If it does, it MUST declare support for the ERC-6909X interface ID, `0x4a31d207`.

An ERC-6909X token MUST implement the Added Token Functions specified below. This includes three additional functions to set allowance and operators, summarized in the following table with the core ERC-6909 `approve` and `setOperator` functions.

| Authentication | Permanent | Temporary |
| ---- | ---- | ---- |
| Native | _`approve`, `setOperator`_ | `temporaryApproveAndCall` |
| EIP-712 Signature | `approveBySig` | `temporaryApproveAndCallBySig` |

An ERC-6909X token MAY implement the Optional Nonce Management Functions specified below.

Additionally, an ERC-6909X token MUST implement ERC-5267 (Retrieval of EIP-712 domain).

### Common Parameters

Each of the three new functions is some form of token approval and receives a subset of the following parameters:

- `owner` (`address`): The owner of the tokens that are being approved. (Implied to be the caller if not present).
- `spender` (`address`): The account that will be able to spend the approved tokens.
- `operator` (`bool`): If `true`, the owner's entire balance of all token IDs is being approved.
- `id` (`uint256`): The token id that is being approved. If `operator` is `true`, this parameter MUST be 0.
- `amount` (`uint256`): The amount of tokens that are approved. If `operator` is `true`, this parameter MUST be 0.
- `target` (`address`): The target contract where the callback SHALL be invoked.
- `data` (`bytes`): The data to be included with the callback.
- `deadline` (`uint48`): A timestamp after which a provided signature MUST be rejected even if otherwise valid.
- `nonce` (`uint256`): A value used by the contract to prevent signature replay.

### Signatures

The functions `approveBySig` and `temporaryApproveAndCallBySig` receive a signature as a `bytes` value.

At least the following signature types and encodings MUST be accepted:

- ECDSA signature encoded as the 65-byte concatenation of `r`, `s`, and `v`.
- ERC-1271 signature (opaque).

This signature MUST be validated as an EIP-712 signature for the contract's domain (see ERC-5267) of an object with the following type:

```solidity
struct ERC6909XApproveAndCall {
    bool temporary;
    address owner;
    address spender;
    bool operator;
    uint256 id;
    uint256 amount;
    address target;
    bytes data;
    uint256 nonce;
    uint48 deadline;
}
```

`temporary` determines whether the signature is intended for `approveBySig` (`temporary = false`) or `temporaryApproveAndCallBySig` (`temporary = true`). A signature MUST be rejected if submitted to the wrong function.

Any `nonce` value previously unused as a nonce by the signer MUST be accepted. A signature MUST be considered invalid if its nonce was previously used. It is RECOMMENDED to choose a random nonce for every new signature.

`deadline` is a timestamp after which the signature MUST be rejected.

The meaning of the remaining fields is as specified in the previous section.

### Added Token Functions

The behavior described below is REQUIRED unless it is explicitly stated otherwise.

#### `temporaryApproveAndCall`

If `operator = false`, sets `amount` as the caller's allowance for `spender` for token `id`. If `operator = true`, reverts if `amount` or `id` are non-zero, and otherwise sets `spender` as an operator for the caller.

With allowance or operator set, invokes the callback `onTemporaryApprove` on `target`, forwarding the parameters `operator`, `id`, `amount`, and `data`, and setting `owner` to the caller. Validates that the callback returns the value `0xb74de3da`.

After the callback returns, resets operator status and allowance to their values prior to this function call.

An implementation MAY use EIP-1153 transient storage to store temporary allowance and operator status.

Returns `true`.

```yaml
- name: temporaryApproveAndCall
  type: function
  stateMutability: nonpayable

  inputs:
    - name: spender
      type: address
    - name: operator
      type: bool
    - name: id
      type: uint256
    - name: amount
      type: uint256
    - name: target
      type: address
    - name: data
      type: bytes

  outputs:
    - name: status
      type: bool
```

#### `temporaryApproveAndCallBySig`

Validates that the signature was produced by `owner` as described in the Signatures section, and that `deadline` is greater than or equal to the block timestamp, then marks the nonce as used.

The rest of this function behaves like `temporaryApproveAndCall`, substituting the signer for the caller.

If `operator = false`, sets `amount` as the signer's allowance for `spender` for token `id`. If `operator = true`, reverts if `amount` or `id` are non-zero, and otherwise sets `spender` as an operator for the signer.

With allowance or operator set, invokes the callback `onTemporaryApprove` on `target`, forwarding the parameters `operator`, `id`, `amount`, and `data`, and setting `owner` to the signer. Validates that the callback returns the value `0xb74de3da`.

After the callback returns, resets operator status and allowance to their values prior to this function call.

An implementation MAY use EIP-1153 transient storage to store temporary allowance and operator status.

Returns `true`.

```yaml
- name: temporaryApproveAndCallBySig
  type: function
  stateMutability: nonpayable

  inputs:
    - name: owner
      type: address
    - name: spender
      type: address
    - name: operator
      type: bool
    - name: id
      type: uint256
    - name: amount
      type: uint256
    - name: target
      type: address
    - name: data
      type: bytes
    - name: deadline
      type: uint48
    - name: nonce
      type: uint256
    - name: signature
      type: bytes

  outputs:
    - name: status
      type: bool
```

#### `approveBySig`

Validates that the signature was produced by `owner` as described in the Signatures section, and that `deadline` is greater than or equal to the block timestamp, then marks the nonce as used.

The rest of this function behaves like ERC-6909 `approve` or `setOperator` (if `operator` is false or true respectively), substituting the signer for the caller.

Returns `true`.

```yaml
- name: approveBySig
  type: function
  stateMutability: nonpayable

  inputs:
    - name: owner
      type: address
    - name: spender
      type: address
    - name: operator
      type: bool
    - name: id
      type: uint256
    - name: amount
      type: uint256
    - name: deadline
      type: uint48
    - name: nonce
      type: uint256
    - name: signature
      type: bytes

  outputs:
    - name: status
      type: bool
```

### Optional Nonce Management Functions

#### `invalidateNonce`

Causes `nonce` to be considered invalid for future signatures by the caller, as if it had been used in an accepted signature.

MUST emit a `NonceInvalidation` event.

_Note that, if this function is not included, it is possible for a signer to invalidate a nonce by generating and submitting a signature to one of the signature validating functions, such as `approveBySig`._

```yaml
- name: invalidateNonce
  type: function
  stateMutability: nonpayable

  inputs:
    - name: nonce
      type: uint256
```

#### `NonceInvalidation`

Emitted when a nonce is invalidated.

MAY be emitted when a nonce is used as part of an accepted signature.

```yaml
- name: NonceInvalidation
  type: event

  inputs:
    - name: owner
      type: address
      indexed: true
    - name: nonce
      type: uint256
      indexed: true
```


### Callback Function

#### `onTemporaryApprove`

A contract MUST implement the callback function `onTemporaryApprove` if it is meant to be used as the `target` parameter of `temporaryApproveAndCall[BySig]`.

The callback MAY revert. Otherwise, it MUST return the value `0xb74de3da` (i.e., the function selector for this function).

```yaml
- name: onTemporaryApprove
  type: function
  stateMutability: nonpayable

  inputs:
    - name: owner
      type: address
    - name: operator
      type: bool
    - name: id
      type: uint256
    - name: amount
      type: uint256
    - name: data
      type: bytes

  outputs:
    - name: ack
      type: bytes4
```

## Rationale

### Unordered Nonces

The use of sequential nonces in other standards like ERC-2612 has proved insufficient, as it implies that a token owner can have only one valid outstanding signature at a time. Different mechanisms have been proposed to address this, such as allowing multiple parallel nonce "timelines" for a single signer. Here we have opted for the approach of removing the sequentiality requirement, which has been adopted by standards like ERC-3009, do to its simplicity and sufficiency.

## Backwards Compatibility

### `Approval` and `OperatorSet` Events

ERC-6909 requires `Approval` and `OperatorSet` events to be emitted when allowance and operator status respectively are set. ERC-6909X tokens SHOULD emit these events as part of temporary approvals for strict compliance. The omission of these events during temporary approvals may confuse indexers that rely on events to track allowances and operators. For example, an indexer that assumes the events are complete may conclude that a spender has zero allowance in a case where in fact it has non-zero allowance.

## Security Considerations

### Frontrunning

The signatures used by this ERC are not by themselves bound to a caller: they can be submitted to the token by anyone, and, in particular, can be taken from the mempool and submitted in a frontrunning transaction. This applies to both `temporaryAppoveAndCallBySig` and `approveBySig`.

If a transaction invokes one of these functions on the token directly, frontrunning can in general only result in wasted gas, unless the callback invoked during temporary approval makes use of the transaction origin, which is controlled by the attacker during frontrunning. (Relying on the transaction origin in a smart contract is considered bad practice.)

If a transaction invokes one of these functions indirectly, in the context of a larger transaction, it is important to consider the following two consequences:

1. An attacker can frontrun the signature and cause the call to the token function to revert (since the signature is already used at that point). In the absence of other frontrunning protection, a contract should allow the token call to revert.
2. An attacker can take the signature and submit it in a different context. Contracts should not interpret a signature as an intent to do anything other than what is explicitly contained in the signature. For example, a signature meant for `approveBySig` should only be interpreted as the intent to set an allowance, and not should not be interpreted as an authorization to spend the approved tokens in any particular way.

The general recommendation is to use temporary approve with a callback to execute additional logic.


## Copyright

Copyright and related rights waived via [CC0](../LICENSE.md).
