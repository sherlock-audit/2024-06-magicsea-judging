Interesting Chili Albatross

High

# Voting and bribe rewards can be hijacked during emergency unlock by already existing positions

## Summary

There are many ways of modifying an already existing staking position : but they exhibit different behavior when an emergency unlock is made active.
`renewLock()` and `extendLock()` are straight away disallowed.
`withdrawFromPosition()` and `emergencyWithdraw()` are allowed but it can only pull out assets and not do anything else.

`addToPosition()` is a different case, it has no restrictions even when the emergency unlock is active. This can be used by an attacker to modify pre-existing position to vote with a new amount in Voter.

This is a big problem because all other ways of starting, or renewing a staking position are blocked.

## Vulnerability Detail

If a position already exists before the emergency unlock started, the owner of such positions can add amounts to them, and use this new amount for voting in the current voting period.

All other existing positions can also vote, but no new amount of assets can enter the voting circle because :

- calling createPosition will set lockDuration to 0 for this new position (which will revert when voting due to being < minimumLockTime) : see [here](https://github.com/sherlock-audit/2024-06-magicsea/blob/42e799446595c542eff9519353d3becc50cdba63/magicsea-staking/src/MlumStaking.sol#L356) and [here](https://github.com/sherlock-audit/2024-06-magicsea/blob/42e799446595c542eff9519353d3becc50cdba63/magicsea-staking/src/Voter.sol#L172)
- calling renewLock and extendLock is straightaway [reverted](https://github.com/sherlock-audit/2024-06-magicsea/blob/42e799446595c542eff9519353d3becc50cdba63/magicsea-staking/src/MlumStaking.sol#L692).

Now because no new assets can enter the voting amount circulation during this period, this creates a monopoly for already existing positions to control the voting outcome with lesser participants on board.

This can be particularly used by an attacker to hijack the voting process in the following steps :

- If the attacker already knows the peculiarty of `addToPosition()`, they can plan in advance and open many many 1 wei staking positions with long lock duration (such that they are eligible for voting based on minimumLockTime etc. in Voter.sol) and wait for the emergency unlock.
- If the emergency unlock happens they suddenly start adding large amounts to their 1 wei positions.
- This increases their amountWithMultiplier(sure multiplier will get limited to 1x but we are talking about the amount itself) and thus they have more voting power.
- emergencyUnlock is meant to be used in an emergency which can happen anytime. If a voting period is running during that time, there is no way to stop the current voting period.
- The attacker can use this new voting power for all their positions to hijack the voting numbers because other users cant open new positions or renew/extend, and honest users are not aware of this loophole in the system via addToPosition()

In this way, an attacker can gain monopoly over the votes during an emergencyUnlock, which ultimately influences the LUM emissions directed to certain pools.

## Impact

Voting process can be manipulated during an emergencyUnlock using pre-existing positions. This also leads to a hijack of the bribe rewards if any bribe rewarders are set for those pools.

High severity because this can be used to unfairly influence the voting outcome and misdirect LUM rewards, which is the most critical property of the system. Also this is unfair to all other voters as they lose out on bribe rewards, and the attacker can get a very high share of it.

## Code Snippet

https://github.com/sherlock-audit/2024-06-magicsea/blob/42e799446595c542eff9519353d3becc50cdba63/magicsea-staking/src/MlumStaking.sol#L397

## Tool used

Manual Review

## Recommendation

This can be solved by simply disallowing any activity apart from emregencyWithdraw during the time when emergencyUnlock is active. Apply the check used in [\_lockPosition()](https://github.com/sherlock-audit/2024-06-magicsea/blob/42e799446595c542eff9519353d3becc50cdba63/magicsea-staking/src/MlumStaking.sol#L692) to addToPosition() too.
