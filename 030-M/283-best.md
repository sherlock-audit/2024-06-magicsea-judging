Icy Basil Seal

Medium

# Withdrawing a stake position will cause bribe rewards to be unclaimable

## Summary

Each MLUM staking position is associated with a (non-transferrable) NFT, associated with one voting ballot. However withdrawing a stake position will burn the NFT, causing bribe rewards to be unclaimable.

## Vulnerability Detail

When users stake MLUM for at least a specified duration and a minimum amount threshold, they receive voting power to direct LUM incentives into LP pools. For each lock positions, a non-transferrable NFT is created.

When a user withdraws their position, the NFT corresponding to the position is burned.

```solidity
function _destroyPosition(uint256 tokenId) internal {
    // burn lsNFT
    delete _stakingPositions[tokenId];
    ERC721Upgradeable._burn(tokenId);
}
```

- Function `withdrawFromPosition()`: https://github.com/sherlock-audit/2024-06-magicsea/blob/main/magicsea-staking/src/MlumStaking.sol#L496
- Internal function `_withdrawFromPosition()`: https://github.com/sherlock-audit/2024-06-magicsea/blob/main/magicsea-staking/src/MlumStaking.sol#L643
- Internal function `_destroyPosition()`: https://github.com/sherlock-audit/2024-06-magicsea/blob/main/magicsea-staking/src/MlumStaking.sol#L602

People can set up bribes to incentivize voters to vote into their pool. Voters who vote into "bribed" pools will earn bribe rewards. 

User can claim the bribe reward through the function `BribeRewarder.claim()`. It will then call into `_modify()`

```solidity
function claim(uint256 tokenId) external override {
    // ...
    for (uint256 i = _startVotingPeriod; i <= endPeriod; ++i) {
        totalAmount += _modify(i, tokenId, 0, true); // @audit this will have an ownership check
    }
    // ...
}
```

https://github.com/sherlock-audit/2024-06-magicsea/blob/main/magicsea-staking/src/rewarders/BribeRewarder.sol#L160

which has an NFT ownership check:

```solidity
function _modify(uint256 periodId, uint256 tokenId, int256 deltaAmount, bool isPayOutReward)
    private
    returns (uint256 rewardAmount)
{
    if (!IVoter(_caller).ownerOf(tokenId, msg.sender)) {
        revert BribeRewarder__NotOwner(); // @audit NFT ownership check will revert if NFT is burned
    }
    // ...
}
```

https://github.com/sherlock-audit/2024-06-magicsea/blob/main/magicsea-staking/src/rewarders/BribeRewarder.sol#L264-L266

https://github.com/sherlock-audit/2024-06-magicsea/blob/main/magicsea-staking/src/Voter.sol#L402-L404

Therefore when a position is fully withdrawn, the bribe rewards became impossible to claim and will remain stuck in the contract.

## Impact

Fully withdrawing a stake position will cause bribe rewards to be unclaimable and remain stuck in the contract

## Code Snippet

When withdrawing, the position is destroyed and the NFT is burned: https://github.com/sherlock-audit/2024-06-magicsea/blob/main/magicsea-staking/src/MlumStaking.sol#L602

When claiming, there is an ownership check: https://github.com/sherlock-audit/2024-06-magicsea/blob/main/magicsea-staking/src/rewarders/BribeRewarder.sol#L264-L266

## Tool used

Manual Review

## Recommendation

When a position casts a vote, the `Voter` contract should record the owner at the time of voting. When the user wants to claim a reward, the ownership check should be that of the voter of that period.

