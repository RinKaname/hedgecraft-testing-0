## 1 — Overview / High level
* Game is a **fund-management simulator** where the player manages a GP (general partner) running multiple strategies (funds), hires staff, handles events, and tries to survive / achieve victory conditions (15+ years with avg ≥20% p.a.).
---
## 2 — Data model (constants & initial state)
### 2.1 Strategy definitions (`STRATEGIES`)
Each strategy entry contains:
* `name` (string)
* `returnPA` — expected annual return in percent (number)
* `volatility` — volatility percent (number)
* `capacity` — capacity in millions (number)
* `correlation` (named `correlation` in UI; code uses `correlation` / `beta`)
* `cost` — launch / annual cost (number)
Concrete strategies and values (examples shown; complete set in file): Market Neutral (8% rpa, vol 5, capacity 2000 etc.), Long/Short (12%), Global Macro (15%), Quant Momentum (20%), Crypto (25%), etc.
### 2.2 Hireable roles (`HIRES`)
Each hire contains:
* `name`
* `cost` — annual cost (number)
* `description` — effect description
Roles and their effects (as text in `description`):
* `cio`: Boosts all strategy returns by 50% (multiplicative). Cost 500,000.
* `cfo`: Cuts GP administrative costs by 50%. Cost 350,000.
* `coo`: Reduces monthly fund operating costs by 50%. Cost 400,000.
* `compliance`: Prevents compliance decay and actively improves score over time. Cost 200,000.
* `fundraiser` (Investor Relations): Unlocks consistent fundraising, doubling inflow potential. Cost 300,000.
* `quantDev`: Reduces portfolio volatility across all strategies by 35%. Cost 450,000.
### 2.3 Market regimes (`MARKET_REGIMES`)
Each regime has:
* `name`, `momentum`, `liquidity`, `volatility`, `fundraising`, `weight`.
Defined regimes: Bull Market (momentum 1.1, liquidity 1.2, volatility 0.7, fundraising 1.3, weight 1.0), Bear Market (momentum -0.4, liquidity -0.3, volatility 1.4, fundraising 0.7), Sideways Grind, Volatility Spike, Liquidity Crisis, Recovery Phase. Regime selection occurs annually with weighted random selection.
### 2.4 Random events (`RANDOM_EVENTS`) and AUM boost events (`AUM_BOOST_EVENTS`)
* Random events include SEC Inspection, GP Bank Account Hacked, Prime Broker Margin Call, LP Due Diligence Wave, Flash Crash, Whistleblower Complaint, Trade Execution Error, Favorable Press Coverage (positive). Each event has typed fields such as `impact`, `aumImpact`, `performanceHit`, `complianceHit`, `lpConfidenceHit`, `grantsFundraisingBoost`, etc.
* AUM boost events: Competitor Blowup (aumBoost 5,000,000), Institutional Mandate Win (aumBoost 10,000,000, lpConfidenceBoost 0.2).
### 2.5 Initial `gameState` fields and defaults
Key state fields and defaults:
* `month`: 0 (0..11)
* `year`: 1
* `gpCash`: 5,000,000
* `aum`: 50,000,000
* `performance`, `cumulativePerformance`, `annualPerformance`, `lastYearPerformance` etc.
* `lpSatisfaction`: 0.8
* `complianceScore`: 1.0 (100%)
* `activeFunds`: ['longShort'] (default active strategy)
* `team`: [] (hired staff keys)
* `history`: [] (recent history entries)
* `marketRegime`: MARKET_REGIMES[0] (initial regime)
* `gameOver`: false, `victory`: false, `lastEvent`: null
* `managementFees`, `performanceFees`, `hwm`: 50,000,000
* `fundraisingBoostMonths`, `lastEvents`, `lastEventCost`
* `yearStartAUM`: 50,000,000
* Accumulators: `accumulatedCapital`, `accumulatedEventCapital`, `accumulatedEventImpact`
* `yearEndHistory`: [], `collapseReason`, `collapseDetails`.
---
## 3 — UI / Interaction toggles (controls affecting mechanics)
* `allocationMethod` control with values: `'capacity'`, `'volatility'`, `'returns'`. Each changes AUM allocation logic: capacity → allocate proportional to `strategy.capacity`; volatility → allocate more to lower volatility (1/vol weighting); returns → allocate to higher expected return.
* Menus & UI flags used to display modal dialogs: `showHireMenu`, `showFundMenu`, `showEventModal`, `showYearEndSummary`, `showTrackRecord`. These do not change core rules but control when the player sees events/year summaries.
* `autoPaused` controls the automatic per-second tick; when false, `processTurn()` executes every 1000ms via `setInterval`.
---
## 4 — Core numeric helpers & display
* `formatCurrency(value)`: converts raw numbers into human readable strings (B, M, K) with `toFixed` formatting.
* `getRoundedValue(value)`: rounds values at K/M/B scales (specific thresholds 1e3, 1e6, 1e9).
---
## 5 — Allocation & cost mechanics
### 5.1 AUM share calculation — `calculateAUMShare(fundKey, method = allocationMethod)`
* If `method === 'capacity'`:
* `totalCapacity = sum(strategy.capacity * 1,000,000 for activeFunds)`
* `aumShare = strategy.capacity * 1,000,000 / totalCapacity` (if totalCapacity > 0)
* If `method === 'volatility'`:
* `totalInverseVol = sum(1 / strategy.volatility for activeFunds)`
* `aumShare = (1 / strategy.volatility) / totalInverseVol`
* If `method === 'returns'`:
* `totalReturns = sum(strategy.returnPA for activeFunds)`
* `aumShare = strategy.returnPA / totalReturns`
* Returns 0 if `activeFunds.length === 0`.
### 5.2 Fund operations cost — `calculateFundOperationsCost(method = allocationMethod)`
* For each `activeFund`:
* `baseCost = strategy.cost / 12` (monthly base)
* `aumShare = calculateAUMShare(fundKey, method)`
* `aumForFund = gameState.aum * aumShare`
* `strategyCapacity = strategy.capacity * 1,000,000`
* `utilizationRatio = aumForFund / strategyCapacity`
* `aumScaleMultiplier = 1 + utilizationRatio * 10`
* `costAfterScaling = baseCost * aumScaleMultiplier`
* `finalCostBeforeDiscount = Math.max(baseCost, costAfterScaling)`
* `cooDiscount = gameState.team.includes('coo') ? 0.5 : 1.0`
* `finalCost = finalCostBeforeDiscount * cooDiscount`
* Sum `finalCost` across funds and return.
### 5.3 GP admin costs (base admin)
* Base admin cost = 8,000 + (`activeFunds.length` × 4,000).
* AUM scaling: `increments = floor((gameState.aum - 50,000,000) / 50,000,000)`
* `aumMultiplier = 1 + increments * 0.10` (±10% per ±$50M from $50M baseline).
* `adminCosts = baseAdminCost * aumMultiplier`.
* If `cfo` hired → `adminCosts = adminCosts * 0.5`.
---
## 6 — Monthly performance mechanics (`calculateMonthlyPerformance`)
Flow and formulas:
1. **Capacity totals & allocation**
* `totalCapacity = sum(strategy.capacity * 1,000,000 for activeFunds)`
* `totalAllocatedAUM = sum(gameState.aum * calculateAUMShare(fundKey) for each activeFund)`
* `capacityUtilization = totalAllocatedAUM / totalCapacity`
2. **Capacity penalty**
* If `capacityUtilization > 1.0`: `capacityPenalty = (capacityUtilization - 1.0) * 0.5`
* Else if `capacityUtilization > 0.8`: `capacityPenalty = (capacityUtilization - 0.8) * 0.2`
* Else `capacityPenalty = 0`
3. **Per-strategy return calculation**
For each active fund:
* `baseReturn = strategy.returnPA / 12 / 100` (monthly decimal)
* `cioBoost = gameState.team.includes('cio') ? 1.5 : 1.0` (multiplicative to baseReturn).
* `expectedReturn = baseReturn * cioBoost`
* `momentumSkew = gameState.marketRegime.momentum`
* `randomFactor = Math.random()`
* `skewedRandom` computed as:
* If `momentumSkew > 0`: `Math.pow(randomFactor, 1 / (1 + momentumSkew * 0.5))`
* Else if `momentumSkew < 0`: `1 - Math.pow(1 - randomFactor, 1 / (1 + Math.abs(momentumSkew) * 0.5))`
* Else `randomFactor`
* `effectiveVolatility = gameState.team.includes('quantDev') ? strategy.volatility * 0.65 : strategy.volatility`
* `volatilitySwing = (skewedRandom - 0.5) * 2 * effectiveVolatility / 100 * gameState.marketRegime.volatility`
* `liquidityConsistency = clamp(0.5, 1.5, 1 + gameState.marketRegime.liquidity * 0.3)`
* `consistencyAdjustedReturn = expectedReturn + volatilitySwing / liquidityConsistency`
* `strategyReturn = consistencyAdjustedReturn - capacityPenalty / 12`
* `cappedStrategyReturn = clamp(-0.30, 0.30, strategyReturn)` (limits -30% to +30% monthly)
* `dollarReturn = allocatedAUM * cappedStrategyReturn`
* Add dollarReturn to totalDollarReturn and accumulate `weightedVol += effectiveVolatility * aumShare`.
4. **Result**
* `totalReturn = totalDollarReturn / gameState.aum` (if aum > 0)
* Function returns `{ return: totalReturn, volatility: weightedVol, capacityUtilization }`.
---
## 7 — Monthly turn flow (`processTurn`)
Top-level flow (guard rails included):
1. **Early exit**
* If `gameState.gameOver` → return (no processing).
2. **Calculate monthly performance**
* Call `calculateMonthlyPerformance()` → receives `monthlyReturn` (decimal) and `capacityUtilization`.
* `newAUM = gameState.aum * (1 + monthlyReturn)`.
3. **LP satisfaction changes**
* `lpSatChange` starts 0; adjusted by monthlyReturn bands:
* monthlyReturn > 0.03 → +0.04
* monthlyReturn > 0.01 → +0.02
* monthlyReturn < -0.05 → -0.08
* monthlyReturn < -0.02 → -0.04
* Annual & last-year performance adjustments:
* `gameState.annualPerformance > 0.15` → +0.03
* `> 0.08` → +0.02
* `< -0.10` → -0.02
* If `yearEndHistory` exists: `lastYearPerformance > 0.20` → +0.02; `> 0.10` → +0.01; `< -0.05` → -0.01
* Compliance effect:
* `complianceScore > 0.8` → +0.01; `< 0.3` → -0.02
* `fundraiser` hire effect: +0.015
* Momentum of last 6 months: if ≥5 positive months → +0.02; if ≤1 positive → -0.02
* New `currentLPSat = clamp(0,1, gameState.lpSatisfaction + lpSatChange)`.
4. **Costs**
* `salaries = sum(HIRES[hire].cost / 12 for hires in gameState.team)`
* `fundCosts = calculateFundOperationsCost()`
* `baseAdminCost = 8000 + gameState.activeFunds.length * 4000` and then scaled by `aumMultiplier` (see §5.3). `cfo` halves admin costs if present.
* `totalCosts = salaries + finalFundCosts + adminCosts`.
5. **Events**
* Event selection probability:
* `eventProbability = totalMonthsPlayed < 12 ? 0.08 : 0.2` (higher after 1 year).
* If not year-end month and random < eventProbability → select an event from `RANDOM_EVENTS` subject to filtering.
* Filtering logic:
* If `compliance` hire exists, availableEvents filtered to remove events with `complianceHit`.
* Else (no compliance staff), if `complianceAfterNaturalChange <= 0.40` → filter events with `complianceHit >= 0.35` (or remove those with high complianceHit). If `<=0.20` → further filter to `complianceHit < 0.15`.
* Avoid repeating `Whistleblower Complaint` or `GP Bank Account Hacked` if they were the most recent event(s). Prevent back-to-back `SEC Inspection` twice in a row.
6. **AUM boost (positive) injection mechanics**
* `baseAumEventChance = 0.05`.
* `boostedAumEventChance = gameState.fundraisingBoostMonths > 0 ? 0.25 : baseAumEventChance` (fundraising boost increases chance).
* If random < boostedAumEventChance and not year-end month → pick `AUM_BOOST_EVENTS` and add `aumBoost` to accumulated capital inflow and `eventCapitalThisMonth`.
7. **Fundraising organic inflows**
* If `hasFundraiser`:
* If `currentLPSat > 0.6 && monthlyReturn > 0` → `accumulatedCapitalThisMonth += gameState.aum * 0.025 * totalFundraisingMultiplier * aumScaleFactor`
* Else:
* If `currentLPSat > 0.75 && monthlyReturn > 0.02` → `accumulatedCapitalThisMonth += gameState.aum * 0.015 * totalFundraisingMultiplier * aumScaleFactor`
* `totalFundraisingMultiplier = gameState.marketRegime.fundraising` (regime-based multiplier).
* `aumScaleFactor` depends on `gameState.aum` tiers:
* `< 100M → 1.0`, `< 500M → 0.85`, `< 1B → 0.70`, `< 5B → 0.55`, else 0.40.
8. **Fees, redemptions, year-end accounting (when `month === 11`)**
* `isEnteringNewYear = gameState.month === 11`
* At year-end:
* `annualReturn = newAnnualPerformance` where `newAnnualPerformance = (1 + gameState.annualPerformance) * (1 + totalMonthReturn) - 1`
* `annualHurdleRate = 0.08`
* If `annualReturn > annualHurdleRate`:
* `excessReturn = annualReturn - annualHurdleRate`
* `performanceFee = gameState.yearStartAUM * excessReturn * 0.2` (note: performance fee calculated on yearStartAUM * excessReturn * 0.2)
* `finalAUM = finalAUM - performanceFee` then `managementFee = finalAUM * 0.02` and `finalAUM = finalAUM - managementFee`. (Management fee set to 2% of finalAUM after subtracting performance fee).
* **Redemptions logic (year-end)**
* Set `redemptionRate` based on `annualReturn` and `currentLPSat`:
* If `annualReturn < -0.2` → redemptionRate = 0.25
* Else if `< -0.1` → 0.12
* Else if `< -0.05` → 0.04
* Else if `< 0 && currentLPSat < 0.3` → 0.10
* Else if `< 0 && currentLPSat < 0.5` → 0.05
* Else if `>= 0 && currentLPSat < 0.3` → 0.03
* Else if `>= 0 && currentLPSat < 0.5` → 0.01
* Cap: if `annualReturn >= 0` and redemptionRate > 0.10 → set redemptionRate = 0.10.
* `redemptions = aumBeforeRedemptions * redemptionRate`
* `finalAUM = aumBeforeRedemptions - redemptions` then `finalAUM += updatedAccumulatedCapital`.
9. **GP cash after year-end**
* `newGPCash = gameState.gpCash + managementFee + performanceFee - totalCosts` (for year-end branch). Otherwise `newGPCash = gameState.gpCash - totalCosts`.
10. **Event impacts during month**
* If `event` chosen:
* `eventImpactCash = event.impact || 0` (negative impact reduces gpCash)
* `eventImpactAUM = event.aumImpact || 0` (affects AUM directly)
* `eventImpactPerf = event.performanceHit || 0` (reduces performance percent)
* `eventImpactLP = (event.lpConfidenceHit || 0) - (event.lpConfidenceBoost || 0)`
* `eventImpactCompliance = gameState.team.includes('compliance') ? 0 : event.complianceHit || 0` (if compliance hire present, complianceHit ignored)
* `newEventCost = eventImpactCash < 0 ? Math.abs(eventImpactCash) : 0`
* If event grants fundraising boost: `newFundraisingBoostMonths = 6`
* If `aumBoostEvent` occurred: it subtracts `lpConfidenceBoost` from `eventImpactLP` and `eventCapitalInflow = aumBoostEvent.aumBoost`.
11. **Accumulators**
* `accumulatedCapital` and `accumulatedEventCapital` are incremented by organic and event-driven inflows respectively; `accumulatedEventImpact` aggregates negative event AUM impacts. These are used in year-end record and finalAUM calculations.
12. **History entry** (month snapshot)
* `historyEntry` includes `month` (1..12), `aum`, `performance` (totalMonthReturn * 100), `gpCash`, `lpSat`, `compliance`. History is sliced to last 24 months when appended.
13. **Modal triggers**
* If event with display content → `setShowEventModal(true)` (display event modal)
* If entering new year and not displayEvent → `setShowYearEndSummary(true)` (show year-end summary modal).
---
## 8 — Failure & victory / Game over conditions
### 8.1 Automatic game-over triggers (checked every `processTurn`)
* If `newGPCash + eventImpactCash < 0` → `gameOver = true` (insolvency).
* Else if `finalCompliance < 0.1` (i.e., compliance score < 10%) → `gameOver = true` (regulatory shutdown).
* Else if `finalAUM < 1,000,000` → `gameOver = true` (fund liquidated). Additional collapseReason logic provided (see below).
* Else if `newLPSat < 0.1` → `gameOver = true` (investor exodus due to LP satisfaction collapse).
### 8.2 Collapse reason determination (when AUM < $1M and at year-end)
If `isEnteringNewYear && yearEndRecord`:
* Calculate `totalFees = yearEndRecord.managementFee + yearEndRecord.performanceFee` and `aumBeforeRedemptions = yearEndRecord.aumBeforeRedemptions`
* If `yearEndRecord.redemptions > aumBeforeRedemptions * 0.3` → `collapseReason = "Mass Redemptions (X% of capital fled)"`
* Else if `totalFees > yearEndRecord.aumBeforeCapital * 0.15` → `collapseReason = "Excessive Fees (Y% drained fund)"`
* Else if `yearEndRecord.annualReturn < -0.2` → `collapseReason = "Catastrophic Performance (Z% annual loss)"`
* Else `collapseReason = 'AUM Below Minimum Threshold'`
* Compose `collapseDetails` with `startingAUM`, `performanceImpact`, `eventImpact`, `managementFee`, `performanceFee`, `redemptions`, `newCapital`, `finalAUM`.
### 8.3 Victory condition
* When `isEnteringNewYear && yearEndRecord`, compute `completedYears = gameState.yearEndHistory.length + 1`.
* If `completedYears >= 15`:
* `allYearRecords = [...gameState.yearEndHistory, yearEndRecord]`
* `avgReturn = average(r.annualReturn across allYearRecords)`
* If `avgReturn >= 0.20` (20% p.a. average) → `victory = true`, `gameOver = true`.
---
## 9 — Hire & Launch actions (player actions & requirements)
### 9.1 Hiring (`hireStaff(hireKey)`)
* Preconditions:
* If `gameState.team.includes(hireKey)` → return (already hired).
* If `gameState.gpCash < HIRES[hireKey].cost` → return (insufficient GP cash).
* Effect:
* `gameState.gpCash -= hire.cost`
* `gameState.team.push(hireKey)`
* Close hire menu.
### 9.2 Launch fund (`launchFund(fundKey)`)
* Preconditions:
* If `gameState.activeFunds.includes(fundKey)` → return.
* If `gameState.gpCash < fund.cost * 12` → return (require one year’s cost up-front).
* Effect:
* `gameState.gpCash -= fund.cost`
* `gameState.activeFunds.push(fundKey)`
* Close fund menu.
---
## 10 — Year-end record structure (`yearEndRecord`)
At year-end, `yearEndRecord` contains:
* `yearStartAUM`
* `aumBeforeFees`
* `aumBeforeCapital` (same as aumBeforeFees per code)
* `aumBeforeRedemptions`
* `capitalDeployed` (`updatedAccumulatedCapital`)
* `aumAfterCapital` (aumBeforeRedemptions + updatedAccumulatedCapital)
* `aumAfterFees` (finalAUM)
* `managementFee`, `performanceFee`, `redemptions`, `redemptionRate`
* `eventCapital` (`updatedAccumulatedEventCapital`)
* `organicCapital` = `updatedAccumulatedCapital - updatedAccumulatedEventCapital`
* `gpCashBefore`, `gpCashAfter`
* `annualReturn`, `year`
This object is stored in `yearEndHistory` and also used in the year-end UI summary.
---
## 11 — UI presentation rules tied to mechanics
* `showEventModal` shows event details including GP cash impact, AUM impact, performance hit, compliance hit, LP satisfaction boost, fundraising boost message, and uses `lastEvent` fields as displayed.
* Track record UI displays `yearEndHistory` and highlights management fee, performance fee, redemptions, GP cash changes, and other numeric summaries from `yearEndRecord`.
* Active strategy cards show expected return modified by `cio` (if hired), volatility modified by `quantDev`, capacity, AUM allocated, utilization percent, base monthly cost, AUM scale multiplier and final cost (with COO discount applied visually and numerically). Colors and thresholds in UI reflect utilization bands (>100% red, >80% yellow).
* GP Operations view details base cost, active funds addition, AUM scaling, and CFO discount effect.
---
## 12 — Randomness, filtering and anti-repeat rules
* Events chosen by random selection subject to filters (compliance hire removes compliance-hit events; low compliance makes some events more/less likely by filtering out events with certain `complianceHit` thresholds).
* Prevents immediate repeat of some events: if lastEvents[0] is 'Whistleblower Complaint' then availableEvents filter removes the same event; similar for `GP Bank Account Hacked`. Also blocks triple `SEC Inspection` if seen twice in a row.
---
## 13 — Time & tick behaviour
* Game advances a month per `processTurn()` call. `useEffect` runs `setInterval(processTurn, 1000)` while `autoPaused` is false and `gameState.gameOver` is false. Each interval increments `month` (wraps at 12) and, on `month === 11`, performs year-end logic and increments `year`.
---
## 14 — Edge cases and clamps
* Many values are clamped:
* `complianceScore` and `lpSatisfaction` are clamped in [0,1].
* `cappedStrategyReturn` is clamped to [-0.30, 0.30] monthly.
* `calculateAUMShare` returns 0 if no activeFunds.
---
## 15 — Persistence & reset
* `resetGame()` fully resets the `gameState` to initial defaults (same as initial state above), pauses auto-play, clears modals and history, and reinitializes variables like gpCash=5M and aum=50M.
---
## 16 — Derived metrics shown in UI (for reference)
* `formatCurrency` used to show AUM, GP cash, costs, and other formatted numeric fields.
* `performance` fields often expressed as percentages ×100 in the UI. Year-end summary shows management fee 2% and performance fee 20% explicitly.
---
## 17 — Implementation notes (as-coded business rules — for engineers)
* Performance fee calculation is based on `yearStartAUM * excessReturn * 0.2` if annualReturn > 0.08; management fee calculated as 2% of finalAUM after performance fee subtraction. (This is the explicit order implemented in code.)
* Several multipliers use `gameState.team.includes(...)` checks for `cio`, `coo`, `cfo`, `quantDev`, `fundraiser`, `compliance`. Hiring effects are persistent and apply immediately after hire (no delay).
* `aumScaleMultiplier` in fund costs uses `1 + utilizationRatio * 10`, which can create large cost escalation when utilizationRatio >> 1 (intended capacity penalty). The `finalCostBeforeDiscount = Math.max(baseCost, costAfterScaling)` ensures cost never goes below base.
