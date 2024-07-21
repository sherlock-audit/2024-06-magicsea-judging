Hollow Paisley Panda

High

# Funds get locked in rewarder due to precision loss

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
