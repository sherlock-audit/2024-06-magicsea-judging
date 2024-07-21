Icy Basil Seal

High

# Wrong call order for `setTopPoolIdsWithWeights`, resulting in wrong distribution of rewards

## Summary

Per the Sherlock rules:

> If the protocol team provides specific information in the README or CODE COMMENTS, that information stands above all judging rules.

The Masterchef contract allows people to stake an admin-selected token in farms to earn LUM rewards. Each two weeks, MLUM stakers can vote on their favorite pools, and the top pools will earn LUM emissions according to the votes. Admin has to call `setTopPoolIdsWithWeights` to set those votes and weights to set the reward emission for the next two weeks.

Per the documented call order for `setTopPoolIdsWithWeights`:

```solidity
/**
* @dev Set farm pools with their weight;
*
* WARNING:
* Caller is responsible to updateAll oldPids on masterChef before using this function
* and also call updateAll for the new pids after.
*
* @param pids - list of pids
* @param weights - list of weights
*/
```

We show that this call order is wrong, and will result in wrong rewards distribution.

## Vulnerability Detail

There is a global parameter `lumPerSecond`, set by the admin. Whenever `updateAll` is called for a set of pools:
- Let the total weight of all votes across all top pools be `totalWeight`
- Let the **current** weight of a pool Pid be `weightPid`. This weight can be set by the admin using `setTopPoolIdsWithWeights`
- Pool Pid will earn `totalLumRewardForPid = (lumPerSecond * weightPid / totalWeight) * (elapsed_time)`, i.e. each second it earns `lumPerSecond` times its percentage of voted weight `weightPid` across the total weight all top pools `totalWeight`.

https://github.com/sherlock-audit/2024-06-magicsea/blob/main/magicsea-staking/src/MasterchefV2.sol#L522-L525

Now, the function `updateAll` does the following:
- For each pool, fetch its weight and calculate its `totalLumRewardForPid` since last update
- Mint that calculated amount of LUM
- `updateAccDebtPerShare` i.e. distribute rewards since the last updated time

https://github.com/sherlock-audit/2024-06-magicsea/blob/main/magicsea-staking/src/MasterchefV2.sol#L526-L528

Per the code comments, the admin is responsible for calling `updateAll()` on the old pools before calling `setTopPoolIdsWithWeights()` for the new pools, and *then* calling `updateAll()` on the new pools.

We claim that, using this call order, a pool will be wrongly updated if it's within the set `newPid` but not in `oldPid`, and the functions are called with this order. Take this example. 

## PoC

Let LUM per second = 1. We assume all farms were created and registered at time 0:
- At time 1000: there are two pools, A and B making it into the top pools. Weight = 1 both. 
  - `updateAll` for oldPid. There are no old Pids
  - `setTopPoolIdsWithWeights`: Pool A and pool B now have weight = 1.
  - `updateAll` for newPid (A and B). 1000 seconds passed, each pool accrued 500 LUM for having 50% weight despite just making it into the top weighted pools
- At time 2000: a new pool C makes it into the pool, while pool B is no longer in the top. Weight = 1 both. 
  - `updateAll` for oldPid (A and B).
    - For pool A, 1000 seconds passed. It earns 500 LUM
    - For pool B, 1000 seconds passed. It earns 500 LUM
  - `setTopPoolIdsWithWeights`: Pool A and pool C now have weight = 1.
  - `updateAll` for newPid (A and C). 
    - For pool A, 0 seconds passed. It earns 0 LUM
    - For pool C, 2000 seconds passed. It earns 1000 LUM

The end result is that, at time 2000:
- Pool A accrued 1000 LUM
- Pool B accrued 1000 LUM
- Pool C accrued 1000 LUM

Where the correct result should be:
- Pool A accrued 500 LUM, it was in the top pools in time 1000 to 2000
- Pool B accrued 500 LUM, it was in the top pools in time 1000 to 2000
- Pool C accrued 0 LUM, it only started being in the top pools from timestamp 2000

In total, 3000 LUM has been distributed from timestamps 1000 to 2000, despite the emission rate should be 1 LUM per second. In fact, LUM has been wrongly distributed since timestamp 1000, as both pool A and B never made it into the top pools but still immediately accrued 500 LUM each.

This is because if a pool is included in an `updateAll` call **after** its weight has been set, its last updated timestamp is still in the past. Therefore when `updateAll` is called, the new weights are applied across the entire interval since it was last updated (i.e. a far point in the past). 

## Coded PoC

We provide a coded PoC to prove the impact of timestamp 1000. We add two farms A and B. LUM per second is set to 1000 wei per second. We also have a single staker Alice depositing into farm A.

We have two tests to compare the results:
- Both tests set the pool weights at time 1000, then output Alice's pending reward right after that.
- There are no `oldPids`, so there's no need to call `updateAll()` on anything before setting weights.
- In one test, we call `updateAll(newPids)` *after* calling `setTopPoolIdsWithWeights()`.
- In the other test, we call `updateAll(newPids)` *before* calling `setTopPoolIdsWithWeights()`.

We output the pending rewards of Alice for comparison.

First, change the function `mint()` of contract `MockERC20` to be the following:

```solidity
function mint(address _to, uint256 _amount) external returns (uint256) {
    _mint(_to, _amount);
    return _amount;
}
```

Then, create a new test file `MasterChefTest.t.sol`:

<details>
  <summary>forge test --match-test testSetPoolWeights -vv</summary>

```solidity
// SPDX-License-Identifier: MIT

pragma solidity ^0.8.20;

import "forge-std/Test.sol";

import "openzeppelin-contracts-upgradeable/access/OwnableUpgradeable.sol";
import {SafeERC20, IERC20} from "openzeppelin/token/ERC20/utils/SafeERC20.sol";

import "../src/transparent/TransparentUpgradeableProxy2Step.sol";

import "openzeppelin/token/ERC721/ERC721.sol";
import "openzeppelin/token/ERC20/ERC20.sol";
import {ERC20Mock} from "./mocks/ERC20.sol";
import {MasterChef} from "../src/MasterChefV2.sol";
import {MlumStaking} from "../src/MlumStaking.sol";
import "../src/Voter.sol";
import "../src/rewarders/BaseRewarder.sol";
import "../src/rewarders/MasterChefRewarder.sol";
import "../src/rewarders/RewarderFactory.sol";
import {IVoter} from "../src/interfaces/IVoter.sol";
import {ILum} from "../src/interfaces/ILum.sol";
import {IRewarderFactory} from "../src/interfaces/IRewarderFactory.sol";

contract MasterChefV2Test is Test {
    address payable immutable DEV = payable(makeAddr("dev"));
    address payable immutable ALICE = payable(makeAddr("alice"));
    address payable immutable BOB = payable(makeAddr("bob"));

    Voter private _voter;
    MlumStaking private _pool;
    MasterChef private _masterChefV2;

    ERC20Mock private farmA;
    ERC20Mock private farmB;
    ERC20Mock private farmC;

    MasterChefRewarder rewarderPoolA;
    MasterChefRewarder rewarderPoolB;
    MasterChefRewarder rewarderPoolC;

    RewarderFactory factory;

    ERC20Mock private _stakingToken;
    ERC20Mock private _rewardToken;
    ERC20Mock private _lumToken;

    uint256[] pIds;
    uint256[] weights;

    function setUp() public {
        vm.prank(DEV);
        _stakingToken = new ERC20Mock("MagicLum", "MLUM", 18);

        vm.prank(DEV);
        _rewardToken = new ERC20Mock("USDT", "USDT", 6);

        vm.prank(DEV);
        address poolImpl = address(new MlumStaking(_stakingToken, _rewardToken));

        _pool = MlumStaking(
            address(
                new TransparentUpgradeableProxy2Step(
                    poolImpl, ProxyAdmin2Step(address(1)), abi.encodeWithSelector(MlumStaking.initialize.selector, DEV)
                )
            )
        );

        address factoryImpl = address(new RewarderFactory());
        factory = RewarderFactory(
            address(
                new TransparentUpgradeableProxy2Step(
                    factoryImpl,
                    ProxyAdmin2Step(address(1)),
                    abi.encodeWithSelector(
                        RewarderFactory.initialize.selector, address(this), new uint8[](0), new address[](0)
                    )
                )
            )
        );

        vm.prank(DEV);
        _lumToken = new ERC20Mock("Lum", "LUM", 18);
        farmA = new ERC20Mock("Farm A", "FARM A", 18);
        farmB = new ERC20Mock("Farm B", "FARM B", 18);
        farmC = new ERC20Mock("Farm C", "FARM C", 18);

        vm.prank(DEV);
        address masterChefImp = address(new MasterChef(ILum(address(_lumToken)), _voter, factory,DEV,1));

        _masterChefV2 = MasterChef(
            address(
                new TransparentUpgradeableProxy2Step(
                    masterChefImp, ProxyAdmin2Step(address(1)), abi.encodeWithSelector(MasterChef.initialize.selector, DEV,DEV)
                )
            )
        );
        vm.prank(DEV);
        _masterChefV2.setLumPerSecond(1000);
        vm.prank(DEV);
        _masterChefV2.setMintLum(true);

        //add 3 farms
        vm.prank(DEV);
        _masterChefV2.add(farmA,IMasterChefRewarder(address(0)));
        vm.prank(DEV);
        _masterChefV2.add(farmB,IMasterChefRewarder(address(0)));
        vm.prank(DEV);
        _masterChefV2.add(farmC,IMasterChefRewarder(address(0)));

        vm.prank(DEV);
        address voterImpl = address(new Voter(_masterChefV2, _pool, factory));

        _voter = Voter(
            address(
                new TransparentUpgradeableProxy2Step(
                    voterImpl, ProxyAdmin2Step(address(1)), abi.encodeWithSelector(Voter.initialize.selector, DEV)
                )
            )
        );

        vm.prank(DEV);
        _voter.updateMinimumLockTime(2 weeks);

        vm.prank(DEV);
        factory.setRewarderImplementation(
            IRewarderFactory.RewarderType.MasterChefRewarder, IRewarder(address(new MasterChefRewarder(address(_masterChefV2))))
        );

        vm.prank(DEV);
        _masterChefV2.setVoter(_voter);
    }

    //SetPoolWeightsTest Correct
    function testSetPoolWeightsCorrect() public {
        pIds.push(0);
        pIds.push(1);
        weights.push(500);
        weights.push(500);

        farmA.mint(ALICE, 2 ether);

        vm.prank(ALICE);
        farmA.approve(address(_masterChefV2), 1 ether);
        vm.prank(ALICE);
        _masterChefV2.deposit(0, 1 ether);

        skip(1000);
        vm.prank(DEV);
        _masterChefV2.updateAll(pIds);

        vm.prank(DEV);
        _voter.setTopPoolIdsWithWeights(pIds,weights);

        (uint256[] memory lumRewards,IERC20[] memory tokens,uint256[] memory extraRewards) = _masterChefV2.getPendingRewards(ALICE, pIds);
        console.log("Alice rewards correct:");
        console.log(lumRewards[0]);
    }

    //SetPoolWeightsTest Wrong
    function testSetPoolWeightsWrong() public {
        pIds.push(0);
        pIds.push(1);
        weights.push(500);
        weights.push(500);

        farmA.mint(ALICE, 2 ether);

        vm.prank(ALICE);
        farmA.approve(address(_masterChefV2), 1 ether);
        vm.prank(ALICE);
        _masterChefV2.deposit(0, 1 ether);

        skip(1000);
        vm.prank(DEV);
        _voter.setTopPoolIdsWithWeights(pIds,weights);

        vm.prank(DEV);
        _masterChefV2.updateAll(pIds);

        (uint256[] memory lumRewards,IERC20[] memory tokens,uint256[] memory extraRewards) = _masterChefV2.getPendingRewards(ALICE, pIds);
        console.log("Alice rewards wrong:");
        console.log(lumRewards[0]);
    }
}
```

</details>

And the results are:

```txt
Ran 2 tests for test/MasterChefTest.t.sol:MasterChefV2Test
[PASS] testSetPoolWeightsCorrect() (gas: 506430)
Logs:
  Alice rewards correct:
  0

[PASS] testSetPoolWeightsWrong() (gas: 589539)
Logs:
  Alice rewards wrong:
  499999

Suite result: ok. 2 passed; 0 failed; 0 skipped; finished in 2.54ms (559.21µs CPU time)
```

As shown, in the "correct" test, Alice has not accrued any rewards right after the new weights are set. However, in the "wrong" test, Alice accrues 499999 reward units right after setting. 

## Impact

Pools that have just made it into the top pools will have already accrued rewards for time intervals it wasn't in the top pools. Rewards are thus severely inflated.

## Code Snippet

https://github.com/sherlock-audit/2024-06-magicsea/blob/main/magicsea-staking/src/Voter.sol#L250-L260

## Tool used

Manual Review

## Recommendation

All of the pools (oldPids and newPids) should be updated, only then weights should be applied.

In other words, the correct call order should be:
- `updateAll` should be called for **all** pools within oldPid or newPid.
- `setTopPoolIdsWithWeights` should then be called.

Additionally, we think it might be better that `setTopPoolIdsWithWeights` itself should just call `updateAll` for all (old and new) pools before updating the pool weights, or at least validate that their last updated timestamp is sufficiently fresh.