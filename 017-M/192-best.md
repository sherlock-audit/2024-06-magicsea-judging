Clever Paisley Ladybug

Medium

# It is possible to earn rewards in `MlumStaking` staking contract without locking tokens for a period of time

## Summary
Due to not checking the `lockDuration` to a minimum value, people can create positions without locking their token for a period of time and earn rewards.

## Vulnerability Detail
To create a new staking position we can call the following function in `MlumStaking` contract:

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

However, as you can see this function dosn't check if the `lockDuration` chosed by the user respect a minimum value or at least grather than 0. Therefore, users can create positions with a lockDuration equal to 0. By doing this, users can keep earning rewards without locking their tokens, and they become able to withdraw at any moment they want.

Here is a proof of concept to reproduce this vulnerability:
```solidity
function testCreatePositionZeroTimeLock() public {
        _stakingToken.mint(BOB, 4 ether);
        _stakingToken.mint(ALICE, 2 ether);

        // step 1: Alice create a position by staking 1 ether with a zero timelock
        vm.startPrank(ALICE);
        _stakingToken.approve(address(_pool), 1 ether);
        _pool.createPosition(1 ether, 0);
        vm.stopPrank();

        assertEq(ERC721(address(_pool)).ownerOf(1), ALICE);

        // step 2: Bob create a position by staking 1 ether for a timelock of 2 days (this step was added to show that rewards are correctly distributed)
        vm.startPrank(BOB);
        _stakingToken.approve(address(_pool), 2 ether);
        _pool.createPosition(1 ether, 2 days);
        vm.stopPrank();

        // send something
        _rewardToken.mint(address(_pool), 100_000_000);

         vm.prank(ALICE);
        _pool.harvestPosition(1);
        assertGt(_rewardToken.balanceOf(ALICE), 0); // here you can see that rewards were actually distributed to Alice even with a timelock of 0

        vm.startPrank(ALICE);
        _pool.withdrawFromPosition(1, 1 ether);
        assertEq(_stakingToken.balanceOf(ALICE),2 ether); // here you can see that alice was able to withdraw all its position without waiting for a timelock
        vm.stopPrank();

        vm.prank(BOB);
        uint256 bobAmount = _pool.pendingRewards(2);
        console.log(bobAmount);
        assertGt(bobAmount, 0); // bob also received its rewards
    }
```
add the previous PoC to MlumStaking.t.sol file to correctly reproduce this vulnerability.

## Impact
This lead to unfair advantage where some users exploit the system to continuously earn rewards without any risk or commitment.

## Code Snippet
https://github.com/sherlock-audit/2024-06-magicsea/blob/main/magicsea-staking/src/MlumStaking.sol#L354-L390

## Tool used
Manual Review

## Recommendation
Check for a minimum value for the `lockDuration` when a new position is created.