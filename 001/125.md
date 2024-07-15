Young Iron Beaver

Medium

# Calling `setLockMultiplierSettings` After Staking Leads to Potential Unfair Reward Distribution

## Summary

The `MlumStaking` contract initializes the `_maxLockMultiplier` with an incorrect value of 20000 (200%), which exceeds the defined constant `MAX_LOCK_MULTIPLIER_LIMIT` of 15000 (150%). This mismatch can lead to unintended and potentially unfair reward distributions among stakers when the multiplier is corrected after staking has begun.

## Vulnerability Detail

In the `MlumStaking` contract, there exists a variable `MAX_LOCK_MULTIPLIER_LIMIT` initially set to 15000 (note that there are no functions to modify this variable, meaning it is immutable). According to the CODE COMMENTS, this indicates a 150% multiplier, which is the upper limit for `maxLockMultiplier`.
```solidity
uint256 public constant MAX_LOCK_MULTIPLIER_LIMIT = 15000; // 150%, high limit for maxLockMultiplier (100 = 1%)
```

This is further confirmed by the `setLockMultiplierSettings` function's CODE COMMENTS and code, which ensure that `_maxLockMultiplier` cannot exceed `MAX_LOCK_MULTIPLIER_LIMIT`.
```solidity
    /**
     * @dev Set lock multiplier settings
     *
     * maxLockMultiplier must be <= MAX_LOCK_MULTIPLIER_LIMIT
     * maxLockMultiplier must be <= _maxGlobalMultiplier - _maxBoostMultiplier
     *
     * Must only be called by the owner
     */
    function setLockMultiplierSettings(uint256 maxLockDuration, uint256 maxLockMultiplier) external {
        require(msg.sender == owner() || msg.sender == _operator, "FORBIDDEN");
        require(maxLockMultiplier <= MAX_LOCK_MULTIPLIER_LIMIT, "too high");
        _maxLockDuration = maxLockDuration;
        _maxLockMultiplier = maxLockMultiplier;

        emit SetLockMultiplierSettings(maxLockDuration, maxLockMultiplier);
    }
```

However, `_maxLockMultiplier` was incorrectly initialized to 20000, or 200%, exceeding the `MAX_LOCK_MULTIPLIER_LIMIT`.

The potential consequence of this erroneous setting is that once staking begins, if the team discovers the error in `_maxLockMultiplier` and wishes to amend it, it must be set to no more than 15000 (i.e., `MAX_LOCK_MULTIPLIER_LIMIT`). This could lead to a permanent unfair distribution of rewards, possibly allowing users who staked before the correction to receive more rewards compared to those who staked later.

For instance:
1. After staking begins, user A calls `createPosition` to stake 100 `stakedToken` with a `lockDuration` of 365 days, at block T. At this point, `_maxLockMultiplier` is 20000 and `_maxLockDuration` is 365 days. Therefore, according to the `getMultiplierByLockDuration` function, the position’s `lockMultiplier` is `_maxLockMultiplier`, or 20000.
```solidity
    function getMultiplierByLockDuration(uint256 lockDuration) public view returns (uint256) {
        if (isUnlocked()) return 0;
        if (_maxLockDuration == 0 || lockDuration == 0) return 0;
        if (lockDuration >= _maxLockDuration) return _maxLockMultiplier;
        return (_maxLockMultiplier * lockDuration) / (_maxLockDuration);
    }
```
Thus, the `amountWithMultiplier` is 300.
```solidity
        uint256 lockMultiplier = getMultiplierByLockDuration(lockDuration);
        uint256 amountWithMultiplier = amount * (lockMultiplier + 1e4) / 1e4;
```

2. In a later block, T+10, the team realizes the error in `_maxLockMultiplier` and calls `setLockMultiplierSettings` to correct it. Since `MAX_LOCK_MULTIPLIER_LIMIT` is 15000, the updated `_maxLockMultiplier` cannot exceed this, and let’s assume it is reset to 10000 while `maxLockDuration` remains 365 days.

3. After the update, in block T+20, user B also calls `createPosition` to stake 100 `stakedToken` with a `lockDuration` of 365 days. At this time, `_maxLockMultiplier` is 10000 and `_maxLockDuration` is 365 days. Therefore, the `lockMultiplier` for this position is 10000, making the `amountWithMultiplier` 200.

4. Clearly, staking the same 100 `stakedToken` for one year, the change in `_maxLockMultiplier` results in user A’s `amountWithMultiplier` being higher than user B's. Consequently, user A will always receive more rewardTokens than user B when rewards are calculated.

This issue arises if there are existing stakes when the `setLockMultiplierSettings` is called to modify `_maxLockMultiplier` and `maxLockDuration`, leading to an unfair distribution of rewards, where staking the same amount and duration of `stakedToken` before and after the modification yields different rewards.

Please note, although we should assume that a trustworthy owner and operator would not modify `maxLockMultiplier` and `maxLockDuration` after staking has started (to avoid unfair reward distribution), according to the related code and CODE COMMENTS (Hierarchy of truth) it is expected that `_maxLockMultiplier` should not exceed `MAX_LOCK_MULTIPLIER_LIMIT`, and since `MAX_LOCK_MULTIPLIER_LIMIT` is immutable and initialized at 15000, clearly initializing `_maxLockMultiplier` at 20000 is a mistake and not in line with the protocol’s expected behavior. This could lead to the correction of this error by the owner or operator after staking has begun, resulting in the unfair reward distribution scenario described.

## Impact

The primary impact of this issue is the potential for an unfair distribution of rewards among stakers. Users who stake before the correction of the `_maxLockMultiplier` can secure a higher multiplier (200% in this scenario), leading to a disproportionately higher reward compared to users who stake after the correction (where the multiplier would be set at 150% or lower). 

## Code Snippet

https://github.com/sherlock-audit/2024-06-magicsea/blob/42e799446595c542eff9519353d3becc50cdba63/magicsea-staking/src/MlumStaking.sol#L69-L73

https://github.com/sherlock-audit/2024-06-magicsea/blob/42e799446595c542eff9519353d3becc50cdba63/magicsea-staking/src/MlumStaking.sol#L262-L278

## Tool used

Manual Review

## Recommendation

It is recommended to modify `_maxLockDuration` to 15000, ensuring it does not exceed the expected `MAX_LOCK_MULTIPLIER_LIMIT`, and align with the behavior described in the comments. Furthermore, if staking has already begun and positions exist, modifications to `maxLockMultiplier` and `maxLockDuration` should be prohibited.

On the other hand, according to CODE COMMENTS, `maxLockMultiplier must be<=_maxGlobalMultiplier - _maxBoostMultiplier`, however, the actual code does not have corresponding logic, and it is recommended to correct it.