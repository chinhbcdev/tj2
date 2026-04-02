1
EA Market Improvement Strategy Overview
(Developer-Oriented Concept Summary)
This document describes the core improvement strategy behind the EA Market execution
framework before explaining Phase 1, Phase 2, and Phase 3.
The system design is based on the concept of “Immune System × Ecosystem”, where a
large population of EAs behaves like species within a dynamic market environment.
The goal is not to predict markets or find a single perfect EA, but to build a system that
can adapt and survive across changing market regimes.
1. Core Problem with Traditional EA Systems
Most traditional EA portfolio systems fail because they focus on:
• optimizing historical returns
• maximizing win rate
• selecting a small number of “best” strategies
However, financial markets are non-stationary, and regime changes are unpredictable.
Strategies that perform well in one environment may fail in another.
In practice, the majority of portfolio losses occur not because strategies lose occasionally, but
because losing strategies are not stopped early enough.
Therefore, the real problem is:
not finding winners, but eliminating destructive strategies while allowing surviving
strategies to continue.
2. System Design Philosophy
EA Market treats EAs as species in an ecosystem.
Each EA has:
2
• a native market habitat (its best regime)
• a survival history
• identifiable failure patterns
There is no universal EA that works in all environments.
Instead, different strategies survive under different market conditions.
The system therefore acts like an immune system, constantly monitoring and regulating the
EA population.
Its responsibilities include:
• detecting dangerous strategies
• temporarily isolating weak strategies
• allowing compatible strategies to dominate exposure
• allocating capital to stronger survivors
3. Market Environment Layer
Before any EA decisions are made, the system classifies the current market environment into
four simple regimes:
Trend Range
Low Volatility High Volatility
Example regimes:
• Trend × Low Volatility
• Trend × High Volatility
• Range × Low Volatility
• Range × High Volatility
The system does not attempt to predict future regimes.
It only labels the current environment.
This classification is used later to determine which EAs are better suited for the current
conditions.
3
4. Why Portfolio Performance Appears to Improve
A key concept in the system is the difference between:
• EA count-based success rate
• capital-weighted success rate
Example:
Assume there are 100 active EAs.
• 55 EAs are currently profitable
• 45 EAs are currently losing
The raw EA success rate is therefore:
55%
This situation typically occurs after Phase 2, where regime-compatible strategies are
prioritized.
However, Phase 3 introduces capital allocation control.
Instead of assigning equal capital to all EAs, the system:
• increases lot size for profitable EAs
• reduces lot size for losing EAs
As a result:
• the EA count success rate remains 55%
• but the majority of capital is allocated to winning strategies
This creates a capital-weighted success probability that exceeds 60%.
In other words:
the number of profitable EAs does not change,
but the system concentrates risk on strategies that are performing better.
4
5. Three-Layer Improvement Architecture
The EA Market system improves portfolio performance through three sequential layers:
Phase Purpose
Phase 1 Remove destructive EAs (risk elimination)
Phase 2 Increase exposure to regime-compatible EAs
Phase 3 Allocate more capital to stronger EAs
Each phase addresses a different problem:
• Phase 1 protects the system from catastrophic losses.
• Phase 2 aligns strategies with the current market environment.
• Phase 3 concentrates capital on the most productive strategies.
Together, these mechanisms transform the system from a static EA collection into a selfadaptive trading ecosystem.
6. Design Principle
The system does not attempt to predict market direction.
Instead, it follows a survival-oriented principle:
Strategies that remain healthy receive more exposure and capital, while fragile
strategies lose influence or are removed.
This survival-based architecture allows the system to adapt continuously as the market
environment evolves.
The following sections describe how this concept is implemented through Phase 1, Phase 2,
and Phase 3.
5
Phase 1 – EA Elimination (Hard Kill / Soft Kill)
Objective
The purpose of Phase 1 is to remove or isolate Expert Advisors (EAs) that exhibit persistent
negative behaviour in order to eliminate deep loss risk from the overall system.
This phase does not aim to maximise profit, but to protect the platform from destructive
strategies.
In a large-scale EA environment (e.g., 10K Demo Platform), it is statistically expected that a
significant portion of EAs will eventually fail due to:
• market regime changes
• overfitting or strategy decay
• structural weaknesses in the trading logic
If these failing EAs remain active, they can generate deep drawdowns and portfolio
instability.
Therefore, Phase 1 introduces a controlled elimination mechanism.
Concept
Each EA continuously receives a Health Score and performance metrics such as:
• drawdown level
• loss acceleration
• profit factor
• stability of returns
• abnormal execution behaviour
Based on these indicators, an EA can transition into one of three states:
1. Active
2. Soft Kill
3. Hard Kill
6
Soft Kill
Soft Kill is a temporary suspension.
Conditions that may trigger Soft Kill include:
• consistent negative returns
• declining performance trend
• temporary incompatibility with current market conditions
When an EA enters Soft Kill:
• trading is paused
• the EA remains monitored
• it may return to Active status if performance recovers
This prevents premature deletion of strategies that may recover under different market
regimes.
Hard Kill
Hard Kill is a permanent removal.
Conditions that trigger Hard Kill typically include:
• extreme drawdown (e.g., Loss > 70%)
• structural failure or abnormal behaviour
• persistent inability to recover
When Hard Kill is triggered:
• the EA is removed from the active trading pool
• the EA will not be reactivated automatically
• historical data is retained for research purposes
Expected Outcome
By continuously applying Hard Kill and Soft Kill rules, the system:
7
• removes strategies that create deep loss risk
• stabilises the EA population
• improves the overall survival rate of active strategies
This phase acts as the first layer of risk control before higher-level processes such as:
• Regime-aware selection (Phase 2)
• Capital allocation optimisation (Phase 3)
Implementation Notes
Developers should design this mechanism as an automated monitoring system that:
• evaluates EA performance periodically (e.g., weekly)
• assigns state transitions based on predefined thresholds
• logs all decisions for transparency and future analysis
The goal of Phase 1 is simple:
quickly discard strategies that are clearly harmful while preserving those that still have
potential.
8
Phase 2 – Regime-Aware Selection (Strategy Exposure Control)
Objective
The objective of Phase 2 is to increase the exposure of EAs that are more compatible with
the current market regime, while reducing the influence of EAs that tend to perform poorly
under the same conditions.
Unlike Phase 1, which removes clearly harmful strategies, Phase 2 focuses on adjusting the
relative importance of surviving EAs.
The goal is not to predict market direction, but to align the EA population with the current
market environment.
Concept
Financial markets operate under different structural conditions, commonly referred to as
market regimes.
Typical simplified regime classifications include:
• Trend Market
• Range Market
• High Volatility
• Low Volatility
Different trading strategies perform differently under these conditions.
For example:
• Trend-following EAs tend to perform well in trend regimes
• Mean-reversion or range strategies perform better in range regimes
• Certain strategies may degrade significantly during high volatility events
Therefore, instead of treating all EAs equally, the system should dynamically prioritise EAs
that historically survive better in the current regime.
Core Principle
9
Phase 2 does not attempt to predict the future market direction.
Instead, it evaluates the conditional survival probability of each EA under the current
market regime.
In other words:
The system does not ask “Where will the market go?”
It asks “Which strategies are least fragile in the current environment?”
Operational Mechanism
Each EA maintains a Regime Performance Profile, which records how the EA historically
behaves under different regimes.
Example:
EA Trend Range High Volatility Low Volatility
EA_A Strong Weak Medium Strong
EA_B Weak Strong Medium Medium
EA_C Medium Medium Strong Weak
When the system detects the current market regime, it adjusts EA exposure accordingly.
Exposure Adjustment
EAs are not immediately removed.
Instead, the system adjusts their relative exposure using a weighting mechanism.
Example (Trend regime detected):
EA Type Weight
Trend-compatible EA Increased exposure
Neutral EA Normal exposure
Range-dependent EA Reduced exposure
10
This approach ensures that:
• suitable EAs have greater influence
• unsuitable EAs remain available but with reduced impact
• the system remains robust if the regime changes
Expected Outcome
By dynamically increasing the exposure of regime-compatible strategies, the system can:
• improve the effective profitability ratio of the EA pool
• reduce the impact of regime-fragile strategies
• stabilise overall portfolio behaviour
This phase represents the second layer of optimisation, following Phase 1 risk elimination
and preceding Phase 3 capital allocation.
Implementation Notes
Developers should implement Phase 2 as a dynamic exposure management system that:
• classifies the current market regime using predefined indicators
• maintains regime-specific performance statistics for each EA
• periodically recalculates EA exposure weights (e.g., weekly)
• applies the exposure adjustments before execution allocation
Importantly, EAs are not permanently removed during this phase.
Instead, their relative influence is adjusted so the system can adapt quickly when market
conditions change.
11
Phase 3 – Capital Allocation (Dynamic Lot Allocation)
Objective
The objective of Phase 3 is to allocate more trading capital to EAs that demonstrate
stronger and more stable performance, while reducing the capital exposure to weaker or
deteriorating strategies.
Unlike Phase 1 (strategy elimination) and Phase 2 (regime-based prioritisation), Phase 3
focuses on how much capital each active EA should control.
The goal is to improve the overall portfolio efficiency by directing risk toward betterperforming strategies.
Concept
Even after removing harmful EAs (Phase 1) and prioritising regime-compatible strategies
(Phase 2), the remaining EAs will still have different levels of performance quality.
Some EAs may show:
• strong profitability
• stable drawdown behaviour
• consistent recovery patterns
Others may still be active but exhibit:
• lower profitability
• higher volatility
• unstable performance patterns
Instead of allocating equal capital to all EAs, the system dynamically adjusts lot sizes
according to each EA’s health and recent performance.
Core Principle
The system does not attempt to perfectly predict which EA will win next.
Instead, it applies a risk-weighting mechanism based on observed performance.
12
In simple terms:
Strategies that demonstrate stronger and more stable behaviour receive a larger share of
capital, while weaker strategies receive less exposure.
This approach improves the capital-weighted success rate of the portfolio, even if the
number of profitable EAs remains unchanged.
Health Score Integration
Each EA maintains a Health Score (0–100) derived from multiple factors such as:
• drawdown stability
• profitability metrics
• regime compatibility
• execution quality
The Health Score is then converted into a lot allocation multiplier.
Example:
Health Score Allocation Multiplier
80 – 100 1.5x
60 – 79 1.0x
40 – 59 0.6x
20 – 39 0.3x
< 20 0 (Soft Kill)
This mechanism ensures that:
• high-quality EAs receive more capital
• weaker strategies gradually lose influence
• total system risk remains controlled
Risk Normalisation
13
To maintain stable portfolio risk, the system must normalise total exposure after applying
allocation multipliers.
In practice:
1. Each EA receives a preliminary multiplier based on its Health Score.
2. The system calculates the total weighted exposure.
3. Lot sizes are scaled so that overall portfolio risk remains constant.
This prevents excessive leverage while still favouring stronger strategies.
Expected Outcome
By dynamically allocating capital to stronger EAs, the system can:
• increase the capital-weighted profitability of the portfolio
• reduce the impact of weaker strategies
• stabilise drawdown behaviour
• improve long-term capital growth
This phase represents the final optimisation layer, following:
• Phase 1: Risk elimination (removing destructive strategies)
• Phase 2: Regime-aware prioritisation (selecting suitable strategies)
Phase 3 ensures that capital is concentrated where it is most productive.
Implementation Notes
Developers should implement Phase 3 as a periodic allocation engine that:
• recalculates Health Scores for all active EAs
• converts scores into allocation multipliers
• normalises total portfolio exposure
• updates lot sizes before the next trading cycle (e.g., weekly SOD)
The allocation system should be smooth and conservative, avoiding abrupt changes in lot
size to maintain portfolio stability.
14