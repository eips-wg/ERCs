---
eip: 6381
title: Public Non-Fungible Token Emote Repository
description: React to any Non-Fungible Tokens using Unicode emojis.
author: Bruno Škvorc (@Swader), Steven Pineda (@steven2308), Stevan Bogosavljevic (@stevyhacker), Jan Turk (@ThunderDeliverer)
discussions-to: https://ethereum-magicians.org/t/eip-6381-emotable-extension-for-non-fungible-tokens/12710
status: Final
type: Standards Track
category: ERC
created: 2023-01-22
requires: 165
---

## Abstract

The Public Non-Fungible Token Emote Repository standard provides an enhanced interactive utility for [ERC-721](../00721.md) and [ERC-1155](../01155.md) by allowing NFTs to be emoted at.

This proposal introduces the ability to react to NFTs using Unicode standardized emoji in a public non-gated repository smart contract that is accessible at the same address in all of the networks.

## Motivation

With NFTs being a widespread form of tokens in the Ethereum ecosystem and being used for a variety of use cases, it is time to standardize additional utility for them. Having the ability for anyone to interact with an NFT introduces an interactive aspect to owning an NFT and unlocks feedback-based NFT mechanics.

This ERC introduces new utilities for [ERC-721](../00721.md) based tokens in the following areas:

- [Interactivity](#interactivity)
- [Feedback based evolution](#feedback-based-evolution)
- [Valuation](#valuation)

### Interactivity

The ability to emote on an NFT introduces the aspect of interactivity to owning an NFT. This can either reflect the admiration for the emoter (person emoting to an NFT) or can be a result of a certain action performed by the token's owner. Accumulating emotes on a token can increase its uniqueness and/or value.

### Feedback based evolution

Standardized on-chain reactions to NFTs allow for feedback based evolution.

Current solutions are either proprietary or off-chain and therefore subject to manipulation and distrust. Having the ability to track the interaction on-chain allows for trust and objective evaluation of a given token. Designing the tokens to evolve when certain emote thresholds are met incentivizes interaction with the token collection.

### Valuation

Current NFT market heavily relies on previous values the token has been sold for, the lowest price of the listed token and the scarcity data provided by the marketplace. There is no real time indication of admiration or desirability of a specific token. Having the ability for users to emote to the tokens adds the possibility of potential buyers and sellers gauging the value of the token based on the impressions the token has collected.

## Specification

The key words “MUST”, “MUST NOT”, “REQUIRED”, “SHALL”, “SHALL NOT”, “SHOULD”, “SHOULD NOT”, “RECOMMENDED”, “MAY”, and “OPTIONAL” in this document are to be interpreted as described in RFC 2119.

```solidity
/// @title ERC-6381 Emotable Extension for Non-Fungible Tokens
/// @dev See https://eips.ethereum.org/EIPS/eip-6381
/// @dev Note: the ERC-165 identifier for this interface is 0xd9fac55a.

pragma solidity ^0.8.16;

interface IERC6381 /*is IERC165*/ {
    /**
     * @notice Used to notify listeners that the token with the specified ID has been emoted to or that the reaction has been revoked.
     * @dev The event MUST only be emitted if the state of the emote is changed.
     * @param emoter Address of the account that emoted or revoked the reaction to the token
     * @param collection Address of the collection smart contract containing the token being emoted to or having the reaction revoked
     * @param tokenId ID of the token
     * @param emoji Unicode identifier of the emoji
     * @param on Boolean value signifying whether the token was emoted to (`true`) or if the reaction has been revoked (`false`)
     */
    event Emoted(
        address indexed emoter,
        address indexed collection,
        uint256 indexed tokenId,
        bytes4 emoji,
        bool on
    );

    /**
     * @notice Used to get the number of emotes for a specific emoji on a token.
     * @param collection Address of the collection containing the token being checked for emoji count
     * @param tokenId ID of the token to check for emoji count
     * @param emoji Unicode identifier of the emoji
     * @return Number of emotes with the emoji on the token
     */
    function emoteCountOf(
        address collection,
        uint256 tokenId,
        bytes4 emoji
    ) external view returns (uint256);

    /**
     * @notice Used to get the number of emotes for a specific emoji on a set of tokens.
     * @param collections An array of addresses of the collections containing the tokens being checked for emoji count
     * @param tokenIds An array of IDs of the tokens to check for emoji count
     * @param emojis An array of unicode identifiers of the emojis
     * @return An array of numbers of emotes with the emoji on the tokens
     */
    function bulkEmoteCountOf(
        address[] memory collections,
        uint256[] memory tokenIds,
        bytes4[] memory emojis
    ) external view returns (uint256[] memory);

    /**
     * @notice Used to get the information on whether the specified address has used a specific emoji on a specific
     *  token.
     * @param emoter Address of the account we are checking for a reaction to a token
     * @param collection Address of the collection smart contract containing the token being checked for emoji reaction
     * @param tokenId ID of the token being checked for emoji reaction
     * @param emoji The ASCII emoji code being checked for reaction
     * @return A boolean value indicating whether the `emoter` has used the `emoji` on the token (`true`) or not
     *  (`false`)
     */
    function hasEmoterUsedEmote(
        address emoter,
        address collection,
        uint256 tokenId,
        bytes4 emoji
    ) external view returns (bool);

    /**
     * @notice Used to get the information on whether the specified addresses have used specific emojis on specific
     *  tokens.
     * @param emoters An array of addresses of the accounts we are checking for reactions to tokens
     * @param collections An array of addresses of the collection smart contracts containing the tokens being checked
     *  for emoji reactions
     * @param tokenIds An array of IDs of the tokens being checked for emoji reactions
     * @param emojis An array of the ASCII emoji codes being checked for reactions
     * @return An array of boolean values indicating whether the `emoter`s has used the `emoji`s on the tokens (`true`)
     *  or not (`false`)
     */
    function haveEmotersUsedEmotes(
        address[] memory emoters,
        address[] memory collections,
        uint256[] memory tokenIds,
        bytes4[] memory emojis
    ) external view returns (bool[] memory);

    /**
     * @notice Used to get the message to be signed by the `emoter` in order for the reaction to be submitted by someone
     *  else.
     * @param collection The address of the collection smart contract containing the token being emoted at
     * @param tokenId ID of the token being emoted
     * @param emoji Unicode identifier of the emoji
     * @param state Boolean value signifying whether to emote (`true`) or undo (`false`) emote
     * @param deadline UNIX timestamp of the deadline for the signature to be submitted
     * @return The message to be signed by the `emoter` in order for the reaction to be submitted by someone else
     */
    function prepareMessageToPresignEmote(
        address collection,
        uint256 tokenId,
        bytes4 emoji,
        bool state,
        uint256 deadline
    ) external view returns (bytes32);

    /**
     * @notice Used to get multiple messages to be signed by the `emoter` in order for the reaction to be submitted by someone
     *  else.
     * @param collections An array of addresses of the collection smart contracts containing the tokens being emoted at
     * @param tokenIds An array of IDs of the tokens being emoted
     * @param emojis An arrau of unicode identifiers of the emojis
     * @param states An array of boolean values signifying whether to emote (`true`) or undo (`false`) emote
     * @param deadlines An array of UNIX timestamps of the deadlines for the signatures to be submitted
     * @return The array of messages to be signed by the `emoter` in order for the reaction to be submitted by someone else
     */
    function bulkPrepareMessagesToPresignEmote(
        address[] memory collections,
        uint256[] memory tokenIds,
        bytes4[] memory emojis,
        bool[] memory states,
        uint256[] memory deadlines
    ) external view returns (bytes32[] memory);

    /**
     * @notice Used to emote or undo an emote on a token.
     * @dev Does nothing if attempting to set a pre-existent state.
     * @dev MUST emit the `Emoted` event is the state of the emote is changed.
     * @param collection Address of the collection containing the token being emoted at
     * @param tokenId ID of the token being emoted
     * @param emoji Unicode identifier of the emoji
     * @param state Boolean value signifying whether to emote (`true`) or undo (`false`) emote
     */
    function emote(
        address collection,
        uint256 tokenId,
        bytes4 emoji,
        bool state
    ) external;

    /**
     * @notice Used to emote or undo an emote on multiple tokens.
     * @dev Does nothing if attempting to set a pre-existent state.
     * @dev MUST emit the `Emoted` event is the state of the emote is changed.
     * @dev MUST revert if the lengths of the `collections`, `tokenIds`, `emojis` and `states` arrays are not equal.
     * @param collections An array of addresses of the collections containing the tokens being emoted at
     * @param tokenIds An array of IDs of the tokens being emoted
     * @param emojis An array of unicode identifiers of the emojis
     * @param states An array of boolean values signifying whether to emote (`true`) or undo (`false`) emote
     */
    function bulkEmote(
        address[] memory collections,
        uint256[] memory tokenIds,
        bytes4[] memory emojis,
        bool[] memory states
    ) external;

    /**
     * @notice Used to emote or undo an emote on someone else's behalf.
     * @dev Does nothing if attempting to set a pre-existent state.
     * @dev MUST emit the `Emoted` event is the state of the emote is changed.
     * @dev MUST revert if the lengths of the `collections`, `tokenIds`, `emojis` and `states` arrays are not equal.
     * @dev MUST revert if the `deadline` has passed.
     * @dev MUST revert if the recovered address is the zero address.
     * @param emoter The address that presigned the emote
     * @param collection The address of the collection smart contract containing the token being emoted at
     * @param tokenId IDs of the token being emoted
     * @param emoji Unicode identifier of the emoji
     * @param state Boolean value signifying whether to emote (`true`) or undo (`false`) emote
     * @param deadline UNIX timestamp of the deadline for the signature to be submitted
     * @param v `v` value of an ECDSA signature of the message obtained via `prepareMessageToPresignEmote`
     * @param r `r` value of an ECDSA signature of the message obtained via `prepareMessageToPresignEmote`
     * @param s `s` value of an ECDSA signature of the message obtained via `prepareMessageToPresignEmote`
     */
    function presignedEmote(
        address emoter,
        address collection,
        uint256 tokenId,
        bytes4 emoji,
        bool state,
        uint256 deadline,
        uint8 v,
        bytes32 r,
        bytes32 s
    ) external;

    /**
     * @notice Used to bulk emote or undo an emote on someone else's behalf.
     * @dev Does nothing if attempting to set a pre-existent state.
     * @dev MUST emit the `Emoted` event is the state of the emote is changed.
     * @dev MUST revert if the lengths of the `collections`, `tokenIds`, `emojis` and `states` arrays are not equal.
     * @dev MUST revert if the `deadline` has passed.
     * @dev MUST revert if the recovered address is the zero address.
     * @param emoters An array of addresses of the accounts that presigned the emotes
     * @param collections An array of addresses of the collections containing the tokens being emoted at
     * @param tokenIds An array of IDs of the tokens being emoted
     * @param emojis An array of unicode identifiers of the emojis
     * @param states An array of boolean values signifying whether to emote (`true`) or undo (`false`) emote
     * @param deadlines UNIX timestamp of the deadline for the signature to be submitted
     * @param v An array of `v` values of an ECDSA signatures of the messages obtained via `prepareMessageToPresignEmote`
     * @param r An array of `r` values of an ECDSA signatures of the messages obtained via `prepareMessageToPresignEmote`
     * @param s An array of `s` values of an ECDSA signatures of the messages obtained via `prepareMessageToPresignEmote`
     */
    function bulkPresignedEmote(
        address[] memory emoters,
        address[] memory collections,
        uint256[] memory tokenIds,
        bytes4[] memory emojis,
        bool[] memory states,
        uint256[] memory deadlines,
        uint8[] memory v,
        bytes32[] memory r,
        bytes32[] memory s
    ) external;
}
```

### Message format for presigned emotes

The message to be signed by the `emoter` in order for the reaction to be submitted by someone else is formatted as follows:

```solidity
keccak256(
        abi.encode(
            DOMAIN_SEPARATOR,
            collection,
            tokenId,
            emoji,
            state,
            deadline
        )
    );
```

The values passed when generating the message to be signed are:

- `DOMAIN_SEPARATOR` - The domain separator of the Emotable repository smart contract
- `collection` - Address of the collection containing the token being emoted at
- `tokenId` - ID of the token being emoted
- `emoji` - Unicode identifier of the emoji
- `state` - Boolean value signifying whether to emote (`true`) or undo (`false`) emote
- `deadline` - UNIX timestamp of the deadline for the signature to be submitted

The `DOMAIN_SEPARATOR` is generated as follows:

```solidity
keccak256(
        abi.encode(
            "ERC-6381: Public Non-Fungible Token Emote Repository",
            "1",
            block.chainid,
            address(this)
        )
    );
```

Each chain, that the Emotable repository smart contract is deployed on, will have a different `DOMAIN_SEPARATOR` value due to chain IDs being different.

### Pre-determined address of the Emotable repository

The address of the Emotable repository smart contract is designed to resemble the function it serves. It starts with `0x311073` which is the abstract representation of `EMOTE`. The address is:

```
0x31107354b61A0412E722455A771bC462901668eA
```

## Rationale

Designing the proposal, we considered the following questions:

1. **Does the proposal support custom emotes or only the Unicode specified ones?**\
The proposal only accepts the Unicode identifier which is a `bytes4` value. This means that while we encourage implementers to add the reactions using standardized emojis, the values not covered by the Unicode standard can be used for custom emotes. The only drawback being that the interface displaying the reactions will have to know what kind of image to render and such additions will probably be limited to the interface or marketplace in which they were made.
2. **Should the proposal use emojis to relay the impressions of NFTs or some other method?**\
The impressions could have been done using user-supplied strings or numeric values, yet we decided to use emojis since they are a well established mean of relaying impressions and emotions.
3. **Should the proposal establish an emotable extension or a common-good repository?**\
Initially we set out to create an emotable extension to be used with any ERC-721 compliant tokens. However, we realized that the proposal would be more useful if it was a common-good repository of emotable tokens. This way, the tokens that can be reacted to are not only the new ones but also the old ones that have been around since before the proposal.\
In line with this decision, we decided to calculate a deterministic address for the repository smart contract. This way, the repository can be used by any NFT collection without the need to search for the address on the given chain.
4. **Should we include only single-action operations, only multi-action operations, or both?**\
We've considered including only single-action operations, where the user is only able to react with a single emoji to a single token, but we decided to include both single-action and multi-action operations. This way, the users can choose whether they want to emote or undo emote on a single token or on multiple tokens at once.\
This decision was made for the long-term viability of the proposal. Based on the gas cost of the network and the number of tokens in the collection, the user can choose the most cost-effective way of emoting.
5. **Should we add the ability to emote on someone else's behalf?**\
While we did not intend to add this as part of the proposal when drafting it, we realized that it would be a useful feature for it. This way, the users can emote on behalf of someone else, for example, if they are not able to do it themselves or if the emote is earned through an off-chain activity.
6. **How do we ensure that emoting on someone else's behalf is legitimate?**\
We could add delegates to the proposal; when a user delegates their right to emote to someone else, the delegate can emote on their behalf. However, this would add a lot of complexity and additional logic to the proposal.\
Using ECDSA signatures, we can ensure that the user has given their consent to emote on their behalf. This way, the user can sign a message with the parameters of the emote and the signature can be submitted by someone else.
7. **Should we add chain ID as a parameter when reacting to a token?**\
During the course of discussion of the proposal, a suggestion arose that we could add chain ID as a parameter when reacting to a token. This would allow the users to emote on the token of one chain on another chain.\
We decided against this as we feel that additional parameter would rarely be used and would add additional cost to the reaction transactions. If the collection smart contract wants to utilize on-chain emotes to tokens they contain, they require the reactions to be recorded on the same chain. Marketplaces and wallets integrating this proposal will rely on reactions to reside in the same chain as well, because if chain ID parameter was supported this would mean that they would need to query the repository smart contract on all of the chains the repository is deployed in order to get the reactions for a given token.\
Additionally, if the collection creator wants users to record their reactions on a different chain, they can still direct the users to do just that. The repository does not validate the existence of the token being reacted to, which in theory means that you can react to non-existent token or to a token that does not exist yet. The likelihood of a different collection existing at the same address on another chain is significantly low, so the users can react using the collection's address on another chain and it is very unlikely that they will unintentionally react to another collection's token.

## Backwards Compatibility

The Emote repository standard is fully compatible with [ERC-721](../00721.md) and with the robust tooling available for implementations of ERC-721 as well as with the existing ERC-721 infrastructure.

## Test Cases

Tests are included in [`emotableRepository.ts`](./assets/test/emotableRepository.ts).

To run them in terminal, you can use the following commands:

```
cd ../assets/eip-6381
npm install
npx hardhat test
```

## Reference Implementation

See [`EmotableRepository.sol`](./assets/contracts/EmotableRepository.sol).

## Security Considerations

The proposal does not envision handling any form of assets from the user, so the assets should not be at risk when interacting with an Emote repository.

The ability to use ECDSA signatures to emote on someone else's behalf introduces the risk of a replay attack, which the format of the message to be signed guards against. The `DOMAIN_SEPARATOR` used in the message to be signed is unique to the repository smart contract of the chain it is deployed on. This means that the signature is invalid on any other chain and the Emote repositories deployed on them should revert the operation if a replay attack is attempted.

Another thing to consider is the ability of presigned message reuse. Since the message includes the signature validity deadline, the message can be reused any number of times before the deadline is reached. The proposal only allows for a single reaction with a given emoji to a specific token to be active, so the presigned message can not be abused to increase the reaction count on the token. However, if the service using the repository relies on the ability to revoke the reaction after certain actions, a valid presigned message can be used to re-react to the token. We suggest that the services using the repository in cnjunction with presigned messages use deadlines that invalidate presigned messages after a reasonalby short period of time.

Caution is advised when dealing with non-audited contracts.

## Copyright

Copyright and related rights waived via [CC0](/LICENSE.md).
