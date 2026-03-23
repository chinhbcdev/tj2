I. Phase 1 – EA Classification (Active / Soft Kill / Hard Kill)
A batch job will be executed weekly (end of week).
For each EA that is currently:
* Active

* Soft Kill

* or not yet labeled

The system will analyze its closed trade history using the following rules:
a) HK1 – Extreme Drawdown
Evaluate overall capital decline:
   * Max Drawdown (based on balance) ≥ 75%

   * Parameter: HK1_MAX_DD_PCT = 0.75

This is evaluated over the entire trading history.
b) HK2 – Weak Recovery After Major Drawdown
Detect cases where an EA fails to recover after a large loss.
Conditions:
      * A drawdown ≥ 50% occurs
→ HK2_MIN_DD_PCT = 0.5

Definitions:
         * Let <n> = number of trades causing the drawdown

         * Let HK2_TRADE_COUNT_RATIO = 2

         * Let HK2_MIN_DD_RECOVER_PCT = 0.5
Rule:
After the drawdown:
            * If the EA executes more than 2 × n trades

            * AND recovery < 50% of the drawdown

→ classify as Hard Kill (HK2)
Example
               * Balance drops from $6000 → $2500 → drawdown = 58%

               * Number of trades during drawdown: n = 20

               * Afterward, EA executes 45 trades > 40 (2 × n)

               * Recovery: $2500 → $3500 → recovery = 29%

Since recovery < 50% → HK2 triggered
c) HK3 – Large Consecutive Losses
Detecting destructive losing streaks.
                  * Parameter: HK3_DD_MAX = 30%

Example:
                     * Initial balance: $5000

                     * 3 consecutive losing trades → total loss = $2500

                     * Drawdown = 50% > 30%

→ Hard Kill
d) HK4 – Single Large Loss Trade
Detect abnormal single trade risk.
                        * Parameter: HK4_MAX_LOSS_PCT = 10%

Example:
                           * Balance = $5000

                           * One trade loss = $600 → 12%

→ Hard Kill
e) SK1 – Moderate Drawdown
                              * Current drawdown ≥ 20%

                              * Parameter: SK1_MAX_DD_PCT = 0.2

→ Soft Kill
f) SK2 – Negative Equity Trend (Short-Term)
Use linear regression on the last 5 trades:
y = A*x + B


Condition:
                                 * A < 0

                                 * |A| / original_balance ≥ 0.02

→ Soft Kill
________________
II. Phase 2 – Market Regime Classification
________________


General Principles
                                    * Each EA operates on a fixed:

                                       * Symbol

                                       * Timeframe

→ Market classification must be symbol + timeframe specific
Time Window
                                          * Market regime is defined over a time window:
→ last N weeks

                                          * Assumption (to be validated):

The probability that next week’s regime is similar to last week is > 50%
Definition
“Current market regime” = regime within time window [t1, t2]
Indicator Handling
Technical indicators fluctuate per bar history.
→ For a time window: Use average value across all candles
a) Data Calculation & Storage
Trend Score
                                             * Compute average ADX(14) across all bars in the window:
                                             * TrendScore = Avg(ADX)




Volatility Score
                                             * Compute: VolatilityScore = Avg(ATR(14) / ATR(100))


Weekly Metrics
For each EA, per week, store data of weekly metrics:
                                             * <Trend Score>

                                             * <Volatility Score>

                                             * <WEEKLY_PNL_PCT>

If no trades → WEEKLY_PNL_PCT = 0
b) Exposure Adjustment (Next Week)
Before adjusting exposure, perform validation:
(1) Regime Stability Check
Check similarity of: (Trend Score, Volatility Score)
If similarity > 55% → considered stable
(2) Regime–Performance Correlation
Check correlation between: (Regime, WEEKLY_PNL_PCT)
Can be positive or negative correlation

Decision Logic
                                                * If BOTH conditions satisfied:
→ Exposure Adjustment is reasonable!

                                                * Otherwise:
→ Mark as “Exposure Adjustment is not reasonable”
________________


III. Phase 3 – Health Score & Capital Allocation
________________


Health Score Definition
Calculated weekly for all active EAs (excluding Hard Kill).
Range: HealthScore ∈ [0, 100]
Components & Weights


Metric Name
	Weight
	Parameter Name
	Normalized Profit Factor(PF)
	25%
	WEIGHT_PF
	Normalized Recovery Factor(RF)
	25%
	WEIGHT_RF
	Normalized Regime Compatibility(RC)
	20%
	WEIGHT_RC
	Normalized Max Drawdown(MaxDD)
	15%
	WEIGHT_MAX_DD
	Normalized Equity Smoothness(ES)
	15%
	WEIGHT_ES
	Health Score Formula
HealthScore = WEIGHT_PF × PF + WEIGHT_MAX_DD × MaxDD + WEIGHT_RF × RF + WEIGHT_ES × ES + WEIGHT_RC × RC
a) Normalized Profit Factor
                                                   * Based on last 10 trades (TRADE_COUNT_FOR_PF)
If total loss = 0 → score = 100
Else: PF = |TotalProfit / TotalLoss|
Normalize across all EAs:
Normalized PF = 100 × (PF - PF_min) / (PF_max - PF_min)
b) Normalized Recovery Factor
                                                   * Based on last 10 trades
RecoveryFactor = NetProfit / MaxDD
If MaxDD = 0 → score = 100
Normalize same as PF


c) Normalized Regime Compatibility
Definitions:
A = Avg PnL in current regime
B = Avg PnL across all regimes


Rules:
                                                   * If B < 0 and A > 0 → score = 100

                                                   * If B < 0 and A < 0:

                                                      * If |A| ≥ |B| → score = 0

                                                      * Else → score = 100 × (1 - A/B)

                                                         * If B > 0 and A < 0 → score = 0

                                                         * If B > 0 and A > 0 → RC = A / B

Then normalize across EAs.
d) Normalized Max Drawdown
                                                            * Based on last 10 trades

If MaxDD = 0 → score = 100
Else: SAFE% = 100 - MaxDD%
Normalize SAFE% → score
e) Normalized Equity Smoothness
                                                               * Based on last 10 trades
Compute: R² (coefficient of determination)
Then: Score = 100 × R²
IV. Additional Development
                                                               * Build a centralized system to collect historical price data (5 symbols)

                                                               * Store in a unified database

                                                               * System must support PnL simulation for applying phase 2, phase 3

V. Questions:
Q1: What is the difference between "Exposure Adjustment" in Phase 2 and "Lot allocation multiplier" in Phase 3?
In this proposed document, the data from Phase 2 will be conditionally used in the "Lot allocation multiplier" in Phase 3. In that case, can "Exposure Adjustment" in Phase 2 be omitted?
If both "Exposure Adjustment" in Phase 2 and "Lot allocation multiplier" in Phase 3 are applied simultaneously, could you please provide an illustrative example scenario?