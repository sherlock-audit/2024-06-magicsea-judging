Smooth Taffy Moth

High

# Any unclaimed rewards will be permanently frozen once the `MasterChefRewarder` contract has been unlinked

## Summary

When setting a new extra rewarder in the `MasterChefV2` contract, any unclaimed rewards remaining in the old extra rewarder will be permanently frozen.

## Vulnerability Detail

When setting a new extra rewarder in the `MasterChefV2` contract, the old extra rewarder is [unlinked](https://github.com/sherlock-audit/2024-06-magicsea/blob/main/magicsea-staking/src/MasterchefV2.sol#L498) and the `_isStopped` flag is set to [true](https://github.com/sherlock-audit/2024-06-magicsea/blob/main/magicsea-staking/src/rewarders/MasterChefRewarder.sol#L48). Even though there may be unclaimed rewards in the old rewarder, users are unable to claim them. Additionally, the contract owner cannot sweep the unclaimed rewards, as the `BaseRewarder.sweep()` function does not include the unclaimed rewards in the sweeping balance(`L213`). As a result, any unclaimed rewards in the old extra rewarder will be permanently frozen and inaccessible.

```solidity
    function sweep(IERC20 token, address account) public virtual override onlyOwner {
        uint256 balance = _balanceOfThis(token);

        if (token == _token()) {
            if (_isStopped) {
213             if (_getTotalSupply() > 0) balance -= _totalUnclaimedRewards;
            } else {
                balance -= _reserve;
            }
        }
        if (balance == 0) revert BaseRewarder__ZeroAmount();

        _safeTransferTo(token, account, balance);

        emit Swept(token, account, balance);
    }
```

## Impact

Any unclaimed rewards remaining in the old extra rewarder will be permanently frozen and inaccessible.

## Code Snippet

https://github.com/sherlock-audit/2024-06-magicsea/blob/main/magicsea-staking/src/rewarders/BaseRewarder.sol#L208-L223

## Tool used

Manual Review

## Recommendation

In the `sweep()` function, the unclaimed rewards should be swept as well.

```diff
    function sweep(IERC20 token, address account) public virtual override onlyOwner {
        uint256 balance = _balanceOfThis(token);

        if (token == _token()) {
            if (_isStopped) {
-               if (_getTotalSupply() > 0) balance -= _totalUnclaimedRewards;
            } else {
                balance -= _reserve;
            }
        }
        if (balance == 0) revert BaseRewarder__ZeroAmount();

        _safeTransferTo(token, account, balance);

        emit Swept(token, account, balance);
    }
```