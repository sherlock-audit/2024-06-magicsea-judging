Soft Mint Lizard

Medium

# Reward tokens without decimals cannot be used

## Summary

Being an OPTIONAL parameter, tokens without decimals can’t be used in the `MlumStaking` contract.

## Vulnerability Detail

According to the README, any ERC20 tokens can be used in all parts of the system, including `MlumStaking`, however, it won’t work with tokens that do not expose decimals function as mentioned here https://eips.ethereum.org/EIPS/eip-20#specification.

```solidity
constructor(IERC20 _stakedToken, IERC20 _rewardToken) {
      _disableInitializers();

      require(address(_stakedToken) != address(0), "init: zero address");
      require(address(_rewardToken) != address(0), "init: zero address");

      stakedToken = _stakedToken;
      rewardToken = _rewardToken;

      uint256 decimalsRewardToken = uint256(IERC20Metadata(address(_rewardToken)).decimals());//@audit here is reverts if _rewardToken doens't have decimals
      require(decimalsRewardToken < 30, "Must be inferior to 30");

      PRECISION_FACTOR = uint256(10 ** (uint256(30) - decimalsRewardToken));

      _stakedSupply = 0;
  }
```

## Impact

Tokens without public decimals will not be able to be used, despite the intention of the team to support all the tokens.

## Code Snippet

https://github.com/sherlock-audit/2024-06-magicsea/blob/main/magicsea-staking/src/MlumStaking.sol#L91

## Tool used

Manual Review

## Recommendation

Use try/catch approach and assign default decimals in case they are not presented.