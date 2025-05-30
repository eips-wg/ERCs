---
eip: 7743
title: Multi-Owner Non-Fungible Tokens (MO-NFT)
description: A new type of non-fungible token that supports multiple owners, allowing shared ownership and transferability among users.
author: James Savechives (@jamesavechives)
discussions-to: https://ethereum-magicians.org/t/discussion-on-eip-7743-multi-owner-non-fungible-tokens-mo-nft/20577
status: Review
type: Standards Track
category: ERC
created: 2024-07-13
---

## Abstract

This ERC proposes a new standard for non-fungible tokens (NFTs) that supports multiple owners. The MO-NFT standard allows a single NFT to have multiple owners, reflecting the shared and distributable nature of digital assets. This model incorporates mechanisms for provider-defined transfer fees and ownership burning, enabling flexible and collaborative ownership structures. It maintains compatibility with the existing [ERC-721](../00721.md) standard to ensure interoperability with current tools and platforms.

## Motivation

Traditional NFTs enforce a single-ownership model, which does not align with the inherent duplicability and collaborative potential of digital assets. MO-NFTs allow for shared ownership, promoting wider distribution and collaboration while maintaining secure access control. The inclusion of provider fees and ownership burning enhances the utility and flexibility of NFTs in representing digital assets and services.

## Specification

### Token Creation and Ownership Model

1. **Minting**:

   - The function `mintToken()` allows the creation of a new MO-NFT. The caller becomes both the initial owner and the provider of the token.

     ```solidity
     function mintToken() public onlyOwner returns (uint256);
     ```

   - A new `tokenId` is generated, and the caller is added to the owners set and recorded as the provider. The `balanceOf` the caller is incremented.

2. **Ownership List**:

   - The MO-NFT maintains a list of owners for each token. Owners are stored in an enumerable set to prevent duplicates and allow efficient lookup.

3. **Provider Role**:

   - The provider is the initial owner who can set and update the `transferValue` fee. Only the provider can modify certain token parameters.

4. **Transfer Mechanism**:

   - Owners can transfer the token to new owners using `transferFrom`. The transfer adds the new owner to the list without removing existing owners and transfers the `transferValue` fee to the provider.

     ```solidity
     function transferFrom(address from, address to, uint256 tokenId) public;
     ```

### Transfer of Ownership

1. **Additive Ownership**:

   - Transferring ownership adds the new owner to the ownership list without removing current owners. This approach reflects the shared nature of digital assets.

2. **Provider Fee Handling**:

   - During a transfer, the specified `transferValue` fee is transferred to the provider. The contract must have sufficient balance to cover this fee.

3. **Burning Ownership**:

   - Owners can remove themselves from the ownership list using the `burn` function.

     ```solidity
     function burn(uint256 tokenId) external;
     ```

### Interface Definitions

**Minting Functions**

- `function mintToken() public onlyOwner returns (uint256);`

- `function provide(string memory assetName, uint256 size, bytes32 fileHash, address provider, uint256 transferValue) external returns (uint256);`

**Transfer Functions**

- `function transferFrom(address from, address to, uint256 tokenId) public;`

- `function safeTransferFrom(address from, address to, uint256 tokenId) public;` *(Disabled or overridden)*

- `function safeTransferFrom(address from, address to, uint256 tokenId, bytes memory data) public;` *(Disabled or overridden)*

**Ownership Management Functions**

- `function isOwner(uint256 tokenId, address account) public view returns (bool);`

- `function getOwnersCount(uint256 tokenId) public view returns (uint256);`

- `function balanceOf(address owner) external view returns (uint256 balance);`

- `function ownerOf(uint256 tokenId) external view returns (address owner);`

**Provider Functions**

- `function setTransferValue(uint256 tokenId, uint256 newTransferValue) external;`

**Burn Function**

- `function burn(uint256 tokenId) external;`

### Events

- `event TokenMinted(uint256 indexed tokenId, address indexed owner);`

- `event TokenTransferred(uint256 indexed tokenId, address indexed from, address indexed to);`

- `event TokenBurned(uint256 indexed tokenId, address indexed owner);`

- `event TransferValueUpdated(uint256 indexed tokenId, uint256 oldTransferValue, uint256 newTransferValue);`

### [ERC-721](../00721.md) Compliance

The MO-NFT standard is designed to be compatible with the [ERC-721](../00721.md) standard. It implements required functions such as `balanceOf`, `ownerOf`, and `transferFrom` from the `ERC721` interface.

- **Approval Functions**: Functions like `approve`, `getApproved`, `setApprovalForAll`, and `isApprovedForAll` are intentionally disabled or overridden, as they do not align with the MO-NFT multi-owner model.

- **Safe Transfer Functions**: The `safeTransferFrom` functions are restricted because traditional ERC-721 transfer safety checks are not applicable when ownership is additive rather than exclusive.

- **Supports Interface**: The `supportsInterface` function ensures that the MO-NFT declares compatibility with the ERC-721 standard, allowing it to be integrated with existing tools and platforms.

  ```solidity
  function supportsInterface(bytes4 interfaceId) public view returns (bool) {
      return interfaceId == type(IERC721).interfaceId || interfaceId == type(IERC165).interfaceId;
  }
  ```

## Rationale

1. **Multi-Ownership Model**:

   - Digital assets are inherently duplicable and can be shared without loss of quality. The multi-owner model allows broader distribution and collaboration while maintaining a unique token identity.

2. **Additive Ownership**:

   - By adding new owners without removing existing ones, we support shared ownership models common in collaborative environments and digital content distribution.

3. **Provider Fee Mechanism**:

   - Incorporating a provider fee incentivizes creators and providers by rewarding them whenever the asset is transferred. This aligns with models where creators receive royalties or fees for their work.

4. **Ownership Burning**:

   - Allowing owners to remove themselves from the ownership list provides flexibility, enabling owners to relinquish rights or convert their digital ownership into real-world assets.

5. **ERC-721 Compatibility**:

   - Maintaining compatibility with ERC-721 allows MO-NFTs to leverage existing infrastructure, tools, and platforms, facilitating adoption and interoperability.

## Backwards Compatibility

While the MO-NFT standard aims to maintain compatibility with ERC-721, certain deviations are necessary due to the multi-owner model:

- **Approval Functions**: Disabled or overridden to prevent their use, as they do not align with the MO-NFT's ownership structure.

- **Safe Transfer Functions**: Restricted because the additive ownership model does not fit the exclusive ownership assumptions in ERC-721.

- **`ownerOf` Function**: Returns the first owner in the owners list for compatibility, but the concept of a single owner does not fully apply.

Developers should be aware of these differences when integrating MO-NFTs into systems designed for standard ERC-721 tokens.

## Test Cases

1. **Minting an MO-NFT and Verifying Initial Ownership**:

   - **Input**:

     - Call `mintToken()` as the provider.

   - **Expected Output**:

     - A new `tokenId` is generated.

     - The caller is added as the first owner.

     - The `balanceOf` the caller increases by 1.

     - The provider is recorded for the token.

     - `TokenMinted` event is emitted.

2. **Transferring an MO-NFT and Verifying Provider Fee Transfer**:

   - **Input**:

     - Call `transferFrom(from, to, tokenId)` where `from` is an existing owner and `to` is a new address.

   - **Expected Output**:

     - The `to` address is added to the owners list.

     - The `transferValue` fee is transferred to the provider.

     - The `balanceOf` of the `to` address increases by 1.

     - `TokenTransferred` event is emitted.

3. **Burning Ownership**:

   - **Input**:

     - An owner calls `burn(tokenId)`.

   - **Expected Output**:

     - The owner is removed from the owners list.

     - The `balanceOf` of the owner decreases by 1.

     - If the owners list becomes empty, the token is effectively non-existent.

     - `TokenBurned` event is emitted.

4. **Setting Transfer Value**:

   - **Input**:

     - The provider calls `setTransferValue(tokenId, newTransferValue)`.

   - **Expected Output**:

     - The `transferValue` is updated in the contract.

     - `TransferValueUpdated` event is emitted.

5. **Failing Transfer to Existing Owner**:

   - **Input**:

     - Attempt to `transferFrom` to an address that is already an owner.

   - **Expected Output**:

     - The transaction reverts with the error `"MO-NFT: Recipient is already an owner"`.

     - No changes to ownership or balances occur.


## Reference Implementation

The full reference implementation code for the MO-NFT standard is included in the EIPs repository under assets folder. This ensures the code is preserved alongside the EIP and remains accessible.

- **Contracts**:

  - [`MONFT.sol`](./assets/MONFT.sol): The base implementation of the MO-NFT standard.

  - [`DigitalAsset.sol`](./assets/DigitalAsset.sol): An extended implementation for digital assets with provider fees.

- **Interfaces**:

  - [`IDigitalAsset.sol`](./assets/IDigitalAsset.sol): Interface defining the functions for digital asset management.

### Key Functions in Reference Implementation

**Minting Tokens**

```solidity
function mintToken() public onlyOwner returns (uint256) {
    _nextTokenId++;

    // Add the sender to the set of owners for the new token
    _owners[_nextTokenId].add(msg.sender);

    // Increment the balance of the owner
    _balances[msg.sender] += 1;

    // Set the provider to the caller
    _providers[_nextTokenId] = msg.sender;

    emit TokenMinted(_nextTokenId, msg.sender);
    return _nextTokenId;
}
```

**Transferring Tokens**

```solidity
function transferFrom(address from, address to, uint256 tokenId) public override {
    require(isOwner(tokenId, msg.sender), "MO-NFT: Caller is not an owner");
    require(to != address(0), "MO-NFT: Transfer to zero address");
    require(!isOwner(tokenId, to), "MO-NFT: Recipient is already an owner");

    // Add the new owner to the set
    _owners[tokenId].add(to);
    _balances[to] += 1;

    // Transfer the transferValue to the provider
    uint256 transferValue = _transferValues[tokenId];
    address provider = _providers[tokenId];
    require(address(this).balance >= transferValue, "Insufficient contract balance");

    (bool success, ) = provider.call{value: transferValue}("");
    require(success, "Transfer to provider failed");

    emit TokenTransferred(tokenId, from, to);
}
```

**Burning Ownership**

```solidity
function burn(uint256 tokenId) external {
    require(isOwner(tokenId, msg.sender), "MO-NFT: Caller is not an owner");

    // Remove the caller from the owners set
    _owners[tokenId].remove(msg.sender);
    _balances[msg.sender] -= 1;

    emit TokenBurned(tokenId, msg.sender);
}
```

## Security Considerations

1. **Reentrancy Attacks**:

   - **Mitigation**: Use the Checks-Effects-Interactions pattern when transferring Ether (e.g., transferring `transferValue` to the provider).

   - **Recommendation**: Consider using `ReentrancyGuard` from OpenZeppelin to prevent reentrant calls.

2. **Integer Overflow and Underflow**:

   - **Mitigation**: Solidity 0.8.x automatically checks for overflows and underflows, throwing exceptions when they occur.

3. **Access Control**:

   - **Ensured By**:

     - Only owners can call transfer functions.

     - Only providers can set the `transferValue`.

     - Use of `require` statements to enforce access control.

4. **Denial of Service (DoS)**:

   - **Consideration**: Functions that iterate over owners could be expensive in terms of gas if the owners list is large.

   - **Mitigation**: Avoid such functions or limit the number of owners.

5. **Data Integrity**:

   - **Ensured By**: Proper use of Solidity's data types and structures, and by emitting events for all state-changing operations for off-chain verification.

6. **Ether Handling**:

   - **Consideration**: Ensure the contract can receive Ether to handle provider payments.

   - **Mitigation**: Implement a `receive()` function to accept Ether.

## Copyright

Copyright and related rights waived via [CC0](/LICENSE.md).
