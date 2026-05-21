## 1 — Overview / High level
* Game is a **fund-management simulator** where the player manages a GP (general partner) running multiple strategies (funds), hires staff, handles random events, and tries to survive / achieve victory conditions.
* **Victory**: 15+ years of operation with 20%+ average annual returns.
* **Failure**: Insolvency (GP cash < 0), regulatory shutdown (compliance < 10%), fund liquidation (AUM < $1M), or investor exodus (LP satisfaction < 10%).
---
## 2 — Data model (constants & initial state)
### 2.1 Strategy definitions (`STRATEGIES`)
10 strategies available:
* `marketNeutral`: 8% return, 5% vol, $2B capacity, $50K cost
* `longShort`: 12% return, 12% vol, $4B capacity, $100K cost
* `globalMacro`: 15% return, 20% vol, $20B capacity, $200K cost
* `creditArb`: 10% return, 8% vol, $8B capacity, $150K cost
* `eventDriven`: 13% return, 10% vol, $4B capacity, $120K cost
* `quantMomentum`: 20% return, 25% vol, $4B capacity, $250K cost
* `options`: 14% return, 18% vol, $3.2B capacity, $180K cost
* `distressed`: 18% return, 15% vol, $6B capacity, $170K cost
* `activist`: 16% return, 14% vol, $3.2B capacity, $200K cost
* `crypto`: 25% return, 35% vol, $2B capacity, $220K cost

Each strategy has: `name`, `returnPA` (expected annual %), `volatility` (%), `capacity` (millions), `correlation` (beta), `cost` (annual fee per month = cost/12).

### 2.2 Hireable roles (`HIRES`)
Six hirable staff members (costs are annual):
* `cio`: Chief Investment Officer. Boosts all strategy returns by 50% (multiplicative). Cost: $500K/year.
* `cfo`: Chief Financial Officer. Cuts GP administrative costs by 50%. Cost: $350K/year.
* `coo`: Chief Operating Officer. Reduces monthly fund operating costs by 50%. Cost: $400K/year.
* `compliance`: Compliance Officer. Prevents compliance score decay and actively improves score (+2% per month). Cost: $200K/year.
* `fundraiser`: Investor Relations / Fundraising. Unlocks consistent fundraising, allowing higher capital inflows. Cost: $300K/year.
* `quantDev`: Quant Developer. Reduces portfolio volatility across all strategies by 35%. Cost: $450K/year.

### 2.3 Market regimes (`MARKET_REGIMES`)
Six market regimes with momentum, liquidity, volatility, fundraising, and weight (probability):
1. **Bull Market**: momentum 1.1, liquidity 1.2, volatility 0.7, fundraising 1.3, weight 1.0
2. **Bear Market**: momentum -0.4, liquidity -0.3, volatility 1.4, fundraising 0.7, weight 1.0
3. **Sideways Grind**: momentum 0.0, liquidity 0.3, volatility 1.0, fundraising 1.0, weight 1.0
4. **Volatility Spike**: momentum -0.1, liquidity 0.2, volatility 1.8, fundraising 0.6, weight 0.5
5. **Liquidity Crisis**: momentum -0.6, liquidity -0.5, volatility 1.6, fundraising 0.4, weight 0.5
6. **Recovery Phase**: momentum 0.8, liquidity 1.0, volatility 1.1, fundraising 1.2, weight 1.0

Market regime changes randomly at year-end (month 11 transition to month 0 of next year).

### 2.4 Random events (`RANDOM_EVENTS`)
Eight random events (triggered during months excluding year-end):
* **SEC Inspection**: -$200K impact, -15% compliance
* **GP Bank Account Hacked**: -$500K impact, -5% performance
* **Prime Broker Margin Call**: -$300K AUM impact, liquidity stress
* **LP Due Diligence Wave**: -$100K impact, time stress
* **Flash Crash**: -$1.5M AUM impact, -30% LP confidence
* **Whistleblower Complaint**: -$400K impact, -35% compliance
* **Trade Execution Error**: -$250K AUM impact, -3% performance
* **Favorable Press Coverage**: grants +6 months fundraising boost, +15% LP confidence

### 2.5 AUM boost events (`AUM_BOOST_EVENTS`)
Two positive capital inflow events (higher chance with fundraising boost):
* **Competitor Blowup**: +$5M AUM boost
* **Institutional Mandate Win**: +$10M AUM boost, +20% LP confidence

### 2.6 Initial `gameState` fields and defaults
* `month`: 0 (0..11, wraps to 0 at year-end)
* `year`: 1
* `gpCash`: $5,000,000
* `aum`: $50,000,000
* `performance`, `cumulativePerformance`, `annualPerformance`, `lastYearPerformance`
* `lpSatisfaction`: 0.8 (80%)
* `complianceScore`: 1.0 (100%)
* `activeFunds`: ['longShort'] (Long/Short Equity starts active)
* `team`: [] (no hired staff initially)
* `history`: [] (rolling 24-month history)
* `marketRegime`: MARKET_REGIMES[0] (Bull Market)
* `gameOver`: false, `victory`: false, `lastEvent`: null
* `managementFees`, `performanceFees`: cumulative totals
* `hwm`: $50,000,000 (high water mark)
* `fundraisingBoostMonths`: 0 (countdown for boost events)
* `lastEvents`: [] (last 2 event names for anti-repeat)
* `lastEventCost`: 0
* `yearStartAUM`: $50,000,000
* `accumulatedCapital`, `accumulatedEventCapital`, `accumulatedEventImpact`: inflow/impact accumulators (reset at year-end)
* `annualPerformance`: 0 (reset at year-end)
* `yearEndHistory`: [] (array of yearEndRecord objects)
* `collapseReason`, `collapseDetails`: failure state metadata

---
## 3 — UI / Interaction toggles (controls affecting mechanics)
* `allocationMethod`: Control with values `'capacity'`, `'volatility'`, `'returns'`. Affects how AUM is distributed across active funds (see §5.1).
* `selectedTab`: Navigation between 'overview', 'portfolio', 'operations', 'analytics' (UI only, no mechanical effect).
* `showHireMenu`, `showFundMenu`, `showEventModal`, `showYearEndSummary`, `showTrackRecord`: Modal display flags (UI only).
* `autoPaused`: Boolean. When false, `processTurn()` runs every 1000ms via `setInterval`. When true, game is paused.
---
## 4 — Core numeric helpers & display
* `formatCurrency(value)`: Converts numbers to human-readable format (e.g., $1.5B, $50.2M, $100K).
  * ≥ $1B: divide by 1e9, toFixed(2), append "B"
  * ≥ $1M: divide by 1e6, toFixed(2), append "M"
  * ≥ $1K: divide by 1e3, toFixed(0), append "K"
  * Else: toFixed(0)
* `getRoundedValue(value)`: Rounds values at K/M/B scales to nearest round number.
---
## 5 — Allocation & cost mechanics
### 5.1 AUM share calculation — `calculateAUMShare(fundKey, method = allocationMethod)`
Determines what fraction of total AUM is allocated to each active fund.

**Capacity method** (default):
* `totalCapacity = sum(strategy.capacity * 1,000,000 for each activeFund)`
* `aumShare = STRATEGIES[fundKey].capacity * 1,000,000 / totalCapacity`
* Allocates proportionally to fund capacity limits.

**Volatility method**:
* `totalInverseVol = sum(1 / STRATEGIES[fk].volatility for each activeFund)`
* `aumShare = (1 / STRATEGIES[fundKey].volatility) / totalInverseVol`
* Favors low-volatility strategies (more AUM to safer funds).

**Returns method**:
* `totalReturns = sum(STRATEGIES[fk].returnPA for each activeFund)`
* `aumShare = STRATEGIES[fundKey].returnPA / totalReturns`
* Favors high-return strategies (more AUM to growth-oriented funds).

**Edge case**: If `activeFunds.length === 0`, returns 0.

### 5.2 Fund operations cost — `calculateFundOperationsCost(method = allocationMethod)`
Per-month cost to run all active strategies.

For each active fund:
1. `baseCost = STRATEGIES[fundKey].cost / 12` (annual cost / 12 months)
2. `aumShare = calculateAUMShare(fundKey, method)`
3. `aumForFund = gameState.aum * aumShare`
4. `strategyCapacity = STRATEGIES[fundKey].capacity * 1,000,000`
5. `utilizationRatio = aumForFund / strategyCapacity`
6. **AUM scale multiplier**: `aumScaleMultiplier = 1 + utilizationRatio * 10`
   * Cost escalates rapidly as AUM approaches capacity (intended penalty for being over-capacity).
   * At 50% utilization: 1 + 0.5 * 10 = 6x cost
   * At 100% utilization: 1 + 1.0 * 10 = 11x cost
7. `costAfterScaling = baseCost * aumScaleMultiplier`
8. `finalCostBeforeDiscount = Math.max(baseCost, costAfterScaling)` (never lower than base cost)
9. **COO discount**: `cooDiscount = gameState.team.includes('coo') ? 0.5 : 1.0`
10. `finalCost = finalCostBeforeDiscount * cooDiscount`

Return sum of `finalCost` across all funds.

### 5.3 GP admin costs (base operations)
Base infrastructure cost to run the GP:

1. `baseAdminCost = 8,000 + gameState.activeFunds.length * 4,000`
   * Fixed $8K/month + $4K per active fund
2. **AUM scaling**:
   * `increments = Math.floor((gameState.aum - 50,000,000) / 50,000,000)`
   * `aumMultiplier = 1 + increments * 0.10` (±10% per ±$50M from $50M baseline)
   * At $50M AUM: 1.0x
   * At $100M AUM: 1.1x
   * At $200M AUM: 1.3x
   * Below $50M: multiplier decreases (e.g., $25M → 0.95x)
3. `adminCosts = baseAdminCost * aumMultiplier`
4. **CFO discount**: If `gameState.team.includes('cfo')`, `adminCosts = adminCosts * 0.5`

---
## 6 — Monthly performance mechanics (`calculateMonthlyPerformance`)
Calculates the portfolio's monthly return before fees, redemptions, or events.

### Step 1: Capacity totals & allocation
* `totalCapacity = sum(STRATEGIES[fk].capacity * 1,000,000 for each activeFund)`
* For each fund: `allocatedAUM = gameState.aum * calculateAUMShare(fundKey)`
* `totalAllocatedAUM = sum(allocatedAUM for each activeFund)` (should equal gameState.aum if no gap)
* `capacityUtilization = totalAllocatedAUM / totalCapacity`

### Step 2: Capacity penalty
Reduces portfolio return if over-capacity:
* If `capacityUtilization > 1.0`: `capacityPenalty = (capacityUtilization - 1.0) * 0.5`
  * At 1.5x capacity: 0.25 (25% return reduction)
* Else if `capacityUtilization > 0.8`: `capacityPenalty = (capacityUtilization - 0.8) * 0.2`
  * At 1.0x capacity: 0.04 (4% reduction)
* Else: `capacityPenalty = 0`

### Step 3: Per-strategy return calculation
For each active fund:
1. `baseReturn = STRATEGIES[fundKey].returnPA / 12 / 100` (convert annual % to monthly decimal)
   * E.g., 12% annual → 0.01 monthly
2. **CIO boost**: `cioBoost = gameState.team.includes('cio') ? 1.5 : 1.0`
   * CIO multiplies expected return by 1.5x (50% boost)
3. `expectedReturn = baseReturn * cioBoost`
4. **Random return with momentum skew**:
   * `randomFactor = Math.random()` (0 to 1)
   * `momentumSkew = gameState.marketRegime.momentum`
   * If `momentumSkew > 0`: `skewedRandom = Math.pow(randomFactor, 1 / (1 + momentumSkew * 0.5))`
     * Bull market biases toward higher returns
   * Else if `momentumSkew < 0`: `skewedRandom = 1 - Math.pow(1 - randomFactor, 1 / (1 + Math.abs(momentumSkew) * 0.5))`
     * Bear market biases toward lower returns
   * Else: `skewedRandom = randomFactor` (neutral)
5. **Volatility adjustment**:
   * `effectiveVolatility = gameState.team.includes('quantDev') ? STRATEGIES[fundKey].volatility * 0.65 : STRATEGIES[fundKey].volatility`
   * Quant Developer reduces volatility by 35%
6. `volatilitySwing = (skewedRandom - 0.5) * 2 * effectiveVolatility / 100 * gameState.marketRegime.volatility`
   * Variance proportional to strategy volatility and market volatility regime
7. **Liquidity consistency**:
   * `liquidityConsistency = clamp(0.5, 1.5, 1 + gameState.marketRegime.liquidity * 0.3)`
   * Divides volatility swing (dampens volatility in tight liquidity conditions)
8. `consistencyAdjustedReturn = expectedReturn + volatilitySwing / liquidityConsistency`
9. `strategyReturn = consistencyAdjustedReturn - capacityPenalty / 12`
   * Apply capacity penalty (amortized monthly)
10. **Cap return**: `cappedStrategyReturn = clamp(-0.30, 0.30, strategyReturn)`
    * No monthly return worse than -30%, better than +30%
11. `dollarReturn = allocatedAUM * cappedStrategyReturn`
12. Accumulate: `totalDollarReturn += dollarReturn`, `weightedVol += effectiveVolatility * aumShare`

### Step 4: Return aggregation
* `totalReturn = totalDollarReturn / gameState.aum` (if aum > 0, else 0)
* Return `{ return: totalReturn, volatility: weightedVol, capacityUtilization }`

---
## 7 — Monthly turn flow (`processTurn`)
Executes every 1000ms when game is not paused. Advances month-by-month, updates all state, checks game-over conditions.

### 7.1 Early exit
* If `gameState.gameOver === true`, return immediately (no processing).

### 7.2 Monthly performance
* `{ return: monthlyReturn, capacityUtilization } = calculateMonthlyPerformance()`
* `newAUM = gameState.aum * (1 + monthlyReturn)`

### 7.3 LP satisfaction changes
`lpSatChange` accumulates adjustments:
* **Monthly performance bands**:
  * monthlyReturn > 0.03 → +0.04
  * 0.01 < monthlyReturn ≤ 0.03 → +0.02
  * -0.02 ≤ monthlyReturn < -0.05 → -0.04
  * monthlyReturn < -0.05 → -0.08
* **Annual performance** (if `gameState.annualPerformance` is tracked):
  * > 0.15 → +0.03
  * 0.08 to 0.15 → +0.02
  * < -0.10 → -0.02
* **Last-year performance** (if `yearEndHistory.length > 0`):
  * `lastYearPerformance > 0.20` → +0.02
  * 0.10 to 0.20 → +0.01
  * < -0.05 → -0.01
* **Compliance score effect**:
  * `complianceScore > 0.8` → +0.01
  * `complianceScore < 0.3` → -0.02
* **Fundraiser hire**: +0.015
* **6-month momentum**:
  * If last 6 months in history: count positive months
  * ≥ 5 positive → +0.02
  * ≤ 1 positive → -0.02
* `currentLPSat = clamp(0, 1, gameState.lpSatisfaction + lpSatChange)`

### 7.4 Monthly costs
* `salaries = sum(HIRES[hire].cost / 12 for each hired staff)`
* `fundCosts = calculateFundOperationsCost()`
* `baseAdminCost = 8,000 + gameState.activeFunds.length * 4,000`
  * Scale: `aumMultiplier = 1 + Math.floor((gameState.aum - 50M) / 50M) * 0.10`
  * `adminCosts = baseAdminCost * aumMultiplier`
  * If CFO hired: `adminCosts *= 0.5`
* `totalCosts = salaries + fundCosts + adminCosts`

### 7.5 Compliance natural change & event filtering
* `complianceChange = gameState.team.includes('compliance') ? 0.02 : -0.005`
  * Compliance Officer adds +2% per month; without, natural decay is -0.5% per month
* `complianceAfterNaturalChange = clamp(0, 1, gameState.complianceScore + complianceChange)`
* Event filtering (§7.6) uses this projected compliance score to decide which events to allow.

### 7.6 Random events
**Event probability**:
* `totalMonthsPlayed = (gameState.year - 1) * 12 + gameState.month`
* If `totalMonthsPlayed < 12`: eventProbability = 0.08 (8%)
* Else: eventProbability = 0.2 (20%)
* **Only triggers** if NOT year-end month (month !== 11) AND `random < eventProbability`

**Event filtering**:
* Start with all `RANDOM_EVENTS`
* If `compliance` hire: remove all events with `complianceHit > 0`
* Else (no compliance hire):
  * If `complianceAfterNaturalChange ≤ 0.40`: filter out events with `complianceHit ≥ 0.35`
  * If `complianceAfterNaturalChange ≤ 0.20`: filter out events with `complianceHit ≥ 0.15`
* **Anti-repeat rules**:
  * If last event was 'Whistleblower Complaint': remove 'Whistleblower Complaint' from available
  * If last event was 'GP Bank Account Hacked': remove 'GP Bank Account Hacked' from available
  * If last 2 events were both 'SEC Inspection': remove 'SEC Inspection' from available
* **Selection**: If `availableEvents.length > 0`, pick one at random

### 7.7 AUM boost (positive capital injection) events
* `baseAumEventChance = 0.05` (5% baseline)
* `boostedAumEventChance = gameState.fundraisingBoostMonths > 0 ? 0.25 : baseAumEventChance`
  * Fundraising boost multiplies chance by 5x
* Only triggers if NOT year-end month AND `random < boostedAumEventChance`
* If triggered: pick one `AUM_BOOST_EVENTS` at random
* `aumBoostEvent.aumBoost` is added to capital inflow

### 7.8 Fundraising organic inflows
Capital raised from existing LPs (not from events):
* **Requires**: No capacity limit (`capacityUtilization < 1.0`) AND LP satisfaction thresholds
* If `fundraiser` hired:
  * If `currentLPSat > 0.6 && monthlyReturn > 0`:
    * `accumulatedCapitalThisMonth += gameState.aum * 0.025 * marketRegime.fundraising * aumScaleFactor`
    * 2.5% of AUM per month in good conditions
* Else (no fundraiser):
  * If `currentLPSat > 0.75 && monthlyReturn > 0.02`:
    * `accumulatedCapitalThisMonth += gameState.aum * 0.015 * marketRegime.fundraising * aumScaleFactor`
    * 1.5% of AUM per month (harder without fundraiser)
* **AUM scale factor** (decreasing returns at scale):
  * AUM < $100M: 1.0
  * AUM < $500M: 0.85
  * AUM < $1B: 0.70
  * AUM < $5B: 0.55
  * AUM ≥ $5B: 0.40

### 7.9 Compliance score update
* `finalCompliance = clamp(0, 1, complianceAfterNaturalChange - eventImpactCompliance)`
  * If event has compliance hit and no compliance officer, it reduces score
  * If compliance officer: event hits are ignored

### 7.10 Year-end accounting (triggered when `month === 11`)
**Only when transitioning from month 11 to month 0 of next year**:

1. `annualReturn = newAnnualPerformance = (1 + gameState.annualPerformance) * (1 + totalMonthReturn) - 1`
   * Compound monthly returns across the year
2. `annualHurdleRate = 0.08` (8%)
3. **Performance fee** (if annual return exceeds hurdle):
   * If `annualReturn > annualHurdleRate`:
     * `excessReturn = annualReturn - annualHurdleRate`
     * `performanceFee = gameState.yearStartAUM * excessReturn * 0.2`
     * (20% of excess return, applied to start-of-year AUM)
4. `finalAUM = finalAUM - performanceFee`
5. **Management fee**:
   * `managementFee = finalAUM * 0.02` (2% of AUM after performance fee)
   * `finalAUM = finalAUM - managementFee`
6. `aumBeforeRedemptions = finalAUM`
7. **Redemption logic**:
   * Determine `redemptionRate` based on performance and LP satisfaction:
     * `annualReturn < -0.2` → 25%
     * `-0.2 ≤ annualReturn < -0.1` → 12%
     * `-0.1 ≤ annualReturn < -0.05` → 4%
     * `-0.05 ≤ annualReturn < 0`:
       * If `currentLPSat < 0.3` → 10%
       * Else if `currentLPSat < 0.5` → 5%
       * Else → 0%
     * `annualReturn ≥ 0`:
       * If `currentLPSat < 0.3` → 3%
       * Else if `currentLPSat < 0.5` → 1%
       * Else → 0%
     * **Cap**: If `annualReturn ≥ 0` and `redemptionRate > 0.10`, cap at 10%
   * `redemptions = aumBeforeRedemptions * redemptionRate`
   * `finalAUM = aumBeforeRedemptions - redemptions`
8. **New capital deployment**:
   * `finalAUM = finalAUM + updatedAccumulatedCapital` (add organic + event inflows)
9. **GP cash (year-end)**:
   * `newGPCash = gameState.gpCash + managementFee + performanceFee - totalCosts`
   * (Fees flow to GP cash; costs subtracted)
10. **Create year-end record**:
    * Capture all metrics for display and victory/collapse logic
    * Reset accumulators: `accumulatedCapital = 0`, `accumulatedEventCapital = 0`, `accumulatedEventImpact = 0`
    * Reset annual performance: `annualPerformance = 0`
    * Set `yearStartAUM = finalAUM` for next year
    * Store in `yearEndHistory`

### 7.11 Event impacts (processed every month if event triggered)
* `eventImpactCash = event.impact || 0` (directly affects GP cash)
* `eventImpactAUM = event.aumImpact || 0` (directly affects AUM)
* `eventImpactLP = (event.lpConfidenceHit || 0) - (event.lpConfidenceBoost || 0)`
  * Negative = loss of confidence; positive = gain
* `eventImpactCompliance = gameState.team.includes('compliance') ? 0 : event.complianceHit || 0`
  * Compliance officer nullifies compliance hits
* If event grants fundraising boost: `fundraisingBoostMonths = 6`
* If AUM boost event: `lpConfidenceHit` is subtracted (double-dips LP satisfaction)

### 7.12 Accumulating metrics
* `accumulatedCapital += accumulatedCapitalThisMonth` (organic + event inflows)
* `accumulatedEventCapital += eventCapitalThisMonth` (event-driven only)
* `accumulatedEventImpact += negativeEventImpactThisMonth` (AUM impacts < 0)

### 7.13 History entry
* Append to `history`:
  ```
  {
    month: gameState.month + 1,      // 1..12
    aum: finalAUM,
    performance: totalMonthReturn * 100,
    gpCash: newGPCash + eventImpactCash,
    lpSat: newLPSat,
    compliance: finalCompliance
  }
  ```
* Keep only last 24 months: `history = history.slice(-24)`

### 7.14 Market regime change (year-end)
* If transitioning to new year (month 11 → 0), pick new regime based on weighted probabilities
* Iterate through `MARKET_REGIMES`, accumulate weights until random threshold crossed

### 7.15 Game-over & victory checks
* **Insolvency**: `newGPCash + eventImpactCash < 0` → `gameOver = true`
* **Regulatory shutdown**: `finalCompliance < 0.1` → `gameOver = true`
* **Fund liquidation**: `finalAUM < 1,000,000` → `gameOver = true` + determine `collapseReason`
* **Investor exodus**: `newLPSat < 0.1` → `gameOver = true`
* **Victory** (only at year-end):
  * `completedYears = yearEndHistory.length + 1`
  * If `completedYears >= 15`:
    * `avgReturn = average(yearEndRecord.annualReturn for all years)`
    * If `avgReturn >= 0.20` (20% p.a.) → `victory = true`, `gameOver = true`

### 7.16 Collapse reason (when AUM < $1M and year-end)
Determine why fund failed:
* `totalFees = managementFee + performanceFee`
* If `redemptions > aumBeforeRedemptions * 0.3` (>30% redeemed):
  * `collapseReason = "Mass Redemptions (X% of capital fled)"`
* Else if `totalFees > aumBeforeCapital * 0.15` (fees > 15% of AUM):
  * `collapseReason = "Excessive Fees (Y% drained fund)"`
* Else if `annualReturn < -0.2` (-20% loss):
  * `collapseReason = "Catastrophic Performance (Z% annual loss)"`
* Else:
  * `collapseReason = "AUM Below Minimum Threshold"`
* Populate `collapseDetails` with breakdown of contributing factors

### 7.17 Final state update
* Increment month: `month = (month + 1) % 12`
* Increment year if transitioning: `year = month === 0 ? year + 1 : year` (no, only happens when month was 11)
* Update all tracked fields in `gameState`
* Trigger UI modals if needed:
  * If event with display content → `showEventModal = true`
  * If year-end and no event → `showYearEndSummary = true`

---
## 8 — Failure & victory conditions
### 8.1 Automatic game-over triggers
Checked each `processTurn()`:
* **Insolvency**: `newGPCash + eventImpactCash < 0`
  * GP (general partner) cash insufficient to cover operations
* **Regulatory shutdown**: `finalCompliance < 0.1` (10%)
  * Compliance score fell below regulatory minimum
* **Fund liquidation**: `finalAUM < 1,000,000` ($1M)
  * AUM too small to sustain infrastructure costs
  * Collapse reason determined as per §7.16
* **Investor exodus**: `newLPSat < 0.1` (10%)
  * LP satisfaction collapsed; investors redeem all capital

### 8.2 Victory condition
* Requires: **15+ years** of operation (at year-end check)
* AND: **20% average annual return** across all completed years
* When both met: `victory = true`, `gameOver = true`
* Compared against legendary hedge fund track records (Jim Simons ~66% over 30 years, Druckenmiller ~30% over 30 years, etc.)

---
## 9 — Player actions (Hire & Launch)
### 9.1 Hiring staff (`hireStaff(hireKey)`)
**Preconditions**:
* Not already hired: `!gameState.team.includes(hireKey)`
* Sufficient GP cash: `gameState.gpCash ≥ HIRES[hireKey].cost`

**Effect**:
* `gameState.gpCash -= HIRES[hireKey].cost` (one-time upfront cost)
* `gameState.team.push(hireKey)` (persistent; immediately affects mechanics)
* Close hire menu

**Notes**: No delay; hire effects are applied immediately in next `processTurn()`.

### 9.2 Launching funds (`launchFund(fundKey)`)
**Preconditions**:
* Not already active: `!gameState.activeFunds.includes(fundKey)`
* Sufficient GP cash: `gameState.gpCash ≥ STRATEGIES[fundKey].cost * 12` (require 12 months' cost)

**Effect**:
* `gameState.gpCash -= STRATEGIES[fundKey].cost * 12`
  * (Note: code comment says `cost` is subtracted, not `cost * 12`; clarify: it's `fund.cost` which is annual, so possibly just monthly? Code shows `fund.cost * 12` check but `fund.cost` deduction. **Assume it's `fund.cost` deducted**.)
* `gameState.activeFunds.push(fundKey)` (immediately contributes to AUM allocation and costs)
* Close fund menu

**Notes**: Fund launch is permanent; you cannot unlaunch a fund.

---
## 10 — Year-end record structure (`yearEndRecord`)
Captured at year-end (month 11 transition) and stored in `yearEndHistory`:
* `yearStartAUM`: AUM at start of year
* `aumBeforeFees`: AUM before performance/management fees
* `aumBeforeCapital`: Same as `aumBeforeFees` (after performance but before redemptions/capital)
* `aumBeforeRedemptions`: AUM after fees, before redemptions
* `aumAfterCapital`: After new capital deployed
* `aumAfterFees`: Final AUM (after all fees)
* `capitalDeployed`: Total new capital added during year
* `managementFee`: 2% of final AUM
* `performanceFee`: 20% of excess return above 8% hurdle, applied to year-start AUM
* `redemptions`: Amount LPs redeemed
* `redemptionRate`: Percentage redeemed
* `eventCapital`: Capital from AUM boost events
* `organicCapital`: `capitalDeployed - eventCapital`
* `gpCashBefore`: GP cash at start of year-end month
* `gpCashAfter`: GP cash after fees
* `annualReturn`: Year's compound return
* `year`: Year number

---
## 11 — UI & presentation
* **Header stats**: AUM, GP cash, selected return metric (YTD/last year/cumulative), LP satisfaction, compliance, fund capacity, fund profile, fee structure, market regime, win condition progress.
* **Event modal**: Displays event name, impacts (cash, AUM, performance, compliance, LP satisfaction, fundraising boost).
* **Year-end summary**: Detailed breakdown of AUM changes, fees, redemptions, capital deployment, GP cash changes.
* **Track record**: List of all completed years with annual return, AUM growth, fees, redemptions.
* **Portfolio tab**: Active strategies with allocation, utilization, costs, breakdowns.
* **Operations tab**: GP cost breakdown, team members and their effects.
* **Analytics tab**: LP satisfaction and compliance score trends over time.

---
## 12 — Randomness & event rules
* **Random returns**: Each month, each strategy's return includes randomness skewed by market momentum (bullish/bearish).
* **Event triggering**: 8% chance in year 1, 20% after, subject to compliance filtering.
* **Anti-repeat**: Prevents consecutive repetition of certain high-impact events (Whistleblower, Bank Hack, triple SEC Inspection).
* **AUM boost chance**: 5% baseline, 25% with fundraising boost (6-month duration from "Favorable Press" event).
* **Market regime**: Weighted random selection at year-end.

---
## 13 — Time & tick behavior
* **Tick interval**: Every 1000ms when `autoPaused = false`
* **Month advancement**: `month = (month + 1) % 12`
* **Year advancement**: When `month` wraps from 11 to 0
* **Game continues** until `gameOver = true` (failure or victory)

---
## 14 — Edge cases & clamps
* `lpSatisfaction` clamped to [0, 1]
* `complianceScore` clamped to [0, 1]
* `cappedStrategyReturn` clamped to [-0.30, 0.30] (monthly)
* `liquidityConsistency` clamped to [0.5, 1.5]
* `calculateAUMShare` returns 0 if no active funds
* AUM never goes negative; clamped at 0 minimum
* GP cash can go negative (triggers insolvency)

---
## 15 — Reset & persistence
* **Reset**: `resetGame()` returns all fields to initial state, pauses autoplay, clears modals, clears history.
  * `gameState` fully re-initialized
  * All modals closed
  * `autoPaused = true`
* **Persistence**: Game state persists in React component memory for the session; no external storage (no localStorage shown in code).

---
## 16 — Derived metrics (UI display only)
* `cumulativePerformance`: Compound return from game start
  * `cumulativePerformance = ((1 + prev / 100) * (1 + monthlyReturn) - 1) * 100`
* `annualPerformance`: Compound return year-to-date, reset at year-end
* `lastYearPerformance`: Annual return of prior completed year (from `yearEndHistory`)
* **Weighted portfolio return**: AUM-weighted average of active fund expected returns (with CIO/QuantDev bonuses)
* **Weighted portfolio volatility**: AUM-weighted average of active fund volatilities (with QuantDev reduction)

---
## 17 — Implementation notes (as-coded)
* **Performance fee calculation**: Based on `yearStartAUM * excessReturn * 0.2` applied at year-end; management fee is 2% of AUM after performance fee deduction.
* **Hiring effects**: Persistent, apply immediately in next turn (e.g., CIO boosts all fund returns, CFO discounts admin costs, COO discounts fund ops costs, Compliance prevents compliance decay and filters events, Fundraiser increases capital inflow rates, QuantDev reduces volatility).
* **Cost escalation**: Fund operating costs scale dramatically with capacity utilization (multiplicand reaches 11x at 100% utilization), creating strong incentive to manage capacity carefully or expand fund capacity.
* **Fundraising dynamics**: Base 1.5% inflow per month without fundraiser (hard conditions); 2.5% with fundraiser (easier conditions). Boost events multiplied 5x when active. Scale factor discourages extreme AUM growth (returns diminish above $100M).
* **LP satisfaction**: Complex formula considering monthly, annual, and multi-year performance, compliance health, momentum, and fundraiser presence. Ensures players must balance returns, stability, and risk.
* **Compliance**: Naturally decays 0.5% monthly without Compliance Officer; improves 2% monthly with Officer. Events can deliver sudden -15% to -35% hits, so hiring Officer is critical risk mitigation if compliance score is low.
* **Market regimes**: Weighted random selection ensures some regimes are rarer (Volatility Spike, Liquidity Crisis each have weight 0.5 vs. weight 1.0 for others). Directly affect return skew (momentum), fee inflow (fundraising), and cost/return variance (volatility).
