Chilly Iris Parakeet

Medium

# Voters can claim their reward immediately in the end of epoch

## Summary
[Based on docs](https://docs.magicsea.finance/protocol/magic/magic-lum-voting)
> Bribes as an additional incentive to vote <ins>can be claimed 24-48 hours after an epoch has ended</ins>. Voters can claim the rewards until the next epoch is ended. Unclaimed rewards will be sent back to the briber. 

Voters for claiming their reward have to call `BribeRewarder::claim` but in the implementation of this function doesn't have any restriction for apply this limitation
## Vulnerability Detail
add this function to Voter.t.sol and before that u have to change setUp function
```diff
+ RewarderFactory _rewardFactory;
function setUp()public {
...
+ address factoryImpl = address(new RewarderFactory());

+        _rewardFactory = RewarderFactory(
+           address(
+                new TransparentUpgradeableProxy2Step(
+                    factoryImpl, ProxyAdmin2Step(address(1)), abi.encodeWithSelector(RewarderFactory.initialize.selector, DEV, 
+ _rewardType, _rewarder)
+                )
+           )
+        );
```
```solidity
    function testClaimRewardsimmediately() public {
        ERC20Mock pool = new ERC20Mock("lp","LP", 18);
        ERC20Mock usdt = new ERC20Mock("USDT","USDT", 6);
        usdt.mint(address(this), 300e6);

        IBribeRewarder _bribeRewarder1 = _rewardFactory.createBribeRewarder(usdt, address(pool));
    
        _createPosition(ALICE);


        
        usdt.approve(address(_bribeRewarder1), 300e6);
        _bribeRewarder1.fundAndBribe(1, 3, 100e6);
        //start first epoch
        vm.prank(DEV);
        _voter.startNewVotingPeriod();
        address[] memory pools = new address[](1);
        pools[0] = address(pool);

        
        vm.prank(ALICE);
        _voter.vote(1, pools, _getDeltaAmounts());

        skip(14 days);
        //end of first epoch
        //start second epoch
        vm.prank(DEV);
        _voter.startNewVotingPeriod();

        vm.prank(ALICE);
        _bribeRewarder1.claim(1);
        assertGt(usdt.balanceOf(ALICE), 0);
    }
```

## Impact
Voters can claim their reward immediately in the end of epoch
## Code Snippet
https://github.com/sherlock-audit/2024-06-magicsea/blob/main/magicsea-staking/src/rewarders/BribeRewarder.sol#L153
## Tool used

Manual Review

## Recommendation
```diff
function claim(uint256 tokenId) external override {
        uint256 endPeriod = IVoter(_caller).getLatestFinishedPeriod();//@audit voters can claim their rewards immeditly after end of epoch
+        (, uint256 endTime) = IVoter(_caller).getPeriodStartEndtime(endPeriod);
+        require(block.timestamp >= endTime + 2 days, "u can claim your reward after 2 days");

         uint256 totalAmount;
         ...
    }
```
