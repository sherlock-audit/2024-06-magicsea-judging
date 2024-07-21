Young Iron Beaver

High

# Precision Loss in lockMultiplier Calculation Leads to Potential Loss of Reward Tokens in MlumStaking Contract

## Summary

The MlumStacking contract has an precision loss vulnerability in the `getMultiplierByLockDuration` function, which determines the lockMultiplier based on the duration a user locks their tokens. When the lockDuration is less than _maxLockDuration, the calculated lockMultiplier is truncated, leading to a lower-than-expected multiplier. This truncation causes a smaller amountWithMultiplier to be computed, ultimately resulting in users receiving fewer rewardTokens than anticipated, causing potential economic losses and unfair reward distribution.

## Vulnerability Detail

In the MlumStaking contract, the `getMultiplierByLockDuration` function returns the corresponding lockMultiplier based on the lockDuration.

```solidity
    function getMultiplierByLockDuration(uint256 lockDuration) public view returns (uint256) {
        // in case of emergency unlock
        if (isUnlocked()) return 0;

        if (_maxLockDuration == 0 || lockDuration == 0) return 0;

        // capped to maxLockDuration
        if (lockDuration >= _maxLockDuration) return _maxLockMultiplier;

        return (_maxLockMultiplier * lockDuration) / (_maxLockDuration);
    }
```

The logic of the function is that if lockDuration is greater than or equal to _maxLockDuration, then lockMultiplier is _maxLockMultiplier. Otherwise, the multiplier is calculated based on the ratio of lockDuration to _maxLockDuration.

The lockMultiplier computed for a position is used to calculate amountWithMultiplier in the `createPosition` and `_updateBoostMultiplierInfoAndRewardDebt` functions as shown below.

```solidity
function createPosition(uint256 amount, uint256 lockDuration) external override nonReentrant {
        // skip
        // calculate bonuses
        uint256 lockMultiplier = getMultiplierByLockDuration(lockDuration);
        uint256 amountWithMultiplier = amount * (lockMultiplier + 1e4) / 1e4;
        // skip
}

function _updateBoostMultiplierInfoAndRewardDebt(StakingPosition storage position) internal {
        // keep the original lock multiplier and recompute current boostPoints multiplier
        uint256 newTotalMultiplier = position.lockMultiplier;
        if (newTotalMultiplier > _maxGlobalMultiplier) newTotalMultiplier = _maxGlobalMultiplier;

        position.totalMultiplier = newTotalMultiplier;
        uint256 amountWithMultiplier = position.amount * (newTotalMultiplier + 1e4) / 1e4;
        // update global supply
        _stakedSupplyWithMultiplier = _stakedSupplyWithMultiplier - position.amountWithMultiplier + amountWithMultiplier;
        position.amountWithMultiplier = amountWithMultiplier;

        position.rewardDebt = amountWithMultiplier * _accRewardsPerShare / PRECISION_FACTOR;
}
```

The position’s amountWithMultiplier will determine the subsequent rewards the position can receive.

```solidity
function _harvestPosition(uint256 tokenId, address to) internal {
        StakingPosition storage position = _stakingPositions[tokenId];

        // compute position's pending rewards
        uint256 pending = position.amountWithMultiplier * _accRewardsPerShare / PRECISION_FACTOR - position.rewardDebt;

        // transfer rewards
        if (pending > 0) {
            // send rewards
            _safeRewardTransfer(to, pending);
        }
        emit HarvestPosition(tokenId, to, pending);
}
```

Therefore, it is evident that the calculations of lockMultiplier and amountWithMultiplier are crucial, affecting the subsequent acquisition of rewardTokens. However, there are precision losses in the calculation of these two important variables, causing users to potentially receive fewer rewardTokens than expected, leading to economic losses.

Let’s illustrate with the default contract values, i.e., _maxLockMultiplier value of 20000 and _maxLockDuration of 365 days. 

Assuming Alice stakes 1 ether of stakedToken for 30 days. In the same time (same block), Bob staked 0.1 ether of stakedToken for 365 days. For Alice, the `getMultiplierByLockDuration` function will calculate and return a lockMultiplier of 1643, with amountWithMultiplier being 1164300000000000000. For Bob, since he staked tokens with the maxLockDuration, his position's lockMultiplier will be 2000, and the amountWithMultiplier will be 300000000000000000.

Then, after the contract is distributed total of 1_000_000_000_000_000 reward tokens, Alice and Bob call 'HarvestPosition' to obtain rewards, which will result in 795123950.010243 and 204876049.989756 reward tokens, respectively.

The poc is as follows:

```solidity
function testgetMultiplierByLockDuration() public {
        vm.startPrank(ALICE);
        _stakingToken.mint(ALICE, 1 ether);
        _stakingToken.approve(address(_pool), 1 ether);
        _pool.createPosition(1 ether, 30 days);
        IMlumStaking.StakingPosition memory p = _pool.getStakingPosition(1);
        emit log_named_uint("lockMultiplier", p.lockMultiplier);
        emit log_named_uint("amountWithMultiplier", p.amountWithMultiplier);

        vm.startPrank(BOB);
        _stakingToken.mint(BOB, 0.1 ether);
        _stakingToken.approve(address(_pool), 0.1 ether);
        _pool.createPosition(0.1 ether, 365 days);
        p = _pool.getStakingPosition(2);
        emit log_named_uint("lockMultiplier", p.lockMultiplier);
        emit log_named_uint("amountWithMultiplier", p.amountWithMultiplier);

        _rewardToken.mint(address(_pool), 1_000_000_000_000_000);

        vm.startPrank(ALICE);
        _pool.harvestPosition(1);
        emit log_named_decimal_uint("Alice reward", _rewardToken.balanceOf(ALICE), _rewardToken.decimals());

        vm.startPrank(BOB);
        _pool.harvestPosition(2);
        emit log_named_decimal_uint("Bob reward", _rewardToken.balanceOf(BOB), _rewardToken.decimals());
}
```

Output:
```solidity
Logs:
        lockMultiplier: 1643
        amountWithMultiplier: 1164300000000000000
        lockMultiplier: 20000
        amountWithMultiplier: 300000000000000000
        Alice reward: 795123950.010243
        Bob reward: 204876049.989756
```

Since the multiplier is scaled by 1e4, a multiplier of 20000 means that the additional boost amount will be twice the original amount, meaning that the final amountWithMultiplier will be three times the original amount. Therefore, here, for Alice, the final amountWithMultiplier will be 1.1643 times the amount, and for Bob, 3 times the amount.

However, due to precision loss in the `getMultiplierByLockDuration` function, the actual lockMultiplier for Alice should be (20000 * 30 * 86400) / (365 * 86400), approximately 1643.835616438356164383, but it truncates to 1643. Therefore, the actual amountWithMultiplier should be approximately 1.1643835616438356164383 times the amount, i.e., 1164383561643835616. Thus, the amountWithMultiplier calculated in the contract is less than the actual expected amount, leading to Alice receiving fewer rewardTokens than expected in subsequent reward calculations (this also means Alice could choose to stake slightly less than 30 days and still receive the same lockMultiplier and amountWithMultiplier). Correspondingly, Bob obtained more reward tokens, which should have belonged to Alice.

## Impact

The precision loss in the `getMultiplierByLockDuration` function may result in a lower calculated lockMultiplier for user positions when the lockDuration is less than _maxLockDuration. This, in turn, leads to a smaller amountWithMultiplier, thereby causing users to receive fewer rewardTokens than expected, potentially resulting in financial losses.

## Code Snippet
https://github.com/sherlock-audit/2024-06-magicsea/blob/42e799446595c542eff9519353d3becc50cdba63/magicsea-staking/src/MlumStaking.sol#L214-L227

https://github.com/sherlock-audit/2024-06-magicsea/blob/42e799446595c542eff9519353d3becc50cdba63/magicsea-staking/src/MlumStaking.sol#L369-L371

https://github.com/sherlock-audit/2024-06-magicsea/blob/42e799446595c542eff9519353d3becc50cdba63/magicsea-staking/src/MlumStaking.sol#L656-L665

## Tool used

Manual Review

## Recommendation

It is suggested to modify the calculation methods of `lockMultiplier` and `amountWithMultiplier` to minimize precision loss. A possible modification is as follows:

```diff
    function getMultiplierByLockDuration(uint256 lockDuration) public view returns (uint256) {
        // in case of emergency unlock
        if (isUnlocked()) return 0;

        if (_maxLockDuration == 0 || lockDuration == 0) return 0;

        // capped to maxLockDuration
-        if (lockDuration >= _maxLockDuration) return _maxLockMultiplier;
+        if (lockDuration >= _maxLockDuration) return _maxLockMultiplier * 1e18;

-        return (_maxLockMultiplier * lockDuration) / (_maxLockDuration);
+       return (_maxLockMultiplier * lockDuration * 1e18) / (_maxLockDuration);
    }

    function createPosition(uint256 amount, uint256 lockDuration) external override nonReentrant {
        // skip
        // calculate bonuses
        uint256 lockMultiplier = getMultiplierByLockDuration(lockDuration);
-        uint256 amountWithMultiplier = amount * (lockMultiplier + 1e4) / 1e4;
+       uint256 amountWithMultiplier = amount + (amount * lockMultiplier / 1e4) / 1e18;
        // skip
```

Then test with the previous poc script.

```solidity
function testgetMultiplierByLockDuration() public {
        vm.startPrank(ALICE);
        _stakingToken.mint(ALICE, 1 ether);
        _stakingToken.approve(address(_pool), 1 ether);
        _pool.createPosition(1 ether, 30 days);
        IMlumStaking.StakingPosition memory p = _pool.getStakingPosition(1);
        emit log_named_uint("lockMultiplier", p.lockMultiplier);
        emit log_named_uint("amountWithMultiplier", p.amountWithMultiplier);

        vm.startPrank(BOB);
        _stakingToken.mint(BOB, 0.1 ether);
        _stakingToken.approve(address(_pool), 0.1 ether);
        _pool.createPosition(0.1 ether, 365 days);
        p = _pool.getStakingPosition(2);
        emit log_named_uint("lockMultiplier", p.lockMultiplier);
        emit log_named_uint("amountWithMultiplier", p.amountWithMultiplier);

        _rewardToken.mint(address(_pool), 1_000_000_000_000_000);

        vm.startPrank(ALICE);
        _pool.harvestPosition(1);
        emit log_named_decimal_uint("Alice reward", _rewardToken.balanceOf(ALICE), _rewardToken.decimals());

        vm.startPrank(BOB);
        _pool.harvestPosition(2);
        emit log_named_decimal_uint("Bob reward", _rewardToken.balanceOf(BOB), _rewardToken.decimals());
}
```

Output:
```solidity
Logs:
        lockMultiplier: 1643835616438356164383
        amountWithMultiplier: 1164383561643835616
        lockMultiplier: 20000000000000000000000
        amountWithMultiplier: 300000000000000000
        Alice reward: 795135640.785781
        Bob reward: 204864359.214218
```

For Alice, the final amountWithMultiplier will be 1164383561643835616, which is more precise than the previously calculated 1164300000000000000.  And Alice will receive 795135640.785781 reward tokens, which is approximately 11690 more than when there was an precision loss in the original calculation.

Moreover, other code of the calculation logic, such as the `_updateBoostMultiplierInfoAndRewardDebt` function, should also be adjusted accordingly.
