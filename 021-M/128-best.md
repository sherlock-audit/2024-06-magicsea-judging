Slow Maroon Gibbon

High

# ` _voter` state variable should be immutable  in `MasterChefV2.sol`

## Summary
The `_voter` state variable is meant to be immutable, but since it is not, it will disrupt the proxy storage.
## Vulnerability Detail
```solidity 
 ILum private immutable _lum;
    //@audit should make this immutable again
    IVoter private _voter; // TODO make immutable again
    IRewarderFactory private immutable _rewarderFactory;
    address private immutable _lbHooksManager;
```
Here the `_voter` variable is supposed to be immutable, We can see that with the TODO comment and the initialization of _voter in the constructor. 

The contract uses a proxy pattern so it should always set the non-immutable variables in the `initialize` function, not the constructor. When we initialize a normal state variable in a constructor it never gets saved to proxy but only to the state variable of the implementation contracts.

## Impact
Incorrect mutability of the function will break the proxy storage layout. 

## Code Snippet
https://github.com/sherlock-audit/2024-06-magicsea/blob/42e799446595c542eff9519353d3becc50cdba63/magicsea-staking/src/MasterchefV2.sol#L35

## Tool used

Manual Review

## Recommendation
Make the `_voter` variable immutable.