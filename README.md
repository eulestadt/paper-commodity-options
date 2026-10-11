# Paper-trading long-dated commodity options for under $8 a month

Research memo, 11 October 2026. This repo is an ideation note: what already exists, what it costs, and how a 12-month “price stays put” option strategy on oil or gas would actually get its data. It is not a trading system, and it is not investment advice.

## Short answer

The simulation engine and the paper broker already exist, and both can be run for $0 a month. Nobody publishes a complete, licensed history of NYMEX energy option chains inside an $8 budget. The workable design is to paper-trade forward on free delayed quotes, record the chain you actually use, and spend only on a narrow historical backfill.

| Piece | Use | Monthly cost |
| --- | --- | --- |
| [NautilusTrader](https://github.com/nautechsystems/nautilus_trader) | Strategy runtime, backtest, and IBKR paper routing | $0 |
| [QuantConnect LEAN](https://github.com/QuantConnect/Lean) | Alternate engine when early exercise and assignment must be modeled explicitly | $0 locally. Their US future-options dataset is a separate paid product |
| Interactive Brokers paper account | Simulated fills in crude and natural-gas futures options | $0 with delayed NYMEX quotes |
| NYMEX top-of-book, only if delayed is not enough | Streaming L1 | $1.55, and a paper account will not earn the commission waiver |
| EIA, FRED, CFTC | Storage, rates, positioning | $0 |
| [Databento](https://databento.com/pricing) CME history | Backfill of specific settlements, quoted before download | Keep a hard cap near $5. The unlimited CME plan is $199 |
| A home machine that wakes once a day | Run the daily mark and the recorder | $0 |

A always-on VPS is optional. This strategy marks once a day for a year. It does not need a server.

## The trade, stated so it can be simulated

“Buy options that oil or gas will not change for 12 months” is a long butterfly on one dated futures contract:

1. Buy one call at a lower strike.
2. Sell two calls at a center strike near the future’s price.
3. Buy one call at a higher strike.
4. Use the expiry closest to 12 months that still has a two-sided market on both wings.

You pay a debit. The most the package can lose is that debit. It is worth the most if the underlying future expires at the center strike. Time decay and a decline in implied volatility work in favor of the position while the future stays between the wings.

A short iron butterfly is the same view entered for a credit. It adds short-option margin and the chance of being assigned a futures position. The debit butterfly matches the wording “buying options,” and it keeps the paper account’s risk equal to the premium.

### Name the contract, not “oil”

The underlier is a specific future, such as December 2027 WTI, rather than the front-month continuous series. The front month of Henry Hub natural gas is seasonal. It moves as the strip rolls from winter to summer even when the market’s price for one particular month is unchanged. A 12-month thesis on natural gas has to name the contract month. Otherwise the backtest will reject a true “that month stays put” view, or accept a false one.

WTI options are a cleaner first market. CME lists American-style Light Sweet Crude Oil options (Globex `LO`) on the `CL` future. Under NYMEX Rule 310 they expire at the close on the third business day before the underlying future expires, and they can be exercised on any business day until then. Consecutive months are listed years forward, with June and December listed further out, so a 12-month expiry is a listed product. Listing is not the same as a tight market. Open interest and the width of the quote thin out as you leave the front of the curve. The strategy needs a liquidity gate: if either wing lacks a real bid and ask, or open interest is negligible, there is no trade that day.

### Use the micro contract in the paper account

Standard WTI options cover 1,000 barrels. One cent of premium is $10. Micro WTI options (`MCO`) cover 100 barrels, settle financially, and are worth $1 per cent of premium. One butterfly on the micro is the right size for a simulated account. Scale to the standard contract only after the marks, the quotes, and the early-exercise behavior look right.

Natural gas is the second market, and only after the thesis is rewritten per contract month. The standard Henry Hub contract is 10,000 MMBtu. American options trade as `ON`. Micro Henry Hub exists for the same sizing reason as micro WTI.

### What the pricer has to get right

Price the option off the futures price with Black-76, then adjust for American exercise. An American call on a future can be exercised early. A European Black-Scholes pricer on the spot price will mis-mark both the premium and the assignment risk on the short center strike.

The paper blotter should store, for the package and for each leg:

- Debit paid, and the live package bid and ask (each leg’s bid or ask on the correct side)
- Settlement mark
- Delta, gamma, vega, and theta of the package
- Days to option expiry, which is earlier than the future’s expiry
- Whether a short leg is in the money enough that early assignment into the future is plausible

Marking the butterfly at mid price will invent an edge. Twelve-month energy wings are often several ticks wide. The simulated fill belongs at the quote you could hit, or it is not a simulation of this market.

## What is already on GitHub

Star counts were read from GitHub on 11 October 2026.

| Project | Stars | What it actually covers | Fit for this strategy |
| --- | --- | --- | --- |
| [vnpy/vnpy](https://github.com/vnpy/vnpy) | 45,815 | Python framework aimed at Chinese futures and the CTP interface | Large and active. CME energy options are not its home venue |
| [nautechsystems/nautilus_trader](https://github.com/nautechsystems/nautilus_trader) | 29,856 | Rust engine, Python strategies. Adapters include Interactive Brokers (futures and futures-option chains, paper port) and Databento | Best match for a coded strategy that paper-trades and can reload history |
| [mementum/backtrader](https://github.com/mementum/backtrader) | 23,452 | Bar-based Python backtester | Strong for single-instrument research. It does not model a futures-option chain, American exercise, or assignment |
| [QuantConnect/Lean](https://github.com/QuantConnect/Lean) | 21,963 | Event-driven engine for equities, equity options, futures, and future options, with margin and exercise models. Apache-2.0. IBKR plugin supports paper trading | Best options model. The downloadable US future-options dataset is contact-only and sized for the whole universe (about 1.5 GB daily, 400 GB at minute resolution) |
| [philipodonnell/paperbroker](https://github.com/philipodonnell/paperbroker) | 281 | Simulated options brokerage and UI | The description matches the request. Last push was April 2018, and it targets equity options |

These are popular and are the wrong market structure for NYMEX futures options: Freqtrade, Hummingbot, and Jesse (crypto), Zipline and its forks (equities), and equity-option sandboxes at Alpaca and Tradier.

Broker front-ends can place the butterfly by hand today. thinkorswim paperMoney and IBKR Trader Workstation’s paper account both understand futures options. They are manual sandboxes. They are a poor place to leave a strategy that has to choose strikes, check wing liquidity, and mark early-exercise risk every day for a year.

TradingView’s paper account is a chart simulator. It does not keep a four-leg commodity option book with greeks and assignment.

### How to read the two engines that do fit

NautilusTrader is the default runtime. One strategy definition can point at a Databento historical catalog for research and at IB Gateway’s paper port (`4002`, or TWS paper on `7497`) for the forward test. The IB adapter loads futures and futures-option chains between a minimum and maximum expiry, which is the shape of a 12-month listing. License is LGPL-3.0.

LEAN is the reference model for option lifecycle: American exercise, assignment into the future, and portfolio margin. Run it locally and feed it data you recorded or bought. QuantConnect’s free cloud plan lists futures and options among its asset classes, which is enough for a smoke test. Local bulk access to US future options is a paid organization feature, and the minute-level universe is hundreds of gigabytes. That dataset is the wrong shape for an $8 cap.

Before trusting a multi-year Nautilus backtest of this butterfly, check one known case: a short American call on a future that is deep in the money before expiry. If the engine never assigns it, switch that study to LEAN or add the exercise rule. The forward paper test at IBKR does not have this ambiguity. IBKR’s paper matcher applies the broker’s own exercise handling.

## Data, and what “all of it” means

A 12-month butterfly needs less data than a market-making stack, and more data than a futures-only backtest.

**To price and mark the position**

- The underlying future itself: bid, ask, settlement, volume, open interest. Symbol is the dated contract (`CLZ27`), never a stitched continuous series, during the life of the option.
- Each of the three strikes, call and put if you later compare a put butterfly: bid, ask, last, settlement, volume, open interest, and the vendor’s implied vol when they publish one.
- The rest of that expiry’s chain, at least daily, so wing width is chosen from real strikes and the skew is visible.
- Contract specs: multiplier, tick, exercise style, option expiry, and the future the option delivers or references.
- A financing curve (FRED treasury or SOFR) for Black-76 discounting.
- The exchange calendar, including the three-business-day gap between option expiry and futures expiry.

**To decide whether the thesis is on**

- EIA weekly petroleum status and weekly natural gas storage. Free API key.
- A realized-volatility measure from the future’s own settlements, plus OVX if you want the listed crude-vol index as context.
- CFTC Commitments of Traders, free files.
- The futures curve from the front month out past the option’s expiry, so a move in the front is not mistaken for a move in the contract you sold the body against.

**What you can skip at this budget**

Full order-book history, tick trades, and every energy product on the exchange. Those are what push CME data into the $199-a-month plan. A butterfly that is re-marked daily never reads them.

### Where each series comes from

Delayed NYMEX quotes are free in an IBKR account. The campus market-data notes describe about a 10-minute delay, plus snapshot quotes at $0.03 for non-equity instruments with a $1 monthly waiver. Snapshots are a poor way to archive a chain: a few dozen strikes, every session, use up the waiver immediately. Delayed streaming while the daily job is connected, written to your own files, is the $0 archive going forward.

Real-time NYMEX top-of-book is listed at $1.55 per month for a non-professional. The listed commission waiver is $20 a month, which a paper account does not generate, so budget the $1.55 if you subscribe. The broader US futures bundle is $10 before add-ons and sits above this cap. Paid market-data subscriptions also expect a funded live account (IBKR describes a $500 balance to keep subscriptions). Delayed data avoids that funding step. Confirm in Client Portal that any paid NYMEX package you select includes futures options before turning it on.

IBKR’s historical API is a reasonable source for recent bars and a poor source for a decade of option surfaces. Pace limits and incomplete option history make it the wrong backfill.

Databento sells CME Globex (CME, CBOT, NYMEX, COMEX), including futures and options, on a usage meter, and quotes the cost before the download. New accounts receive $125 in credits, usable on historical data, expiring after six months. Daily bars or settlement statistics for one micro-WTI expiry and its wings are a small request. Unrestricted live CME access on the Standard plan moved to $199 a month on 22 June 2026. That plan solves a different problem.

Barchart, CME DataMine, and exchange-direct cloud feeds are the professional versions of the same history. They are the right vendors if the study later needs the full surface. They are not the $8 path.

Public websites that show a delayed chain are fine for a human to look at. Scraping them into a research store breaks the vendor’s terms and produces gaps. Use a licensed feed for anything the strategy will trust.

## A daily job, once this leaves the memo

The software left to write is small, because the broker simulator should stay at IBKR.

1. **Chain recorder.** Once each session, after the energy settlement window, pull the micro-WTI future that is about 12 months out and every listed strike of that option expiry. Store bid, ask, settlement, volume, and open interest. This file is the dataset from that day forward.
2. **Butterfly builder.** Center strike is the future’s prior settlement, rounded to a listed strike. Wing width is the narrowest listed distance at which both wings have a two-sided quote and open interest above a floor you choose (start by inspecting the chain; do not hard-code a width from crude’s front-month market). Skip the day when that floor fails.
3. **Paper order.** One three-leg limit order, debit capped. Never a market order. Size is one micro contract.
4. **Blotter.** Package bid, ask, and theoretical value; greeks; distance of the future to each strike; a flag when a short leg is deep in the money.
5. **Thesis log.** EIA storage surprise and the future’s realized volatility, written next to the position so a later review can tell “the vol view was wrong” from “the fill model was wrong.”

Hold to expiry for the first paper cohort. That is the experiment. Early exits (future outside the wings, package bid already near max value) can be a second cohort, so the two rules are not mixed in one result.

Backfill, if you want one before the year of paper trading starts: ask Databento for a quote on daily statistics for `MCO` and the matching micro future, for the strikes and the two or three years you care about. Accept the quote only if it lands inside the credit or inside the monthly cap. If the quote is for the full book of every energy option, narrow the request. You do not need that file.

## Budget that stays under $8

| Choice | Cost | When to pick it |
| --- | --- | --- |
| Delayed IBKR paper, local Nautilus, EIA/FRED/CFTC, record your own chain | $0 | Default. The honest test of a 12-month option is the next 12 months |
| Add NYMEX L1 | $1.55 | You want live quotes and you have accepted the funding requirement for paid data |
| Add a capped Databento backfill | inside a $5 cap, after the $125 credit | You want a historical rehearsal before the paper year |
| QuantConnect future-options subscription, Databento CME Standard, a futures depth feed | $10 to $199 and up | Out of scope for this constraint |

Running IB Gateway on a small VPS is possible and usually annoying. Gateway wants a desktop session. A machine you already use, with a scheduled daily run, fits the strategy better.

## What this memo is not recommending

Building a new matching engine, a new greeks library, or a new options brokerage UI. LEAN, Nautilus, and IBKR paper already split that work. The missing piece on a tight budget is a narrow, licensed history plus a recorder that starts now.

Starting on natural gas front-month continuous prices. Seasonality will dominate the result.

Backtesting the butterfly on the future’s price alone, with a made-up implied volatility. The trade’s P&L is the option premium. A spot backtest cannot see the wings, the skew, or the assignment.

Paying for real-time depth. A position held for a year does not read the order book.

## Open points before any code

- Futures and futures-options permissions on the IBKR account. Paper trading follows those permissions. Approval is a process, separate from the monthly cost.
- The open-interest floor and the maximum wing quote width, chosen after looking at a live `MCO` chain about 12 months out.
- Whether the first study is paper-forward only ($0) or paper-forward plus a quoted Databento backfill.
- A single known-answer test for early exercise before any historical P&L is treated as real.

## Sources checked on 11 October 2026

- QuantConnect LEAN: <https://github.com/QuantConnect/Lean>, <https://www.lean.io/lean/>
- LEAN US future options bulk data: <https://www.quantconnect.com/docs/v2/lean-cli/datasets/quantconnect/download-in-bulk/us-future-options>
- NautilusTrader and its IBKR adapter: <https://github.com/nautechsystems/nautilus_trader>, <https://nautilustrader.io/docs/latest/integrations/interactive_brokers/>
- IBKR market data pricing: <https://www.interactivebrokers.com/en/pricing/market-data-pricing.php>
- IBKR permissions and snapshot quotes: <https://www.interactivebrokers.com/campus/trading-lessons/trade-permissions-mkt/>
- NYMEX exchange-fee schedule, including crude and natural-gas futures options: <https://www.interactivebrokers.com/en/accounts/fees/NYMEX.php>
- NYMEX Rule 310, Light Sweet Crude Oil Option: <https://www.cmegroup.com/content/dam/cmegroup/rulebook/NYMEX/3/310.pdf>
- CME energy options codes, including `LO`, `MCO`, and `ON`: <https://www.cmegroup.com/markets/energy/energy-options.html> and <https://www.cmegroup.com/markets/energy/files/wti-crude-options-fact-card.pdf>
- Databento CME options and pricing: <https://databento.com/options>, <https://databento.com/pricing>, <https://databento.com/docs/faqs/usage-pricing-and-data-credits>, and the 29 May 2026 subscription update <https://databento.com/blog/updates-to-subscription-pricing>
