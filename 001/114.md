Deep Rose Mandrill

Medium

# Voter receives the bribe reward for `all` voting period, regardless of how many votingPeriod he voted for

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