---
eip: 7710
title: Smart Contract Delegation
description: Interfaces for consistently delegating capabilities to other contracts or EOAs.
author: Ryan McPeck (@McOso), Dan Finlay (@DanFinlay), Rob Dawson (@rojotek), Derek Chiang (@derekchiang)
discussions-to: https://ethereum-magicians.org/t/towards-more-conversational-wallet-connections-a-proposal-for-the-redeemdelegation-interface/16690
status: Draft
type: Standards Track
category: ERC
created: 2024-05-20
requires: 1271, 7579
---

## Abstract

This proposal introduces a standard way for smart contracts to delegate capabilities to other smart contracts
or Externally Owned Accounts (EOAs).  The delegating contract (delegator) must be able to authorize a
`DelegationManager` contract to call the delegator to execute the desired action.

This framework empowers a delegating contract with the ability to delegate any actions it has the authority to perform,
thereby enabling more flexible and scalable contract interactions. This standard outlines the
minimal interface necessary to facilitate such delegation.

Additionally, this proposal is compatible with [ERC-4337](../04337.md), although its implementation does not
necessitate [ERC-4337](../04337.md).

## Motivation

The development of smart contracts on Ethereum has led to a diverse array of decentralized applications (dApps)
that leverage composability to interact with one another in innovative ways. While current smart contracts are
indeed capable of working together, enabling these interactions, especially in the realm of sharing capabilities
or permissions, remains a tedious and often gas-expensive process, which lacks backwards compatibility.

Currently, for a smart contract to interact with or utilize the functionality of another, it typically requires
hardcoded permissions or the development of bespoke, intermediary contracts. This not only increases the complexity and
development time but also results in higher deployment and execution gas costs. Moreover, the rigid nature of these
interactions limits the ability to adapt to new requirements or to delegate specific, limited permissions in a dynamic
manner.

The proposed standard aims to simplify and standardize the process of delegation between contracts, reducing the
operational complexity and gas costs associated with shared capabilities. By establishing a common framework for
delegating permissions, we can streamline interactions within the Ethereum ecosystem, making contracts more flexible,
cost-effective, and adaptable to the needs of diverse applications. This opens up new possibilities for collaboration
and innovation, allowing dApps to leverage each other's strengths in a more seamless and efficient manner.

## Specification

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "NOT
RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in RFC 2119 and RFC 8174.

### Terms

- A **Delegator** is a smart contract that can create a delegation.
- A **Delegation Manager** is a singleton smart contract that is responsible for validating delegation authority and
  calling on the *Delegator* to execute an action. It implements the `ERC7710Manager` interface.
- A **delegation** is an authority given to another address to perform a specific action.
- A **delegate** is a smart contract, smart contract account, or EOA that has authority to redeem a delegation.
- A **redeemer** is a *delegate* that is using a delegation.

### Overview

#### Redeeming a Delegation

When a delegate wishes to redeem a delegation, they call the `redeemDelegations` function on the Delegation Manager and
pass in the action they want to execute and the proof of authority (ie delegation) which they are executing on behalf
of. The Delegation Manager then verifies the delegation's validity and, if valid, calls the privileged function on the
Delegator which executes the specified capability on behalf of the Delegator.

![diagram showing the flow of redeemDelegations](./assets/diagram.svg)

### Interfaces

#### `ERC7710Manager.sol`

The Delegation Manager MUST implement the `redeemDelegations` which will be responsible for validating the delegations
being redeemed, and will then call the delegators to execute the actions.

The bytes array `_permissionContexts` passed in as a parameter to the `redeemDelegations` function contains the authority to execute a
specific action on behalf of the delegating contract.

The bytes32 array `_modes` and the bytes array `_executionCallDatas` passed in as parameters to the `redeemDelegations` function are arrays of `mode` and `executionCalldata`, which are defined precisely in [ERC-7579](../07579.md) (under the "Execution Behavior" section).  Briefly, `mode` encodes the "behavior" of the execution, which could be a single call, a batch call, and others.  `executionCallData` encodes the data of the execution, which typically includes at least a `target`, a `value`, and a `to` address.

```solidity
pragma solidity 0.8.23;

/**
 * @title ERC7710Manager
 * @notice Interface for Delegation Manager that exposes the redeemDelegations function.
 */
interface ERC7710Manager {
    /**
     * @notice This method validates the provided permission contexts and executes the execution if the caller has authority to do so.
     * @dev the structure of the _permissionContexts bytes[] is determined by the specific Delegation Manager implementation
     * @param _permissionContexts the data used to validate the authority given to execute the corresponding execution.
     * @param _action the action to be executed
     * @param _modes the array of modes to execute the related executioncallData
     * @param _executionCallDatas the array of encoded executions to be executed
     */
    function redeemDelegations(bytes[] calldata _permissionContexts, bytes32[] calldata _modes, bytes[] calldata _executionCallDatas) external;
}
```

## Rationale

The design of this ERC is motivated by the need to introduce standardized, secure, and efficient mechanisms for
delegation within the Ethereum ecosystem. Several considerations were taken into account:

**Flexibility and Scalability**: The proposed interfaces are designed to be minimal yet powerful, allowing contracts to
delegate a wide range of actions without imposing a heavy implementation burden. This balance aims to encourage
widespread adoption and innovation.

**Interoperability**: Compatibility with existing standards, such as [ERC-1271](../01271.md) and [ERC-4337](../04337.md), ensures that this approach
can be seamlessly integrated into the current Ethereum infrastructure. This encourages adoption and leverages existing
security practices.

**Usability**: By enabling contracts to delegate specific actions to others, we open the door to more user-friendly
DApps that can perform a variety of tasks on behalf of users, reducing the need for constant user interaction and
enhancing the overall user experience.

This ERC represents a step towards a more interconnected and flexible Ethereum ecosystem, where smart contracts can more
effectively collaborate and adapt to users' needs.

### Execution Interface

A previous iteration of this spec defined `Action` as a simple `(target, value, data)` tuple, and defined a specific
execution interface on the delegator that is `executeDelegatedAction(Action _action)` which the Delegation Manager is
supposed to call.

That approach had a few downsides:

- Existing smart accounts won't be compatible with this spec (unless they happen to implement the execution interface).
- The execution behavior is limited to a single call, since `Action` could only encode a single call.  It made complex
  execution behaviors such as batching, delegatecall, and CREATE2 impossible.

To solve the first issue, we decided to remove the requirement for the delegator to implement any specific interface.
Rather, we rely on the Delegation Manager to correctly call the delegator, and we rely on the fact that a delegator would
only create `_permissionContexts` for a Delegation Manager that knows how to correctly call it.

To solve the second issue, we decied to adopt the execution interface from [ERC-7579](../07579.md), which had to solve a similar problem
within the context of modular smart accounts: defining a standardized execution interface that can support many types of
executions.

## Security Considerations

The introduction of customizable authorization terms requires careful consideration of how authorization data is
structured and interpreted. Potential security risks include the misinterpretation of authorization terms and
unauthorized actions being taken if the interface is not properly implemented. It is recommended that applications
implementing this interface undergo thorough security audits to ensure that authorization terms are handled securely.

Needs discussion. <!-- TODO -->

## Copyright

Copyright and related rights waived via [CC0](/LICENSE.md).
