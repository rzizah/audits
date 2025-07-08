
| ID                                                                                        | Title                                                                             | Severity |
| ----------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------- | -------- |
| [M-01](2025-03-symm-io-stacking.md#m-01-initialization-vulnerability-leading-to-denial-of-service-in-symmvesting-contract) | Initialization Vulnerability Leading to Denial of Service in SymmVesting Contract | medium   |


## [M-01] Initialization Vulnerability Leading to Denial of Service in SymmVesting Contract

### Summary

The `SymmVesting` contract suffers from a denial of service (DoS) vulnerability due to improper usage of the `initializer` modifier instead of `onlyInitializing` in its inheritance chain. The conflict arises when the contract's `initialize` function attempts to invoke the `__vesting_init` function of `vesting` contract, `SymmVesting`, through `__vesting_init(...)` . Since both contracts utilize the `initializer` modifier, the process fails, leaving the `SymmVesting` contract in an unusable state after deployment via a proxy. This effectively causes a denial of service, preventing any use of the contract.

### Root Cause

- In short using `initializer` modifier instead of `onlyInitializing`.

The vulnerability is caused by the interaction between the `initializer` modifier in both the `SymmVesting` and `vesting` contracts. When the `initialize` function of `SymmVesting` is called, it first executes its own initialization logic, where the `initializer` modifier is invoked for the first time. Subsequently, it attempts to call the `__vesting_init` method of the parent contract, `vesting`, using `__vesting_init(...)`

### Internal Pre-conditions

- The `SymmVesting` contract is deployed using a proxy contract.
- The `SymmVesting` contract inherits from `vesting`.
- Both the `SymmVesting` and `vesting` contracts contain `initialize` and `__vesting_init` functions with the `initializer` modifier.

### External Pre-conditions

- The deployment occurs via a proxy contract, where the proxy contract is expected to call the `__vesting_init` function.
    
- An attacker or user attempts to deploy and initialize the `SymmVesting` contract through this proxy, triggering the initialization process.
    

### Attack Path

1. Deploy the `SymmVesting` contract behind a proxy.
2. Call the `initialize` function of `SymmVesting`.
3. The `initialize` function calls `__vesting_init(...)`, which invokes `vesting`'s `__vesting_init` function.
4. The `initializer` modifier in `vesting`'s `__vesting_init` function detects that the contract has already been initialized (due to the previous call in `SymmVesting`), causing a reversion.

### Impact

- The `SymmVesting` contract cannot be properly initialized, leading to a complete denial of service.
- Key contract functionalities cannot be executed, halting operations.

### PoC

_No response_

### Mitigation

1. use `onlyInitializing` instead of `initializer` Modifier from `vesting::__vesting_init` function This will prevent the conflict caused by the call from the child contract.
