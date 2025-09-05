# Introduction to Bonds — The Basics (India-Ready, BondFlow Style)

Bonds = lend money to an issuer (government or company), earn periodic interest (coupon), and get your principal back at maturity. Simple idea, powerful in portfolios.

Think of it like: You lend your friend ₹1,000 today; they pay you ₹50 each year and return ₹1,000 after 5 years. A bond is that—formalized, tradable, and rated.

## Quick Snapshot
- What you get: Regular interest (coupon) + principal back at maturity
- What can change: Market price (up/down), yield, and sometimes when the bond can be called back
- Why people buy: Income, stability, diversification, and defined cashflows
- What to watch: Interest rates, credit quality, liquidity, taxes, and embedded options

---

## Key Characteristics (with examples)

| Term | What it means | Why it matters | Example |
|---|---|---|---|
| Face/Par Value | Principal repaid at maturity | Basis for coupon calculation | ₹1,000 par → coupon paid on ₹1,000 |
| Coupon Rate | Annual interest % on par | Your income stream | 8% → ₹80 per year, often in 2 x ₹40 |
| Coupon Payment | Actual periodic payout | Cash flow frequency | Semi-annual: ₹40 every 6 months |
| Maturity Date | When issuer repays par | Defines time horizon | 5-year bond issued 2023 → matures 2028 |
| Issue Price | Price at issuance (par/discount/premium) | Affects initial yield | Issued at ₹980 (discount) or ₹1,020 (premium) |
| Yield | Return considering price, coupon, time | Realistic return metric | Price down → yield up (and vice versa) |

Yield flavors:
- Current Yield = Annual coupon / Current price
- Yield to Maturity (YTM) = Annualized total return if held to maturity (assumes reinvestment of coupons at same rate)
- Yield to Call (YTC) = Same as YTM but to the call date (for callable bonds)

---

## Types of Bonds (India context)
- Government Bonds (G-Secs): Sovereign risk; benchmark for rates; usually lowest credit risk
- SDLs: State Development Loans; state government bonds
- PSU/AAA Corporate Bonds: High quality; lower yields than mid-grade corporates
- Corporate Bonds (AA/A/BBB): Higher yields; higher credit risk; check covenants and ratings
- Tax-Free Bonds (legacy issues): Some older issues have tax-free interest—scarce and often at premium
- Perpetuals (AT1/Perp): No fixed maturity; callable; coupon can be discretionary—advanced only
- Zero-Coupon Bonds (Strips): No coupons; sold at discount; return realized at maturity
- Floating-Rate Bonds: Coupons reset to a benchmark (like MIBOR) + spread

Embedded options to note:
- Callable: Issuer can redeem early (you face reinvestment risk if called)
- Puttable: You can exit early at par (investor-friendly)
- Convertible: Can convert into equity under conditions

---

## How Bond Returns Work

Total Return ≈ Coupon income + Price change ± Accrued interest − Taxes/Fees

Scenarios:
- Held to maturity: You typically get all coupons + par back → return ≈ YTM at purchase
- Sell before maturity: You lock in market price then → gain or loss depends on where yields moved and credit developments

Price–Yield relationship (always inverse):
- If yields rise by 1%, bond prices fall
- If yields fall by 1%, bond prices rise

Rule of thumb via duration:
- Approx % Price Change ≈ −Duration × ΔYield
- Example: Duration = 4.0; Yield +1% → Price ≈ −4% (approx)

---

## Clean vs Dirty Price (Accrued Interest)

- Clean Price: Quoted price excluding accrued interest
- Dirty Price: What you pay/receive = Clean Price + Accrued Interest

Simple example (semi-annual 8% on ₹1,000 → ₹40 per half-year):
- Midway through the coupon period, accrued ≈ ₹20
- If clean price = ₹960 → dirty price ≈ ₹960 + ₹20 = ₹980

Note: Actual calculation uses a day-count convention (e.g., ACT/ACT, 30/360). BondFlow will show both clean and dirty transparently.

---

## Bond Math Quickies

1) Current Yield
- Current Yield = Annual coupon / Current price
- Example: Coupon ₹80, Price ₹960 → 80/960 = 8.33%

2) Approximate YTM (handy estimator)
- YTM ≈ [Coupon + (Par − Price)/Years] / [(Par + Price)/2]
- Example: Par 1,000; Coupon 80; Price 960; Years 5
  - YTM ≈ (80 + (1,000 − 960)/5) / (1,000 + 960)/2 ≈ (80 + 8) / 980 ≈ 8.98%
- If price is ₹1,050:
  - YTM ≈ (80 + (1,000 − 1,050)/5) / (1,000 + 1,050)/2 ≈ (80 − 10) / 1,025 ≈ 6.83%

3) Duration intuition
- Modified Duration ≈ % price change per 1% yield move
- Higher duration = more interest rate sensitivity

---

## Risks to Know (beyond the basics)

- Interest Rate Risk: Prices drop when yields rise; larger for higher-duration bonds
- Credit Risk: Issuer may miss payments; watch ratings, outlooks, and covenants
- Liquidity Risk: Hard to sell quickly at fair price; BondFlow aims to improve this via RFQ and fractional depth
- Reinvestment Risk: Coupons may be reinvested at lower future rates
- Call/Prepayment Risk: Callable bonds can be taken away when rates fall
- Inflation Risk: Real purchasing power of coupons/principal can erode
- Concentration Risk: Overweight one issuer/sector/tenor magnifies shocks
- Operational/Settlement Risk: Reduced with regulated rails, escrow, and depository settlement

---

## How to Read a Bond Card (at a glance)

| Field | Meaning | What to check |
|---|---|---|
| ISIN | Unique ID of the bond | Used for settlement and lookup |
| Issuer | Company/PSU/Govt | Quality, sector exposure |
| Coupon | Fixed or floating rate | Frequency and day-count |
| Maturity/Call | Final date and call/put options | Call schedule can cap upside |
| Price (Clean/Dirty) | Market price vs what you pay | Accrued interest clarity |
| YTM/YTC | Return to maturity/call | Compare vs peers and your horizon |
| Rating | Credit quality (AAA → BBB/Below) | Trends and outlook changes |
| Lot size | Minimum tradable unit | Fractional access via BondFlow |
| Liquidity | Indicative depth/spreads | Lower spread is better |
| Tax | Interest and capital gains | See Tax Basics below |

---

## Cashflow Example (Fixed-Rate)

Assume: ₹1,000 par, 8% coupon, semi-annual, 3 years

| Date | Cashflow |
|---|---|
| 2025-06-30 | ₹40 |
| 2025-12-31 | ₹40 |
| 2026-06-30 | ₹40 |
| 2026-12-31 | ₹40 |
| 2027-06-30 | ₹40 |
| 2027-12-31 | ₹1,040 (coupon + principal) |

Hold to maturity and reinvest coupons at YTM → realized return ≈ YTM at purchase.

---

## Duration & Convexity (just enough for action)
- Duration: First-order sensitivity to rate changes; good for small moves
- Convexity: Second-order effect; cushions large rate moves; higher convexity bonds lose less when rates rise sharply (all else equal)
- Portfolio tip: If you think rates may rise, consider shorter duration; if you think rates will fall, longer duration benefits more

---

## Credit Ratings (shortcut guide)

| Category | Meaning | Typical Issuers |
|---|---|---|
| AAA | Highest safety | Sovereign, top PSUs, blue-chip corporates |
| AA | Very strong | Large stable corporates/PSUs |
| A | Strong but more cyclical | Mid-to-large corporates |
| BBB | Adequate; watch cycles | Smaller/leveraged firms |
| Below BBB | Speculative | High yield/risk |

Note: Ratings can change. Always track rating actions and covenants.

---

## Taxes (high-level; consult your advisor)

- Interest: Typically taxed as income at your slab; TDS may apply as per law
- Capital Gains (India): Depends on listed/unlisted status and holding period; rules can change
- TDS/Form 16A: Platform shows gross and net; statements provided for filing support

BondFlow will present tax flags and indicative post-tax yields where feasible.

---

## Why Bonds vs FDs vs Equity?

- Bonds vs FDs: Bonds can be traded (market gains/losses possible) and often offer higher yields for higher credit risk; FDs are simpler but illiquid before maturity without penalty
- Bonds vs Equity: Bonds usually lower volatility and defined cashflows; equity has higher upside potential with higher risk

Portfolio idea: Core bonds for stability/income + equities for growth = smoother long-term ride.

---

## Common Mistakes (and fixes)

- Chasing only high coupon → Check YTM and credit risk (spread vs peers)
- Ignoring call risk → A juicy coupon might be called away next year
- Forgetting accrued interest → Always compare on clean price and YTM
- One-issuer obsession → Diversify maturities, sectors, and ratings
- Selling in panic on rate spikes → If horizon matches maturity, consider holding through

---

## BondFlow Angle (how we help)
- Fractional Access: Small ticket sizes so you can ladder and diversify
- Transparent Pricing: Clean vs dirty; accrued interest auto-calculated
- RFQ Liquidity: Institutional-grade routing to tighten spreads
- Compliance & Safety: KYC/AML by design; depository settlement; on-chain audit trails
- Smart Tools: Duration/convexity, YTM comparisons, call schedules, and alerts
- Education: Bite-sized explainers, calculators, and risk nudges in-app

---

## Mini Quiz (2-minute confidence boost)
1) If price falls from ₹1,000 to ₹950 on an 8% coupon bond, what happens to YTM? → It increases.
2) A 5-year bond with duration ≈ 4. If yields jump 1%, rough price change? → About −4%.
3) Callable in 1 year at par with a high coupon and falling rates: risk? → Reinvestment risk; likely to be called.
