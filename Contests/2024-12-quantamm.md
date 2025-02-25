# QuantAMM - Findings Report

# Table of contents
- ## [Contest Summary](#contest-summary)
- ## [Results Summary](#results-summary)
- ## High Risk Findings
    - ### [H-01. fees sent to QuantAMMAdmin is stuck forever as there is no function to retrieve them](#H-01)
    - ### [H-02. Huge precision loss leading to error calculation of adminFee](#H-02)
    - ### [H-03. Wrong calculation of gradient](#H-03)
    - ### [H-04. Wrong calculation of Variance](#H-04)
    - ### [H-05. loss of ownerFee as there is no way to retrieve them](#H-05)
    - ### [H-06. user can bypass paying fees by updating his position to another account then remove liquidity](#H-06)
- ## Medium Risk Findings
    - ### [M-01. user can bypass paying fees](#M-01)
    - ### [M-02. length of poolsFeeData can by bypassed](#M-02)
    - ### [M-03. User bypass paying fees due to wrong calculation](#M-03)
- ## Low Risk Findings
    - ### [L-01. missing implementation for a function to change upliftFee](#L-01)
    - ### [L-02.  wrong value added to blockTimestampDeposit in afterUpdate](#L-02)


# <a id='contest-summary'></a>Contest Summary

### Sponsor: QuantAMM

### Dates: Dec 20th, 2024 - Jan 15th, 2025

[See more contest details here](https://codehawks.cyfrin.io/c/2024-12-quantamm)

# <a id='results-summary'></a>Results Summary

### Number of findings:
- High: 6
- Medium: 3
- Low: 2


# High Risk Findings

## <a id='H-01'></a>H-01. fees sent to QuantAMMAdmin is stuck forever as there is no function to retrieve them            



## Summary

When a user removes liquidity from the pool he pays fees and a portion of the fees is liquidity added back for `QuantAMMAdmin`

Those funds are stuck in, and can't be removed as there is no mapping or NFT minted for those shares so any attempt to removeliquidity will revert

## Vulnerability Details

Here is the call inside `onAfterREmoveLiquidity` to add fees to `quantAMMAdmin`

```solidity
File: UpliftOnlyExample.sol
536:         if (localData.adminFeePercent > 0) {
537:             _vault.addLiquidity(
538:                 AddLiquidityParams({
539:                     pool: localData.pool,
540:                     to: IUpdateWeightRunner(_updateWeightRunner).getQuantAMMAdmin(),
541:                     maxAmountsIn: localData.accruedQuantAMMFees,
542:                     minBptAmountOut: localData.feeAmount.mulDown(localData.adminFeePercent) / 1e18,
543:                     kind: AddLiquidityKind.PROPORTIONAL,
544:                     userData: bytes("")
545:                 })
```

In `removeLiquidityProportional` there is a check for the length of `poolsFeeData` in this case, would be zero and any attempt to remove will revert

```solidity
File: UpliftOnlyExample.sol
265:     function removeLiquidityProportional(
266:         uint256 bptAmountIn,
267:         uint256[] memory minAmountsOut,
268:         bool wethIsEth,
269:         address pool
270:     ) external payable saveSender(msg.sender) returns (uint256[] memory amountsOut) {
271:         uint depositLength = poolsFeeData[pool][msg.sender].length;
272: 
273:         if (depositLength == 0) {
274:             revert WithdrawalByNonOwner(msg.sender, pool, bptAmountIn);
275:         }
```

### POC

Add this anywhere in `UpliftExample.t.sol`

```solidity
    function testRemoveLiquidity_Stuck_Admin_Fees() public {
        vm.prank(address(vaultAdmin));
        updateWeightRunner.setQuantAMMUpliftFeeTake(0.5e18);
        vm.stopPrank();

        // Add liquidity so bob has BPT to remove liquidity.
        uint256[] memory maxAmountsIn = [dai.balanceOf(bob), usdc.balanceOf(bob)].toMemoryArray();

        vm.prank(bob);
        upliftOnlyRouter.addLiquidityProportional(pool, maxAmountsIn, bptAmount, false, bytes(""));
        vm.stopPrank();

        uint256[] memory minAmountsOut = [uint256(0), uint256(0)].toMemoryArray();

        vm.startPrank(bob);
        upliftOnlyRouter.removeLiquidityProportional(bptAmount, minAmountsOut, false, pool);
        vm.stopPrank();
        BaseVaultTest.Balances memory balancesAfter = getBalances(updateWeightRunner.getQuantAMMAdmin());

        //trying to remove liquidity added to QuantAMMAdmin with the value added from bob removing liquidity the remove attempt will revert with `WithdrawalByNonOwner` error
        vm.prank(updateWeightRunner.getQuantAMMAdmin());
        vm.expectRevert();
        upliftOnlyRouter.removeLiquidityProportional(500000000000000000, minAmountsOut, false, pool);
        vm.stopPrank();

    }
```

## Impact

* **Loss of Funds**: Fees added as liquidity for `QuantAMMAdmin` are stuck and cannot be removed.

## Tools Used

manual review

## Recommendations

* create a new array while creating as done for normal addition

## <a id='H-02'></a>H-02. Huge precision loss leading to error calculation of adminFee            



## Summary

In `onAfterSwap` function the calculation of `adminFee` is as follow

```solidity
File: UpliftOnlyExample.sol
334:                 if (quantAMMFeeTake > 0) {
335:                     uint256 adminFee = hookFee / (1e18 / quantAMMFeeTake);
```

Which is incorrect due to the multiple-division

The right equation should be:

```solidity
uint256 adminFee = hookFee * quantAMMFeeTake / 1e18;
```

## POC

Add this test on any test file ex: UpliftExample.t.sol

```solidity
     //test calculate wrong admin fee and the right one
     function test_calc_fee() public {
         //assume quantAMMFeeTake is 500000000000000009
         uint256 quantAMMFeeTake = 500000000000000009;
         //assume hookFee is 500000000000000000
         uint256 hookFee = 500000000000000000;
         uint256 wrongAdminFee = hookFee / (1e18 / quantAMMFeeTake);
         uint256 rightAdminFee = hookFee * quantAMMFeeTake / 1e18;
         assertNotEq(wrongAdminFee,rightAdminFee);
     }
```

you'll notice that the wrong Fee was 5e17 while the correct one was 250000000000000004\
which is a huge loss and an inaccurate calculation of the Fee

## Impact

* precision loss leads to a loss of funds to the owner.

```solidity
ownerFee = hookFee - adminFee;
```

## Tools Used

Manual review

## Recommendations

Use the correct equation as follows

```diff
--                 uint256 adminFee = hookFee / (1e18 / quantAMMFeeTake);
++                 uint256 adminFee = hookFee * quantAMMFeeTake / 1e18;
```

## <a id='H-03'></a>H-03. Wrong calculation of gradient            



## Summary

In gradient calculation if we have for example 6 gradients they get packed into 3 in `intermediateGradientStates`

Calculating the gradient incase of vector lambda it packs with the wrong index forwarding `i` instead of `locals.storageArrayIndex` for `intermediateGradientStates` which assign wrong values of the gradient.

The gradient calculation is to unpack `intermediateGradientStates` then enter a loop and calculate the new values and packing them again in `intermediateGradientStates`\
the loop go as follow

* getting the value for `i` and `i+1`
* Calculating then packing them with index of `storageArrayIndex` in `intermediateGradientStates`
* then i+=2 and ++locals.storageArrayIndex

a simple chart to simplify the process:

| intermediateGradientStates | \[0] | \[1] | \[2] |
| :------------------------- | :--- | :--- | :--- |
| stored gradient \[i]       | 0,1  | 2,3  | 4,5  |

incase of a vector wrong index is used for `intermediateGradientStates` `i` instead of `storageArrayIndex` so values is packed as follow

| intermediateGradientStates | \[0] | \[1]                              | \[2] | \[3]                               | \[4] |
| :------------------------- | :--- | :-------------------------------- | :--- | :--------------------------------- | :--- |
| stored gradient            | 0,1  | OldGradient (no new value stored) | 2,3  | 0 (no value as length should be 3) | 4,5  |

incase number of assets 7 the index is done correctly for `intermediateGradientStates` but overwrite one value due to wrong index used in the loop

| intermediateGradientStates | \[0] | \[1]                          | \[2] | \[3]                   | \[4] |
| :------------------------- | :--- | :---------------------------- | :--- | :--------------------- | :--- |
| stored gradient            | 0,1  | OldGradient (no value stored) | 2,3  | Last IntermediateValue | 4,5  |

[Affected code](https://github.com/Cyfrin/2024-12-quantamm/blob/475398c75c631ad9ba79b69c1b7b61ff91cc2701/pkg/pool-quantamm/contracts/rules/base/QuantammGradientBasedRule.sol#L156-L157)

```solidity
File: QuantammGradientBasedRule.sol
156:                 intermediateGradientStates[_poolParameters.pool][i] = _quantAMMPackTwo128(
157:                     locals.intermediateGradientState[i],
158:                     locals.secondIntermediateValue
159:                 );
160:                 unchecked {
161:                     i += 2;
162:                     ++locals.storageArrayIndex;
163:                 }
164:             }
```

## Impact

* broken calculation giving wrong calculations affecting all function depends on it
  * QuantAMMGradientBasedRule
  * PowerChannelUpdateRule
  * MomentumUpdateRule
  * ChannelFollowingUpdateRule
  * AntiMomentumUpdateRule
* all roles will calculate wrong weights which is the core of the protocol and break all its logic

## Tools Used

Manual review

## Recommendations

Use the correct value for index

```diff
File: QuantammGradientBasedRule.sol
--                 intermediateGradientStates[_poolParameters.pool][i] = _quantAMMPackTwo128(
++                 intermediateGradientStates[_poolParameters.pool][locals.storageArrayIndex] = _quantAMMPackTwo128(

```

## <a id='H-04'></a>H-04. Wrong calculation of Variance            



## Summary

In variance calculation it packs each 2 variance in 1 and if number of assets is odd then last will be appended directly at the end.

Incase of odd number and a vector lambda the check for `notDivisbleByTwo` is missing leading to wrong values being calculated

logic goes as follow

* if length is even ex: 8

| intermediateVarianceStates | \[0] | \[1] | \[2] | \[3] |
| :------------------------- | :--- | :--- | :--- | :--- |
| indexes packed             | 0,1  | 2,3  | 4,5  | 6,7  |

```Solidity
- loop go untill 7
	- with i being 0,2,4,6
	- secondIndex 1,3,5,7
```

* if length is 7

| intermediateVarianceStates | \[0] | \[1] | \[2] | \[3] |
| :------------------------- | :--- | :--- | :--- | :--- |
| indexes packed             | 0,1  | 2,3  | 4,5  | 6    |

```Solidity
- loop go untill 5
	- i is 0,2,4
	- secondIndex 1,3,5
- this store 6 values and the 7th stored separetly out of the loop
```

With vector lambda subtracting 1 from nMinusOne is missing leading to loop go as follow

| intermediateVarianceStates | \[0] | \[1] | \[2] | \[3] | \[4] |
| :------------------------- | :--- | :--- | :--- | :--- | :--- |
| indexes packed             | 0,1  | 2,3  | 4,5  | 6,7  | 8    |

with values of index 7,8 is zero leading to wrong calculation and mismatching of future variance calculation\
\- wrong values will be calculated\
\- admin can't assign the correct values as length won't match his inputted data

```solidity
File: QuantammVarianceBasedRule.sol
196:     /// @param _numberOfAssets the number of assets in the pool
197:     function _setIntermediateVariance(
198:         address _poolAddress,
199:         int256[] memory _initialValues,
200:         uint _numberOfAssets
201:     ) internal {
202:         uint storeLength = intermediateVarianceStates[_poolAddress].length;
```

1. wrong values stored as in this example storage length is retrieved from tokens length and %2 !=0 so value will be retrieved in this example index\[3] that holds a packed value without unpacking and the wrong value will be used

## Impact

* broken calculation giving wrong calculation affecting all function depends on it
  * MinimumVarianceUpdateRule

## Tools Used

manual review

## Recommendations

apply the correct check incase of odd number incsae of vector lambda

```diff
++            if (locals.notDivisibleByTwo) {
++                unchecked {
++                    --locals.nMinusOne;
++                }
++            }
```

## <a id='H-05'></a>H-05. loss of ownerFee as there is no way to retrieve them            



## Summary

In `UpliftOnlyExample::onAfterSwap` there is fees paid to owner that sent to the `UpliftOnlyExample` contract that has no function to retrieve the fees from it leading to fees being stuck in the contract forever

## Vulnerability Details

In `UpliftOnlyExample::onAfterSwap` if it is ownerFee it being sent to the address of the router which has no function to take out those fees

```solidity
File: UpliftOnlyExample.sol
342:                 if (ownerFee > 0) {
343:                     _vault.sendTo(feeToken, address(this), ownerFee);
```

## Impact

* Loss of funds
* funds stuck in the contract forever

## Tools Used

manual review

## Recommendations

implement a function to retrieve the fees

## <a id='H-06'></a>H-06. user can bypass paying fees by updating his position to another account then remove liquidity            



## Summary

User add liquidity to the protocol and if there is a positive change in price he pays fees that depend on `BptAmount` he is withdrawing

he can bypass paying this fee by updating his position to another account

```solidity
File: UpliftOnlyExample.sol
576:     function afterUpdate(address _from, address _to, uint256 _tokenID) public {
// CODE
605:         if (tokenIdIndexFound) {
606:             if (_to != address(0)) {
607:                 // Update the deposit value to the current value of the pool in base currency (e.g. USD) and the block index to the current block number
608:                 //vault.transferLPTokens(_from, _to, feeDataArray[i].amount);
609:                 feeDataArray[tokenIdIndex].lpTokenDepositValue = lpTokenDepositValueNow;
610:                 feeDataArray[tokenIdIndex].blockTimestampDeposit = uint32(block.number);
611:                 feeDataArray[tokenIdIndex].upliftFeeBps = upliftFeeBps;
```

as seen above the logic of updating the current `lpTokenDepositValueNow` is recorded

#### Scenario

* User added liquidity where `lpTokenDepositValueNow` = 2e18
* Time passes and `lpTokenDepositValueNow` = 10e18 (user should pay fees for that positive change )
* user updated his position to his other account recording the current `lpTokenDepositValueNow` then remove liquidity
* then a user removes liquidity with recorded `lpTokenDepositValueNow` of 10e18
* user pays the minimum fee as there is no positive change in price

```solidity
File: UpliftOnlyExample.sol
471:         for (uint256 i = localData.feeDataArrayLength - 1; i >= 0; --i) {
472:             localData.lpTokenDepositValue = feeDataArray[i].lpTokenDepositValue;
473: 
474:             localData.lpTokenDepositValueChange =
475:                 (int256(localData.lpTokenDepositValueNow) - int256(localData.lpTokenDepositValue)) /
476:                 int256(localData.lpTokenDepositValue);
477: 
478:             uint256 feePerLP;
479:             // if the pool has increased in value since the deposit, the fee is calculated based on the deposit value
480:             if (localData.lpTokenDepositValueChange > 0) {
481:                 feePerLP =
482:                     (uint256(localData.lpTokenDepositValueChange) * (uint256(feeDataArray[i].upliftFeeBps) * 1e18)) /
483:                     10000;
484:             }
485:             // if the pool has decreased in value since the deposit, the fee is calculated based on the base value - see wp
486:             else {
487:                 //in most cases this should be a normal swap fee amount.
488:                 //there always myst be at least the swap fee amount to avoid deposit/withdraw attack surgace.
489:                 feePerLP = (uint256(minWithdrawalFeeBps) * 1e18) / 10000;
490:             }
```

## Impact

Loss of funds/fees

## Tools Used

manual review

## Recommendations

incase of update the first recorded value should be the same

    
# Medium Risk Findings

## <a id='M-01'></a>M-01. user can bypass paying fees            



## Summary

in `UpliftOnlyExample` user can bypass paying fees by waiting for the `upliftFeeBps` to be law then swap to his other account and withdraw with the few fee

this report assumes implementation of the function specified by the sponsor [here](https://youtu.be/xNhk01UDKjo?t=2451) has been done and solved

#### scenario

`upliftFeeBps` is vthe alue of the fee charged while withdrawing the value can be changed but not for already added BptAmount

```solidity
File: UpliftOnlyExample.sol
606:             if (_to != address(0)) {
607:                 // Update the deposit value to the current value of the pool in base currency (e.g. USD) and the block index to the current block number
608:                 //vault.transferLPTokens(_from, _to, feeDataArray[i].amount);
609:                 feeDataArray[tokenIdIndex].lpTokenDepositValue = lpTokenDepositValueNow;
610:                 feeDataArray[tokenIdIndex].blockTimestampDeposit = uint32(block.number);
611:@>               feeDataArray[tokenIdIndex].upliftFeeBps = upliftFeeBps;
```

1. Assume Bob added liquidity at `upliftFeeBps` = 5000
2. then he wanted to the withdraw creator to change the fee to 100 (this value is for future deposits)
3. bob call update the position to his other account which will store the new fee value of 100
4. after that, he can withdraw from that account with a few fee

## Impact

* loss of (funds / fees)

## Tools Used

manual review

## Recommendations

in case of swap\
when liquidity is added the fee should stay as is and not changed to the new fee

## <a id='M-02'></a>M-02. length of poolsFeeData can by bypassed            



## Summary

When user add liquidity length of the poolsFeeData array can't exceed 100 to mitigate DDOS issus but this check is forgotten in afterUpdate

#### scenario

* bob has already 100 deposits
* an update happens many times
* bob array length keeps getting bigger with no restriction of 100

## Vulnerability Details

## Impact

* leads to DDOS attacks (which the protocol is trying to mitigate)
* out of gas issues causing user funds to get stuck

## Tools Used

manual review

## Recommendations

in afterUpdate function add this check

```diff
++        if (poolsFeeData[pool][_to].length >= 100) {
++            revert TooManyDeposits(pool, msg.sender);
++        }
```

## <a id='M-03'></a>M-03. User bypass paying fees due to wrong calculation            



## Summary

In `onAfterRemoveLiquidity` function if there is a positive change in price user pays fees to the hook depending on the `bptamount` being removed

however, due to wrong calculation, a round down happens and the user can take profit without paying any fees.

## Vulnerability Details

The equation to calculate if there is a positive change in price or not is calculated as follows

```solidity
File: UpliftOnlyExample.sol
474:             localData.lpTokenDepositValueChange =
475:                 (int256(localData.lpTokenDepositValueNow) - int256(localData.lpTokenDepositValue)) /
476:                 int256(localData.lpTokenDepositValue);

```

The equation has a severe issue as the user can take profit up to (lpTokenDepositValue \* 2) - 1\
ex: if the price in the deposit was 2 then at the time of removing liquidity user can remove up to 3 without paying fees and pays fees as there is no profit happened

### POC

Add this test at the end of `UpliftExample.t.sol` contract\
it has comments showing the right amount that should have been subtracted and the wrong amount that was applied

```solidity
    //test user can take up to near double profit without paying fees
    function testRemoveLiquidityWithProtocolTakeNearDoublePositivePriceChangeWithoutPayingFees() public {
        vm.prank(address(vaultAdmin));
        updateWeightRunner.setQuantAMMUpliftFeeTake(0.5e18);
        vm.stopPrank();
        // Add liquidity so bob has BPT to remove liquidity.
        uint256[] memory maxAmountsIn = [dai.balanceOf(bob), usdc.balanceOf(bob)].toMemoryArray();
        vm.prank(bob);
        upliftOnlyRouter.addLiquidityProportional(pool, maxAmountsIn, bptAmount, false, bytes(""));
        vm.stopPrank();

        //mock prices to be 1.9 OldPrice
        int256[] memory prices = new int256[]();
        for (uint256 i = 0; i < tokens.length; ++i) {
            prices[i] = int256(i) * 1.9e18;
        }
        updateWeightRunner.setMockPrices(pool, prices);

        uint256[] memory minAmountsOut = [uint256(0), uint256(0)].toMemoryArray();

        BaseVaultTest.Balances memory balancesBefore = getBalances(updateWeightRunner.getQuantAMMAdmin());

        vm.startPrank(bob);
        upliftOnlyRouter.removeLiquidityProportional(bptAmount, minAmountsOut, false, pool);
        vm.stopPrank();
        BaseVaultTest.Balances memory balancesAfter = getBalances(updateWeightRunner.getQuantAMMAdmin());

        // The right fee amount that must be taken is as follow
        uint256 feeAmountAmountPercentpsitivechange = ((bptAmount / 2) *
            ((uint256(upliftOnlyRouter.upliftFeeBps()) * 1e18) / 10000)) / ((bptAmount / 2));

        uint256 amountOutPositiveChange = (bptAmount / 2).mulDown((1e18 - feeAmountAmountPercentpsitivechange));

        // Calculation Of Fees Incase of negative change of price 
        uint256 feeAmountAmountPercent = ((bptAmount / 2) *
            ((uint256(upliftOnlyRouter.minWithdrawalFeeBps()) * 1e18) / 10000)) / ((bptAmount / 2));
        uint256 amountOut = (bptAmount / 2).mulDown((1e18 - feeAmountAmountPercent));

        // Bob gets original liquidity with no fee applied because of not exceeding the double amount.
        assertEq(
            balancesAfter.bobTokens[daiIdx] - balancesBefore.bobTokens[daiIdx],
            amountOut,
            "bob's DAI amount is wrong"
        );
        assertEq(
            balancesAfter.bobTokens[usdcIdx] - balancesBefore.bobTokens[usdcIdx],
            amountOut,
            "bob's USDC amount is wrong"
        );

        // the fees taken doesn't match the right fee amount
        assertNotEq(
            balancesAfter.bobTokens[daiIdx] - balancesBefore.bobTokens[daiIdx],
            amountOutPositiveChange,
            "bob's DAI amount is wrong"
        );
        assertNotEq(
            balancesAfter.bobTokens[usdcIdx] - balancesBefore.bobTokens[usdcIdx],
            amountOutPositiveChange,
            "bob's USDC amount is wrong"
        );
    }
```

## Impact

* Loss Of funds as the wrong fee amount is being calculated.

## Tools Used

manual review

## Recommendations

apply this adjustment\
this code applies a check if ( lpTokenDepositValueNow > lpTokenDepositValue )\
to apply whether the fee for positive change or negative using the same original formula

```diff

--            localData.lpTokenDepositValueChange =
--                (int256(localData.lpTokenDepositValueNow) - ----int256(localData.lpTokenDepositValue)) /
--                int256(localData.lpTokenDepositValue);
--
--            uint256 feePerLP;
--            // if the pool has increased in value since the deposit, the fee is calculated based on the deposit value
--            if (localData.lpTokenDepositValueChange > 0) {
--                feePerLP =
--                    (uint256(localData.lpTokenDepositValueChange) * (uint256(feeDataArray[i].upliftFeeBps) * 1e18)) /
--                   10000;
--            }
--            // if the pool has decreased in value since the deposit, the fee is calculated based on the base value - see wp
--            else {
--                //in most cases this should be a normal swap fee amount.
--                //there always myst be at least the swap fee amount to avoid deposit/withdraw attack surgace.
--                feePerLP = (uint256(minWithdrawalFeeBps) * 1e18) / 10000;
--            }

  

++        uint256 feePerLP;
++        
++        // If the LP token deposit value has increased
++        if (localData.lpTokenDepositValueNow > localData.lpTokenDepositValue) {
++            // Calculate the percentage change in value
++            localData.lpTokenDepositValueChange = 
++                (1e18 * (int256(localData.lpTokenDepositValueNow) - int256(localData.lpTokenDepositValue))) /
++                int256(localData.lpTokenDepositValue);
++        
++            // Calculate the fee per LP based on the uplift fee
++            // 1e18 is used to maintain precision
++            feePerLP = 
++                (uint256(localData.lpTokenDepositValueChange) * uint256(feeDataArray[i].upliftFeeBps)) /
++                10000;
++        } else {
++            // If the pool value has decreased since deposit, calculate a base fee
++            // This ensures there's always at least the swap fee to avoid attacks during deposit/withdrawal cycles
++            feePerLP = (uint256(minWithdrawalFeeBps) * 1e18) / 10000;
++        }
```


# Low Risk Findings

## <a id='L-01'></a>L-01. missing implementation for a function to change upliftFee            



## Summary

`upliftFeeBps` Is the uplift fee in basis points (1/10000) for the pool\
the pool creator should have the ability to change this value for future deposits as stated by the sponsor [here](https://youtu.be/xNhk01UDKjo?t=2451) but there is no function anywhere to do so It is also stated by the sponsor[ ](https://youtu.be/xNhk01UDKjo?t=2451)to flag any missing implementation

#### scenario

pool creator set the `upliftFeeBps` to 0 as a start then adjust it later then after deployment and some deposits he found out he couldn't change the value

this could be all the way around with fees being too high and not being set back to decent value

## Impact

* function not implemented breaks contract design forcing the creator to static `upliftFeeBps`
* loss of (funds / fees) to the creator

## Tools Used

manual review

## Recommendations

implement a function to change `upliftFeeBps`

```diff
++    function changeUpliftFeeBps(uint256 newUpliftFeeBps) external onlyowner {
++		uint256 oldUpliftFeeBps = upliftFeeBps;
++      require(newUpliftFeeBps <= 10000, "upliftFeeBps must be less than 10000")
++		upliftFeeBps = newUpliftFeeBps;	
++		emit upliftFeeBpsChanged(upliftFeeBps,oldUpliftFeeBps)
++	}
```

## <a id='L-02'></a>L-02.  wrong value added to blockTimestampDeposit in afterUpdate            



## Summary

In `afterUpdate` function the `feeDataArray` holds `blockTimestampDeposit` which is the timestamp of when the index of this array has been started

In `addLiquidityProportional` it hold `block.timestamp` value but\
In `afterUpdate` it hold `block.number`

## Impact

* Function is broken and not working as intended

## Tools Used

Manual review

## Recommendations

Use `block.timestamp`



