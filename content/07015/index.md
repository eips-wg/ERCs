---
eip: 7015
title: NFT Creator Attribution
description: Extending NFTs with cryptographically secured creator attribution.
author: indreams (@strollinghome)
discussions-to: https://ethereum-magicians.org/t/eip-authorship-attribution-for-erc721/14244
status: Review
type: Standards Track
category: ERC
created: 2023-05-11
requires: 55, 155, 712, 721, 1155
---

## Abstract

This Ethereum Improvement Proposal aims to solve the issue of creator attribution for Non-Fungible Token (NFT) standards ([ERC-721](../00721.md), [ERC-1155](../01155.md)). To achieve this, this EIP proposes a mechanism where the NFT creator signs the required parameters for the NFT creation, including the NFT metadata in a hash along with any other relevant information. The signed parameters and the signature are then validated and emitted during the deployment transaction, which allows the NFT to validate the creator and NFT platforms to attribute creatorship correctly. This method ensures that even if a different wallet sends the deployment transaction, the correct account is attributed as the creator.

## Motivation

Current NFT platforms assume that the wallet deploying the smart contract is the creator of the NFT, leading to a misattribution in cases where a different wallet sends the deployment transaction. This happens often when working with smart wallet accounts, and new contract deployment strategies such as the first collector deploying the NFT contract. This proposal aims to solve the problem by allowing creators to sign the parameters required for NFT creation so that any wallet can send the deployment transaction with an signal in a verifiable way who is the creator.

## Specification

The keywords “MUST,” “MUST NOT,” “REQUIRED,” “SHALL,” “SHALL NOT,” “SHOULD,” “SHOULD NOT,” “RECOMMENDED,” “MAY,” and “OPTIONAL” in this document are to be interpreted as described in RFC 2119.

ERC-721 and ERC-1155 compliant contracts MAY implement this NFT Creator Attribution extension to provide a standard event to be emitted that defines the NFT creator at the time of contract creation.

This EIP takes advantage of the fact that contract addresses can be precomputed before a contract is deployed. Whether the NFT contract is deployed through another contract (a factory) or through an EOA, the creator can be correctly attributed using this specification.

**Signing Mechanism**

Creator consent is given by signing an [EIP-712](../00712.md) compatible message; all signatures compliant with this EIP MUST include all fields defined. The struct signed can be any arbitrary data that defines how to create the token; it must hashed in an EIP-712 compatible format with a proper EIP-712 domain.

The following shows some examples of structs that could be encoded into `structHash` (defined below):

```solidity
// example struct that can be encoded in `structHash`; defines that a token can be created with a metadataUri and price:

struct TokenCreation {
  string metadataUri;
  uint256 price;
  uint256 nonce;
}
```

**Signature Validation**

Creator attribution is given through a signature verification that MUST be verified by the NFT contract being deployed and an event that MUST be emitted by the NFT contract during the deployment transaction. The event includes all the necessary fields for reconstructing the signed digest and validating the signature to ensure it matches the specified creator. The event name is `CreatorAttribution` and includes the following fields:

- `structHash`: hashed information for deploying the NFT contract (e.g. name, symbol, admins etc). This corresponds to the value `hashStruct` as defined in the [EIP-712 definition of hashStruct](../00712.md#definition-of-hashstruct) standard.
- `domainName`: the domain name of the contract verifying the signature (for EIP-712 signature validation).
- `version`: the version of the contract verifying the signature (for EIP-712 signature validation)
- `creator`: the creator's account
- `signature`: the creator’s signature

The event is defined as follows:

```solidity
event CreatorAttribution(
  bytes32 structHash,
  string domainName,
  string version,
  address creator,
  bytes signature
);
```

Note that although the `chainId` parameters is necessary for [EIP-712](../00712.md) signatures, we omit the parameter from the event as it can be inferred through the transaction data. Similarly, the `verifyingContract` parameter for signature verification is omitted since it MUST be the same as the `emitter` field in the transaction. `emitter` MUST be the token.

A platform can verify the validity of the creator attribution by reconstructing the signature digest with the parameters emitted and recovering the signer from the `signature` parameter. The recovered signer MUST match the `creator` emitted in the event. If `CreatorAttribution` event is present creator and the signature is validated correctly, attribution MUST be given to the `creator` instead of the account that submitted the transaction.

### Reference Implementation

#### Example signature validator

```solidity
pragma solidity 0.8.20;
import "@openzeppelin/contracts/utils/cryptography/EIP712.sol";
import "@openzeppelin/contracts/utils/cryptography/ECDSA.sol";
import "@openzeppelin/contracts/interfaces/IERC1271.sol";

abstract contract ERC7015 is EIP712 {
  error Invalid_Signature();
  event CreatorAttribution(
    bytes32 structHash,
    string domainName,
    string version,
    address creator,
    bytes signature
  );

  /// @notice Define magic value to verify smart contract signatures (ERC1271).
  bytes4 internal constant MAGIC_VALUE =
    bytes4(keccak256("isValidSignature(bytes32,bytes)"));

  function _validateSignature(
    bytes32 structHash,
    address creator,
    bytes memory signature
  ) internal {
    if (!_isValid(structHash, creator, signature)) revert Invalid_Signature();
    emit CreatorAttribution(structHash, "ERC7015", "1", creator, signature);
  }

  function _isValid(
    bytes32 structHash,
    address signer,
    bytes memory signature
  ) internal view returns (bool) {
    require(signer != address(0), "cannot validate");

    bytes32 digest = _hashTypedDataV4(structHash);

    // if smart contract is the signer, verify using ERC-1271 smart-contract
    /// signature verification method
    if (signer.code.length != 0) {
      try IERC1271(signer).isValidSignature(digest, signature) returns (
        bytes4 magicValue
      ) {
        return MAGIC_VALUE == magicValue;
      } catch {
        return false;
      }
    }

    // otherwise, recover signer and validate that it matches the expected
    // signer
    address recoveredSigner = ECDSA.recover(digest, signature);
    return recoveredSigner == signer;
  }
}
```

## Rationale

By standardizing the `CreatorAttribution` event, this EIP enables platforms to ascertain creator attribution without relying on implicit assumptions. Establishing a standard for creator attribution empowers platforms to manage the complex aspects of deploying contracts while preserving accurate onchain creator information. This approach ensures a more reliable and transparent method for identifying NFT creators, fostering trust among participants in the NFT ecosystem.

[ERC-5375](../05375.md) attempts to solve the same issue and although offchain data offers improved backward compatibility, ensuring accurate and immutable creator attribution is vital for NFTs. A standardized onchain method for creator attribution is inherently more reliable and secure.

In contrast to this proposal, ERC-5375 does not facilitate specifying creators for all tokens within an NFT collection, which is a prevalent practice, particularly in emerging use cases.

Both this proposal and ERC-5375 share similar limitations regarding address-based creator attribution:

> The standard defines a protocol to verify that a certain *address* provided consent. However, it does not guarantee that the address corresponds to the expected creator […]. Proving a link between an address and the entity behind it is beyond the scope of this document.

## Backwards Compatibility

Since the standard requires an event to be emitted during the NFTs deployment transaction, existing NFTs cannot implement this standard.

## Security Considerations

A potential attack exploiting this proposal could involve deceiving creators into signing creator attribution consent messages unintentionally. Consequently, creators MUST ensure that all signature fields correspond to the necessary ones before signing.

## Copyright

Copyright and related rights waived via [CC0](/LICENSE.md).
