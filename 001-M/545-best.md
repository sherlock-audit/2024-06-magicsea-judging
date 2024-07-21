Damaged Tartan Worm

Medium

# Lack of support for fee on transfer, rebasing and tokens with balance modifications outside of transfers.

## Summary

The protocol wants to work with various ERC20 tokens, but in certain cases doesn't provide the needed support for tokens that charge a fee on transfer, tokens that rebase (negatively/positively) and overall, tokens with balance modifications outside of transfers.

## Vulnerability Detail

The protocol wants to work with various ERC20 tokens, but still handles various transfers without querying the amount transferred and amount received, which can lead a host of accounting issues and the likes downstream.

For instance, In MasterchefV2.sol during withdrawals or particularly emergency withdrawals, the last user to withdraw all tokens will face issues as the amount registered to his name might be signifiacntly lesser than the token balance in the contract, which as a result will cause the withdrawal functions to fail. Or the protocol risks having to send extra funds from their pocket to coverup for these extra losses due to the fees. 
This is because on deposit, the amount entered is deposited as is, without accounting for potential fees.

```solidity
    function deposit(uint256 pid, uint256 amount) external override {
        _modify(pid, msg.sender, amount.toInt256(), false);

        if (amount > 0) _farms[pid].token.safeTransferFrom(msg.sender, address(this), amount);
    }
```
Some tokens like stETH have a [1 wei corner case ](https://docs.lido.fi/guides/lido-tokens-integration-guide/#1-2-wei-corner-case) in which during transfers the amount that actually gets sent is actually a bit less than what has been specified in the transaction. 

## Impact
On a QA severity level, tokens received by users will be less than emitted to them in the event.
On medium severity level, accounting issues, potential inability of last users to withdraw, potential loss of funds from tokens with airdrops, etc.
## Code Snippet

https://github.com/sherlock-audit/2024-06-magicsea/blob/42e799446595c542eff9519353d3becc50cdba63/magicsea-staking/src/MasterchefV2.sol#L287

https://github.com/sherlock-audit/2024-06-magicsea/blob/42e799446595c542eff9519353d3becc50cdba63/magicsea-staking/src/MasterchefV2.sol#L298

https://github.com/sherlock-audit/2024-06-magicsea/blob/42e799446595c542eff9519353d3becc50cdba63/magicsea-staking/src/MasterchefV2.sol#L309

https://github.com/sherlock-audit/2024-06-magicsea/blob/42e799446595c542eff9519353d3becc50cdba63/magicsea-staking/src/MasterchefV2.sol#L334

https://github.com/sherlock-audit/2024-06-magicsea/blob/42e799446595c542eff9519353d3becc50cdba63/magicsea-staking/src/MlumStaking.sol#L559

https://github.com/sherlock-audit/2024-06-magicsea/blob/42e799446595c542eff9519353d3becc50cdba63/magicsea-staking/src/MlumStaking.sol#L649

https://github.com/sherlock-audit/2024-06-magicsea/blob/42e799446595c542eff9519353d3becc50cdba63/magicsea-staking/src/MlumStaking.sol#L744

https://github.com/sherlock-audit/2024-06-magicsea/blob/42e799446595c542eff9519353d3becc50cdba63/magicsea-staking/src/MlumStaking.sol#L747

https://github.com/sherlock-audit/2024-06-magicsea/blob/42e799446595c542eff9519353d3becc50cdba63/magicsea-staking/src/rewarders/BribeRewarder.sol#L120

## Tool used
Manual Code Review

## Recommendation

Recommend implementing a measure like that of the `_transferSupportingFeeOnTransfer` function that can correctly handle these transfers. A sweep function can also be created to help with positive rebases and airdrops.