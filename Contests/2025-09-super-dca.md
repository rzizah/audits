
| ID                                                                                                                                                          | Title                                                                                                                                              | Severity |
| ----------------------------------------------------------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------- | -------- |
| [H-01](collectFees function will not function correclty with ETH-DCA pair)                                                                                  | collectFees function will not function correclty with ETH-DCA pair                                                                                 | High     |
| [H-02](Lost rewards due to lastRewardIndex being updated without accruing rewards for past staked funds)                                                    | Lost rewards due to lastRewardIndex being updated without accruing rewards for past staked funds                                                   | High     |
| [m-01](New staker gains unearned rewards after zero staking period)                                                                                         | New staker gains unearned rewards after zero staking period                                                                                        | Medium   |
| [M-02]( Changing Mint Rate will incorrectly Lead to wrong incease or decrease of all past Reward sense last accrual as it dosn't update the reward indixes) | Changing Mint Rate will incorrectly Lead to wrong incease or decrease of all past Reward sense last accrual as it dosn't update the reward indixes | Medium   |


# [H-01] collectFees function will not function correclty with ETH-DCA pair


## Summary

Lack of ETH handling in fee collection will cause fee collection to revert for ETH-DCA pools, preventing the owner from collecting the earned fees from listed NFT positions as the contract cannot correctly read ETH balances

## Root Cause

In `SuperDCAListing.sol:305-306` , and `SuperDCAListing.sol:323-324`

```solidity
File: SuperDCAListing.sol
304:         // Record token balances before fee collection
305:         uint256 balance0Before = IERC20(Currency.unwrap(token0)).balanceOf(recipient);
306:         uint256 balance1Before = IERC20(Currency.unwrap(token1)).balanceOf(recipient);

// CODE

322:         // Calculate and emit the collected amounts
323:         uint256 balance0After = IERC20(Currency.unwrap(token0)).balanceOf(recipient);
324:         uint256 balance1After = IERC20(Currency.unwrap(token1)).balanceOf(recipient);
```

The fee collection uses `IERC20.balanceOf()` for both tokens, but this will revert for ETH since it's not an ERC20 token.

## Internal Pre-conditions

1. An nftId for an ETH-DCA pool need to be listed through the listing mechanism 
2. The ETH-DCA pool needs to accumulate fees that can be collected

## Attack Path 

1. LP provides ETH-DCA liquidity and and nftId is listed with `list()` function
2. Pool accumulates fees over time 
3. Owner calls `collectFees()` to send fees to recipient.
4. Transaction reverts when trying to call `balanceOf()` on ETH address (0x0)
5. Earned fees can't be collected for the pair

## Impact

- Earned fees for ETH-DCA pools cannot be collected.
- `collectFees` can't handle native ETH so it won't function with ETH-DCA pools

## Mitigation

Add special handling for ETH by checking if the token address matches the native ETH address:

```diff
++    if (Currency.unwrap(token0) == address(0)) {
++        balance0Before = recipient.balance;
++    } else {
++        balance0Before = IERC20(Currency.unwrap(token0)).balanceOf(recipient);
++    }
```




# [H-02] Lost rewards due to lastRewardIndex being updated without accruing rewards for past staked funds

## Summary

State overwrite in staking operations causes reward loss for users when `stake/unstake` and `accrueReward` are called in the same block or when the `stake/unstake` operations happens before `accrueReward` as it overwrite critical reward tracking state without accumulating pending rewards.

## Root Cause

In `SuperDCAStaking.sol` the stake function updates `info.lastRewardIndex` to latest value of `rewardIndex` that was updated in `_updateRewardIndex` function wich also update `lastMinted` state variable without first collecting pending rewards, causing any accumulated rewards  for the same token to be lost when stake/unstake happen before accrueReward.

## Internal Pre-conditions

1. User must have staked tokens in a token bucket
2. stake/unstake called before  gauge calling accrueReward
3. Rewards must have accumulated since the last update

## Attack Path

1. User has staked tokens and rewards have accumulated
2. User calls stake() or unstake() which updates lastRewardIndex and lastMinted
	1. Attack could happen in 2 ways, A malicious user frontrunning gauge call of accrueReward by staking small amounts or 
	2. Rewards is lost when any user stake some amount for a token that is already staked but gauge yet hasn't called `accruereward`
3. The pending rewards that should have been calculated based on previous stakes are lost

## Impact

Stakers lose their accumulated rewards whenever stake/unstake being called before accrueReward. This can be significant as it affects all rewards accumulated since the last accrual.

## PoC
 add those functions in `PreviewPending` contract and run 
 `forge test --mc PreviewPending --mt testLostRewardsOnSameBlockOperations -vvv`
 
```solidity
    function testLostRewardsOnSameBlockOperations() public {

        // First stake to initialize
        SuperDCAStakingTest.setUp();

        // wraping time after initialization so states could be updated
        vm.warp(block.timestamp + 1);
        
        // 1. User stakes more tokens
        _stake(user, tokenA, 500e18);
        
        // Time passes, rewards accumulate
        vm.warp(block.timestamp + 1 days);
        
        // In the same block:
        // 2. User stakes more tokens
        _stake(user, tokenA, 500e18);
        
        // 3. previewPending is the same as if Gauge calls accrueReward
        uint256 rewards = staking.previewPending(tokenA);
        
        console.log(rewards);

    }

    function testLostRewardsOnSameBlockOperations1() public {


        // First stake to initialize
        SuperDCAStakingTest.setUp();

        // wraping time after initialization so states could be updated
        vm.warp(block.timestamp + 1);
        
        // 1. User stakes more tokens
        _stake(user, tokenA, 500e18);
        
        // Time passes, rewards accumulate
        vm.warp(block.timestamp + 1 days);
        
        // 2. previewPending is the same as if Gauge calls accrueReward
        uint256 rewards = staking.previewPending(tokenA);
        
        console.log(rewards);
    }
```

logs
```bash
super-dca-gauge$ forge test --mc PreviewPending --mt testLostRewardsOnSameBlockOperations -vvv
[⠰] Compiling...
[⠰] Compiling 1 files with Solc 0.8.26
[⠑] Solc 0.8.26 finished in 1.92s
Compiler run successful!

Ran 2 tests for test/SuperDCAStaking.t.sol:PreviewPending
[PASS] testLostRewardsOnSameBlockOperations() (gas: 3121858)
Logs:
  0

[PASS] testLostRewardsOnSameBlockOperations1() (gas: 3068247)
Logs:
  8640000

Suite result: ok. 2 passed; 0 failed; 0 skipped; finished in 2.63ms (1.99ms CPU time)
```

As expected and explained in the report calling stake before accruereward will lead to rewards loss
## Mitigation

- Gauge must accrue rewards before updating state in stake/unstake operations.



# [M-01] New staker gains unearned rewards after zero staking period

## Summary

Lack of state update during zero staking periods will cause inflated rewards for 1st new staker as they can claim rewards for time periods when no tokens were staked in the system

## Root Cause

In `SuperDCAStaking.sol` the `_updateRewardIndex()` function doesn't update `lastMinted` when `totalStakedAmount` is zero, allowing new stakers to claim rewards for periods when no one was staking.

## Internal Pre-conditions

1. All stakers need to unstake, bringing `totalStakedAmount` to zero
2. Time needs to pass with zero total staked amount
3. A new staker needs to stake tokens

## Attack Path

1. All users unstake their tokens, bringing `totalStakedAmount` to 0
2. Time passes (e.g. 1 hour/day/week/month) with no stakers
3. Attacker stakes tokens
4. In same block, gauge calls `accrueReward()`
5. Attacker receives rewards for the entire time when no one was staking

## Impact

The protocol suffers loss of rewards proportional to zero-staking time periods. The attacker gains these unearned rewards that should have been voided.

- broken invariant

> No accrual leakage on stake/unstake:  
> Immediately after stake/unstake for token T, tokenRewardInfos[T].lastRewardIndex == rewardIndex, so getPendingRewards(T) == 0 until time advances.

## PoC

add this test in `PreviewPending` contract and run
` forge test --mc PreviewPending --mt testZeroStakingPeriodRewards -vvv`
```solidity
    function testZeroStakingPeriodRewards() public {

        // First stake to initialize
        SuperDCAStakingTest.setUp();

        // wraping time after initialization so states could be updated
        vm.warp(block.timestamp + 1);

        // 1. adding stakes for tokenA
        _stake(user, tokenA, 500e18);
        
        // Time passes, rewards accumulate
        vm.warp(block.timestamp + 1 days);

        // 2. unstaking all funds for tokenA that now `totalStakedAmount` is zero
        _unstake(user, tokenA, 500e18);

        console.log("totalStakedAmount value: ", staking.totalStakedAmount());

        // Time passes, no rewards should accumulate as there is 0 stakes
        vm.warp(block.timestamp + 1 days);
        
        // In the same block:
        // 3. User stakes tokens
        _stake(user, tokenA, 1000e18);
        
        // now the user will claim rewards for all the time eg: 1 days. while he is just staked at this block
        // 4. previewPending is the same as if Gauge calls accrueReward
        uint256 rewards = staking.previewPending(tokenA);
        console.log("accumulated rewards for the passed day", rewards);

    }
```

logs
```bash

Ran 1 test for test/SuperDCAStaking.t.sol:PreviewPending
[PASS] testZeroStakingPeriodRewards() (gas: 3182515)
Logs:
  totalStakedAmount value:  0
  accumulated rewards for the passed day 8640000

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 2.17ms (1.01ms CPU time)
```


## Mitigation

Update `lastMinted` even when `totalStakedAmount` is zero:

```diff
function _updateRewardIndex() internal {
++    // Always update lastMinted regardless of totalStakedAmount
++    lastMinted = block.timestamp;
    
	  // Return early if no stakes exist or no time has passed
      if (totalStakedAmount == 0) return;
```




# [M-02]Changing Mint Rate will incorrectly Lead to wrong incease or decrease of all past Reward sense last accrual as it dosn't update the reward indixes

## Summary

Lack of reward index synchronization during mint rate changes will cause incorrect reward calculations for stakers as the new rate will be applied retroactively to the entire period since last update

## Root Cause

In `SuperDCAStaking.sol` the mint rate update doesn't force a reward index update before changing the rate, causing the new rate to be applied to all time elapsed since the last update.

## Internal Pre-conditions

1. Time needs to pass without reward accrual (no accrueReward/stake/unstake calls)
2. Admin or gauge needs to modify the mint rate
3. Total staked amount must be greater than 0

## Attack Path

1. Initial state: mintRate = 100, users are staking
2. No reward accrual happens for aperiod of time could be hours/days/weeks
3. Admin increases mintRate to 200
4. When `accrueReward` is finally called:
    - Elapsed time ex: 30 days
    - Uses new mintRate (200) for entire period
    - Rewards are 2x what they should be as the rewards should be calculated with mintrate of 100 and only apply the mintrate for time after the change not the past amount of time

## Impact

Incorrect reward amounts will be calculated. If mintRate increases, they get more rewards than intended. If mintRate decreases, they get fewer rewards than earned. The impact scales with the time between updates and the magnitude of rate change.

## PoC

add this function in `PreviewPending` contract in SuperDCAStaking.t.sol
and run the command
`forge test --mc PreviewPending --mt  testWrongRewardsMintRateChange -vvv`

```solidity

    function testWrongRewardsMintRateChange() public {

        // First stake to initialize
        SuperDCAStakingTest.setUp();


        // 1. adding stakes for tokenA
        _stake(user, tokenA, 500e18);
        
        // Time passes, rewards accumulate
        vm.warp(block.timestamp + 1 days);

        uint256 rewardsBefore = staking.previewPending(tokenA);
        console.log("rewards before the changing mintrate", rewardsBefore);

        // 2. changing mintrate
        vm.prank(admin);
        staking.setMintRate(200);
        // now the user will claim rewards for all the time eg: 1 days. while he is just staked at this block
        // 4. previewPending is the same as if Gauge calls accrueReward
        uint256 rewards = staking.previewPending(tokenA);
        console.log("rewards after changing mintrate", rewards);

    }
```


## Mitigation

Force reward index update before changing mint rate
