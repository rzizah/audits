## Summary

| ID                                                                                                                             | Title                                                                                            | Severity |
| ------------------------------------------------------------------------------------------------------------------------------ | ------------------------------------------------------------------------------------------------ | -------- |
| [M-01](2024-08-Midas.md#m-01-discrepancies-between-the-spec-and-the-code-redeem-mtbill-for-usdc-pulled-from-buidl-edge-case) | discrepancies between the spec and the code 'redeem mTBILL for USDC pulled from BUIDL' Edge case | Medium   |



## [M-01] discrepancies between the spec and the code 'redeem mTBILL for USDC pulled from BUIDL' Edge case


## Summary

The edge case of 'User can instantly redeem mTBILL for USDC pulled from BUIDL' isn't implemented anywhere in the contracts.

## Vulnerability Detail

Quoted from [(Special Contract Version) User can instantly redeem mTBILL for USDC pulled from BUIDL](https://ludicrous-rate-748.notion.site/Special-Contract-Version-User-can-instantly-redeem-mTBILL-for-USDC-pulled-from-BUIDL-927832e82a874221996c1edcc1d94b17)

> ### Edge case
> 
> - Redeem the full BUIDL balance in the smartcontract if the BUIDL balance will be less than 250k post transaction (as 250k is the minimum). Make this **250k threshold a parameter** that can be adjusted by the admin

###### The edge case isn't implemented and should be implemented

## Impact

Not handling the Edge case

## Code Snippet

## Tool used

Manual Review

## Recommendation

- Assign a state variable that can be adjusted by the admin With initial value of 250,000.

```solidity
uint256 Threshold = 250,000
```

- Create a setter function to adjust its value,

```solidity
    function setThresholdAmount(uint256 newValue) external onlyVaultAdmin {
        Threshold = newValue;
        emit SetThresholdAmount(msg.sender, newValue);
    }
```

- and an event in the interface.

```solidity
    event SetThresholdAmount(address indexed caller, uint256 Threshold);

```

- Create an if statement to check for this edge case in the redeem function.
