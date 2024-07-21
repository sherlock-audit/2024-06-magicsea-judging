Bitter Seaweed Eagle

High

# The first staker inside `MlumStaking` will receive all of the rewards and all of the consecutive stakers will not receive any

## Summary
Because of the way `rewardDebt` is calculated and the way the pool is updated the first person to stake tokens after the pool has been supplied with reward tokens will receive more rewards than they were supposed to and steal all of the other users' rewards

## Vulnerability Detail

Let's say Alice is the first person to call `MlumStaking::createPosition` after the contract has been supplied with 100 USD which will represent the reward tokens. She creates a position by depositing 1eth worth of staking token, which in our case will be MLUM for 1 day lock duration. The first thing that will happen inside the function will be to call `_updatePool`

```solidity
    /**
     * @dev Updates rewards states of this pool to be up-to-date
     */
    function _updatePool() internal {
        uint256 accRewardsPerShare = _accRewardsPerShare;
        uint256 rewardBalance = rewardToken.balanceOf(address(this));
        uint256 lastRewardBalance = _lastRewardBalance;

        // recompute accRewardsPerShare if not up to date
@>      if (lastRewardBalance == rewardBalance || _stakedSupply == 0) {
            return;
        }

        uint256 accruedReward = rewardBalance - lastRewardBalance;
        _accRewardsPerShare =
            accRewardsPerShare + ((accruedReward * (PRECISION_FACTOR)) / (_stakedSupplyWithMultiplier));

        _lastRewardBalance = rewardBalance;

        emit PoolUpdated(_currentBlockTimestamp(), accRewardsPerShare);
    }
```

Because of the fact that the current `_stakedSupply` is 0 the function will return without nothing happening. Continuing the function execution

```solidity
    /**
     * @dev Create a staking position (lsNFT) with an optional lockDuration
     */
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
@>          rewardDebt: amountWithMultiplier * (_accRewardsPerShare) / (PRECISION_FACTOR),
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

Everything else will execute as expected apart from `rewardDebt` which will be set to 0 because `_accRewardsPerShare` is still 0.

The next user to deposit will be Bob who will deposit 2eth worth of MLUM. This time when `_updatePool` is called the execution will reach the end of the function because `rewardBalance /= 0 ` and `_lastRewardBalance=0` and `_stakedSupply` is no longer 0.

```solidity
    /**
     * @dev Updates rewards states of this pool to be up-to-date
     */
    function _updatePool() internal {
        uint256 accRewardsPerShare = _accRewardsPerShare;
        uint256 rewardBalance = rewardToken.balanceOf(address(this));
        uint256 lastRewardBalance = _lastRewardBalance;

        // recompute accRewardsPerShare if not up to date
        if (lastRewardBalance == rewardBalance || _stakedSupply == 0) {
            return;
        }

        uint256 accruedReward = rewardBalance - lastRewardBalance;
        _accRewardsPerShare =
            accRewardsPerShare + ((accruedReward * (PRECISION_FACTOR)) / (_stakedSupplyWithMultiplier));

        _lastRewardBalance = rewardBalance;

        emit PoolUpdated(_currentBlockTimestamp(), accRewardsPerShare);
    }
```
`accuredRewards` wil be calculated as ` 100*10^6- 0  = 100*10^6`.
Then `_accRewardPerShare = 0 + ((100*10^6 * (10^24)) / (1005400000000000000) = 99462900338173`

And then `rewardDebt = 2010800000000000000 * 99462900338173 / 10^24 = 199985660417` using the new _accRewardPerShare.

Now if we call `harvestPosition` for Bob, his pending rewards will be

` uint256 pending = position.amountWithMultiplier * _accRewardsPerShare / PRECISION_FACTOR - position.rewardDebt;`
`pending = 2010800000000000000  * 99462900338173 / 10^24 - 199985660417`. But as you can see from the formuals `position.rewardDebt = pending` which means the result will be 0. That means Bob will receive 0 reward tokens.

Taking a look at Alice's situation, when she calls `harvestPosition`:
`pending = 1005400000000000000 * 99462900338173  / 10^24 - 0 =  99999999` meaning the user will receive 99$ out of the 100$ deposited.
## Impact

POC
```solidity

    function testInitialHarvest() public{
        _stakingToken.mint(ALICE, 1 ether);
        _stakingToken.mint(BOB, 2 ether);

        _rewardToken.mint(address(_pool), 100_000_000);  

        vm.startPrank(ALICE);
        _stakingToken.approve(address(_pool), 1 ether);
        _pool.createPosition(1 ether,1 days);
        vm.stopPrank();


        vm.startPrank(BOB);
        _stakingToken.approve(address(_pool), 2 ether);
        _pool.createPosition(2 ether,1 days);
        vm.stopPrank();
        

        vm.prank(ALICE);
        _pool.harvestPosition(1);
        console.logUint(_rewardToken.balanceOf(ALICE));

        vm.prank(BOB);
        _pool.harvestPosition(2);
        console.logUint(_rewardToken.balanceOf(BOB));

        vm.assertGt(_rewardToken.balanceOf(ALICE),_rewardToken.balanceOf(BOB));
        vm.assertEq(_rewardToken.balanceOf(ALICE),99_999_999);
        vm.assertEq(_rewardToken.balanceOf(BOB),0);
    }
```
## Code Snippet
https://github.com/sherlock-audit/2024-06-magicsea/blob/main/magicsea-staking/src/MlumStaking.sol#L354-L390
https://github.com/sherlock-audit/2024-06-magicsea/blob/main/magicsea-staking/src/MlumStaking.sol#L442-L448
https://github.com/sherlock-audit/2024-06-magicsea/blob/main/magicsea-staking/src/MlumStaking.sol#L470-L489
## Tool used

Manual Review
VS Code
Foundry

## Recommendation

Find another way of calculating the `rewardDebt` and how many reward tokens the user must receive