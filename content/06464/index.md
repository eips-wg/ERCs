---
eip: 6464
title: Multi-operator, per-token ERC-721 approvals.
description: Extends ERC-721 to allow token owners to approve multiple operators to control their assets on a per-token basis.
author: Cristian Espinoza (@crisgarner), Simon Fremaux (@dievardump), David Huber (@cxkoda), and Arran Schlosberg (@aschlosberg)
discussions-to: https://ethereum-magicians.org/t/fine-grained-erc721-approval-for-multiple-operators/12796
status: Stagnant
type: Standards Track
category: ERC
created: 2023-02-02
requires: 165, 721
---

## Abstract

[ERC-721](../00721.md) did not foresee the approval of multiple operators to manage a specific token on behalf of its owner. This lead to the establishment of `setApprovalForAll()` as the predominant way to authorise operators, which affords the approved address control over all assets and creates an unnecessarily broad security risk that has already been exploited in a multitude of phishing attacks. The presented EIP extends ERC-721 by introducing a fine-grained, on-chain approval mechanism that allows owners to authorise multiple, specific operators on a per-token basis; this removes unnecessary access permissions and shrinks the surface for exploits to a minimum. The provided reference implementation further enables cheap revocation of all approvals on a per-owner or per-token basis.

## Motivation

The NFT standard defined in ERC-721 allows token owners to "approve" arbitrary addresses to control their tokens—the approved addresses are known as "operators". Two types of approval were defined:

1. `approve(address,uint256)` provides a mechanism for only a single operator to be approved for a given `tokenId`; and
2. `setApprovalForAll(address,bool)` toggles whether an operator is approved for *every* token owned by `msg.sender`.

With the introduction of multiple NFT marketplaces, the ability to approve multiple operators for a particular token is necessary if sellers wish to allow each marketplace to transfer a token upon sale. There is, however, no mechanism for achieving this without using `setApprovalForAll()`. This is in conflict with the principle of least privilege and creates an attack vector that is exploited by phishing for malicious (i.e. zero-cost) sell-side signatures that are executed by legitimate marketplace contracts.

This EIP therefore defines a fine-grained approach for approving multiple operators but scoped to specific token(s).

### Goals

1. Ease of adoption for marketplaces; requires minimal changes to existing workflows.
2. Ease of adoption for off-chain approval-indexing services.
3. Simple revocation of approvals; i.e. not requiring one per grant.

### Non-goals

1. Security measures for protecting NFTs other than through limiting the scope of operator approvals.
2. Compatibility with [ERC-1155](../01155.md) semi-fungible tokens. However we note that the mechanisms described herein are also applicable to ERC-1155 token *types* without requiring approval for all other types.

## Specification

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in RFC 2119 and RFC 8174.

To comply with this EIP, a contract MUST implement `IERC6464` (defined herein) and the `ERC165` and `ERC721` interfaces; see [ERC-165](../00165.md) and ERC-721 respectively.

```solidity
/**
 * @notice Extends ERC-721 to include per-token approval for multiple operators.
 * @dev Off-chain indexers of approvals SHOULD assume that an operator is approved if either of `ERC721.Approval(…)` or
 * `ERC721.ApprovalForAll(…, true)` events are witnessed without the corresponding revocation(s), even if an
 * `ExplicitApprovalFor(…, false)` is emitted.
 * @dev TODO: the ERC-165 identifier for this interface is TBD.
 */
interface IERC6464 is ERC721 {
    /**
     * @notice Emitted when approval is explicitly granted or revoked for a token.
     */
    event ExplicitApprovalFor(
        address indexed operator,
        uint256 indexed tokenId,
        bool approved
    );

    /**
     * @notice Emitted when all explicit approvals, as granted by either `setExplicitApprovalFor()` function, are
     * revoked for all tokens.
     * @dev MUST be emitted upon calls to `revokeAllExplicitApprovals()`.
     */
    event AllExplicitApprovalsRevoked(address indexed owner);

    /**
     * @notice Emitted when all explicit approvals, as granted by either `setExplicitApprovalFor()` function, are
     * revoked for the specific token.
     * @param owner MUST be `ownerOf(tokenId)` as per ERC721; in the case of revocation due to transfer, this MUST be
     * the `from` address expected to be emitted in the respective `ERC721.Transfer()` event.
     */
    event AllExplicitApprovalsRevoked(
        address indexed owner,
        uint256 indexed tokenId
    );

    /**
     * @notice Approves the operator to manage the asset on behalf of its owner.
     * @dev Throws if `msg.sender` is not the current NFT owner, or an authorised operator of the current owner.
     * @dev Approvals set via this method MUST be revoked upon transfer of the token to a new owner; equivalent to
     * calling `revokeAllExplicitApprovals(tokenId)`, including associated events.
     * @dev MUST emit `ApprovalFor(operator, tokenId, approved)`.
     * @dev MUST NOT have an effect on any standard ERC721 approval setters / getters.
     */
    function setExplicitApproval(
        address operator,
        uint256 tokenId,
        bool approved
    ) external;

    /**
     * @notice Approves the operator to manage the token(s) on behalf of their owner.
     * @dev MUST be equivalent to calling `setExplicitApprovalFor(operator, tokenId, approved)` for each `tokenId` in
     * the array.
     */
    function setExplicitApproval(
        address operator,
        uint256[] memory tokenIds,
        bool approved
    ) external;

    /**
     * @notice Revokes all explicit approvals granted by `msg.sender`.
     * @dev MUST emit `AllExplicitApprovalsRevoked(msg.sender)`.
     */
    function revokeAllExplicitApprovals() external;

    /**
     * @notice Revokes all excplicit approvals granted for the specified token.
     * @dev Throws if `msg.sender` is not the current NFT owner, or an authorised operator of the current owner.
     * @dev MUST emit `AllExplicitApprovalsRevoked(msg.sender, tokenId)`.
     */
    function revokeAllExplicitApprovals(uint256 tokenId) external;

    /**
     * @notice Query whether an address is an approved operator for a token.
     */
    function isExplicitlyApprovedFor(address operator, uint256 tokenId)
        external
        view
        returns (bool);
}

interface IERC6464AnyApproval is ERC721 {
    /**
     * @notice Returns true if any of the following criteria are met:
     * 1. `isExplicitlyApprovedFor(operator, tokenId) == true`; OR
     * 2. `isApprovedForAll(ownerOf(tokenId), operator) == true`; OR
     * 3. `getApproved(tokenId) == operator`.
     * @dev The criteria MUST be extended if other mechanism(s) for approving operators are introduced. The criteria
     * MUST include all approval approaches.
     */
    function isApprovedFor(address operator, uint256 tokenId)
        external
        view
        returns (bool);
}
```

## Rationale

### Draft notes to be expanded upon

1. Approvals granted via the newly introduced methods are called *explicit* as a means of easily distinguishing them from those granted via the standard `ERC721.approve()` and `ERC721.setApprovalForAll()` functions. However they follow the same intent: authorising operators to act on the owner's behalf.
2. Abstracting `isApprovedFor()` into `IERC6464AnyApproval` interface, as against keeping it in `IERC6464` allows for modularity of plain `IERC6464` implementations while also standardising the interface for checking approvals when interfacing with specific implementations and any future approval EIPs.
3. Inclusion of an indexed owner address in `AllExplicitApprovalsRevoked(address,uint256)` assists off-chain indexing of existing approvals.
4. Re `IERC6464AnyApproval`: With an increasing number of approval mechanisms it becomes cumbersome for marketplaces to integrate with them since they have to query multiple interfaces to check if they are approved to manage tokens. This provides a streamlined interface, intended to simplify data ingestion for them.

<!--
  The rationale fleshes out the specification by describing what motivated the design and why particular design decisions were made. It should describe alternate designs that were considered and related work, e.g. how the feature is supported in other languages.

  The current placeholder is acceptable for a draft.

  TODO: Remove this comment before submitting
-->

## Backwards Compatibility

This extension was written to allow for the smallest change possible to the original ERC-721 spec while still providing a mechanism to grant, revoke and track approvals of multiple operators on a per-token basis.

Extended contracts remain fully compatible with all existing platforms.

**Note** the `Security Considerations` sub-section on `Other risks` regarding interplay of approval types.

## Reference Implementation

TODO: add internal link to assets directory when the implementation is in place.

An efficient mechanism for broad revocation of approvals via incrementing nonces is included.

## Security Considerations

### Threat model

### Mitigations

### Other risks

TODO: Interplay with `setApprovalForAll()`.

<!--
  All EIPs must contain a section that discusses the security implications/considerations relevant to the proposed change. Include information that might be important for security discussions, surfaces risks and can be used throughout the life cycle of the proposal. For example, include security-relevant design decisions, concerns, important discussions, implementation-specific guidance and pitfalls, an outline of threats and risks and how they are being addressed. EIP submissions missing the "Security Considerations" section will be rejected. An EIP cannot proceed to status "Final" without a Security Considerations discussion deemed sufficient by the reviewers.

  The current placeholder is acceptable for a draft.

  TODO: Remove this comment before submitting
-->

Needs discussion.

## Copyright

Copyright and related rights waived via [CC0](/LICENSE.md).
