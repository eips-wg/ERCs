---
eip: 5216
title: ERC-1155 Allowance Extension
description: Extension for ERC-1155 secure approvals
author: Iván Mañús (@ivanmmurciaua), Juan Carlos Cantó (@EscuelaCryptoES)
discussions-to: https://ethereum-magicians.org/t/eip-erc1155-approval-by-amount/9898
status: Last Call
last-call-deadline: 2022-11-12
type: Standards Track
category: ERC
created: 2022-07-11
requires: 20, 165, 1155
---

## Abstract

This ERC defines standard functions for granular approval of [ERC-1155](../01155.md) tokens by both `id` and `amount`. This ERC extends [ERC-1155](../01155.md).

## Motivation

[ERC-1155](../01155.md)'s popularity means that multi-token management transactions occur on a daily basis. Although it can be used as a more comprehensive alternative to [ERC-721](../00721.md), ERC-1155 is most commonly used as intended: creating multiple `id`s, each with multiple tokens. While many projects interface with these semi-fungible tokens, by far the most common interactions are with NFT marketplaces.

Due to the nature of the blockchain, programming errors or malicious operators can cause permanent loss of funds. It is therefore essential that transactions are as trustless as possible. ERC-1155 uses the `setApprovalForAll` function, which approves ALL tokens with a specific `id`. This system has obvious minimum required trust flaws. This ERC combines ideas from [ERC-20](../00020.md) and [ERC-721](../00721.md) in order to create a trust mechanism where an owner can allow a third party, such as a marketplace, to approve a limited (instead of unlimited) number of tokens of one `id`.

## Specification

The keywords “MUST”, “MUST NOT”, “REQUIRED”, “SHALL”, “SHALL NOT”, “SHOULD”, “SHOULD NOT”, “RECOMMENDED”, “MAY”, and “OPTIONAL” in this document are to be interpreted as described in RFC 2119.

Contracts using this ERC MUST implement the `IERC5216` interface.

### Interface implementation

```solidity
/**
 * @title ERC-1155 Allowance Extension
 * Note: the ERC-165 identifier for this interface is 0x1be07d74
 */
interface IERC5216 is IERC1155 {

    /**
     * @notice Emitted when `account` grants or revokes permission to `operator` to transfer their tokens, according to
     * `id` and with an amount: `amount`.
     */
    event Approval(address indexed account, address indexed operator, uint256 id, uint256 amount);

    /**
     * @notice Grants permission to `operator` to transfer the caller's tokens, according to `id`, and an amount: `amount`.
     * Emits an {Approval} event.
     *
     * Requirements:
     * - `operator` cannot be the caller.
     */
    function approve(address operator, uint256 id, uint256 amount) external;

    /**
     * @notice Returns the amount allocated to `operator` approved to transfer `account`'s tokens, according to `id`.
     */
    function allowance(address account, address operator, uint256 id) external view returns (uint256);
}
```

The `approve(address operator, uint256 id, uint256 amount)` function MUST be either `public` or `external`.

The `allowance(address account, address operator, uint256 id)` function MUST be either `public` or `external` and MUST be `view`.

The `safeTrasferFrom` function (as defined by ERC-1155) MUST:

- Not revert if the user has approved `msg.sender` with a sufficient `amount`
- Subtract the transferred amount of tokens from the approved amount if `msg.sender` is not approved with `setApprovalForAll`

In addition, the `safeBatchTransferFrom` MUST:

- Add an extra condition that checks if the `allowance` of all `ids` have the approved `amounts` (See `_checkApprovalForBatch` function reference implementation)

The `Approval` event MUST be emitted when a certain number of tokens are approved.

The `supportsInterface` method MUST return `true` when called with `0x1be07d74`.

## Rationale

The name "ERC-1155 Allowance Extension" was chosen because it is a succinct description of this ERC. Users can approve their tokens by `id` and `amount` to `operator`s.

By having a way to approve and revoke in a manner similar to [ERC-20](../00020.md), the trust level can be more directly managed by users:

- Using the `approve` function, users can approve an operator to spend an `amount` of tokens for each `id`.
- Using the `allowance` function, users can see the approval that an operator has for each `id`.

The [ERC-20](../00020.md) name patterns were used due to similarities with [ERC-20](../00020.md) approvals.

## Backwards Compatibility

This standard is compatible with [ERC-1155](../01155.md).

## Reference Implementation

The reference implementation can be found [here](./assets/ERC5216.sol).

## Security Considerations

Users of this ERC must thoroughly consider the amount of tokens they give permission to `operators`, and should revoke unused authorizations.

## Copyright

Copyright and related rights waived via [CC0](/LICENSE.md).
