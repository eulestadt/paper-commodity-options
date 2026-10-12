# IBKR setup checklist (paper trading + API)

Use this while we walk through the browser. You enter every password and personal detail yourself.

## Path

1. **Open a live individual account** (IBKR Pro). Paper trading is unlocked after the live account is approved. Funding is not required to unlock paper; funding is required later for paid market-data subscriptions.
2. **Request a Paper Trading account** in Client Portal → Settings → Account Configuration → Paper Trading Account. Create a separate paper username and password.
3. **Trading permissions** on the live account (paper mirrors them): US Stocks, US Options (for stock options such as MU), and US Futures / Futures Options (for energy products). Approval can take one to a few business days.
4. **Install IB Gateway** (lighter than full Trader Workstation) for API access. Log in with the **paper** credentials and choose Paper Trading. Socket port should stay **4002**.
5. In Gateway: Configuration → Settings → API → Settings → enable **ActiveX and Socket Clients**, leave **Read-Only API unchecked**, keep localhost-only on.

## Ports (do not mix)

| Session | TWS | IB Gateway |
| --- | --- | --- |
| Live | 7496 | 4001 |
| Paper | 7497 | 4002 |

Any automation we build must target the paper port only until you explicitly say otherwise.

## After paper is live

- Confirm the paper username (often looks like `p……`) in Client Portal.
- Log into paper once in Gateway so the API socket is listening.
- Dump one option chain (for example micro WTI or MU LEAPs) with delayed data before placing any paper orders.
