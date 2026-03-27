EA Portfolio System – Implementation Proposal v2.0
0. Core Design Principle
This system is not designed to predict market direction.
Its purpose is to improve survival and capital efficiency of the EA population.
The architecture is based on three layers:
Phase 1: eliminate destructive strategies 
Phase 2: increase exposure to regime-compatible strategies 
Phase 3: allocate more capital to stronger strategies 
Key principle:
The system should kill only what is structurally harmful,
pause what is temporarily weak,
and allocate more capital to what is currently healthy.

I. Phase 1 – EA State Classification
(Active / Soft Kill / Hard Kill)
1. Execution Cycle
Soft Kill evaluation: weekly 
Hard Kill evaluation: every 3 to 4 weeks 
Evaluation target: 
Active EAs 
Soft Kill EAs 
unlabeled EAs 
This separation is important:
Soft Kill reacts to short-term deterioration 
Hard Kill should only be used for irreversible failure 

2. EA States
Active
EA is currently tradable 
Can be used in allocation logic 
Soft Kill
EA is temporarily paused 
EA is not removed 
EA may return if conditions improve 
Hard Kill
EA is permanently retired 
EA is excluded from future execution 
Historical records remain for research only 

3. Hard Kill Rules
(Only for irreversible failure)
HK1 – Extreme Lifetime Drawdown
Trigger Hard Kill if:
Max Drawdown (balance-based) ≥ 75% 
Parameter:
HK1_MAX_DD_PCT = 0.75 
This is a structural failure and should remain Hard Kill.

HK2 – Failed Recovery After Major Drawdown
Trigger Hard Kill if all conditions are met:
a drawdown ≥ 50% occurred 
let n = number of trades during the drawdown period 
after the drawdown, EA executes more than 2 × n trades 
recovery remains less than 50% of the drawdown amount 
Parameters:
HK2_MIN_DD_PCT = 0.50 
HK2_TRADE_COUNT_RATIO = 2 
HK2_MIN_DD_RECOVER_PCT = 0.50 
This represents an EA that is not merely weak, but unable to recover after sufficient time.

4. Soft Kill Rules
(Temporary weakness / regime mismatch)
SK1 – Moderate Current Drawdown
Trigger Soft Kill if:
Current drawdown ≥ 20% 
Parameter:
SK1_MAX_DD_PCT = 0.20 

SK2 – Negative Short-Term Equity Trend
Using the most recent 5 closed trades, fit a simple linear regression:
y=Ax+B
Trigger Soft Kill if:
slope A < 0 
and decline magnitude is material relative to original balance 
Parameter:
SK2_MIN_SLOPE_PCT = 0.02 

SK3 – Large Consecutive Losses
Previous proposal used this as Hard Kill.
In v2.0, this is changed to Soft Kill.
Trigger Soft Kill if:
consecutive losing trades produce drawdown ≥ 30% 
Parameter:
SK3_DD_MAX = 0.30 
Reason:
This may be caused by temporary regime mismatch and should not be treated as irreversible failure.

SK4 – Single Large Loss Trade
Previous proposal used this as Hard Kill.
In v2.0, this is changed to Soft Kill.
Trigger Soft Kill if:
one trade loss ≥ 10% of balance 
Parameter:
SK4_MAX_LOSS_PCT = 0.10 
Reason:
Single-trade shock may reflect market condition mismatch or abnormal volatility, not necessarily permanent structural failure.

5. Reactivation Rule for Soft Kill EA
A Soft Kill EA may return to Active only if:
current drawdown falls below Soft Kill threshold 
short-term equity slope becomes non-negative 
regime compatibility condition is acceptable 
no Hard Kill condition has been triggered 
This reactivation check is performed weekly.

II. Phase 2 – Regime-Aware Exposure Adjustment
1. Objective
Phase 2 does not try to predict the next regime.
Instead, it answers:
Under the current market environment, which EAs are relatively less fragile?
This phase adjusts relative exposure priority, not capital size.

2. Market Regime Definition
Each EA is linked to:
fixed symbol 
fixed timeframe 
Therefore regime classification must be symbol + timeframe specific.
We define the current regime using a recent rolling window:
default: last 1 week 
optional research window: 2 to 4 weeks 

3. Regime Axes
Trend Score
Use:
average ADX(14) over the time window 
TrendScore=Avg(ADX14)
Volatility Score
Use:
VolatilityScore=Avg(ATR(14)/ATR(100))

4. Regime Labels
Each weekly regime is classified into one of four states:
Trend × High Vol 
Trend × Low Vol 
Range × High Vol 
Range × Low Vol 
This should be stored per symbol + timeframe + week.

5. Weekly Regime Performance Storage
For each EA, per week, store:
TrendScore 
VolatilityScore 
RegimeLabel 
WEEKLY_PNL_PCT 
If no trades occurred:
WEEKLY_PNL_PCT = 0 

6. Regime Compatibility Score
For each EA, calculate compatibility with current regime based on historical weekly data.
Suggested concept:
compare average performance in current regime 
versus average performance across all regimes 
This produces a Regime Compatibility score, but it should be used only as an input to exposure ranking.

7. Important v2.0 Change
Remove “Regime Stability Check”
The previous proposal assumed:
next week’s regime is likely similar to last week
This assumption is not reliable enough and should not be part of the decision logic.
Therefore:
Do not gate exposure adjustment by regime persistence probability 
Always compute regime compatibility from current regime label only 

8. Output of Phase 2
Phase 2 should output:
Exposure Weight, not lot size 
Example:
Regime Compatibility
Exposure Weight
Strong
1.3
Neutral
1.0
Weak
0.7
Very Weak
0.4

This weight is passed into Phase 3.

III. Phase 3 – Health Score & Capital Allocation
1. Objective
Phase 3 determines:
How much capital should each active EA receive?
It uses:
Health Score 
Exposure Weight from Phase 2 
Final lot size is calculated only here.

2. Important v2.0 Change
Phase 2 and Phase 3 must remain separate
Phase 2 = logical exposure priority 
Phase 3 = capital / lot allocation 
These should not be merged into one black-box step.

3. Health Score Definition
Calculated weekly for:
Active EAs 
Soft Kill EAs under observation 
excluding Hard Kill EAs 
Range:
HealthScore∈[0,100]

4. Health Score Components
Metric
Weight
Profit Factor (PF)
25%
Recovery Factor (RF)
25%
Regime Compatibility (RC)
20%
Max Drawdown Safety
15%
Equity Smoothness (ES)
15%

Formula:
HealthScore=0.25PF+0.25RF+0.20RC+0.15MaxDD+0.15ES

5. Important v2.0 Change
Fixed Scoring Instead of Min-Max Normalization
The previous min-max normalization across all EAs is removed.
Reason:
unstable week to week 
sensitive to outliers 
hard to explain 
creates distorted scores 
Instead, use fixed score mapping functions.

6. Component Scoring Rules
PF Score
Based on:
last 2 to 4 weeks 
not just last 10 trades 
Suggested mapping:
PF
Score
≤ 1.0
30
1.2
50
1.5
70
2.0
85
≥ 3.0
100


RF Score
Recovery Factor:
RF=NetProfit/MaxDD
Suggested mapping:
RF
Score
≤ 0
20
0.5
40
1.0
60
1.5
80
≥ 2.0
100


RC Score
Derived from regime compatibility.
Suggested mapping:
Compatibility
Score
Very weak
20
Weak
40
Neutral
60
Good
80
Strong
100


MaxDD Safety Score
Use recent MaxDD.
Example mapping:
MaxDD
Score
≥ 40%
10
30%
30
20%
50
10%
80
0%
100


Equity Smoothness Score
Use R² on recent equity curve.
ES=100×R2
This is acceptable as proposed.

7. Lot Allocation Multiplier
Health Score is converted into a lot multiplier.
Example:
Health Score
Multiplier
80–100
1.5
60–79
1.0
40–59
0.6
20–39
0.3
< 20
0.0


8. Final Lot Formula
FinalLot=BaseLot×ExposureWeight×HealthMultiplier
Where:
BaseLot = default lot size 
ExposureWeight = Phase 2 output 
HealthMultiplier = Phase 3 output 

9. Risk Normalization
After calculating all final lots, apply normalization so that:
total portfolio exposure remains within allowed limit 
risk budget is not unintentionally increased 
This is mandatory.

IV. Additional Development
1. Centralized Market Data System
Build centralized system for:
5 symbols initially 
unified historical price storage 
regime analysis support 
simulation support 

2. Weekly EA Analytics Table
Store weekly EA metrics:
weekly pnl 
regime label 
drawdown 
PF 
RF 
R² 
soft/hard kill state 
health score 
exposure weight 
final lot multiplier 
This table will be the core of analysis and backtesting.

V. Answer to Q1
Difference between Phase 2 Exposure Adjustment and Phase 3 Lot Allocation
Phase 2
Determines:
Which EAs should be placed more in front under current market conditions?
This is a relative priority decision based on regime.
Phase 3
Determines:
How much capital should be allocated to each EA?
This is a capital allocation decision based on broader health.
Therefore:
Phase 2 should not be omitted 
Phase 3 should not absorb Phase 2 completely 
They solve different problems.

Example Scenario
Assume the market is currently Trend × High Vol.
EA_A
strong trend compatibility 
moderate PF 
stable DD 
Phase 2:
ExposureWeight = 1.3 
Phase 3:
HealthMultiplier = 1.0 
Final:
BaseLot × 1.3 × 1.0 

EA_B
weak trend compatibility 
high PF from earlier weeks 
acceptable DD 
Phase 2:
ExposureWeight = 0.7 
Phase 3:
HealthMultiplier = 1.2 
Final:
BaseLot × 0.7 × 1.2 
This means:
EA_B is still healthy 
but it is not well suited to the current regime 
so it receives less total capital than EA_A 
This is the correct behavior.

Final Recommendation
This v2.0 structure should be used as the implementation baseline because it:
preserves REA potential 
prevents excessive Hard Kill 
keeps Phase 2 and Phase 3 logically separate 
makes Health Score stable and explainable 
is consistent with the EM philosophy of survival and adaptation 
