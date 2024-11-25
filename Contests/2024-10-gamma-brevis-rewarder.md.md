


# [H-01] users can't claim his rewards of epochs with the same distributionId

### Summary

When creating a distribution (distribution range) distributionId could include multiple epochs

Then at `gammarewarder::handleProofResult` line [GammaRewarder.sol#L209-L210](https://github.com/sherlock-audit/2024-10-gamma-rewarder/blob/475f7fbd0f7c2717ed585a67632e9a675b51c306/GammaRewarder/contracts/GammaRewarder.sol#L209-L210)

```solidity
require(claim.amount == 0 , "Already claimed reward.");
```

the require reverts if the user has claimed an epoch before in the same distribution range `distributionId`

### Root Cause

```solidity
require(claim.amount == 0 , "Already claimed reward.");
```

### Internal pre-conditions

The user claimed his rewards of one epoch in the same distribution range

### Attack Path

- User claims an epoch normally with the distributionId.
- User tries to claim another epoch in the same distribution range `distributionId`.
- Claim revert due to require.

### Impact

loss of funds for users that claimed an epoch and can't claim for other epochs with the same distribution range.

### mitigation

remove the require and apply the appropriate check.