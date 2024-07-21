Future Green Frog

Medium

# Missing Duplicate Registration Check in onRegister Function

## Summary

The `onRegister()` function in the `Voter.sol` contract is missing a check to prevent duplicate registrations of rewarders for the same token and pool combination.

## Vulnerability Detail

The `onRegister()` function allows `IBribeRewarder` contracts to register themselves. While the function correctly verifies if the rewarder is from the allowed `rewarderFactory`, it does not check if the rewarder token and pool combination are already registered. This omission can lead to multiple registrations of the same rewarder, potentially disrupting the bribe reward system.

```solidity
        (address pool, uint256[] memory periods) = rewarder.getBribePeriods();
        for (uint256 i = 0; i < periods.length; ++i) {
            // TODO check if rewarder token + pool  is already registered

            require(periods[i] >= currentPeriodId, "wrong period");
            require(_bribesPerPriod[periods[i]][pool].length + 1 <= Constants.MAX_BRIBES_PER_POOL, "too much bribes");
            _bribesPerPriod[periods[i]][pool].push(rewarder);
        }
    }
```

## Impact

Duplicate rewarders for the same token and pool combination can be registered, leading to incorrect bribe calculations and unfair distribution of rewards. This can result in a disproportionate allocation of rewards and degrade the fairness and accuracy of the voting system.

## Code Snippet

https://github.com/sherlock-audit/2024-06-magicsea/blob/7fd1a65b76d50f1bf2555c699ef06cde2b646674/magicsea-staking/src/Voter.sol#L130-L144

## Tool used

Manual Review

## Recommendation

Implement duplicate registration check before pushing `rewarder`.
