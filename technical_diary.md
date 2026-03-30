# TECHNICAL DIARY: Crypto Derivatives Options Trading
## Source: RendaTurbinada / Serious Money Course (Brazilian Portuguese, recorded ~April 2024 – June 2024, with later sessions in March 2025)
## Purpose: AI-consumable reference for algorithmic trading decisions on Deribit

---

## TABLE OF CONTENTS
1. Market Infrastructure
2. Core Concepts & Definitions
3. Strategy Encyclopedia
4. Risk Management Framework
5. Platform Operations (Deribit)
6. Market Timing & Indicators
7. Algorithm Implementation Notes

---

## 1. MARKET INFRASTRUCTURE

### 1.1 Exchange Roles

| Exchange | Role | Asset Coverage |
|---|---|---|
| Bybit | Spot purchase / fiat on-ramp | BTC, ETH, SOL, USDT, USDC, BRL (PIX) |
| Deribit | Derivatives trading (primary) | BTC, ETH, SOL options + futures |

### 1.2 Fiat-to-Deribit Capital Flow

```
BRL (PIX) → Bybit
  → Convert BRL to USDT
  → Convert USDT to SOL (Solana)
  → Withdraw SOL via Solana network to Deribit deposit address
  → Deribit Spot: SOL → USDC
  → Deribit Spot: USDC → BTC
  → BTC ready for derivatives margin
```

**Rationale for SOL routing:**
- Solana network fees: near-zero (cents of USD)
- Solana network speed: near-instant (< 1 minute typically)
- Bitcoin network fees: high and variable, especially during congestion
- Deribit supports SOL, USDC, BTC, ETH, USDT, MATIC (Ethereum network only — no Polygon fee saving)

**Minimum practical starting capital:** ~$150–$200 USD equivalent (1 SOL minimum withdrawal from Bybit)

**Minimum contract sizes on Deribit:**
- Futures: $10 USD minimum ticket
- Options: 0.1 BTC minimum contract

### 1.3 Brazilian ETF Alternatives (Non-Deribit)

These are equity-wrapper products on B3 (Brazilian stock exchange) — no derivatives access:
- **QBTC11**: Bitcoin ETF (Brazil)
- **HASH11**: Crypto index ETF (Brazil)
- **EBTC11**: Bitcoin ETF (Brazil)
- **US Spot BTC ETFs**: Approved January 2024 (BlackRock IBIT, Fidelity FBTC, etc.)

Note: ETFs do NOT allow derivatives strategies. Deribit is required for all strategies in this course.

### 1.4 Deribit Account Structure

- **Sub-accounts (subaccounts):** Deribit natively supports multiple sub-accounts per master account.
- **Best practice:** Create one sub-account per strategy to isolate margin, track P&L independently, and prevent cross-contamination.
- **Naming convention (suggested):** Use strategy name as sub-account label (e.g., "CoveredCall", "IronCondor", "FuturesSpread").
- **Margin type:** Set to **Standard Portfolio Margin** for structured options operations — this allows the exchange to recognize that one leg hedges another, reducing total margin requirement.

### 1.5 Settlement

- All Deribit derivatives settle in **BTC** (for BTC contracts) or **ETH** (for ETH contracts).
- Settlement time: **08:00 UTC / 05:00 BRT (Brasília Time)** on expiry date.
- Post-settlement: Convert BTC profit to USDC via Deribit Spot if USD denomination preferred.
- Stablecoins available on Deribit: USDC (~$1.0007) and USDT (~$1.0001) — minor depeg acceptable.

---

## 2. CORE CONCEPTS & DEFINITIONS

### 2.1 Derivatives Fundamentals

**Derivative:** A financial contract whose value derives from an underlying asset (BTC, ETH). Buying a derivative does NOT mean owning the underlying asset.

**Option:** A contract granting the right (but not obligation) to buy or sell an asset at a specified price (strike) on or before a specified date (expiry).

**Futures Contract:** An agreement to buy/sell an asset at a predetermined price at a future date. Obligatory (unlike options).

**Perpetual Contract (Perp):** A futures contract with no expiry date. Price tracks spot closely via funding rate mechanism.

### 2.2 Option Types

| Type | Definition | Buyer profits when | Seller profits when |
|---|---|---|---|
| Call | Right to BUY at strike | Price > Strike | Price < Strike |
| Put | Right to SELL at strike | Price < Strike | Price > Strike |

### 2.3 Moneyness

| Term | Abbreviation | Call Condition | Put Condition |
|---|---|---|---|
| In the Money | ITM | Spot > Strike | Spot < Strike |
| At the Money | ATM | Spot ≈ Strike | Spot ≈ Strike |
| Out of the Money | OTM | Spot < Strike | Spot > Strike |

**Delta as moneyness proxy:**
- ITM Call: Delta > 0.5
- ATM Call: Delta ≈ 0.5
- OTM Call: Delta < 0.5
- Delta 0.3 ≈ ~30% probability of expiring ITM (used as sell zone in Iron Condor)
- Delta 0.1 ≈ ~10% probability of expiring ITM (used as hedge/wing in Iron Condor)

### 2.4 The Greeks

| Greek | Symbol | Definition | Sign for Long Call | Sign for Long Put |
|---|---|---|---|---|
| Delta | Δ | Rate of option price change per $1 move in underlying | Positive (0 to 1) | Negative (-1 to 0) |
| Gamma | Γ | Rate of Delta change per $1 move in underlying | Positive | Positive |
| Theta | Θ | Time decay — option value lost per day | Negative (decay) | Negative (decay) |
| Vega | V | Sensitivity to 1% change in implied volatility | Positive | Positive |
| Rho | ρ | Sensitivity to interest rate changes | Minor in crypto | Minor in crypto |

**Practical implications:**
- **Theta** is the option seller's friend: sold options lose value daily if price stays flat.
- **Vega** means high-IV environments favor selling options (collect larger premium); low-IV environments favor buying options (cheaper premium).
- **Delta 0.55 call (ITM):** Accelerates gain more than spot alone in bull market — used in Booster and Adapted Bull Spread strategies.

### 2.5 Implied Volatility (IV)

- Displayed in Deribit option chain as a percentage per strike and type.
- High IV → expensive premiums → favorable for selling strategies (Iron Condor, Covered Call).
- Low IV → cheap premiums → favorable for buying strategies (Booster, Bull Spread).
- **IV input for probability calculation:** Extract from Deribit option chain column "Implied Vol (Bid side)" for the selected strike.

### 2.6 Premium / Basis

**Premium (option):** The price paid to buy an option, denominated in BTC fractions. Represents maximum loss for buyer, maximum gain for seller (in isolation).

**Basis (futures):** The annualized yield spread between a futures contract price and the spot (or perpetual) price.

```
Basis (%) = ((Futures Price - Spot Price) / Spot Price) × (365 / Days to Expiry) × 100
```

**Example (from Aula 4, April 2024):**
- BTC Spot: $63,000
- BTC Sep Futures: $65,300
- Days to expiry: ~150 days
- Basis ≈ ((65,300 - 63,000) / 63,000) × (365/150) × 100 ≈ 8.9% annualized

**Displayed basis on Deribit:** Shown as annualized percentage next to each futures contract for easy comparison.

### 2.7 Payoff Graph

A graphical representation of profit/loss (Y-axis) vs. underlying price at expiry (X-axis).
- **Green line:** Current mark-to-market P&L if closed immediately.
- **Purple line:** Expected P&L at expiry given current market conditions.
- Generated by Deribit's **Position Builder** tool.

### 2.8 Short Synthetic

Combination of: **Buy Put (ATM) + Sell Call (same strike, same expiry)**

Effect: Economically equivalent to being short the underlying asset. If the trader also holds the underlying asset in equal quantity, the combined position is **delta-neutral** (Protected Capital strategy).

### 2.9 Spread

**Spread:** The price difference between two related contracts. In futures context, spread = difference between two futures contracts (or between futures and perp).

**High spread:** Futures price significantly above spot/perp → sell the expensive contract.
**Low spread:** Futures price near spot/perp → buy the cheap contract, expect spread to widen.

---

## 3. STRATEGY ENCYCLOPEDIA

Each strategy entry contains: Name, Description, Market Condition, Payoff Structure, Risk/Reward, Entry Conditions, Exit Conditions, Parameters, and Platform Notes.

---

### 3.1 STRATEGY: Futures Sale (Venda de Futuros)

**Category:** Fixed-income equivalent, directionally neutral (covered)

**Description:**
Sell a BTC futures contract at a price above the current spot while holding the equivalent BTC in custody. The premium (basis) is locked in as profit regardless of price direction, as long as held to expiry.

**Market Condition:** Any market condition where futures trade at a premium to spot. Most effective in bull markets (high contango) where premium reaches 15–22%+ annualized.

**Payoff Structure:**
```
Profit (at expiry) = Futures_Sale_Price - Spot_Purchase_Price
P&L is fixed at entry. No delta risk if held to expiry.
Denominated in USD terms (BTC quantity may decrease if BTC price rises).
```

**Risk/Reward:**
- Maximum gain: Basis locked at entry (e.g., 7.6% to 22% annualized in example periods)
- Maximum loss: Effectively zero if held to expiry AND futures price > acquisition price
- Risk of gain-in-USD / loss-in-BTC: If BTC price rises sharply, fewer satoshis needed to meet the dollar obligation → BTC balance decreases

**Entry Conditions:**
1. Futures price > BTC acquisition cost (average purchase price)
2. Futures price > current spot price (contango, not backwardation)
3. Annualized basis rate > risk-free alternative (e.g., > Brazilian poupança rate of 6.17%)
4. Select contract with HIGHEST annualized basis across all available expiries

**Exit Conditions:**
- **Hold to expiry:** Guaranteed profit (automatic settlement)
- **Early exit (recompra):** Repurchase the sold futures at current price. Profitable if market fell (bought back cheaper). Not guaranteed if market rose. Reason to exit early: capital needed elsewhere, or spread has moved favorably.

**Parameters:**
- Contract selection: Compare annualized basis across all BTC futures (Jun, Sep, Dec, Mar); choose highest
- Position size: Match exactly to BTC held in sub-account (no leverage recommended)
- Minimum viable premium: > transfer cost from Bybit to Deribit
- Deribit menu path: Futures → BTC → [Select expiry] → Sell (right panel, enter USD amount)

**Example (April 2024):**
- BTC Spot: $63,000 | Sep Futures: $65,300 | Basis: ~$2,300 = 3.66% to Sep = ~8.9% annualized
- Worst week reference: 7.6% annualized (still > Brazilian savings account)
- Best observed period: 40–50% annualized (post-halving bull run)

**Algorithm Notes:**
- Scan all BTC futures expiries at market open; select max annualized basis
- Position size = floor(available_BTC / 1 BTC) or fractional in USD equivalent
- Monitor: if basis inverts (backwardation), do NOT enter; consider exit if already in trade

---

### 3.2 STRATEGY: Futures Spread — Buy Spread (Compra de Spread)

**Category:** Market-neutral, non-directional

**Description:**
Simultaneously BUY a long-dated futures contract (cheap, near spot) AND SELL a short-dated futures contract (or perpetual). Profit when the spread WIDENS (long-dated premium increases relative to short-dated).

**Market Condition:** Low spread environment — futures prices across expiries are compressed and close to each other (convergence). Expect reversion to mean wider spread.

**Payoff Structure:**
```
Entry: Buy FL (Futures Long-dated) + Sell FS (Futures Short-dated or Perp)
Exit: Sell FL + Buy FS
Profit = (FL_exit - FL_entry) - (FS_exit - FS_entry)
      = Spread_exit - Spread_entry
Profit denominated in BTC (cryptocurrency).
```

**Risk/Reward:**
- Maximum loss: Near zero (positions offset each other; only risk is operational error or funding)
- Maximum gain: Unlimited theoretically, limited in practice by market mean reversion
- Leverage possible but not recommended (amplifies operational error risk)
- Risk of liquidation: Only if "dispernado" (one leg open without the other)

**Entry Conditions:**
1. Spread between long-dated futures and short-dated futures/perp is BELOW historical average
2. Minimum 2 months remaining on the long-dated contract (allow time for spread to widen)
3. Visual confirmation via tools like Dell Gain (shows all futures premiums on one chart)

**Exit Conditions:**
1. Spread returns to historical average or above → close both legs simultaneously
2. Contract near expiry: Roll the short-dated leg before expiry (close short leg, open new short leg in next month)
3. Opportunistic early exit if spread widens rapidly (days, not months, as observed April 15–21 example)

**Parameters:**
- Short leg choice: Perpetual (subject to funding) vs. Short-dated futures (subject to rolling, no funding risk)
- Roll timing: Before expiry of short leg; open equivalent contract in next available month
- Position size: Equal USD value on both legs (maintain market neutrality)
- Tracking: Record USD balance at entry; compare at exit; profit is in BTC

**Perpetual Leg Warning:** Perpetual contracts pay/receive funding every 8 hours. In bull markets, short perp position will PAY funding (cost). Use short-dated futures leg instead to avoid this exposure.

**Algorithm Notes:**
- Monitor spread ratio: spread = (FL_price - perp_price) / perp_price
- Define "low spread" threshold: e.g., spread < 0.5% of spot price (below mean)
- Auto-detect when spread crosses threshold for entry
- Track roll dates; execute roll 3–5 days before expiry of short leg

---

### 3.3 STRATEGY: Futures Spread — Sell Spread (Venda de Spread)

**Category:** Market-neutral, non-directional

**Description:**
Mirror of Buy Spread. SELL the long-dated futures contract (expensive) and BUY the short-dated contract (or perpetual). Profit when spread NARROWS (convergence).

**Market Condition:** High spread environment — long-dated futures trading at significant premium above short-dated/perp. Expect mean reversion (compression).

**Payoff Structure:**
```
Entry: Sell FL + Buy FS (or Perp)
Exit: Buy FL + Sell FS
Profit = (FL_entry - FL_exit) - (FS_entry - FS_exit)
       = Spread_entry - Spread_exit
```

**Entry/Exit Conditions:** Mirror of Buy Spread, with spread being HIGH rather than LOW.

**Cyclic Application:**
```
Spread HIGH → Execute Sell Spread → wait for compression
Spread LOW  → Execute Buy Spread  → wait for expansion
Repeat cyclically
```

---

### 3.4 STRATEGY: Funding Rate Harvest (Coleta de Taxa de Funding)

**Category:** Carry trade, directionally neutral

**Description:**
Exploit the funding rate mechanism of perpetual contracts. When funding is strongly positive (bull market, majority long), SHORT the perpetual to RECEIVE funding payments from longs. Offset directional risk with owned BTC.

**Funding Rate Mechanics:**
- Positive rate: Longs pay Shorts → Short perp to collect
- Negative rate: Shorts pay Longs → Long perp to collect
- Rate resets: Every 8 hours on Deribit
- Correlation: Funding rate positively correlated with BTC price direction

**Entry Conditions:**
1. Funding rate is POSITIVE and meaningfully high (threshold: > 10% annualized; best observed: 40–50% annualized)
2. Market in sustained bull phase (high demand for long exposure)
3. Trader holds equivalent BTC in account (covered short — no net directional risk)

**Position Setup:**
```
Hold: X BTC in Deribit account
Short: X USD equivalent in BTC perpetual contract
Net delta: ~0 (market neutral)
Income: Funding payments received every 8 hours
```

**Exit Conditions:**
1. Funding rate drops toward zero or goes negative
2. Opportunity cost: Better strategy available
3. Market direction changes (not a forced exit for a covered position, but monitor)

**Risk:**
- Funding can flip negative → now paying instead of receiving
- Monitor continuously; exit promptly when rate becomes unfavorable

**Algorithm Notes:**
- Poll funding rate from Deribit API every hour
- Entry signal: 8-hour_funding_rate > 0.003 (≈ ~40% annualized)
- Exit signal: 8-hour_funding_rate < 0.001 (≈ ~13% annualized)
- Current rate from Deribit: displayed on perpetual contract page

---

### 3.5 STRATEGY: Covered Call — OTM (Venda Coberta de Call — Fora do Dinheiro)

**Category:** Income generation, mildly bullish to neutral to bearish

**Description:**
Sell an OTM call option against owned BTC. Collect premium. If BTC stays below strike at expiry, keep 100% of premium (in BTC/satoshis). If BTC exceeds strike, sell BTC at strike price (still profitable if strike > acquisition cost).

**Analogy:** Insurance company model — collect premium in exchange for defined obligation.

**Market Condition:** Sideways (lateral) market, mild downtrend (accumulate satoshis), or mild uptrend below strike probability threshold.

**Payoff Structure:**
```
Scenario 1 (BTC < Strike at expiry):
  Profit = +Premium_received (in BTC)
  BTC position: Unchanged

Scenario 2 (BTC > Strike at expiry):
  Profit = (Strike - Acquisition_price) + Premium_received
  BTC position: Effectively sold at Strike price
  Net result: Still profitable if Strike > Acquisition_cost

Scenario 3 (BTC falls significantly):
  USD value decreases, but BTC count INCREASES (premium adds satoshis)
  Result in satoshis: Positive
  Result in USD: Negative
```

**Risk/Reward:**
- Maximum gain: Premium collected (capped) — if price stays below strike
- Maximum loss: Opportunity cost only (upside above strike is forgone); no capital loss if strike > entry price
- "Bad scenarios": (1) BTC rises far above strike (leave money on table), (2) BTC falls (USD loss, satoshi gain)

**Entry Conditions:**
1. Hold BTC in Deribit account (the "coverage" / collateral)
2. BTC acquisition price < target strike (ensures profitability at exercise)
3. Market: lateral, mild downtrend, or mild uptrend with price unlikely to reach strike
4. Target probability of NOT being exercised: 70%–80% (use probability calculator)
5. Strike selection: Typically OTM at 70–90% probability of not being exercised

**Strike Selection Process:**
```
1. Open options chain in Deribit (Options → BTC → Select expiry)
2. Note BTC current price (e.g., $63,250)
3. Choose candidate strike (e.g., $70,000)
4. Extract Implied Volatility (IV) from Deribit chain (bid-side IV column)
5. Input to probability calculator:
   - Current price: $63,250
   - Strike (target price): $70,000
   - Days to expiry: 7
   - Volatility: 55%
6. Read output: P(price > strike) should be ≤ 20–30%
7. Adjust strike down if probability too low (increase premium); up if too high (reduce risk)
```

**Recommended probability calculator:** Referenced in course as external website (takes: spot, strike, days, IV as inputs).

**Parameters:**
- Expiry: Daily, weekly, bi-weekly, monthly — all available on Deribit
- Minimum size: 0.1 BTC
- Premium example (1 week, $70,000 strike, BTC at $63,250): $139 per BTC = $13.90 per 0.1 BTC
- Premium example (1 week, $66,000 strike, BTC at $63,250): ~$505 per BTC = $50.50 per 0.1 BTC
- Premium example (ATM $63,000 strike): $1,579 per BTC (much higher but higher risk)

**Execution on Deribit:**
```
1. Options → BTC → Select expiry date
2. Find strike in central column
3. Click on "Bid" price in the CALL column (left side)
4. Boleta opens: verify contract, expiry, strike, premium
5. Set quantity (minimum 0.1 BTC)
6. Click SELL → Confirm
```

**Exit Conditions:**
- At expiry: Automatic settlement at 08:00 UTC
- Early exit (optional): Buy back the call at market price (close position)
- If BTC approaches strike: Consider defensive actions (see variant below)

**Variants:**

**Variant A — ATM Covered Call with Futures Leverage:**
- Buy additional 40% BTC exposure via futures/perp (margin-financed, no upfront cost)
- Sell ATM call (~Delta 0.5) — much higher premium ($1,579 example)
- The futures position serves as additional coverage for the ATM call
- Higher premium but higher exercise risk → not for beginners

**Variant B — OTM with Stop-Loss Protection:**
- Sell OTM call as normal
- Place a STOP BUY order: if BTC reaches the strike level, automatically buy BTC
- The bought BTC appreciates along with the rising market, covering the call obligation
- Risk: BTC may reverse after triggering stop → net result: short call + long BTC above current market
- Must CANCEL stop order after expiry if not triggered

**Algorithm Notes:**
- Weekly strategy: Select strike with P(exercise) < 25% using Black-Scholes probability
- Monitor: If BTC rises within 2% of strike intraday, evaluate early buyback or roll up
- Roll: If approaching strike before expiry, close and re-sell at higher strike + later expiry (roll up and out)

---

### 3.6 STRATEGY: Protected Capital (Capital Garantido / Capital Protegido)

**Category:** Capital protection, zero-loss structure

**Description:**
Combine buying a put (protection against downside) and selling a call (limits upside) at the same or nearby strike and same expiry. The call premium received finances the put purchase. Net cost = zero (or positive). The BTC holding neutralizes the short call + long put synthetic short position. Result: guaranteed USD value preservation + capped upside participation.

**Also known as:** Collar (in traditional finance), Short Synthetic + Long Underlying.

**Market Condition:** After sustained BTC price appreciation. Price has risen significantly; trader wants to lock in USD value while remaining invested. Suitable for part of portfolio while remainder stays unhedged.

**Payoff Structure:**
```
Components:
  Long BTC (underlying)
  Long Put @ Strike K (insurance vs. downside)
  Short Call @ Strike K (or higher) (limits upside, finances put)

If at expiry: BTC < K → Put ITM, Call OTM
  Put pays: K - BTC_price per BTC
  Call expires worthless: keep premium
  Long BTC: worth BTC_price
  Net USD = K (effectively sold at strike) — capital protected

If at expiry: BTC > K (for same-strike version)
  Put expires worthless
  Call exercised: must sell at K → capped gain
  Net USD = K + Net_credit_received

If Put_strike ≠ Call_strike (asymmetric version):
  Buy Put at lower strike (e.g., $62,000)
  Sell Call at higher strike (e.g., $70,000)
  Protected range: below $62,000 put protects
  Gain range: $62,000 to $70,000 → full participation
  Above $70,000: gain capped
  Net credit = Call_premium - Put_premium > 0 (profit)
```

**Risk/Reward:**
- Downside: Zero USD loss (protected). BTC quantity may decrease.
- Upside: Capped at call strike. Profit in the range [spot, call_strike].
- Net cost: Zero (self-financing) or positive (net credit received).

**Entry Conditions:**
1. BTC price has risen substantially (cycle peak concern)
2. Put_premium ≈ Call_premium (self-financing). The call sold must pay at least as much as the put costs.
3. Both legs MUST be same expiry for synchronized unwinding.
4. Opportunity is NOT always available — requires specific IV skew / price relationship.

**Example from course (Strike $65,000, Sep expiry):**
```
Buy Put @ $65,000: $10,000 per BTC
Sell Call @ $65,000: $11,000 per BTC
Net credit: +$1,000 per BTC (pure profit, zero risk)
Hold to expiry; collect $1,000 per BTC automatically
```

**Asymmetric example:**
```
Buy Put @ $62,000: $8,500 per BTC
Sell Call @ $70,000: $9,243 per BTC
Net credit: +$743 per BTC
Participation range: $62,000 – $70,000
```

**Algorithm Notes:**
- Screen daily: for each BTC expiry, find pairs where Call_bid > Put_ask at same strike
- Net_credit = Call_bid - Put_ask; enter only if Net_credit > transaction_cost
- This opportunity is rare; automate the scan across all strikes and expiries
- Verify: BTC_held_in_account ≥ contract_size (must be covered/backed)

---

### 3.7 STRATEGY: Booster (Potencializador de Alta)

**Category:** Directional bullish, enhanced upside

**Description:**
Amplify gains in a bull market by combining owned BTC + long ITM/ATM call + two short OTM calls. Gains are doubled in a defined price range; above that range, gains plateau (but do not reverse). Downside loss equals normal BTC price decline (linear, no extra downside protection).

**Market Condition:** Strong bullish bias. Trader confident market will rise to a specific target range.

**Components:**
```
1. Hold: 1 BTC (underlying, provides one unit of upside)
2. Buy: 1 Call ITM or ATM (provides second unit of upside → "double exposure")
3. Sell: 2 Calls OTM (same strike, above current price)
   - Finances the ITM call purchase
   - Caps upside at the OTM strike
   - Covered by (1) the owned BTC and (2) the long call
```

**Payoff Structure:**
```
Zone 1: BTC < OTM strike
  BTC position gains linearly
  Long ITM call gains (accelerated near ATM)
  Net: ~2x normal gain (doubled)

Zone 2: BTC > OTM strike (calls exercised)
  Both short calls exercised → covered by BTC + long call
  Gain = gain_up_to_OTM_strike (frozen, no further gain above OTM)
  No loss above OTM strike

Zone 3: BTC falls
  Long call loses value (limited to premium paid)
  BTC loses value linearly
  Net: linear loss = normal BTC loss (no leverage on downside)
```

**Risk/Reward:**
- Upside: Double gain up to OTM strike; then flat above that level
- Downside: Normal linear loss (same as holding BTC alone)
- Cost: Net near-zero (2x short calls finance the long ITM call)

**Entry Conditions:**
1. Strong bullish conviction (this strategy does NOT protect against downside)
2. Low to moderate IV environment (long call is cheaper when IV is low)
3. Identify ITM call and 2x OTM calls where: 2 × OTM_premium ≥ ITM_call_premium

**Example (from course, BTC at ~$64,000):**
```
Buy Call @ $63,000 (ITM, ~ATM): Cost $4,700 per BTC ($477 per 0.1 BTC)
Sell 2× Call @ $68,000 (OTM): Receive 2 × $2,600 = $5,200 per BTC
Net credit: $5,200 - $4,700 = +$500 (zero-cost structure, slight credit)
Doubled gain zone: $64,000 – $68,000
```

**Algorithm Notes:**
- Compute: 2 × OTM_premium - ITM_premium → must be ≥ 0 for zero-cost entry
- Delta of long call should be ≥ 0.5 (ITM or ATM)
- OTM strike should be at desired price target (where "take profit" would be set)
- Payoff verification in Position Builder before entry

---

### 3.8 STRATEGY: Bull Spread with Calls (Trava de Alta com Call)

**Category:** Directional bullish, limited risk/reward

**Description:**
Buy a call ITM (or ATM) and sell a call at a higher strike (OTM). Both same expiry. Net debit paid upfront is the maximum loss. Profit is capped at the difference between strikes minus net debit.

**Market Condition:** Bullish bias. Simpler than Booster; lower premium cost but also lower gain potential.

**Components:**
```
Buy: Call @ Strike K1 (lower, ITM/ATM) — long exposure
Sell: Call @ Strike K2 (higher, OTM) — reduces cost, caps upside
Same expiry required.
```

**Payoff Structure:**
```
Max Profit = K2 - K1 - Net_Debit
Max Loss = Net_Debit (paid at entry)
Breakeven = K1 + Net_Debit

At expiry:
  BTC < K1: Lose Net_Debit
  K1 < BTC < K2: Profit increases linearly
  BTC > K2: Max profit realized
```

**Risk/Reward:**
- Maximum loss: Net debit (cost of spread) — paid upfront, no additional risk
- Maximum gain: K2 - K1 - Net_Debit
- Unlike naked calls: Loss is ALWAYS LIMITED

**Entry Conditions:**
1. Bullish market outlook
2. Minimize net debit (cost) by choosing strikes carefully
3. Prefer K1 to be ITM (Delta ~0.55) for faster acceleration in bull move
4. K2 should be at target level where gain is satisfactory

**Monitoring:**
- Can hold to expiry (automatic settlement)
- Can close early if maximum profit zone is reached before expiry (price already above K2)
- No mandatory continuous monitoring (defined risk profile)

---

### 3.9 STRATEGY: Bear Spread with Puts (Trava de Baixa com Put)

**Category:** Directional bearish, limited risk/reward

**Description:**
Mirror of Bull Spread using puts. Buy a put ITM (strike above current price) and sell a put OTM (strike below current price). Same expiry. Profit if price falls.

**Components:**
```
Buy: Put @ Strike K1 (higher, ITM for puts = strike > spot) — long downside
Sell: Put @ Strike K2 (lower, OTM for puts = strike < spot) — reduces cost, caps gain
Same expiry required.
```

**Moneyness for puts (CRITICAL DISTINCTION):**
- ITM put: Strike > Spot (put is worth something now)
- OTM put: Strike < Spot (put has no intrinsic value)
- ATM put: Strike ≈ Spot

**Payoff Structure:**
```
Max Profit = K1 - K2 - Net_Debit
Max Loss = Net_Debit
Breakeven = K1 - Net_Debit

At expiry:
  BTC > K1: Lose Net_Debit
  K2 < BTC < K1: Profit increases as price falls
  BTC < K2: Max profit realized
```

**Market Condition:** Bearish bias. Use when expecting moderate price decline.

---

### 3.10 STRATEGY: Bull Spread with Puts (Trava de Alta com Put)

**Category:** Directional bullish, credit spread (receives premium)

**Description:**
Sell a put ATM/ITM (higher strike, closer to money) and buy a put OTM (lower strike, further from money). Receive net CREDIT at entry. Profit if price stays flat or rises. Maximum loss if price falls below the bought put strike.

**Components:**
```
Sell: Put @ Strike K1 (higher, ATM or slightly ITM) — receives premium
Buy: Put @ Strike K2 (lower, OTM) — insurance, limits loss
Same expiry.
```

**Payoff Structure:**
```
Max Profit = Net_Credit (received at entry)
Max Loss = (K1 - K2) - Net_Credit
Breakeven = K1 - Net_Credit

At expiry:
  BTC > K1: Keep full credit (max profit)
  K2 < BTC < K1: Partial profit/loss
  BTC < K2: Max loss (limited to spread width - credit)
```

**Scenarios where you profit:**
1. Market rises (both puts expire OTM)
2. Market stays flat (same)
3. Market declines moderately (price stays above K2)

**Live example (from Aula 12, March 22, 2025, BTC ~$84,196):**
```
Expiry: March 28 (6 days)
Sell Put @ $82,000: Receive 0.01 BTC per contract
Buy Put @ $78,000: Pay 0.003 BTC per contract
Net credit: 0.007 BTC per contract (0.1 BTC size)
Account: 0.03 BTC ($2,525)
Margin type: Standard Portfolio Margin
```

**Entry Conditions:**
1. Bullish or neutral outlook
2. Select K1 near ATM (higher premium received)
3. K2 below K1 by ~$4,000–$6,000 (wider spread = more risk, more credit)
4. Net credit > transaction costs

**Algorithm Notes:**
- This is a CREDIT spread: money received at entry, risk is on the downside
- Margin requirement: exchange holds (K1 - K2 - credit) as collateral
- Profit probability: same as covered call (70–80% target when K1 is near ATM)

---

### 3.11 STRATEGY: Bear Spread with Calls (Trava de Baixa com Call)

**Category:** Directional bearish, credit spread

**Description:**
Sell a call ATM/near-money and buy a call OTM (higher strike). Receive net credit. Profit if market stays flat or falls. Loss if market rises above bought call strike.

**Components:**
```
Sell: Call @ Strike K1 (lower, ATM or near-money) — receives premium
Buy: Call @ Strike K2 (higher, OTM) — insurance against strong rally
Same expiry.
```

**Payoff:**
```
Max Profit = Net_Credit
Max Loss = (K2 - K1) - Net_Credit
Breakeven = K1 + Net_Credit

Profit scenarios: market falls, stays flat, rises only modestly (< K1)
```

**Live example (from Aula 13, April 4 expiry, BTC ~$84,200):**
```
Sell Call @ $87,000: Receive 0.02 BTC
Buy Call @ $92,000: Pay 0.008 BTC
Net credit: 0.012 BTC (insurance = buy at $92k)
Two-week expiry
```

---

### 3.12 STRATEGY: Iron Condor

**Category:** Volatility sell, market-neutral, range-bound

**Description:**
Combine a Bull Spread with Puts + Bear Spread with Calls at the same expiry. Creates a profit zone in the middle and loss zones on extreme moves (both capped). Optimal when IV is high (expensive premiums) and price expected to remain in a range.

**Components:**
```
Sell Put @ Delta 0.30 (K_put_sell) — lower middle strike
Buy Put @ Delta 0.10 (K_put_buy) — far lower strike (wing/protection)
Sell Call @ Delta 0.30 (K_call_sell) — upper middle strike
Buy Call @ Delta 0.10 (K_call_buy) — far upper strike (wing/protection)
All same expiry.
```

**Payoff Structure:**
```
Zone: BTC between K_put_sell and K_call_sell → Max profit (keep all premium)
Zone: K_put_buy < BTC < K_put_sell → Declining profit / approaching max loss
Zone: K_call_sell < BTC < K_call_buy → Declining profit / approaching max loss
Zone: BTC < K_put_buy or BTC > K_call_buy → Max loss (capped by wings)

Max Profit = Total_Net_Credit_received
Max Loss = Spread_width - Net_Credit
```

**Market Condition:**
1. IV recently elevated (high premiums to sell)
2. Expectation of price STABILIZATION / range-bound action
3. No strong directional bias

**Entry Conditions:**
1. Sell strikes at Delta ~0.30 (≈70% probability of expiring worthless)
2. Buy wings at Delta ~0.10 (≈90% probability of expiring worthless; cheap insurance)
3. Net credit must be meaningful relative to spread width

**Live example (from Aula 14, April 24 recording, expiry May 9, BTC ~$92,000 context):**
```
Buy Put @ $86,000 (Delta ≈ 0.10–0.12) — lower wing
Sell Put @ $92,000 (Delta ≈ 0.30) — inner lower strike
Sell Call @ $99,000 (Delta ≈ 0.30) — inner upper strike
Buy Call @ $105,000 (Delta ≈ 0.10–0.12) — upper wing
Profit zone: $85,000 – $104,000 (~$19,000 wide range)
```

**Execution sequence on Deribit:**
```
1. Options → BTC → Select expiry
2. Buy Call wing first (Delta 0.10) — lowest cost first
3. Sell Call inner strike (Delta 0.30)
4. Buy Put wing (Delta 0.10)
5. Sell Put inner strike (Delta 0.30)
All at 0.1 BTC size minimum.
```

**Risk Management for Iron Condor:**
- If price approaches inner sell strike: consider closing that threatened leg early
- "Manejo" (position management) possible during vigência (lifetime of trade)
- Max loss is always defined and limited by the wings

**Algorithm Notes:**
- Entry trigger: IV_rank > 60th percentile (high IV environment)
- Delta targets: Sell at |Delta| = 0.28–0.32; Buy wings at |Delta| = 0.08–0.12
- Monitor: if spot touches K_sell ± 2%, alert for potential roll or close
- Theta decay is the profit engine — this is a net-short-Vega strategy

---

### 3.13 STRATEGY: Asymmetric Wings (Asas Quebradas / Asymmetric Wings)

**Category:** Volatility buy/neutral, directional bias possible

**Description:**
A "broken-wing butterfly" using only calls. Buy 1 deep ITM call + Buy 1 OTM call + Sell 2 ATM calls. Creates a profit zone around current price with asymmetric protection. Can be tilted to protect more against one direction.

**Components (Bull-tilted version, as taught):**
```
Buy: 1 Call ITM @ Strike K1 (5% below spot — strong ITM)
Sell: 2 Calls ATM @ Strike K2 (≈ spot price)
Buy: 1 Call OTM @ Strike K3 (3% above spot)
```

**Net position:** 1 long ITM + 1 long OTM - 2 short ATM
**Note:** Quantity: 0.1 BTC each for the ITMs, 0.2 BTC for the ATM sells

**Payoff Structure:**
```
Profit zone: approximately from (K1 + net_debit) to K3
Peak profit at K2 (ATM strike — the sold options)
Below K1: loss (but capped by the deep ITM call)
Above K3: loss (but capped by OTM call)
The asymmetry: More protection on upside (OTM call wing) than downside (ITM deeper)
```

**Market Condition:** Low IV environment (cheap to buy calls). Expect price to stay NEAR current level but with slight upward tilt.

**Entry Conditions:**
1. IV is low (options cheap)
2. Price expected to remain in a $5,000–$8,000 range around current level
3. Slight bullish bias: more protection allocated to upside

**Live example (from Aula 15, May 18 recording, BTC ~$105,360, expiry May 30):**
```
Buy Call @ $100,000 (ITM, 5% below spot): 0.1 BTC @ 0.0650
Sell 2x Call @ $105,000 (ATM, ≈ spot): 0.2 BTC @ 0.0335 each
Buy Call @ $109,000 (OTM, ~3.5% above): 0.1 BTC @ 0.0195
Net cost: 0.0650 + 0.0195 - (2 × 0.0335) = 0.0175 BTC per 0.1 position
Profit zone: ~$101,000 – $108,400
Margin used: ~6% of account
```

**Algorithm Notes:**
- This is a LOW-VEGA strategy (benefits from IV compression after entry)
- Margin requirement is minimal (~6% of account for 12-day trade)
- Use Position Builder to verify payoff shape before entry
- Monitor: if price breaks outside profit zone, evaluate roll or close

---

### 3.14 STRATEGY: Adapted Bull Spread (Trava de Alta com Call Adaptada — Zero Cost)

**Category:** Directional bullish, complex multi-leg, zero-cost or credit entry

**Description:**
An adapted version of the standard Bull Spread with Calls designed to achieve zero net cost or a net credit at entry. Combines three spreads: (1) Bull Spread with Call (main directional bet), (2) Bull Spread with Put (to offset cost of #1), (3) Bear Spread with Call (generates the financing credit).

**Market Condition:** Bullish bias, but trader unwilling to pay debit for a standard bull spread. Offers protection against modest declines due to net credit received.

**Components:**
```
Leg Group 1 — Bull Spread with Call (directional exposure):
  Buy Call @ Delta ~0.55 (ITM, K_call_buy)
  Sell Call @ Delta ~0.40 (slightly OTM, K_call_sell_1)

Leg Group 2 — Bull Spread with Put (to offset Leg Group 1 cost):
  Sell Put @ Delta ~0.30 (K_put_sell)
  Buy Put @ Delta ~0.20 (K_put_buy)

Leg Group 3 — Bear Spread with Call (generates net credit):
  Sell Call @ Delta ~0.30 (K_call_sell_2, OTM)
  Buy Call @ Delta ~0.20 (K_call_buy_2, further OTM)

Total structure: 6 legs, same expiry.
```

**Payoff Structure:**
```
Net credit received → immediate protection against small decline
Profit zone: from [spot - ~1.33%] up to [target price ~K_call_sell_1 region]
Above target: profit decreases (partially given back by Leg Group 3)
Above upper limit (~K_call_buy_2 + credit buffer): breaks even → no further loss
Below K_put_buy: maximum loss (limited, defined by put spread width)
```

**Live example (from Aula 16, June 4 recording, BTC ~$105,440, expiry June 20):**
```
Group 1 (Bull Call Spread): Buy @ $104,000 (Δ0.55), Sell @ $107,000 (Δ0.41)
Group 2 (Bull Put Spread): Sell @ $101,000 (Δ0.30), Buy @ $98,000 (Δ0.19)
Group 3 (Bear Call Spread): Sell @ $109,000 (Δ0.36), Buy @ $112,000 (Δ0.21)
Net: Receives credit
Max profit near ~$107,900
Breakeven on upside ~$111,900 (no loss above this either)
Protection: ~$4,000 downside buffer before any loss (to ~$100,900)
Delta: Positive (bullish structure)
```

**Entry Conditions:**
1. Bullish market outlook with high confidence
2. Net structure must produce zero cost or net credit (verify in Position Builder)
3. All 6 legs same expiry (16-day example above)
4. Delta of structure should be positive (~0.3–0.6 aggregate)

**Risk:**
- Loss only if market falls more than ~1.33% AND stays there
- Upside loss: plateau above max-profit strike, eventually breaks even above upper bound
- No loss above uppermost wing (fully defined risk)

---

### 3.15 STRATEGY: Perpetual Hedge (Venda de Perpétuo — Hedge Total)

**Category:** Portfolio protection, market-neutral

**Description:**
Sell a perpetual futures contract in equal size to the BTC held in the same account. Creates a perfectly hedged (delta-zero) position. Used during periods of uncertainty, travel, or when protecting USD value of holdings without selling to stablecoins.

**Components:**
```
Long: X BTC (held in Deribit account)
Short: X USD equivalent in BTC Perpetual contract
Net delta: ~0
```

**Effect:**
```
If BTC rises: Long BTC gains, Short Perp loses → net ~0
If BTC falls: Long BTC loses, Short Perp gains → net ~0
Result: USD value preserved regardless of BTC price direction
```

**Advantages over stablecoin conversion:**
- No stablecoin depeg risk (USDT/USDC historically depegged briefly during FTX crisis)
- No need to convert out and back in (saves trading fees)
- Maintains BTC custody position

**Risks:**
- Funding rate: If funding is negative while short, you PAY funding (cost)
- Operational risk: If perp position size ≠ BTC position size, imperfect hedge
- Must be within same sub-account for margin offset

**Exit:**
- Close perp short when ready to re-engage market exposure
- Review after each major market event

---

## 4. RISK MANAGEMENT FRAMEWORK

### 4.1 Core Principles (from Aula 10)

**Rule 1 — No Leverage (Primary Rule)**
- Avoid leverage entirely unless using market-neutral structures (futures spread, perpetual hedge)
- Leveraged directional positions risk liquidation
- Liquidation causes: financial loss, psychological trauma, exit from market
- Only acceptable leverage: within fully-hedged structures (both-sides positions)

**Rule 2 — Sub-account Isolation**
- One sub-account per strategy
- Transfer only required capital per sub-account
- Track P&L independently per strategy
- Prevents cross-margin contamination

**Rule 3 — Margin Monitoring**
```
Maintenance Margin (MM): 0% (no positions) → 100% (liquidation threshold)
Target: MM < 70% at all times when holding open positions
Initial Margin (IM): Tracks capacity to open new positions
Target: IM kept low to maintain maneuverability
```

Margin type for structured options: **Standard Portfolio Margin** (recognizes hedged structures, reduces collateral requirements).

**Rule 4 — Trend Alignment**
- Always operate WITH the prevailing trend for directional strategies
- For counter-trend: use only NEUTRAL (non-directional) structures
- Analysis framework: Monthly chart → Weekly chart → Daily chart
- Identify: Higher highs + higher lows = uptrend; Lower highs + lower lows = downtrend

**Rule 5 — Structured Operations Preferred**
- Never sell options "naked" (dry sales / venda seca)
- Naked call: unlimited loss if price rises above strike
- Naked put: unlimited loss if price falls below strike
- ALWAYS combine with an offsetting buy to create defined-risk structures

### 4.2 Insurance Techniques

**Insurance A — Partial Futures Hedge:**
- Sell a small percentage (10–20%) of BTC position as futures short
- If market falls: short position profits, partially offsets BTC loss
- If market rises: long BTC gains dominate; small short is a modest cost

**Insurance B — Cheap OTM Put Purchase:**
- Buy OTM puts as "lottery tickets" against catastrophic downside
- Small fixed cost (premium)
- In a crash event: puts become valuable, can be sold for large profit
- Offsets some of the portfolio loss

**Insurance C — Structured Operation Design:**
- Combine buy + sell within every trade to eliminate naked risk
- Caps gain but defines and limits loss in all scenarios

### 4.3 Black Swan Protection

Reference events that underline the need for hedging:
- FTX collapse (Nov 2022): Second-largest crypto exchange failed unexpectedly → sudden -30%+ BTC drop
- Terra/LUNA collapse: Stablecoin UST depegged, cascaded to market crash
- USDC brief depeg (March 2023, Silicon Valley Bank crisis)
- USDT brief depeg (multiple occasions)

**Protocol during crises:** Perpetual hedge (Sell Perp = BTC size) neutralizes all price risk without stablecoin exposure.

### 4.4 Position Sizing Guidelines

| Strategy | Recommended BTC allocation |
|---|---|
| Futures Sale | 100% of sub-account BTC |
| Covered Call | 100% of sub-account BTC (coverage) |
| Protected Capital | 100% (must hold underlying) |
| Iron Condor | 10–30% (margin-efficient) |
| Booster | 100% + long call (high conviction only) |
| Bull/Bear Spread | Any size (defined risk = full deployment acceptable) |
| Perpetual Hedge | 100% (must match BTC position size exactly) |

### 4.5 Starting Capital Recommendations

- **Testnet (practice):** Start with 0.2 BTC equivalent on Deribit testnet
- **Live minimum (options selling):** 0.1 BTC ($8,000+ at typical prices)
- **Futures minimum:** $10 USD ticket
- **Universal advice:** Start small, verify execution, scale up only after confirmed understanding
- **Operational test:** When transferring to Deribit for first time, send a small test amount first, confirm receipt, then send full amount

### 4.6 Security Protocol (Wallet Safety)

**Critical vulnerability: clipboard hijacker malware**
- Malware replaces copied wallet addresses with attacker's address
- Two known victims personally referenced by course instructors
- **Mitigation:** Always verify LAST 5 DIGITS of destination address after paste before confirming
- **Bybit 2FA:** Adding new withdrawal address requires email confirmation (adds second verification layer)
- Never paste and send without verification step

---

## 5. PLATFORM OPERATIONS (DERIBIT)

### 5.1 Interface Navigation

**Main Menu Items:**
```
Dashboard (Home icon): Account summary, open positions, open orders, P&L
Spot: Limited spot market (BTC, ETH, SOL, USDC, USDT) — used for SOL→USDC→BTC conversion
Futures: Perpetual and dated futures contracts
Options: Options chain by expiry and strike
Position Builder: Simulation tool (grid icon / "little squares")
Wallet: Multi-currency balances, deposit/withdraw functions
```

**Sub-account management:**
- Create sub-accounts from account settings
- Each sub-account has independent margin, positions, and P&L tracking
- Transfer funds between sub-accounts via internal transfer

### 5.2 Options Chain Reading

Columns (left to right for a given expiry):
```
[CALLS side]                    [STRIKE]           [PUTS side]
Bid | Ask | IV(bid) | Delta     [Strike Price]     Delta | IV(ask) | Bid | Ask
```

**How to use:**
- Click on Bid price in Call column → opens sell boleta (order ticket)
- Click on Ask price in Call column → opens buy boleta
- Reverse for Put column (Bid = someone willing to buy puts)
- Double-click on strike to open the order ticket directly

**Expiry availability for BTC options:**
- Daily: Every day
- Weekly: Fridays
- Monthly: Last Friday of month
- Quarterly: March, June, September, December
- Further dated: Up to ~18 months forward (e.g., March 2025, June 2025)

### 5.3 Order Types

**Market Order:** Executes immediately at best available price.

**Limit Order:** Executes only at specified price or better. Stays in order book until filled or cancelled.

**Stop Order:** Triggers a market or limit order when price reaches a specified level.
- Used in Covered Call Variant B: Stop BUY if BTC reaches call strike → protects against exercise

### 5.4 Position Builder Tool

**Access:** Dashboard → Grid icon (top right area) → Position Builder

**Features:**
- Add any combination of instruments (futures, options — calls/puts, any strike/expiry)
- Displays payoff chart: current P&L (green) and expiry P&L (purple) vs. price
- Can toggle between BTC-denominated and USD-denominated P&L view
- Sync with live account: shows current portfolio payoff
- Restrictions: Multi-expiry combinations may distort chart (use same expiry when possible)

**Usage workflow:**
```
1. Open Position Builder
2. Add instruments from left panel (search by strike, expiry, type)
3. Set direction (buy/sell) and quantity for each
4. View payoff chart → verify risk/reward before executing live
5. Execute trades in main interface once satisfied
```

### 5.5 Margin System

**Maintenance Margin (MM):**
- 0% = no positions
- 100% = liquidation triggered
- Target: keep below 70%

**Initial Margin (IM):**
- Reflects capacity to open new positions
- Keep low for operational flexibility

**Standard Portfolio Margin:**
- Required setting for structured options operations
- Exchange recognizes that buy + sell legs hedge each other
- Result: Lower margin requirement vs. treating each leg independently
- Configure in account settings before placing structured trades

### 5.6 Settlement Details

- **Time:** 08:00 UTC daily (05:00 BRT)
- **Method:** Cash-settled in BTC (not physical delivery)
- **Index price:** Deribit's own BTC index (average of multiple exchanges)
- **Auto-settlement:** At expiry, positions close automatically; P&L credited/debited in BTC
- **Post-settlement action:** Convert BTC gains to USDC via Spot market if USD denomination required

### 5.7 Futures Interface

**Access:** Futures → BTC → [Select contract]

**Available BTC futures (as of recording, April 2024):**
- BTC-PERPETUAL (no expiry)
- BTC-28JUN24, BTC-27SEP24, BTC-27DEC24, BTC-28MAR25
- Each has: Current price, 24h change, volume, basis (premium %)

**Annualized basis display:** Deribit shows premium as annualized % for each contract. Compare across contracts to select optimal entry.

**Order ticket (boleta):**
```
Field: USD Amount to sell
Field: Price (auto-filled from order book; adjust if needed)
Button: SELL (red) / BUY (green)
Placed order shows in: Open Orders tab
```

### 5.8 Testnet (Paper Trading)

- Available at testnet.deribit.com
- Claim free test BTC (0.2 BTC suggested by platform)
- Full interface replica — all features available
- Use to practice strategy execution before live deployment
- No financial risk; test transactions do not affect real account

---

## 6. MARKET TIMING & INDICATORS

### 6.1 Funding Rate as Market Signal

**Data source:** Perpetual contract page on Deribit (displayed in real-time)

**Interpretation table:**
| Funding Rate | Market Condition | Action |
|---|---|---|
| High positive (> 0.03%/8h = ~40% annualized) | Extreme bull, crowded longs | Short perp to collect funding (covered) |
| Moderate positive (0.01–0.03%/8h) | Bullish | Monitor; not compelling enough |
| Near zero (< 0.01%/8h) | Neutral/uncertain | No funding opportunity |
| Negative (< 0) | Bearish; longs paid by shorts | Long perp to collect (but bearish = caution) |

**Historical reference (course recordings):**
- April 2024: Rates near zero / slightly negative (post-ATH consolidation)
- ~1 month prior (March 2024): 40–50% annualized (many students collected)
- Correlation: Rate moves with BTC price direction (positive = price rising)

### 6.2 Futures Spread (Basis) as Market Signal

**Tool:** Dell Gain platform (or equivalent futures curve visualization) — shows premium of all contracts on one chart.

**Signal interpretation:**
| Spread State | Description | Action |
|---|---|---|
| Low spread (compressed) | All futures prices near each other and near spot | Buy Spread (long FL, short FS) — expect expansion |
| High spread (expanded) | Long-dated futures far above spot | Sell Spread (short FL, long FS) — expect compression |
| Inverted (backwardation) | Futures below spot | Do NOT enter Futures Sale strategy; market bearish |

**Example timing observation (April 2024):**
- April 15–17: Spreads compressed (convergence)
- April 21: Spreads expanded (diverged)
- Window: 4 days from low to high spread → fast opportunity

### 6.3 Implied Volatility (IV) as Strategy Selector

| IV Level | Interpretation | Favored Strategies |
|---|---|---|
| High IV (elevated) | Options expensive; market fearful | Iron Condor, Covered Call, Protected Capital |
| Low IV (depressed) | Options cheap; market complacent | Booster, Bull/Bear Spread, Asymmetric Wings |
| Rising IV | Directional move expected | Avoid selling strategies; favor buying |
| Falling IV | Stabilization expected | Favor selling strategies (theta/vega income) |

**IV source:** Deribit options chain, displayed per strike per expiry.

### 6.4 BTC Price Cycle Context

**Bitcoin halving cycle (referenced throughout course):**
- Halving events historically precede bull phases (6–18 months)
- April 2024 halving: Recent; course recorded during post-halving period
- Typical cycle: Accumulation → Bull run → ATH → Distribution → Bear market
- Course taught during transition from accumulation/early bull to mid-bull

**Strategy selection by cycle phase:**
| Cycle Phase | Recommended Strategies |
|---|---|
| Accumulation / Sideways | Covered Call (OTM), Futures Sale, Iron Condor |
| Early Bull | Booster, Bull Spread, Adapted Bull Spread |
| Peak Bull / Euphoria | Protected Capital, Perpetual Hedge, Futures Sale (collect basis) |
| Distribution / Correction | Bear Spread, Iron Condor, Covered Call, Funding harvest (short) |
| Bear Market | Bear Spread with Put, Iron Condor, Futures Sale (if in backwardation: avoid) |

### 6.5 Technical Analysis Integration

**Recommended timeframes for trend identification:**
1. Monthly chart: Macro trend (bull/bear cycle)
2. Weekly chart: Intermediate trend (rallies within trend)
3. Daily chart: Entry timing and strike selection

**Key signals:**
- Higher highs + higher lows = confirmed uptrend → use bullish strategies
- Lower highs + lower lows = confirmed downtrend → use bearish or neutral strategies
- Price consolidating near support = accumulation → Covered Call (OTM), Iron Condor

**Principle:** Always operate WITH the dominant trend for directional strategies. Counter-trend operations: use neutral structures only (no directional risk).

---

## 7. ALGORITHM IMPLEMENTATION NOTES

### 7.1 Data Requirements

**Real-time feeds needed:**
```python
# Deribit API endpoints (REST + WebSocket)
DERIBIT_BASE = "https://www.deribit.com/api/v2"

# Key data points to poll:
{
  "btc_spot_index": "/public/get_index_price",
  "funding_rate_current": "/public/get_funding_rate_value",  # for PERPETUAL
  "funding_rate_history": "/public/get_funding_rate_history",
  "futures_instruments": "/public/get_instruments",  # kind=future
  "options_instruments": "/public/get_instruments",  # kind=option
  "options_chain": "/public/get_book_summary_by_currency",
  "ticker": "/public/ticker",  # per instrument
  "account_margin": "/private/get_account_summary",
  "open_positions": "/private/get_positions",
  "order_book": "/public/get_order_book"
}
```

### 7.2 Strategy Selection Logic (Decision Tree)

```
INPUT: current_btc_price, funding_rate, futures_basis[], iv_rank, trend_direction

IF funding_rate > 0.003 (8h rate ≈ 40% annualized) AND btc_held > 0:
  → EXECUTE: Funding Rate Harvest (short perp, size = btc_held)

ELIF futures_basis_max > transfer_cost AND trend != BEARISH:
  → EVALUATE: Futures Sale (select max basis contract)
  → ENTER if basis > threshold (e.g., > 5% annualized)

ELIF futures_spread == LOW AND time_to_expiry_long >= 60 days:
  → EXECUTE: Buy Futures Spread

ELIF futures_spread == HIGH:
  → EXECUTE: Sell Futures Spread

ELIF iv_rank > 60 AND trend == NEUTRAL:
  → EVALUATE: Iron Condor
  → SELECT strikes: sell at |delta| ≈ 0.30, buy wings at |delta| ≈ 0.10

ELIF iv_rank < 40 AND trend == BULLISH:
  → EVALUATE: Booster OR Bull Spread OR Adapted Bull Spread
  → Booster if: strong conviction, 2x_OTM_premium >= ITM_premium
  → Adapted Bull Spread if: want zero-cost entry

ELIF trend == NEUTRAL or MILDLY_BEARISH:
  → EVALUATE: Covered Call OTM
  → SELECT strike: P(exercise) < 25%, premium > 0.5% per week

ELIF capital_protection_needed AND call_bid > put_ask:
  → EXECUTE: Protected Capital (find self-financing collar)
```

### 7.3 Iron Condor Parameter Calculation

```python
def select_iron_condor_strikes(options_chain, target_delta_sell=0.30, target_delta_wing=0.10):
    calls = [o for o in options_chain if o['type'] == 'call']
    puts = [o for o in options_chain if o['type'] == 'put']

    # Find sell strikes (closest to target delta)
    call_sell = min(calls, key=lambda x: abs(x['delta'] - target_delta_sell))
    put_sell = min(puts, key=lambda x: abs(abs(x['delta']) - target_delta_sell))

    # Find wing strikes (closest to target delta for protection)
    call_wing = min(calls, key=lambda x: abs(x['delta'] - target_delta_wing))
    put_wing = min(puts, key=lambda x: abs(abs(x['delta']) - target_delta_wing))

    net_credit = (call_sell['bid'] + put_sell['bid']) - (call_wing['ask'] + put_wing['ask'])
    max_loss = (call_wing['strike'] - call_sell['strike']) - net_credit  # per BTC

    return {
        'put_wing': put_wing['strike'],
        'put_sell': put_sell['strike'],
        'call_sell': call_sell['strike'],
        'call_wing': call_wing['strike'],
        'net_credit_btc': net_credit,
        'max_loss_btc': max_loss,
        'profit_zone': (put_sell['strike'], call_sell['strike'])
    }
```

### 7.4 Covered Call Strike Selection

```python
import math
from scipy.stats import norm

def black_scholes_prob_above_strike(spot, strike, days, iv_annualized):
    """
    Returns probability that spot > strike at expiry.
    Uses log-normal assumption (Black-Scholes framework).
    """
    T = days / 365.0
    sigma = iv_annualized / 100.0
    d2 = (math.log(spot / strike) + (-0.5 * sigma**2) * T) / (sigma * math.sqrt(T))
    prob_above = norm.cdf(-d2)
    return prob_above

def select_covered_call_strike(options_chain, spot, days,
                                max_exercise_prob=0.25):
    """
    Select strike with highest premium where P(exercise) < max_exercise_prob.
    Course recommendation: 20–30% max exercise probability.
    """
    candidates = []
    for option in options_chain:
        if option['type'] != 'call' or option['strike'] <= spot:
            continue
        iv = option['iv_bid'] / 100.0
        prob = black_scholes_prob_above_strike(spot, option['strike'], days, iv * 100)
        if prob <= max_exercise_prob:
            candidates.append({
                'strike': option['strike'],
                'premium_btc': option['bid'],
                'prob_exercise': prob,
                'iv': iv
            })
    # Select highest premium within acceptable risk
    if candidates:
        return max(candidates, key=lambda x: x['premium_btc'])
    return None
```

### 7.5 Annualized Basis Calculator

```python
def annualized_basis(futures_price, spot_price, days_to_expiry):
    """
    Calculates annualized basis (premium) of a futures contract.
    Used to compare contracts and select highest-yielding one.
    """
    raw_basis = (futures_price - spot_price) / spot_price
    annualized = raw_basis * (365 / days_to_expiry) * 100
    return annualized  # in percent

def select_best_futures_contract(futures_list, spot_price):
    """
    futures_list: [{'expiry_date': date, 'price': float}, ...]
    Returns contract with highest annualized basis above minimum threshold.
    """
    from datetime import datetime
    today = datetime.utcnow()
    MIN_BASIS_PCT = 5.0  # Only enter if basis > 5% annualized

    results = []
    for f in futures_list:
        days = (f['expiry_date'] - today).days
        if days <= 0:
            continue
        basis = annualized_basis(f['price'], spot_price, days)
        if basis > MIN_BASIS_PCT:
            results.append({'contract': f, 'basis_annualized': basis, 'days': days})

    return max(results, key=lambda x: x['basis_annualized']) if results else None
```

### 7.6 Margin Monitoring

```python
def check_margin_safety(account_summary, max_mm_pct=70):
    """
    Maintenance margin must stay below 70% of account.
    Alerts if approaching limit.
    Returns: 'SAFE', 'WARNING', 'DANGER', 'LIQUIDATION_IMMINENT'
    """
    mm = account_summary['maintenance_margin_pct']

    if mm < 50:
        return 'SAFE'
    elif mm < max_mm_pct:  # 70% threshold
        return 'WARNING'  # approaching; reduce positions
    elif mm < 85:
        return 'DANGER'  # close losing positions immediately
    else:
        return 'LIQUIDATION_IMMINENT'  # emergency hedge or close all
```

### 7.7 Funding Rate Monitoring

```python
def evaluate_funding_opportunity(current_8h_rate, btc_position_size):
    """
    current_8h_rate: float (e.g., 0.003 = 0.3% per 8 hours)
    btc_position_size: float (BTC held in account)
    Returns recommended action.
    """
    annualized = current_8h_rate * 3 * 365 * 100  # 3 periods/day × 365 days

    if current_8h_rate > 0.003:  # >~40% annualized
        return {
            'action': 'SHORT_PERP',
            'size_usd': btc_position_size,  # match BTC position exactly
            'annualized_yield_pct': annualized,
            'reason': 'High funding rate; collect by shorting perp (covered)'
        }
    elif current_8h_rate > 0.001:  # >~13% annualized
        return {'action': 'MONITOR', 'annualized_yield_pct': annualized}
    elif current_8h_rate <= 0:
        return {'action': 'AVOID_SHORT', 'reason': 'Negative funding; shorts pay longs'}
    else:
        return {'action': 'HOLD', 'annualized_yield_pct': annualized}
```

### 7.8 Key Operational Rules for Algorithms

```
RULE 1: Never submit a sell order without a corresponding buy order in any structured strategy.
        Verify both legs before confirming either. "Dispernado" (one-legged) = liquidation risk.

RULE 2: Always verify destination wallet address (last 5 chars) before any BTC/SOL transfer.
        Clipboard hijacker risk is real and irreversible.

RULE 3: Settlement window is 08:00 UTC. Do NOT have open positions that you intend to let
        expire for more than 15 minutes before settlement without monitoring.

RULE 4: Rolling procedure for futures spread short leg:
        - Trigger: 5 days before expiry of short-dated leg
        - Action: Close short-dated leg AND open new short-dated leg in next month
        - Do NOT close long-dated leg during roll

RULE 5: All profits from BTC derivatives are paid in BTC.
        If USD denomination required: immediately convert via Deribit Spot (BTC → USDC)
        after settlement at 08:00 UTC.

RULE 6: Sub-account isolation — never mix strategies in same sub-account.
        Margin calculations will be unreliable; P&L tracking impossible.

RULE 7: For Standard Portfolio Margin to apply correctly, positions must demonstrate
        hedge relationship (buys offset sells). Pure long positions receive less benefit.

RULE 8: Minimum contract size is 0.1 BTC for options.
        Any calculation that produces a position below 0.1 BTC should be rounded up to 0.1 BTC
        or the trade should be skipped.

RULE 9: When entering Iron Condor, always start with the BUY orders (wings) first.
        This establishes the hedged status before the sell orders are submitted.
        Sequence: Buy Call wing → Sell Call inner → Buy Put wing → Sell Put inner.

RULE 10: Position Builder MUST be used to verify payoff of any new structured strategy
         before live execution. Never skip simulation step.
```

### 7.9 Conversion Workflow Automation

```
Trigger: New capital injection from Bybit

Step 1: Detect SOL deposit confirmed on Deribit (1 blockchain confirmation = ~1 second on Solana)
Step 2: Execute Deribit Spot trade: SOL → USDC (at market)
Step 3: Execute Deribit Spot trade: USDC → BTC (at market)
Step 4: Allocate BTC to target sub-account(s) per strategy allocation rules
Step 5: Log entry BTC quantity and USD equivalent for P&L baseline

Transfer cost estimate: < $0.01 USD per transfer via Solana network
Timing: Near-instant (seconds to minutes)
```

### 7.10 Strategy Performance Metrics to Track

```
Per Strategy / Sub-account:
  - Entry date and BTC balance (baseline)
  - Current BTC balance
  - USD equivalent at entry vs. current
  - Delta BTC (satoshi accumulation)
  - Delta USD (dollar P&L)
  - Premium collected (for selling strategies)
  - Number of times exercised (for covered call)
  - Annualized return (extrapolated)
  - Funding paid/received (perpetual strategies)

Aggregate:
  - Total portfolio delta (market exposure)
  - Total margin utilization (MM%)
  - Total BTC held vs. derivative exposure
  - Net funding cost/income
  - Comparison vs. benchmark: Brazilian savings rate (Selic/Poupança)
```

---

## APPENDIX A: GLOSSARY

| Term (PT) | Term (EN) | Definition |
|---|---|---|
| Taxa de Funding | Funding Rate | Periodic payment between longs and shorts in perpetual contracts |
| Spread de Futuros | Futures Spread | Price differential between futures contracts or between futures and spot |
| Venda Coberta | Covered Call | Selling a call option against owned underlying asset |
| Capital Garantido / Protegido | Protected Capital / Collar | Zero-cost structure combining long put + short call on owned BTC |
| Booster | Booster / Ratio Spread | Enhanced upside through owned BTC + long call + 2x short OTM calls |
| Trava de Alta | Bull Spread | Long lower call + short higher call (or short lower put + long lower put) |
| Trava de Baixa | Bear Spread | Long higher put + short lower put (or short lower call + long higher call) |
| Iron Condor | Iron Condor | Bull put spread + bear call spread combined at same expiry |
| Asas Quebradas | Asymmetric Wings / Broken-Wing Butterfly | Unequal butterfly structure with asymmetric protection |
| Trava de Alta com Call Adaptada | Adapted Bull Spread | Zero-cost 6-leg structure with credit entry for bullish market |
| Venda de Perpétuo | Perpetual Hedge | Short perpetual contract equal to BTC held → delta zero |
| Vencimento | Expiry / Maturity | Date when option/futures contract settles |
| Strike | Strike Price | Exercise price of an option contract |
| Prêmio | Premium | Price of an option contract |
| Basis | Basis | Annualized yield spread between futures and spot |
| Contrato Perpétuo | Perpetual Contract | Futures with no expiry; price-anchored via funding rate |
| OTM | Out of the Money | Call: strike > spot; Put: strike < spot |
| ITM | In the Money | Call: strike < spot; Put: strike > spot |
| ATM | At the Money | Strike ≈ spot |
| Exercício | Exercise | When option buyer exercises the contract at expiry |
| Satoshis | Satoshis | Smallest BTC unit (1 BTC = 100,000,000 satoshis) |
| Dispernado | Dispernado (PT slang) | One-legged spread — one leg open without the hedge leg |
| Boleta | Order Ticket | Trade entry form/dialog on Deribit |
| Rolagem | Roll | Closing a near-expiry position and reopening equivalent in later month |
| Subaccount | Sub-account | Isolated account within Deribit for strategy separation |
| MM | Maintenance Margin | Margin threshold for liquidation risk (100% = liquidation) |
| IM | Initial Margin | Available margin for opening new positions |
| Manejo | Position Management | Adjustments made to open positions to optimize outcome |
| Lastro | Collateral / Coverage | The underlying BTC backing a short options position |

---

## APPENDIX B: QUICK REFERENCE — STRATEGY SELECTION MATRIX

| Market Condition | IV Level | Recommended Strategy | Notes |
|---|---|---|---|
| Strongly bullish | Low | Booster | Double gains up to target |
| Bullish | Low | Bull Spread with Call | Simple, defined risk |
| Bullish (free entry) | Any | Adapted Bull Spread | Complex 6-leg; zero cost |
| Bullish (credit entry) | Any | Bull Spread with Put | Receive credit; profit if flat/up |
| Lateral (range-bound) | High | Iron Condor | Sell expensive premiums |
| Lateral | Any | Covered Call OTM | Income; accumulate satoshis |
| Lateral | Any | Futures Sale | Fixed-income equivalent in USD |
| Bearish | Low | Bear Spread with Put | Simple, defined risk |
| Bearish (credit entry) | Any | Bear Spread with Call | Receive credit; profit if flat/down |
| Capital protection needed | Any | Protected Capital (Collar) | Self-financing; locks USD value |
| Capital protection needed | Any | Perpetual Hedge | Full delta hedge; no stablecoin risk |
| High funding rate | — | Funding Harvest (Short Perp) | Carry income; market-neutral |
| Low futures spread | — | Buy Futures Spread | Expect spread expansion |
| High futures spread | — | Sell Futures Spread | Expect spread compression |
| Any (fixed income) | — | Futures Sale | Locks basis as annualized yield |

---

*Technical Diary compiled from Aulas 1–18 of the RendaTurbinada / Serious Money course.*
*All strategies taught are intended for Deribit derivatives exchange.*
*All monetary examples are denominated in USD unless otherwise specified.*
*BTC prices referenced in course: $63,000–$105,000 range (April 2024 – June 2024, with a March 2025 session at ~$84,196).*
