Elegant Vanilla Crane

Medium

# Rewards might get stuck when approved actor renews a position

## Summary

When an approved actor calls the harvest function, the rewards get sent to the user (staker). However, when the approved actor renews the user’s position, they receive the rewards instead.

If the approved actor is a smart contract (e.g., a keeper), the funds might get stuck forever or go to the wrong user, such as a Chainlink keeper.

## Vulnerability Detail

Suppose Alice mints an NFT by creating a position and approves Bob to use it.

- When Bob calls `harvestPosition` with Alice’s `tokenId`, Alice will receive the rewards (as intended)
- When Bob calls `renewLockPosition` with Alice’s `tokenId`, Bob will receive the rewards. The internal function `_lockPosition`, which is called by `renewLockPosition`, also harvests the position before updating the lock duration. Unlike the harvest function, `_lockPosition` [sends the rewards to `msg.sender`](https://github.com/sherlock-audit/2024-06-magicsea/blob/main/magicsea-staking/src/MlumStaking.sol#L710) instead of the token owner.

This bug exists in both `renewLockPosition` and `extendLockPosition`, as they both call `_lockPosition`, which includes the wrong receiver.

### PoC

To run this test, add it into `MlumStaking.t.sol`.

```solidity
function testVuln_ApprovedActorReceivesRewardsWhenRenewingPosition() public {
    // setup pool
    uint256 _amount = 100e18;
    uint256 lockTime = 1 days;

    _rewardToken.mint(address(_pool), 100_000e6);
    _stakingToken.mint(ALICE, _amount);

    // alice creates new position
    vm.startPrank(ALICE);
    _stakingToken.approve(address(_pool), _amount);
    _pool.createPosition(_amount, lockTime);
    vm.stopPrank();

    // alice approves bob
    vm.prank(ALICE);
    _pool.approve(BOB, 1);

    skip(1 hours);

    // for simplicity of the PoC we use a static call
    // IMlumStaking doesn't include `renewLockPosition(uint256)`
    uint256 bobBefore = _rewardToken.balanceOf(BOB);
    vm.prank(BOB);
    address(_pool).call(abi.encodeWithSignature("renewLockPosition(uint256)", 1));

    // Bob receivew the rewards, instead of alice
    assertGt(_rewardToken.balanceOf(BOB), bobBefore);
}
```

## Impact

Possible loss of reward tokens

## Code Snippet

https://github.com/sherlock-audit/2024-06-magicsea/blob/main/magicsea-staking/src/MlumStaking.sol#L710

## Tool used

Manual Review, Foundry

## Recommendation

Change `_lockPosition()` in `MlumStaking.sol`  to use the owner of the position instead of `msg.sender`.

```solidity
function _lockPosition(uint256 tokenId, uint256 lockDuration, bool resetInitial) internal {
    ...
-   _harvestPosition(tokenId, msg.sender);
+   _harvestPosition(tokenId, _ownerOf(tokenId));
    ...
}
```
