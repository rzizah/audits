

| ID                                                                                                                                   | Title                                                                                                                        | Severity |
| ------------------------------------------------------------------------------------------------------------------------------------ | ---------------------------------------------------------------------------------------------------------------------------- | -------- |
| [L-01](l-01-stakers-would-avoid-slashing-effects-incase-the-timing-of-unstaking-came-right-before-reward-distribution-in-succinctstaking) | Stakers would avoid slashing effects incase the timing of unstaking came right before reward distribution in SuccinctStaking | LOW      |
| [I-01](i-01-users-will-still-get-slashed-after-their-unstake-period-ends)                                                                 | Users will still get slashed after their Unstake period ends                                                                 | INFO     |

## [L-01] Stakers would avoid slashing effects incase the timing of unstaking came right before reward distribution in SuccinctStaking
### Description

In the `_unstake()` function of @contracts/src/SuccinctStaking.sol, there is a critical timing vulnerability that allows stakers to completely avoid the effects of slashing when the timing of their unstaking to came with reward distributions. The issue stems from the reward/slashing adjustment logic that only considers the net change in `iPROVE` value, not the individual effects of slashing and rewards.

The root cause is in the comparison logic at line 484 where the function determines how much `iPROVE` to redeem based on whether `iPROVEReceived` is greater than or less than `_iPROVESnapshot`. This creates an edge case where if rewards are distributed shortly after a slashing event occurs during the unstaking period, the positive effect of rewards can mask or completely offset the negative effect of slashing.

The vulnerable logic flow is:

1. When `requestUnstake()` is called, `_iPROVESnapshot` captures the expected `iPROVE` value
2. During the unstaking period, both slashing and reward distribution can occur
3. In `_unstake()`, only the net result is evaluated: if `iPROVEReceived > _iPROVESnapshot`, rewards are assumed; if `iPROVEReceived < _iPROVESnapshot`, slashing is assumed
4. However, if both events occur with rewards exceeding slashing losses, the staker receives the snapshot amount and avoids slashing entirely

This creates a scenario where the timing of reward distribution can completely negate the intended economic penalty of slashing, undermining the security model of the staking system.

### Proof of Concept

run `forge test --mt test_Unstake_WhenSlashDuringUnstakePeriod` you'll notice test will pass and the user won't get affected by the slash

```
function test_Unstake_WhenSlashDuringUnstakePeriod() public {        // Staker stakes with Alice prover.        uint256 stakeAmount = STAKER_PROVE_AMOUNT;        _stake(STAKER_1, ALICE_PROVER, stakeAmount);
        // Request unstake for the full balance.        uint256 stPROVEBalance = IERC20(STAKING).balanceOf(STAKER_1);        vm.prank(STAKER_1);        ISuccinctStaking(STAKING).requestUnstake(stPROVEBalance);
        // Get the snapshot $iPROVE amount from the unstake request.        uint256 iPROVESnapshot =            ISuccinctStaking(STAKING).unstakeRequests(STAKER_1)[0].iPROVESnapshot;
        // During unstake period, a slash occurs.        // Calculate the $iPROVE amount for the slash (1% of the prover's assets).        uint256 proverAssets = IERC4626(ALICE_PROVER).totalAssets();        uint256 slashAmountiPROVE = proverAssets / 100; // 1% slash in $iPROVE terms        vm.prank(OWNER);        MockVApp(VAPP).processSlash(ALICE_PROVER, slashAmountiPROVE);
        // Process the slash after slash period.        skip(SLASH_PERIOD);        vm.prank(OWNER);        ISuccinctStaking(STAKING).finishSlash(ALICE_PROVER, 0);
        // After unstake period, finish unstake.        skip(UNSTAKE_PERIOD);
        // Add more rewards while tokens are in the unstake queue        MockVApp(VAPP).processFulfillment(ALICE_PROVER, proverAssets );
        // Withdraw these rewards too to increase prover's $iPROVE balance.        {            (uint256 protocolFee, uint256 stakerReward, uint256 ownerReward) =                _calculateFullRewardSplit(STAKER_PROVE_AMOUNT / 4);            _withdrawFromVApp(FEE_VAULT, protocolFee);            _withdrawFromVApp(ALICE, ownerReward);            _withdrawFromVApp(ALICE_PROVER, stakerReward);        }
        // The staker Won't receive less $PROVE due to the slash. and the rewards        vm.prank(STAKER_1);        uint256 proveBalanceBefore = IERC20(PROVE).balanceOf(STAKER_1);        uint256 proveReceived = ISuccinctStaking(STAKING).finishUnstake(STAKER_1);        uint256 proveBalanceAfter = IERC20(PROVE).balanceOf(STAKER_1);
        //the user won't get affected affected with slashing and would receive the full amount        assertEq(iPROVESnapshot,proveReceived);
    }
```

### Recommendation

The current approach of using net change to determine rewards vs slashing is fundamentally flawed. The system should track slashing and rewards separately during the unstaking period to ensure both effects are properly applied.

Consider implementing a more robust tracking mechanism that:

1. Records all slashing events that affect a prover during active unstaking periods
2. Applies slashing proportionally to all unstaking claims based on when they were created
3. Separates the handling of rewards and slashing so they cannot offset each other

A potential approach would be to track cumulative slashing per prover and apply it proportionally to unstaking claims based on their timestamp, ensuring that slashing effects cannot be masked by subsequent reward distributions.

## [I-01] Users will still get slashed after their Unstake period ends
### Summary

Users will still get slashed even when their unstake period finished but they didn't call `finishUnstake()` yet

### Finding Description

`SuccinctStaking` contract has a bug where the user can call `requestUnstake()` and wait the unstake period where there are no slashing requests to him.

After the the unstake period passes he shouldn't be susceptible to any slashing

The problem is that if there is a slash request that happened after the unstake period finished but before user calling `finishUnstake()` the user will still get hit by the slashing and will have to wait the slashing duration till it gets applied or cancelled

since user can't finish his unstake if there is any pending slash requests

```
if (slashClaims[prover].length > 0) revert ProverHasSlashRequest();
```

And user will get hit by the new slash that happened after his unstake period ends, as shown here

```
if (iPROVEReceived > _iPROVESnapshot) {            // Rewards were earned during unstaking. Return the excess to the prover.            uint256 excess = iPROVEReceived - _iPROVESnapshot;            IERC20(iProve).safeTransfer(_prover, excess);            iPROVE = _iPROVESnapshot;        } else {            // Either no change or slashing occurred. Staker gets what's available.            iPROVE = iPROVEReceived;        }
```

### Impact Explanation

Medium: loss of funds to the user

### Likelihood Explanation

Medium: requires certain conditions to happen

### Proof of Concept

```
function test_Slash_After_unstake_period_finishes() public {        uint256 stakeAmount = STAKER_PROVE_AMOUNT;                // Stake to Alice prover        uint256 shares = _permitAndStake(STAKER_1, STAKER_1_PK, ALICE_PROVER, stakeAmount);
        // Check state after staking        assertEq(SuccinctStaking(STAKING).balanceOf(STAKER_1), stakeAmount);        assertEq(SuccinctStaking(STAKING).staked(STAKER_1), stakeAmount);
        // Slash the prover for all of their stake        uint256 slashAmount = SuccinctStaking(STAKING).proverStaked(ALICE_PROVER);        _requestUnstake(STAKER_1, shares);        skip(UNSTAKE_PERIOD);
        _completeSlash(ALICE_PROVER, slashAmount);
        vm.expectRevert();        // no shares to withdraw, all shares have been slashed        _completeUnstake(STAKER_1, stakeAmount);  
        // Verify fully slashed, even when full unstake period passed beforethe slash request
        assertEq(SuccinctStaking(STAKING).balanceOf(STAKER_1), 0);
        assertEq(SuccinctStaking(STAKING).staked(STAKER_1), 0);
        assertEq(IERC20(PROVE).balanceOf(STAKER_1), 0);
        assertEq(IERC20(I_PROVE).balanceOf(ALICE_PROVER), 0);
        assertEq(IERC20(PROVE).balanceOf(I_PROVE), 0);
    }
```

### Recommendation

track the slashing request with timestamps
