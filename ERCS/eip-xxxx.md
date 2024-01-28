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

First note that interaction with a protocol under this pattern requires a token holder to send two separate transactions: 1) a transaction invoking the token's `approve` function to allow a protocol contract to spend some of their balance, and 2) a transaction to invoke the desired protocol operation, which will pull the approved tokens using `transferFrom`. Each additional transaction adds multiple costs to the usage of a protocol, such as gas fees, friction, and cognitive overhead.

To minimize these costs, it has become common to implement the "infinite approval" pattern, where the token holder is encouraged to approve an effectively or actually infinite amount of tokens to the protocol, so that `approve` transactions are not required beyond the first interaction with it. This has significant security consequences. The protocol contract is granted potentially unrestricted access to the user's assets for an indefinite amount of time, far beyond what is needed for an interaction with the protocol, overly exposing users to smart contract risk (i.e., bugs that may later be discovered and exploited).

ERC-6909 has adopted both of these patterns in its core interface, but as a new token standard and potential new token ecosystem it presents an opportunity to improve on the satus quo. The goal of this ERC is to develop an alternative pattern that addresses the friction points without resorting to infinite approval.

## Specification

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in RFC 2119 and RFC 8174.

### Overview

An ERC-6909 token MAY implement the extension described here, known as "ERC-6909X".

An ERC-6909X token MUST implement the Added Token Functions specified below. This includes three additional functions to set allowance and operators, summarized in the following table with the core ERC-6909 `approve` and `setOperator` functions.

| Authentication | Permanent | Temporary |
| ---- | ---- | ---- |
| Native | _`approve`, `setOperator`_ | `temporaryApproveAndCall` |
| EIP-712 Signature | `approveBySig` | `temporaryApproveAndCallBySig` |

Additionally, an ERC-6909X token MUST implement ERC-5267 (Retrieval of EIP-712 domain), and ERC-165 (Standard Interface Detection) declaring support for the ERC-6909X interface ID `0xeb858add`.

### Common Parameters

Each of the three new functions is some form of token approval and receives a subset of the following parameters:

- `owner` (`address`): The owner of the tokens that are being approved. (Implied to be the caller if not present).
- `spender` (`address`): The account that will be able to spend the approved tokens.
- `operator` (`bool`): If `true`, the owner's entire balance of all token IDs is being approved.
- `id` (`uint256`): The token id that is being approved. If `operator` is `true`, this parameter MUST be 0.
- `amount` (`uint256`): The amount of tokens that are approved. If `operator` is `true`, this parameter MUST be 0.
- `target` (`address`): The target contract where the callback SHALL be invoked.
- `data` (`bytes`): The data to be included with the callback.
- `deadline` (`uint48`): A timestamp after which a provided signature MUST be rejected even if valid.

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

The meaning of each field is as specified in the previous section. Additionally, `temporary` determines whether the signature is intended for `approveBySig` (`temporary = false`) or `temporaryApproveAndCallBySig` (`temporary = true`), and a signature MUST be considered invalid if submitted to the wrong function. `nonce` MUST be the next available nonce for the owner, and the contract MUST consider invalid a signature that reuses a nonce.

### Added Token Functions

The behavior described below is REQUIRED unless explicitly described otherwise.

#### `temporaryApproveAndCall`

If `operator = false`, sets `amount` as the caller's allowance for `spender` for token `id`. If `operator = true`, reverts if `amount` or `id` are non-zero, and otherwise sets `spender` as an operator for the caller.

With allowance or operator set, invokes the callback `onTemporaryApprove` on `target`, forwarding the parameters `operator`, `id`, `amount`, and `data`, and setting `owner` to the caller. Validates that the callback returns the value `0xb74de3da`.

After the callback returns, resets operator status and allowance to their values prior to this function call.

An implementation MAY use EIP-1153 transient storage to store temporary allowance and operator status.

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
    - name: signature
      type: bytes

  outputs:
    - name: status
      type: bool
```

#### `approveBySig`

Validates that the signature was produced by `owner` as described in the Signatures section, and that `deadline` is greater than or equal to the block timestamp, then marks the nonce as used.

The rest of this function behaves like ERC-6909 `approve`, substituting the signer for the caller.

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
    - name: signature
      type: bytes

  outputs:
    - name: status
      type: bool
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

<!--
  The rationale fleshes out the specification by describing what motivated the design and why particular design decisions were made. It should describe alternate designs that were considered and related work, e.g. how the feature is supported in other languages.

  The current placeholder is acceptable for a draft.

  TODO: Remove this comment before submitting
-->

TBD

## Backwards Compatibility

<!--

  This section is optional.

  All EIPs that introduce backwards incompatibilities must include a section describing these incompatibilities and their severity. The EIP must explain how the author proposes to deal with these incompatibilities. EIP submissions without a sufficient backwards compatibility treatise may be rejected outright.

  The current placeholder is acceptable for a draft.

  TODO: Remove this comment before submitting
-->

No backward compatibility issues found.

## Reference Implementation

<!--
  This section is optional.

  The Reference Implementation section should include a minimal implementation that assists in understanding or implementing this specification. It should not include project build files. The reference implementation is not a replacement for the Specification section, and the proposal should still be understandable without it.
  If the reference implementation is too large to reasonably be included inline, then consider adding it as one or more files in `../assets/eip-####/`. External links will not be allowed.

  TODO: Remove this comment before submitting
-->

## Security Considerations

<!--
  All EIPs must contain a section that discusses the security implications/considerations relevant to the proposed change. Include information that might be important for security discussions, surfaces risks and can be used throughout the life cycle of the proposal. For example, include security-relevant design decisions, concerns, important discussions, implementation-specific guidance and pitfalls, an outline of threats and risks and how they are being addressed. EIP submissions missing the "Security Considerations" section will be rejected. An EIP cannot proceed to status "Final" without a Security Considerations discussion deemed sufficient by the reviewers.

  The current placeholder is acceptable for a draft.

  TODO: Remove this comment before submitting
-->

Needs discussion.

## Copyright

Copyright and related rights waived via [CC0](../LICENSE.md).
