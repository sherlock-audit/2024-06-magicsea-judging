Furry Viridian Copperhead

High

# Adding genuine BribeRewarder contract instances to a pool in order to incentivize users can be DOSed

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