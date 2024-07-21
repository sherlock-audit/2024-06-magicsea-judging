Long Walnut Sloth

Medium

# setLumPerSecond does not update pools

## Summary

## Vulnerability Detail

`_lumPerSecond` is used to calculate LUM emissions in MasterChefV2. Rewards are distributed every time someone stakes, claims, or withdraws their tokens. However, if `_lumPerSecond` is updated without calling `updateAll`, all pools will distribute incorrect amount of LUM next time they are updated.

## Impact

Too many or too little LUM will be emitted, because new `_lumPerSecond` is retroactively applied to the period before its update.

## Code Snippet
https://github.com/sherlock-audit/2024-06-magicsea/blob/main/magicsea-staking/src/MasterchefV2.sol#L347-L355
## Tool used

Manual Review

## Recommendation
```diff
    function setLumPerSecond(uint96 lumPerSecond) external override onlyOwner {
        if (lumPerSecond > Constants.MAX_LUM_PER_SECOND) revert MasterChef__InvalidLumPerSecond();
-       // _updateAll(_voter.getTopPoolIds()); // todo remove this
+       _updateAll(_voter.getTopPoolIds()); 
``` 