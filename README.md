accrueInterest Flow & Algorithmic Breakdown
A deep dive into how the vault measures elapsed time, calls the interest model, splits fees, mints staker rewards, and updates internal accounting state.

Core Responsibilities
• Measure elapsed time since last accrual and skip if zero.

• Invoke interestModel.calculateInterest with vault metrics.

• Compute local/global reserve fees and remaining staker interest.

• Mint to vault, update debt, timestamps, and borrow rate.

State Touchpoints
• totalPaidDebt: grows by full interest amount.

• accruedLocalReserves / accruedGlobalReserves: accumulate their respective fees.

• lastAccrue / lastBorrowRateMantissa: refreshed on success.

• cachedGlobalFeeBps: synced with factory fee for the vault.


Algorithmic Breakdown
Each numbered step corresponds to a logical chunk of the Solidity function, highlighting the invariants and accounting decisions.

Time-Window Check: Read block.timestamp and compare it to lastAccrue. If no time has passed, exit immediately to avoid unnecessary work.
Gas Budget Capture: Record gasleft() before calling the model. This value is used after the call to determine whether the transaction failed due to insufficient gas.
Interest Model Invocation: Pass totalPaidDebt, lastBorrowRateMantissa, timeElapsed, expRate, and free-debt ratios into interestModel.calculateInterest. The model returns the current borrow rate and interest accrued over the window.
Reserve Fee Extraction: Multiply interest by local/global fee BPS to compute localReserveFee and globalReserveFee. These are added to their respective accrued balances and subtracted from the interest pool before rewarding stakers.
Staker Reward Logic: Compare total staked coins against totalPaidDebt. If stakers are undercollateralized relative to debt, cap their reward to the borrow rate and funnel the residual interest into local reserves. Otherwise, distribute the entire post-fee interest to stakers.
State Refresh: Increase totalPaidDebt by the full interest amount, update lastAccrue and lastBorrowRateMantissa, and sync cachedGlobalFeeBps with the factory’s current fee.
Gas-Safety Fallback: If the model call reverts, check whether the caller provided at least INTEREST_CALCULATION_GAS_REQUIREMENT gas. If not, revert with a clear message; otherwise, propagate the underlying revert.
Key Accounting Formulas
Express the fee math and staker reward logic in a compact, equation-friendly form.

Reserve Fee Extraction

localReserveFee
=
⌊
interest
×
feeBps
10000
⌋
globalReserveFee
=
⌊
interest
×
cachedGlobalFeeBps
10000
⌋
localReserveFee=⌊interest× 
10000
feeBps
​
 ⌋
globalReserveFee=⌊interest× 
10000
cachedGlobalFeeBps
​
 ⌋
Staker Reward Allocation

interestAfterFees
=
interest
−
localReserveFee
−
globalReserveFee
stakerInterest
=
{
interestAfterFees
×
totalStaked
totalPaidDebt
if totalStaked
<
totalPaidDebt
interestAfterFees
otherwise
interestAfterFees=interest−localReserveFee−globalReserveFee
stakerInterest={ 
interestAfterFees× 
totalPaidDebt
totalStaked
​
 
interestAfterFees
​
  
if totalStaked<totalPaidDebt
otherwise
​
 
Debt & Rate Updates

totalPaidDebt
new
=
totalPaidDebt
old
+
interest
lastBorrowRateMantissa
=
currBorrowRate
totalPaidDebt 
new
​
 =totalPaidDebt 
old
​
 +interest
lastBorrowRateMantissa=currBorrowRate
Edge Cases & Invariants
Practical considerations that keep the accrual mechanism safe and predictable.

• Zero-Time: Returning early when timeElapsed == 0 prevents reentrancy or state churn on back-to-back calls.

• Debt vs. Staked Ratio: Comparing totalStaked with totalPaidDebt ensures stakers never receive more than the borrow rate when their collateral is thin.

• Reserve Fees: Local and global reserves grow only when interest is accrued, aligning protocol revenue with utilization.

• Gas Accountability: The gas-before call plus the explicit revert message prevents ambiguous failures when the caller skimps on gas.
