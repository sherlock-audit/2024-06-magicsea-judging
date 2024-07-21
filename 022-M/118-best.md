Crazy Ceramic Huskie

Medium

# `MlumStaking::`Improper Handling of `_stakedToken` and `_rewardToken`

## Summary
The MlumStaking contract has a vulnerability where the `_stakedToken` and `_rewardToken` can be set to the same token. This issue arises because the contract does not check if `_stakedToken` is the same as `_rewardToken` in the constructor. If both tokens are the same, the reward calculation mechanism in the `_updatePool()` function will malfunction. This happens because the contract uses the current balance of the reward token for reward calculations, and if the staked token is the same, the reward balance will include staked tokens, leading to incorrect reward distributions.

## Vulnerability Detail
In the MlumStaking contract, the `_stakedToken` and `_rewardToken` are set in the constructor. If `_stakedToken == _rewardToken` is used, there will be serious complications. This is possible because the `_stakedToken` is not checked to ensure it is not the same as the `_rewardToken` address in [MlumStaking::constructor](https://github.com/sherlock-audit/2024-06-magicsea/blob/42e799446595c542eff9519353d3becc50cdba63/magicsea-staking/src/MlumStaking.sol#L82-L97).
```solidity
    constructor(IERC20 _stakedToken, IERC20 _rewardToken) {
        _disableInitializers();

        require(address(_stakedToken) != address(0), "init: zero address");
        require(address(_rewardToken) != address(0), "init: zero address");

@>       stakedToken = _stakedToken;
@>       rewardToken = _rewardToken;

        uint256 decimalsRewardToken = uint256(IERC20Metadata(address(_rewardToken)).decimals());
        require(decimalsRewardToken < 30, "Must be inferior to 30");

        PRECISION_FACTOR = uint256(10 ** (uint256(30) - decimalsRewardToken));

        _stakedSupply = 0;
    }
```
When the `_stakedToken` is the same token as `_rewardToken`, reward calculation for that pool in the `_updatePool()` function can be incorrect. This is because the current balance of the `rewardToken` in the contract is used in the calculation of the reward. Since the `stakedToken` is the same token as the reward, the reward minted to the contract will inflate the value of total supply, causing the reward of that pool to be less than what it should be.
```solidity
  function _updatePool() internal {
        uint256 accRewardsPerShare = _accRewardsPerShare;
        uint256 rewardBalance = rewardToken.balanceOf(address(this));
        uint256 lastRewardBalance = _lastRewardBalance;

        // recompute accRewardsPerShare if not up to date
        if (lastRewardBalance == rewardBalance || _stakedSupply == 0) {
            return;
        }

        uint256 accruedReward = rewardBalance - lastRewardBalance;
        _accRewardsPerShare =
            accRewardsPerShare + ((accruedReward * (PRECISION_FACTOR)) / (_stakedSupplyWithMultiplier));

        _lastRewardBalance = rewardBalance;

        emit PoolUpdated(_currentBlockTimestamp(), accRewardsPerShare);
    }
```

## Impact
The vulnerability can lead to incorrect reward calculations, causing users to receive less reward than they should. This miscalculation occurs because the staked tokens inflate the reward balance, which is then used in the reward distribution logic. Consequently, the actual rewards distributed to stakers will be lower than intended, undermining the integrity and reliability of the staking mechanism.

## Code Snippet
https://github.com/sherlock-audit/2024-06-magicsea/blob/42e799446595c542eff9519353d3becc50cdba63/magicsea-staking/src/MlumStaking.sol#L82-L97

## Tool used
Manual Review

## Recommendation
```diff
    constructor(IERC20 _stakedToken, IERC20 _rewardToken) {
        _disableInitializers();

        require(address(_stakedToken) != address(0), "init: zero address");
        require(address(_rewardToken) != address(0), "init: zero address");
+       require(address(_stakedToken) != address(_rewardToken), "init: _stakedToken == _rewardToken");

        stakedToken = _stakedToken;
        rewardToken = _rewardToken;

        uint256 decimalsRewardToken = uint256(IERC20Metadata(address(_rewardToken)).decimals());
        require(decimalsRewardToken < 30, "Must be inferior to 30");

        PRECISION_FACTOR = uint256(10 ** (uint256(30) - decimalsRewardToken));

        _stakedSupply = 0;
    }

```
