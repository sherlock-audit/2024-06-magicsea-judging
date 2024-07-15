Slow Maroon Gibbon

High

# Loss of reward if `emergencyWithdraw` is called

## Summary
If a staker calls the emergency withdrawal, they will lose their rewards, and the rewards will get stuck in the contract instead of being distributed to other stakers.

## Vulnerability Detail
When a staker calls `MlumStaking:emergencyWithdraw`, they will receive their staked amount without accrued rewards. These reward tokens should go to the other stakers. However, they will get stuck in the contract if another staker calls `harvestPosition` before an emergency withdrawal is called.

### POC

Here is how the flow works:

- DEV stakes 1 MLUM, and ALICE stakes 100 MLUM.
- 100 reward tokens are deposited.
- DEV calls `harvestPosition` to get their rewards.
- After some time, the owners enable `_emergencyUnlock` due to a short-term emergency.
- ALICE decides to withdraw her stake because of the emergency.
- `_emergencyUnlock` is then turned off again.
- At this moment, DEV is the only solo staker who has been staking from the beginning, and there are still some reward tokens in the contract. ALICE’s unclaimed rewards should go to DEV, but this will not happen in this case.
- DEV could have received these rewards if he had not called `harvestPosition` before ALICE called `emergencyWithdraw`, creating a discrepancy in the reward calculations.

Here is a coded POC with the console to show how this plays out. 
Paste this is `MlumStaking.t.sol`

```solidity
 function testRewardLoss() public {
        _stakingToken.mint(ALICE, 100 ether);
        _stakingToken.mint(DEV, 1 ether);
        // DEV create a lock of 1 MLUM for 1 day
        vm.startPrank(DEV);
        _stakingToken.approve(address(_pool), 1 ether);
        _pool.createPosition(1 ether, 1 days);
        vm.stopPrank();
        // Alice also  creates a lock  of 100 MLUM for  1 second
        vm.startPrank(ALICE);
        _stakingToken.approve(address(_pool), 100 ether);
        _pool.createPosition(100 ether, 1);
        vm.stopPrank();

        _rewardToken.mint(address(_pool), 100_000_000); // 100 rewards sent to pool

        // DEV harvest his reward
        vm.startPrank(DEV);
        _pool.harvestPosition(1);
        vm.stopPrank();

        console2.log(
            "Rewards of DEV after initial harvest: ",
            _rewardToken.balanceOf(DEV)
        );
        skip(1);

        // Now the alice decided emergency withdraw because of emergency
        vm.startPrank(ALICE);
        console2.log("Alice pending Rewards: ", _pool.pendingRewards(2));
        _pool.emergencyWithdraw(2);
        vm.stopPrank();

        console2.log(
            "Rewards for ALICE After withdraw: ",
            _rewardToken.balanceOf(ALICE)
        );

        // At this point because ALICE didn't claim her rewards the alice rewards should now be free and DEV should be able to get that after the emergency is over after 1 week
        skip(1 weeks);
        vm.startPrank(DEV);
        _pool.harvestPosition(1);
        vm.stopPrank();
        console2.log(
            "Rewards of DEV after emergency is over: ",
            _rewardToken.balanceOf(DEV)
        );
    }
```

### Output 
```bash
Ran 1 test for test/MlumStaking.t.sol:MlumStakingTest
[PASS] testRewardLoss() (gas: 816233)
Logs:
  Rewards of DEV after initial harvest:  995392
  Alice pending Rewards:  99004607
  Rewards for ALICE After withdraw:  0
  Rewards of DEV after emergency is over:  995392 // This should be more
```
## Impact
Reward tokens are effectively lost when they could be used to incentivize stakers.

This should be viewed as a loss of funds, and a high severity rating is appropriate.

## Code Snippet
https://github.com/sherlock-audit/2024-06-magicsea/blob/42e799446595c542eff9519353d3becc50cdba63/magicsea-staking/src/MlumStaking.sol#L536
## Tool used

Manual Review

## Recommendation
There should be a mechanism to distribute remaining rewards to the stakers if extra funds are left due to `emergencyWithdraw`.
