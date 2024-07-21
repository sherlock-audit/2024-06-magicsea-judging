Abundant Pickle Rattlesnake

Medium

# DoS of `Voter.sol#createFarms()` due to incorrect validation of `MasterchefV2.sol#add()` function

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