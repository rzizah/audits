
| ID                                                           | Title                                                | Severity |
| ------------------------------------------------------------ | ---------------------------------------------------- | -------- |
| [M-01](infrared-contracts.md#m-01-malicious-user-can-dos-updaterewardsdurationforvault) | malicious user can DOS updateRewardsDurationForVault | Medium   |

## [M-01] malicious user can DOS updateRewardsDurationForVault
### Summary

the `updateRewardsDurationForVault` is to Update the duration of the reward for a specific reward token on a specific vault

However, any malicious user can DOS this function frontrunning it by calling `addIncentives` that update the `periodFinish` as follows

```
File: MultiRewards.sol329:         rewardData[_rewardsToken].lastUpdateTime = block.timestamp;330:@>       rewardData[_rewardsToken].periodFinish =331:             block.timestamp + rewardData[_rewardsToken].rewardsDuration;332:         emit RewardAdded(_rewardsToken, reward);
```

which will then make the update revert due to this check

```
File: MultiRewards.sol360:     function _setRewardsDuration(361:         address _rewardsToken,362:         uint256 _rewardsDuration363:     ) internal {364:         require(365:@>           block.timestamp > rewardData[_rewardsToken].periodFinish,366:             "Reward period still active"367:         );368:
```

### Impact Explanation

- DOS of governor `updateRewardDurationForVault`
- DOS depends on the `rewardsDuration` and can be repeated indefensibly as its very cheap

### Likelihood Explanation

- high as repeating attack is very cheap with high impact on the update DOSing the function forever

### Recommendation (optional)

- implement a good defense mechanism to prevent such attack as private mempool or when a governor wants to change the duration won't get affected by user inputs and overwrite it
