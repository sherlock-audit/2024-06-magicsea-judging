Dapper Basil Crab

Medium

# Inconsistent check in `harvestPositionsTo()` function

## Summary
Inconsistent check in `harvestPositionsTo()` function limits the ability of approved address to harvest on behalf of owner.

## Vulnerability Detail
In the function `harvestPositionsTo()`, function `_requireOnlyApprovedOrOwnerOf()` allows owner or approved address to harvest for the position.

However, the check `(msg.sender == tokenOwner && msg.sender == to)` only allowing the caller to be token owner. Thus these 2 checks are contradicted.
```solidity
function harvestPositionsTo(uint256[] calldata tokenIds, address to) external override nonReentrant {
    _updatePool();

    uint256 length = tokenIds.length;

    for (uint256 i = 0; i < length; ++i) {
        uint256 tokenId = tokenIds[i];
        _requireOnlyApprovedOrOwnerOf(tokenId);
        address tokenOwner = ERC721Upgradeable.ownerOf(tokenId);
        // if sender is the current owner, must also be the harvest dst address
        // if sender is approved, current owner must be a contract
        // @audit not consistent with _requireOnlyApprovedOrOwnerOf()
        require(
            (msg.sender == tokenOwner && msg.sender == to), // legacy || tokenOwner.isContract() 
            "FORBIDDEN"
        );

        _harvestPosition(tokenId, to);
        _updateBoostMultiplierInfoAndRewardDebt(_stakingPositions[tokenId]);
    }
}
```

## Impact
Contradictions in the function `harvestPositionsTo()`. Approved address cannot call `harvestPositionsTo()` on behalf of NFT owner.

## Code Snippet
https://github.com/sherlock-audit/2024-06-magicsea/blob/main/magicsea-staking/src/MlumStaking.sol#L475-L484

## Tool used

Manual Review

## Recommendation
The intended check in function `harvestPositionsTo()` might be, changing `&&` to `||`
```diff
require(
-    (msg.sender == tokenOwner && msg.sender == to), // legacy || tokenOwner.isContract() 
+    (msg.sender == tokenOwner || msg.sender == to), // legacy || tokenOwner.isContract() 
    "FORBIDDEN"
);
```

