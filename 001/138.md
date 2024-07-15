Slow Sapphire Barracuda

Medium

# `MlumStaking::addToPosition` should assing the amount multiplier based on the new lock duration instead of initial lock duration.

## Summary
There are two separate issues that make necessary to assign the multiplier based on the new lock duration:
1. First, when users add tokens to their position via `MlumStaking::addToPosition`, the new remaining time for the lock duration is recalculated as the amount-weighted lock duration. However, when the remaining time for an existing deposit is 0, this term is omitted, allowing users to retain the same amount multiplier with a reduced lock time. Consider the following sequence of actions:
    - Alice creates a position by calling `MlumStaking::createPosition`depositing 1 ether and a lock time of 365 days
    - After the 365 days elapse, Alice adds another 1 ether to her position. The snippet below illustrates how the new lock time for the position is calculated:
        ```javascript
                uint256 avgDuration = (remainingLockTime *
                    position.amount +
                    amountToAdd *
                    position.initialLockDuration) / (position.amount + amountToAdd);
                position.startLockTime = _currentBlockTimestamp();
                position.lockDuration = avgDuration;

                // lock multiplier stays the same
                position.lockMultiplier = getMultiplierByLockDuration(
                    position.initialLockDuration
                );
        ```

    - The result will be: `(0*1 ether + 1 ether*365 days)/ 2 ether`, therefore Alice will need to wait just half a year, while the multiplier remains unchanged.  

2. Second, the missalignment between this function and `MlumStaking::renewLockPosition` creates an arbitrage opportunity for users, allowing them to reassign the lock multiplier to the initial duration if it is more beneficial. Consider the following scenario:

    - Alice creates a position with an initial lock duration of 365 days. The multiplier will be 3.
    - Then after 9 months, the lock duration is updated, let's say she adds to the position just 1 wei. The new lock duration is ≈ 90 days.
    - After another 30 days, she wants to renew her position for another 90 days. Then she calls `MlumStaking::renewLockPosition`. The new amount multiplier will be calculated as ≈ `1+90/365*2 < 3`.
    - Since it is not in her interest to have a lower multiplier than originally, then she adds just 1 wei to her position. The new multiplier will be 3 again.

## Vulnerability Detail

## Impact
You may find below the coded PoC corresponding to each of the aforementioned scenarios:

<details>

<summary> See PoC for scenario 1 </summary>
Place in `MlumStaking.t.sol`.

```javascript
    function testLockDurationReduced() public {
        _stakingToken.mint(ALICE, 2 ether);

        vm.startPrank(ALICE);
        _stakingToken.approve(address(_pool), 1 ether);
        _pool.createPosition(1 ether, 365 days);
        vm.stopPrank();

        // check lockduration
        MlumStaking.StakingPosition memory position = _pool.getStakingPosition(
            1
        );
        assertEq(position.lockDuration, 365 days);

        skip(365 days);

        // add to position should take calc. avg. lock duration
        vm.startPrank(ALICE);
        _stakingToken.approve(address(_pool), 1 ether);
        _pool.addToPosition(1, 1 ether);
        vm.stopPrank();

        position = _pool.getStakingPosition(1);

        assertEq(position.lockDuration, 365 days / 2);
        assertEq(position.amountWithMultiplier, 2 ether * 3);
    }
```

</details>

<details>

<summary> See PoC for scenario 2 </summary>
Place in `MlumStaking.t.sol`.

```javascript
    function testExtendLockAndAdd() public {
        _stakingToken.mint(ALICE, 2 ether);

        vm.startPrank(ALICE);
        _stakingToken.approve(address(_pool), 2 ether);
        _pool.createPosition(1 ether, 365 days);
        vm.stopPrank();

        // check lockduration
        MlumStaking.StakingPosition memory position = _pool.getStakingPosition(
            1
        );
        assertEq(position.lockDuration, 365 days);

        skip(365 days - 90 days);
        vm.startPrank(ALICE);
        _pool.addToPosition(1, 1 wei); // lock duration ≈ 90 days
        skip(30 days);
        _pool.renewLockPosition(1); // multiplier ≈ 1+90/365*2
        position = _pool.getStakingPosition(1);
        assertEq(position.lockDuration, 7776000);
        assertEq(position.amountWithMultiplier, 1493100000000000001);

        _pool.addToPosition(1, 1 wei); // multiplier = 3
        position = _pool.getStakingPosition(1);
        assertEq(position.lockDuration, 7776000);
        assertEq(position.amountWithMultiplier, 3000000000000000006);
    }
```

</details>

## Code Snippet
https://github.com/sherlock-audit/2024-06-magicsea/blob/main/magicsea-staking/src/MlumStaking.sol#L409-L417

https://github.com/sherlock-audit/2024-06-magicsea/blob/main/magicsea-staking/src/MlumStaking.sol#L509-L514

https://github.com/sherlock-audit/2024-06-magicsea/blob/main/magicsea-staking/src/MlumStaking.sol#L714
## Tool used

Manual Review

## Recommendation
Assign new multiplier in `MlumStaking::addToPosition` based on lock duration rather than initial lock duration.