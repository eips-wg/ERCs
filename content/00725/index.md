---
eip: 725
title: General data key/value store and execution
description: An interface for a smart contract based account with attachable data key/value store
author: Fabian Vogelsteller (@frozeman), Tyler Yasaka (@tyleryasaka)
discussions-to: https://ethereum-magicians.org/t/discussion-for-eip725/12158
status: Draft
type: Standards Track
category: ERC
created: 2017-10-02
requires: 165, 173
---

## Abstract

The following describes two standards that allow for a generic data storage in a smart contract and a generic execution through a smart contract. These can be used separately or in conjunction and can serve as building blocks for smart contract accounts, upgradable metadata, and other means.

## Motivation

The initial motivation came out of the need to create a smart contract account system that's flexible enough to be viable long-term but also defined enough to be standardized. They are a generic set of two standardized building blocks to be used in all forms of smart contracts.

This standard consists of two sub-standards, a generic data key/value store (`ERC725Y`) and a generic execute function (`ERC725X`). Both of these in combination allow for a very flexible and long-lasting account system. The account version of `ERC725` is standardized under `LSP0-ERC725Account`.

These standards (`ERC725` X and Y) can also be used separately as `ERC725Y` can be used to enhance NFTs and Token metadata or other types of smart contracts. `ERC725X` allows for a generic execution through a smart contract, functioning as an account or actor.

## Specification

### Ownership

This contract is controlled by a single owner. The owner can be a smart contract or an external account.
This standard requires [ERC-173](../00173.md) and SHOULD implement the functions:

- `owner() view`
- `transferOwnership(address newOwner)`

And the event:

- `OwnershipTransferred(address indexed previousOwner, address indexed newOwner)`

---

### `ERC725X`

**`ERC725X`** interface id according to [ERC-165](../00165.md): `0x7545acac`.

Smart contracts implementing the `ERC725X` standard MUST implement the [ERC-165](../00165.md) `supportsInterface(..)` function and MUST support the `ERC165` and `ERC725X` interface ids.

### `ERC725X` Methods

Smart contracts implementing the `ERC725X` standard SHOULD implement all of the functions listed below:

#### execute

```solidity
function execute(uint256 operationType, address target, uint256 value, bytes memory data) external payable returns(bytes memory)
```

Function Selector: `0x44c028fe`

Executes a call on any other smart contracts or address, transfers the blockchains native token, or deploys a new smart contract.


_Parameters:_

- `operationType`: the operation type used to execute.
- `target`: the smart contract or address to call. `target` will be unused if a contract is created (operation types 1 and 2).
- `value`: the amount of native tokens to transfer (in Wei).
- `data`: the call data, or the creation bytecode of the contract to deploy.


_Requirements:_

- MUST only be called by the current owner of the contract.
- MUST revert when the execution or the contract creation fails.
- `target` SHOULD be address(0) in case of contract creation with `CREATE` and `CREATE2` (operation types 1 and 2).
- `value` SHOULD be zero in case of `STATICCALL` or `DELEGATECALL` (operation types 3 and 4).


_Returns:_ `bytes` , the returned data of the called function, or the address of the contract deployed (operation types 1 and 2).

**Triggers Event:** [ContractCreated](#contractcreated), [Executed](#executed)

The following `operationType` COULD exist:

- `0` for `CALL`
- `1` for `CREATE`
- `2` for `CREATE2`
- `3` for `STATICCALL`
- `4` for `DELEGATECALL` - **NOTE** This is a potentially dangerous operation type

Others may be added in the future.

#### data parameter

- For operationType, `CALL`, `STATICCALL` and `DELEGATECALL` the data field can be random bytes or an abi-encoded function call.

- For operationType, `CREATE` the `data` field is the creation bytecode of the contract to deploy appended with the constructor argument(s) abi-encoded.

- For operationType, `CREATE2` the `data` field is the creation bytecode of the contract to deploy appended with:
  1. the constructor argument(s) abi-encoded
  2. a `bytes32` salt.

```
data = <contract-creation-code> + <abi-encoded-constructor-arguments> + <bytes32-salt>
```

> See [EIP-1014: Skinny CREATE2](../01014.md) for more information.

#### executeBatch

```solidity
function executeBatch(uint256[] memory operationsType, address[] memory targets, uint256[] memory values, bytes[] memory datas) external payable returns(bytes[] memory)
```

Function Selector: `0x31858452`

Executes a batch of calls on any other smart contracts, transfers the blockchain native token, or deploys a new smart contract.

_Parameters:_

- `operationsType`: the list of operations type used to execute.
- `targets`: the list of addresses to call. `targets` will be unused if a contract is created (operation types 1 and 2).
- `values`: the list of native token amounts to transfer (in Wei).
- `datas`: the list of call data, or the creation bytecode of the contract to deploy.

_Requirements:_

- Parameters array MUST have the same length.
- MUST only be called by the current owner of the contract.
- MUST revert when the execution or the contract creation fails.
- `target` SHOULD be address(0) in case of contract creation with `CREATE` and `CREATE2` (operation types 1 and 2).
- `value` SHOULD be zero in case of `STATICCALL` or `DELEGATECALL` (operation types 3 and 4).

_Returns:_ `bytes[]` , array list of returned data of the called function, or the address(es) of the contract deployed (operation types 1 and 2).

**Triggers Event:** [ContractCreated](#contractcreated), [Executed](#executed) on each call iteration

### `ERC725X` Events

#### Executed

```solidity
event Executed(uint256 indexed operationType, address indexed target, uint256 indexed value, bytes4 data);
```

MUST be triggered when `execute` creates a new call using the `operationType` `0`, `3`, `4`.

#### ContractCreated

```solidity
event ContractCreated(uint256 indexed operationType, address indexed contractAddress, uint256 indexed value, bytes32 salt);
```

MUST be triggered when `execute` creates a new contract using the `operationType` `1`, `2`.

---

### `ERC725Y`

**`ERC725Y`** interface id according to [ERC-165](../00165.md): `0x629aa694`.

Smart contracts implementing the `ERC725Y` standard MUST implement the [ERC-165](../00165.md) `supportsInterface(..)` function and MUST support the `ERC165` and `ERC725Y` interface ids.

### `ERC725Y` Methods

Smart contracts implementing the `ERC725Y` standard MUST implement all of the functions listed below:

#### getData

```solidity
function getData(bytes32 dataKey) external view returns(bytes memory)
```

Function Selector: `0x54f6127f`

Gets the data set for the given data key.

_Parameters:_

- `dataKey`: the data key which value to retrieve.

_Returns:_ `bytes` , The data for the requested data key.

#### getDataBatch

```solidity
function getDataBatch(bytes32[] memory dataKeys) external view returns(bytes[] memory)
```

Function Selector: `0xdedff9c6`

Gets array of data at multiple given data keys.

_Parameters:_

- `dataKeys`: the data keys which values to retrieve.

_Returns:_ `bytes[]` , array of data values for the requested data keys.

#### setData

```solidity
function setData(bytes32 dataKey, bytes memory dataValue) external
```

Function Selector: `0x7f23690c`

Sets data as bytes in the storage for a single data key. 

_Parameters:_

- `dataKey`: the data key which value to set.
- `dataValue`: the data to store.

_Requirements:_

- MUST only be called by the current owner of the contract.

**Triggers Event:** [DataChanged](#datachanged)

#### setDataBatch

```solidity
function setDataBatch(bytes32[] memory dataKeys, bytes[] memory dataValues) external
```

Function Selector: `0x97902421`

Sets array of data at multiple data keys. MUST only be called by the current owner of the contract.

_Parameters:_

- `dataKeys`: the data keys which values to set.
- `dataValues`: the array of bytes to set.

_Requirements:_

- Array parameters MUST have the same length.
- MUST only be called by the current owner of the contract.

**Triggers Event:** [DataChanged](#datachanged)

### `ERC725Y` Events

#### DataChanged

```solidity
event DataChanged(bytes32 indexed dataKey, bytes dataValue)
```

MUST be triggered when a data key was successfully set.

### `ERC725Y` Data keys

Data keys, are the way to retrieve values via `getData()`. These `bytes32` values can be freely chosen, or defined by a standard.
A common way to define data keys is the hash of a word, e.g. `keccak256('ERCXXXMyNewKeyType')` which results in: `0x6935a24ea384927f250ee0b954ed498cd9203fc5d2bf95c735e52e6ca675e047`

The `LSP2-ERC725JSONSchema` standard is a more explicit `ERC725Y` data key standard, that defines key types and value types, and their encoding and decoding.

## Rationale

The generic way of storing data keys with values was chosen to allow upgradability over time. Stored data values can be changed over time. Other smart contract protocols can then interpret this data in new ways and react to interactions from a `ERC725` smart contract differently.

The data stored in an `ERC725Y` smart contract is not only readable/writable by off-chain applications, but also by other smart contracts. Function overloading was used to allow for the retrievable of single and multiple keys, to keep gas costs minimal for both use cases.

## Backwards Compatibility

All contracts since `ERC725v2` from 2018/19 should be compatible with the current version of the standard. Mainly interface ID and Event parameters have changed, while `getData(bytes32[])` and `setData(bytes32[], bytes[])` was added as an efficient way to set/get multiple keys at once. The same applies to execution, as `execute(..[])` was added as an efficient way to batch calls.

From 2023 onward, overloading was removed from `ERC-725` (including `ERC725-X` and `ERC725-Y`). This is because, while overloading is accommodated in Solidity, it isn't broadly supported across most blockchain languages. In order to make the standard language-independent, it was decided to shift from overloading to simply attach the term "Batch" to the functions that accept an array as parameters.

## Reference Implementation

Reference implementations can be found in [`ERC725.sol`](./assets/ERC725.sol).

## Security Considerations

This contract allows generic executions, therefore special care needs to be taken to prevent re-entrancy attacks and other forms of call chain attacks.

When using the operation type `4` for `delegatecall`, it is important to consider that the called contracts can alter the state of the calling contract and also change owner variables and `ERC725Y` data storage entries at will. Additionally calls to `selfdestruct` are possible and other harmful state-changing operations.

### Solidity Interfaces

```solidity
// SPDX-License-Identifier: CC0-1.0

pragma solidity >=0.5.0 <0.7.0;

// ERC165 identifier: `0x7545acac`
interface IERC725X  /* is ERC165, ERC173 */ {

    event Executed(uint256 indexed operationType, address indexed target, uint256 indexed  value, bytes4 data);
    event ContractCreated(uint256 indexed operationType, address indexed contractAddress, uint256 indexed value, bytes32 salt);


    function execute(uint256 operationType, address target, uint256 value, bytes memory data) external payable returns(bytes memory);

    function executeBatch(uint256[] memory operationsType, address[] memory targets, uint256[] memory values, bytes memory datas) external payable returns(bytes[] memory);
}

// ERC165 identifier: `0x629aa694`
interface IERC725Y /* is ERC165, ERC173 */ {
    
    event DataChanged(bytes32 indexed dataKey, bytes dataValue);

    function getData(bytes32 dataKey) external view returns(bytes memory);
    function getDataBatch(bytes32[] memory dataKeys) external view returns(bytes[] memory);

    function setData(bytes32 dataKey, bytes memory dataValue) external;
    function setDataBatch(bytes32[] memory dataKeys, bytes[] memory dataValues) external;
}
interface IERC725 /* is IERC725X, IERC725Y */ {
}
```

## Copyright

Copyright and related rights waived via [CC0](/LICENSE.md).
