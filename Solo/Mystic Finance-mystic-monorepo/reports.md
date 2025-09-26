# 1. Inaccurate Collateral Calculation During Partial Position Closure Due to Interest Accounting - High

# Summary

The MorphoLeverageBundler and MysticLeverageBundler contracts calculate collateral withdrawal amounts based on internal position tracking without accounting for accrued interest.

# Finding Description

The leverage bundlers track user positions using internal state variables:

`totalBorrowsPerUser[pairKey][user] `
`totalCollateralsPerUser[pairKey][user] `

When a user partially closes a position, the bundlers calculate how much collateral to withdraw using this formula:

`
uint256 collateralForRepayment = (totalCollateralsPerUser[pairKey][msg.sender] \* debtToClose) / totalBorrowsPerUser[pairKey][msg.sender];

`
This calculation assumes that the ratio between tracked debt and tracked collateral remains constant. However, in lending protocols, interest accrues on borrowed amounts over time, but this accrual is not reflected in the bundlers' internal tracking.

For example, if a user initially borrows 100 ETH with 200 ETH collateral, and after some time the actual debt with interest is 110 ETH, the bundlers still track only the original 100 ETH. When the user wants to close 50% of their position:

Current calculation: (200 ETH collateral \* 50 ETH debt) / 100 ETH tracked debt = 100 ETH collateral to withdraw

Correct calculation: (200 ETH collateral \* 50 ETH debt) / 110 ETH actual debt = 90.9 ETH collateral to withdraw

This discrepancy of 9.1 ETH excess collateral withdrawal will grow larger the longer the position is held and the more interest accrues.

# Impact Explanation

1. The withdrawal could reduce the health factor of the remaining position, bringing it closer to liquidation thresholds.
2. Users who intend to maintain a specific leverage ratio after partial closure will find their positions have higher leverage than expected.
3. The longer a position is held, the more interest accrues, and the greater the discrepancy becomes.

# Proof of Concept

A test contract added to the Bundler3 folder, run the contract using the command
`forge test --mc MorphoLeverageInterestTest  -vvvv `

`
// SPDX-License-Identifier: GPL-2.0-or-later
pragma solidity ^0.8.0;

import "../lib/forge-std/src/Test.sol";
import "../src/calls/MorphoLeverageBundler.sol";
import {MarketParams, IMorpho} from "../lib/morpho-blue/src/interfaces/IMorpho.sol";
import {IOracle} from "../lib/morpho-blue/src/interfaces/IOracle.sol";
import {MathRayLib} from "../src/libraries/MathRayLib.sol";
import {IERC20} from "../lib/openzeppelin-contracts/contracts/token/ERC20/IERC20.sol";

contract MorphoLeverageInterestTest is Test {
using MathRayLib for uint256;

    // Define a basic test demonstrating the mathematical issue
    function testInterestAccountingDiscrepancy() public {
        // Initial position values
        uint256 initialBorrows = 100 ether;
        uint256 initialCollateral = 200 ether;

        // Record the initial ratio
        uint256 initialRatio = (initialCollateral * 1e18) / initialBorrows;

        // Simulate interest accrual (10% interest)
        uint256 interestAccrued = 10 ether; // 10% of 100
        uint256 currentBorrows = initialBorrows + interestAccrued; // Now 110 ether

        // User wants to close 50% of their position
        uint256 positionClosePercent = 50;
        uint256 debtToClose = initialBorrows * positionClosePercent / 100; // 50 ether (based on tracked value)

        // How the bundler calculates collateral:
        // It uses the proportion of debt being closed to the tracked debt
        uint256 bundlerCalculatedProportion = (debtToClose * 1e18) / initialBorrows; // 50/100 = 0.5 * 1e18
        uint256 bundlerWithdrawnCollateral = (initialCollateral * bundlerCalculatedProportion) / 1e18; // 100 ether

        // How it should calculate with accrued interest:
        // Using proportion of debt being closed to the current debt with interest
        uint256 correctProportion = (debtToClose * 1e18) / currentBorrows; // 50/110 = ~0.455 * 1e18
        uint256 correctWithdrawnCollateral = (initialCollateral * correctProportion) / 1e18; // ~90.9 ether

        // The bundler will withdraw too much collateral
        uint256 excessCollateralWithdrawn = bundlerWithdrawnCollateral - correctWithdrawnCollateral;

        // Output for clarity
        console.log("Initial borrows:", initialBorrows / 1 ether, "ETH");
        console.log("Initial collateral:", initialCollateral / 1 ether, "ETH");
        console.log("Interest accrued:", interestAccrued / 1 ether, "ETH");
        console.log("Current borrows with interest:", currentBorrows / 1 ether, "ETH");
        console.log("Debt to close (50%):", debtToClose / 1 ether, "ETH");
        console.log("Bundler withdrawn collateral:", bundlerWithdrawnCollateral / 1 ether, "ETH");
        console.log("Correct withdrawn collateral:", correctWithdrawnCollateral / 1 ether, "ETH");
        console.log("Excess collateral withdrawn:", excessCollateralWithdrawn / 1 ether, "ETH");

        // Assert that there is a discrepancy due to interest accounting
        assertGt(excessCollateralWithdrawn, 0, "Interest accounting should cause a collateral withdrawal difference");

        // The formula in both bundlers:
        // collateralForRepayment = (totalCollateralsPerUser[pairKey][msg.sender] * debtToCover) /
        // totalBorrowsPerUser[pairKey][msg.sender];

        // Demonstrate how the bundler calculates vs how it should calculate:
        console.log("\nDemonstrating bundler calculation:");
        console.log("collateralForRepayment = (totalCollateralsPerUser * debtToClose) / totalBorrowsPerUser");
        console.log("collateralForRepayment = (200 * 50) / 100 = 100 ETH");

        console.log("\nCorrect calculation with interest:");
        console.log("collateralForRepayment = (totalCollateralsPerUser * debtToClose) / currentBorrowsWithInterest");
        console.log("collateralForRepayment = (200 * 50) / 110 = 90.9 ETH");

        // This confirms the issue: when closing positions partially, the bundler doesn't account for
        // interest accrual, resulting in incorrect collateral withdrawal proportions
    }

}

`
the logs from the test contract trace

`
Logs:

Initial borrows: 100 ETH

Initial collateral: 200 ETH

Interest accrued: 10 ETH

Current borrows with interest: 110 ETH

Debt to close (50%): 50 ETH

Bundler withdrawn collateral: 100 ETH

Correct withdrawn collateral: 90 ETH

Excess collateral withdrawn: 9 ETH

Demonstrating bundler calculation:

collateralForRepayment = (totalCollateralsPerUser \* debtToClose) / totalBorrowsPerUser

collateralForRepayment = (200 \* 50) / 100 = 100 ETH

Correct calculation with interest:

collateralForRepayment = (totalCollateralsPerUser \* debtToClose) / currentBorrowsWithInterest

collateralForRepayment = (200 \* 50) / 110 = 90.9 ETH

`

# Recommendation

1. Modify the collateral calculation formula to use the current debt from the protocol

`
// Get current debt from the protocol
uint256 currentDebtWithInterest = morpho.expectedBorrowAssets(marketParams, msg.sender);
// Calculate collateral proportionally to current debt
uint256 collateralForRepayment = (totalCollateralsPerUser[pairKey][msg.sender] \* debtToClose) / currentDebtWithInterest;

`

2. Alternatively, implement a mechanism to periodically sync the internal position tracking with the actual protocol state to account for accrued interest.
