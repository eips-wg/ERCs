---
eip: 6366
title: Permission Token
description: A token that holds the permission of an address in an ecosystem
author: Chiro (@chiro-hiro), Victor Dusart (@vdusart)
discussions-to: https://ethereum-magicians.org/t/eip-6366-a-standard-for-permission-token/9105
status: Review
type: Standards Track
category: ERC
created: 2022-01-19
requires: 6617
---

## Abstract

This EIP offers an alternative to Access Control Lists (ACLs) for granting authorization and enhancing security. A `uint256` is used to store permission of given address in a ecosystem. Each permission is represented by a single bit in a `uint256` as described in [ERC-6617](../06617.md). Bitwise operators and bitmasks are used to determine the access right which is much more efficient and flexible than `string` or `keccak256` comparison.

## Motivation

Special roles like `Owner`, `Operator`, `Manager`, `Validator` are common for many smart contracts because permissioned addresses are used to administer and manage them. It is difficult to audit and maintain these system since these permissions are not managed in a single smart contract.

Since permissions and roles are reflected by the permission token balance of the relevant account in the given ecosystem, cross-interactivity between many ecosystems will be made simpler.

## Specification

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in RFC 2119 and RFC 8174.

_Note_ The following specifications use syntax from Solidity `0.8.7` (or above)

### Core Interface

Compliant contracts MUST implement `IEIP6366Core`.

It is RECOMMENDED to define each permission as a power of `2` so that we can check for the relationship between sets of permissions using [ERC-6617](../06617.md).

```solidity
interface IEIP6366Core {
  /**
   * MUST trigger when `_permission` are transferred, including `zero` permission transfers.
   * @param _from           Permission owner
   * @param _to             Permission receiver
   * @param _permission     Transferred subset permission of permission owner
   */
  event Transfer(address indexed _from, address indexed _to, uint256 indexed _permission);

  /**
   * MUST trigger on any successful call to `approve(address _delegatee, uint256 _permission)`.
   * @param _owner          Permission owner
   * @param _delegatee      Delegatee
   * @param _permission     Approved subset permission of permission owner
   */
  event Approval(address indexed _owner, address indexed _delegatee, uint256 indexed _permission);

  /**
   * Transfers a subset `_permission` of permission to address `_to`.
   * The function SHOULD revert if the message caller’s account permission does not have the subset
   * of the transferring permissions. The function SHOULD revert if any of transferring permissions are
   * existing on target `_to` address.
   * @param _to             Permission receiver
   * @param _permission     Subset permission of permission owner
   */
  function transfer(address _to, uint256 _permission) external returns (bool success);

  /**
   * Allows `_delegatee` to act for the permission owner's behalf, up to the `_permission`.
   * If this function is called again it overwrites the current granted with `_permission`.
   * `approve()` method SHOULD `revert` if granting `_permission` permission is not
   * a subset of all available permissions of permission owner.
   * @param _delegatee      Delegatee
   * @param _permission     Subset permission of permission owner
   */
  function approve(address _delegatee, uint256 _permission) external returns (bool success);

  /**
   * Returns the permissions of the given `_owner` address.
   */
  function permissionOf(address _owner) external view returns (uint256 permission);

  /**
   * Returns `true` if `_required` is a subset of `_permission` otherwise return `false`.
   * @param _permission     Checking permission set
   * @param _required       Required set of permission
   */
  function permissionRequire(uint256 _permission, uint256 _required) external view returns (bool isPermissioned);

  /**
   * Returns `true` if `_required` permission is a subset of `_actor`'s permissions or a subset of his delegated
   * permission granted by the `_owner`.
   * @param _owner          Permission owner
   * @param _actor          Actor who acts on behalf of the owner
   * @param _required       Required set of permission
   */
  function hasPermission(address _owner, address _actor, uint256 _required) external view returns (bool isPermissioned);

  /**
   * Returns the subset permission of the `_owner` address were granted to `_delegatee` address.
   * @param _owner          Permission owner
   * @param _delegatee      Delegatee
   */
  function delegated(address _owner, address _delegatee) external view returns (uint256 permission);
}
```

### Metadata Interface

It is RECOMMENDED for compliant contracts to implement the optional extension `IEIP6617Meta`.

SHOULD define a description for the base permissions and main combinaison.

SHOULD NOT define a description for every subcombinaison of permissions possible.

### Error Interface

Compatible tokens MAY implement `IEIP6366Error` as defined below:

```solidity
interface IEIP6366Error {
  /**
   * The owner or actor does not have the required permission
   */
  error AccessDenied(address _owner, address _actor, uint256 _permission);

  /**
   * Conflict between permission set
   */
  error DuplicatedPermission(uint256 _permission);

  /**
   * Data out of range
   */
  error OutOfRange();
}
```

## Rationale

Needs discussion.

## Reference Implementation

First implementation could be found here:

- [ERC-6366 Core implementation](./assets/contracts/EIP6366Core.sol)

## Security Considerations

Need more discussion.

## Copyright

Copyright and related rights waived via [CC0](/LICENSE.md).
