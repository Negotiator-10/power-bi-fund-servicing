# Power BI: Fund Servicing Analytics

Three Power BI and Excel projects that reproduce the daily reporting and controls cycle of fund servicing and trading operations: **NAV oversight**, **cash and position reconciliation**, and **trade reconciliation** for a proprietary trading desk. Each reproduces a realistic daily operational cycle rather than a trading strategy.

All figures are synthetic and generated for demonstration only.

---

## Project 1: NAV Oversight and Fund Accounting

Folder: `01_NAV_Oversight`

Recomputes each fund's NAV per share from raw accounting inputs (investments, cash, receivables, payables, accrued fees, shares) and flags any daily strike that breaches a per-fund tolerance or fails a data quality check before the NAV is released.

- 6 funds across equity, fixed income, balanced and money market
- 64 business days (April to June 2026), 384 daily strikes
- Per-fund NAV tolerances in basis points (10 bps money market to 500 bps equity)
- Four control checks: NAV move breach, stale price, negative cash, large flow
- Built in Power BI with Power Query, a star-schema data model, DAX measures, and row-level security

Files:
- `NAV_Oversight.pbix` (dashboard on the computed model)
- `NAV_Oversight_FromRaw.pbix` (rebuilt end to end from the raw accounting inputs)
- `NAV_Oversight_Model.xlsx` (formula-driven Excel model)
- `data/` source CSVs

## Project 2: Cash and Position Reconciliation

Folder: `02_Reconciliation`

Reconciles an internal book of record against a custodian statement, position by position and cash by cash, classifies every mismatch into a break type, and reports match rate, break market value exposure, and aging.

- ~126 positions across 4 funds plus cash
- Matching engine on a Fund and Security key
- Break types: missing in internal, missing in custodian, quantity, price, cash, residual market value
- Match rate, break exposure and aging surfaced in a Power BI dashboard with drill-through and row-level security

Files:
- `Cash_Position_Reconciliation.pbix` (dashboard on the computed model)
- `Reconciliation_Matching_Engine.pbix` (matching engine rebuilt natively in Power Query from the two raw books, a full-outer merge on a Fund and Security key with break classification in M)
- `Reconciliation_Model.xlsx` (formula-driven Excel model)
- `data/` source CSVs

## Project 3: Trade Reconciliation (Proprietary Trading Desk)

Folder: `03_Trade_Reconciliation`

Reconciles a proprietary desk's internal OMS trade blotter against the exchange and clearing confirmations for one trading day, matched trade by trade on a TradeID, classifies every mismatch into a break type, and separates break count from break notional exposure.

- 462 trades for a single day across equities, index futures and options, on NSE and BSE
- Matching engine on a TradeID key, ordered classification (presence, then side, quantity, price, fees)
- Break types: missing in exchange, missing in internal (unbooked fill), side, quantity, price, fees
- Match rate, break notional exposure, and a dedicated unbooked-trade count surfaced for the operations desk
- A prop book targets a much cleaner match rate (98 percent plus) than a fund custody reconciliation

Files:
- `Trade_Reconciliation_Model.xlsx` (formula-driven Excel model: raw blotter and confirmations, live break classification, and a summary)
- `data/` source CSVs (internal blotter, exchange confirmations, the computed reconciliation, and the break register)

---

## How they fit together

Projects 1 and 2 cover one coherent daily cycle a fund servicing team runs: confirm holdings against the custodian (reconciliation), then price and oversee the NAV before release. Project 3 applies the same exception-driven matching discipline to a proprietary trading desk, where the internal blotter is tied out against the exchange before settlement. All three recompute everything and surface only what needs a human to act on.

## Tools

Power BI (Power Query, data modelling, DAX, row-level security), Excel.

## A note on the .pbix files

Each `.pbix` opens and displays from data embedded at build time. Because the files were built against local paths, the **Refresh** button will not find the source CSVs after cloning; the dashboards still render correctly without refreshing. To re-enable refresh, repoint the Power Query source to the `data/` folder in your clone.
