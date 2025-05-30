---
eip: 7884
title: Operation Router
description: A protocol that enables smart contracts to redirect write operations to external systems.
author: Lucas Picollo (@pikonha), Alex Netto (@alextnetto), Nick Johnson (@arachnid)
discussions-to: https://ethereum-magicians.org/t/operation-router/22633
status: Draft
type: Standards Track
category: ERC
created: 2025-01-23
---

## Abstract

This EIP introduces a protocol that enables smart contracts to redirect write operations to external systems. The protocol defines a standardized way for contracts to indicate that an operation should be handled by either a contract deployed to an L2 chain, to the L1, or an off-chain database, providing an entry point for easy developer experience and client implementations.

## Motivation

As the Ethereum ecosystem grows, there is an increasing need for efficient ways to manage data storage across different layers and systems.

This protocol addresses these challenges by:

- Providing a gas-efficient way to determine operation handlers through view functions
- Enabling seamless integration with L2 solutions and off-chain databases
- Maintaining strong security guarantees through typed signatures and standardized interfaces

## Specification

### Core Components

The protocol consists of three main components:

1. A view function named interface `getOperationHandler` for determining operation handlers that can be one of the following types:
a. `OperationHandledOnchain` for on-chain handlers
b. `OperationHandledOffchain` for off-chain handlers through a gateway
2. A standardized message format for off-chain storage authorization

### Interface

```solidity
interface OperationRouter {

  /** 
    * @dev Error to raise when an encoded function that is not supported
    * @dev is received on the getOperationHandler function
    */
  error FunctionNotSupported();

  /**
    * @dev Error to raise when mutations are being deferred onchain
    * that being the layer 1 or a layer 2
    * @param chainId Chain ID to perform the deferred mutation to.
    * @param contractAddress Contract Address at which the deferred mutation should transact with.
    */
  error OperationHandledOnchain(
      uint256 chainId,
      address contractAddress
  );

  /**
    * @notice Struct used to define the domain of the typed data signature, defined in EIP-712.
    * @param name The user friendly name of the contract that the signature corresponds to.
    * @param version The version of domain object being used.
    * @param chainId The ID of the chain that the signature corresponds to
    * @param verifyingContract The address of the contract that the signature pertains to.
    */
  struct DomainData {
      string name;
      string version;
      uint64 chainId;
      address verifyingContract;
  }

  /**
    * @notice Struct used to define the message context for off-chain storage authorization
    * @param data The original ABI encoded function call
    * @param sender The address of the user performing the mutation (msg.sender).
    * @param expirationTimestamp The timestamp at which the mutation will expire.
    */
  struct MessageData {
      bytes data;
      address sender;
      uint256 expirationTimestamp;
  }

  /**
    * @dev Error to raise when mutations are being deferred to an Offchain entity
    * @param sender the EIP-712 domain definition
    * @param url URL to request to perform the off-chain mutation
    * @param data The original ABI encoded function call along with authorization context
    */
  error OperationHandledOffchain(
      DomainData sender,
      string url,
      MessageData data
  );

  /**
    * @notice Determines the appropriate handler for an encoded function call
    * @param encodedFunction The ABI encoded function call
    */
  function getOperationHandler(bytes calldata encodedFunction) external view;
}

```

The onchain flow is specified as follows:

![](./assets/d1.svg)

It is important to notice that the `getOperationHandler` relies on the given argument, the encoded function, to specify which contract will the request be redirected to, therefore, it is unable to address `multicall` transactions that could lead to different destination contracts. That means that `multicall` that is known will be redirected to different contracts should be handled in a sequential way by first calling the `getOperationHandler` and then making the actual transaction to the returned contract.

#### Database flow

The HTTP request made to the gateway follows the same standard proposed by the [EIP-3668](./eip-3668) where the URL receives `/{sender}/{data}.json` enabling an API to behave just like an smart contract would. However, the [EIP-712 Typed Signature](../00712.md) was introduced to enable authentication.

![](./assets/d2.svg)

### Implementation Example

The contract deployed to the L1 MUST implement the `getOperationHandler` to act as a router redirecting the requests to the respective handler.

```solidity
contract OperationRouterExample {
    function getOperationHandler(bytes calldata encodedFunction) external view {
        bytes4 selector = bytes4(encodedFunction[:4]);

        if (selector == bytes4(keccak256("setText(bytes32, string)"))) {
            revert OperationHandledOffchain(
                DomainData(
                    "IdentityResolver",
                    "1",
                    1,
                    address(this)
                ),
                "https://api.example.com/profile",
                MessageData(
                    encodedFunction,
                    msg.sender,
                    block.timestamp + 1 hours
                )
            );
        }

        if (selector == bytes4(keccak256("setAddress(bytes32,address)"))) {
            revert OperationHandledOnchain(
                10,
                address(0x123...789)
            );
        }
    }
}

```

The client implementation would look as follows:

```tsx
 try {
  const calldata = {
    functionName: 'setText',
    abi,
    args: [key, value],
    address,
    account
  }
 
  await client.readContract({
    functionName: 'getOperationHandler',
    abi,
    args: [encodeFunctionData(calldata)],
  })
} catch (err) {
  const data = getRevertErrorData(err)

  switch (data?.errorName) {
    case 'OperationHandledOffchain': {
      const [domain, url, message] = errorResult.args as [
        DomainData,
        string,
        MessageData,
      ]
      await handleDBStorage({ domain, url, message, signer })
    }
    case 'OperationHandledOnchain': {
      const [chainId, contractAddress] = data.args as [bigint, `0x${string}`]

      const l2Client = createPublicClient({
        chain: getChain(Number(chainId)),
        transport: http(),
      }).extend(walletActions)

      const { request } = await l2Client.simulateContract({
        ...calldata,
        address: contractAddress,
      })
      await l2Client.writeContract(request)
    }
    default:
      console.error('error registering domain: ', { err })
  }
```

## Rationale

The standard aims to enable offchain writing operations, designed to be a complement for the CCIP-Read ([ERC-3668](./eip-3668)) which is already widely adopted by the community.

## Backwards Compatibility

This EIP is fully backward compatible as it:

- Introduces new interfaces that don't conflict with existing ones
- Uses view functions to gather offchain information
- Can be implemented alongside existing storage patterns

## Security Considerations

### Handler Validation

Off-chain handlers must:

- Verify EIP-712 signatures
- Implement proper access controls
- Handle concurrent modifications safely

### General Recommendations

- Implement rate limiting for off-chain handlers
- Use secure transport (HTTPS) for off-chain communications
- Monitor for unusual patterns that might indicate attacks
- Implement proper error handling for failed transactions

## Copyright

Copyright and related rights waived via [CC0](/LICENSE.md).
