# If this moves from a memo to a paper account

The first running slice is a daily recorder and a one-contract micro-WTI butterfly in an IBKR paper account. No new brokerage, no cloud cluster, no full-surface purchase.

## Order of work

1. Open the IBKR account path that unlocks a paper login, and request futures plus futures-options permissions. Confirm the paper Gateway answers on port `4002`.
2. With delayed data only, dump one `MCO` chain about 12 months out. Record the bid, ask, and open interest of the at-the-money strike and of strikes one, two, and three listings away. That dump sets the wing width and the open-interest floor. Those two numbers are inputs, not guesses.
3. Run NautilusTrader against the paper port. Load the future and the futures-option chain with an expiry window around 12 months.
4. Submit one three-leg call butterfly as a limit debit, size one micro. Persist the order, the fills, and the package bid/ask.
5. Schedule the same dump after the energy settlement, every session. Append to a local parquet or csv store. That store is the history.
6. Add EIA weekly crude stocks and a 20-day realized volatility of the underlying future beside the blotter, so the thesis and the fill are reviewed separately.
7. Only then, request a Databento quote for daily statistics on that same micro future and the strikes the rule would have held. Buy it if the quote fits in the signup credit or under the monthly cap.

## Early-exercise check

Before reading any backtest P&L as evidence:

- Pick a historical date when a short American WTI call was deep in the money with time left.
- Confirm the engine either assigns the future or documents that it is ignoring early exercise.
- If it ignores exercise, treat historical P&L as a European approximation and keep the IBKR paper account as the record that includes assignment.

## Explicitly later

Natural gas, standard-size `LO` (1,000 barrels), iron condors, and any live order. Each one changes margin, seasonality, or the data budget.
