Swift Sage Gerbil

High

# A voter lose bribe rewards if another voter voted before claim.

## Summary
A voter lose bribe rewards if another voter voted before claim.

## Vulnerability Detail
This problem is related to design architecture.
In `BribeRewarder.sol`, the `_lastUpdateTimestamp` is used to calculate the unclaimed rewards for `periodId`, but it is not dependent on `periodId`.
Therefore, once `_lastUpdateTimestamp` has been updated to the next period, there is no way to calculate the unclaimed rewards for the previous period.

The following is the modified test code for PoC.
```solidity
    function testDepositMultiple() public {
        ERC20Mock(address(rewardToken)).mint(address(this), 20e18);
        ERC20Mock(address(rewardToken)).approve(address(rewarder), 20e18);

        rewarder.fundAndBribe(1, 2, 10e18);

        _voterMock.setCurrentPeriod(1);
        _voterMock.setStartAndEndTime(0, 100);

        // time: 0
        vm.warp(0);
        vm.prank(address(_voterMock));
        rewarder.deposit(1, 1, 0.2e18);

        assertEq(0, rewarder.getPendingReward(1));

        // time: 50, seconds join
        vm.warp(50);
        vm.prank(address(_voterMock));
        rewarder.deposit(1, 2, 0.2e18);

        // time: 100
        vm.warp(100);
        _voterMock.setCurrentPeriod(2);
        _voterMock.setStartAndEndTime(0, 100);
        _voterMock.setLatestFinishedPeriod(1);

        // @audit-info next period started
        vm.warp(150);

        // 1 -> [0,50] -> 1: 0.5
        // 2 -> [50,100] -> 1: 0.25 + 0.5, 2: 0.25

        assertEq(7500000000000000000, rewarder.getPendingReward(1));
        assertEq(2500000000000000000, rewarder.getPendingReward(2));

        // @audit-info Another voter votes before claim.
        vm.prank(address(_voterMock));
        rewarder.deposit(2, 3, 0.1e18);

        // @audit-info The expected rewards decreased much
        assertEq(5000000000000000000, rewarder.getPendingReward(1));
        assertEq(0, rewarder.getPendingReward(2));

        vm.prank(alice);
        rewarder.claim(1);

        vm.prank(bob);
        rewarder.claim(2);

        // @audit-info The claimed rewards decreased too.
        assertEq(5000000000000000000, rewardToken.balanceOf(alice));
        assertEq(0, rewardToken.balanceOf(bob));

        assertEq(7500000000000000000, rewardToken.balanceOf(alice), "balance of alice should be 75e17 but 50e17");
        assertEq(2500000000000000000, rewardToken.balanceOf(bob), "balance of bob should be 25e17 but 0");
    }
```
The claimed rewards amount of alice and bob for 1st period are originally `75e17` and `25e17`, respectively.
But if a voter votes for 2nd period before alice and bob claim their rewards for 1st period, the claimed rewards amount of alice and bob will be decreased to `50e17` and `zero`, respectively.
It means that the rewards for [50,100] of 1st period are will not be claimed.

And the following is the test command and result.
```bash
$ forge test --match-test testDepositMultiple

Failing tests:
Encountered 1 failing test in test/BribeRewarder.t.sol:BribeRewarderTest
[FAIL. Reason: balance of alice should be 75e17 but 50e17: 7500000000000000000 != 5000000000000000000] testDepositMultiple() (gas: 767761)
```
As shown above, if anyone votes before alice and bob claim their rewards, the rewards of alice and bob will be decreased.

## Impact
Voters lose bribe rewards if another voter voted before claim.
And such cases can occur frequently enough.

## Code Snippet
- [magicsea-staking/src/rewarders/BribeRewarder.sol#L65](https://github.com/sherlock-audit/2024-06-magicsea/tree/main/magicsea-staking/src/rewarders/BribeRewarder.sol#L65)
- [magicsea-staking/src/rewarders/BribeRewarder.sol#L260-L298](https://github.com/sherlock-audit/2024-06-magicsea/tree/main/magicsea-staking/src/rewarders/BribeRewarder.sol#L260-L298)

## Tool used
Manual Review

## Recommendation
Change the `_lastUpdateTimestamp` state variable to be dependent on `periodId`.
For instance, change it to mapping variable such as `mapping(uint256 periodId => uint256) _lastUpdateTimestamp;`.
