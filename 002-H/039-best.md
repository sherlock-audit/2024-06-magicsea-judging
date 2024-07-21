Future Mandarin Unicorn

High

# Non-functional vote() if there is one bribe rewarder for this pool

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