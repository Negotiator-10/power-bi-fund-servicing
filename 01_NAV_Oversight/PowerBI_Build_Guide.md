# Power BI Build Guide: NAV Oversight Dashboard

This guide builds a Power BI report on top of the NAV Oversight model. The Excel
model already computes and validates every NAV and exception, so Power BI can
connect straight to it and focus on presentation. A second section gives the DAX
if you want to rebuild the logic natively inside Power BI.

Recommended path: build Option A first so you have a working dashboard, then add
the DAX measures from Option B to show data-model depth.

---

## Data source

Files (all under this project folder):

- `NAV_Oversight_Model.xlsx`, sheet `NAV Engine` (the computed daily strikes)
- `data\fund_master.csv` (fund reference data)
- `data\capital_activity.csv` (subscriptions and redemptions)

---

## Option A: load the computed CSV (fast, correct)

Important: load `data\nav_engine.csv`, NOT the Excel sheet. Power BI reads a file
without opening it in Excel, and the Excel model's computed columns are formulas
whose results are not cached, so they import as blanks. `nav_engine.csv` has every
value already calculated and also carries the fund reference data, so you do not
need a separate Fund Master query. (If you would rather use the Excel file, open it
in Excel once and save it first, which writes the results in.)

1. Home, Get Data, Text/CSV, select `data\nav_engine.csv`.
2. Click Transform Data. On the query:
   - Confirm `Date` is Date type, `NAVPerShare` and money columns are Decimal,
     `NAVChangePct` is Percentage, `FlagCount` and `ToleranceBps` are Whole Number.
   - Rename the query to `NAV`.
3. Close and Apply. Every column, including FundName, Strategy and the flags, is
   populated. The Fund Master relationship is not needed; Strategy is in this table.

### Data model

- Create a dedicated date table (Modeling, New Table):

  ```DAX
  DimDate =
  ADDCOLUMNS (
      CALENDAR ( DATE ( 2026, 4, 1 ), DATE ( 2026, 6, 30 ) ),
      "Year", YEAR ( [Date] ),
      "Month", FORMAT ( [Date], "MMM YYYY" ),
      "MonthNo", YEAR ( [Date] ) * 100 + MONTH ( [Date] )
  )
  ```

- Mark `DimDate` as a date table on the `Date` column.
- Relationships:
  - `DimDate[Date]` one-to-many to `NAV[Date]`
  - `Fund Master[FundCode]` one-to-many to `NAV[FundCode]`
  - `Fund Master[FundCode]` one-to-many to `Flows[FundCode]`
- Sort `DimDate[Month]` by `MonthNo`.

### Core measures

```DAX
Fund Strikes Reviewed = COUNTROWS ( NAV )

-- exceptions are counted at the flag level: one strike can trip two checks, so
-- Total Exceptions (9) can exceed the number of flagged strikes (8). This ties to
-- the exceptions-by-type and exceptions-by-fund visuals, which also count flags.
Total Exceptions = SUM ( NAV[FlagCount] )

Strikes Flagged = CALCULATE ( COUNTROWS ( NAV ), NAV[FlagCount] >= 1 )

NAV Move Breaches =
CALCULATE ( COUNTROWS ( NAV ), NAV[BreachFlag] = "NAV MOVE BREACH" )

Match Rate =
DIVIDE ( [Fund Strikes Reviewed] - [Strikes Flagged], [Fund Strikes Reviewed] )

Latest Net Assets =
VAR d = MAX ( NAV[Date] )
RETURN CALCULATE ( SUM ( NAV[NetAssets] ), NAV[Date] = d )

Exception Rate = DIVIDE ( [Strikes Flagged], [Fund Strikes Reviewed] )
```

---

## Report layout

One page, four zones. Use the fund navy `#1F3864` as the accent and a light grey
canvas. Keep to two decimals on percentages and no decimals on currency.

### Top KPI row (four cards)

| Card | Measure | Format |
|---|---|---|
| Fund strikes reviewed | Fund Strikes Reviewed | whole number |
| Total exceptions | Total Exceptions | whole number |
| NAV move breaches | NAV Move Breaches | whole number |
| Total net assets | Latest Net Assets | currency, no decimals |

### Left column

- **NAV per share trend**: line chart. Axis `DimDate[Date]`, value
  `Average of NAVPerShare`, legend `NAV[FundCode]`. This is the headline visual,
  so give it the most space.
- **Net assets by fund**: clustered bar. Axis `FundCode`, value `Latest Net Assets`.

### Right column

- **Exceptions by type**: clustered column. So the count ties to Total Exceptions
  (9), build a proper exceptions table first. In Power Query, right-click the `NAV`
  query, Reference, keep only `Date`, `FundCode` and the four flag columns
  (`BreachFlag`, `StalePriceFlag`, `NegCashFlag`, `LargeFlowFlag`), select those
  four columns, Transform, Unpivot Columns, then filter the resulting Value column
  to remove blanks. Name it `Exceptions` and Close and Apply. Relate
  `Exceptions[FundCode]` to `Fund Master[FundCode]`. The chart is then axis
  `Exceptions[Value]`, value `Count of Value`. It sums to 9.
- **Exception detail table**: table visual showing Date, FundCode, FundName,
  NAVPerShare, NAVChangePct, ToleranceBps, FlagCount, PrimaryException. Add a
  visual-level filter `FlagCount is greater than or equal to 1` so only flagged
  strikes show (8 rows; the 28 May FIIG row shows FlagCount 2). Sort by Date
  descending. Apply conditional formatting: background red where PrimaryException
  contains BREACH or NEGATIVE.

### Slicers (top of page)

- `DimDate[Month]` as a dropdown
- `Fund Master[Strategy]` as buttons
- `Fund Master[FundCode]` as a dropdown

---

## Option B: rebuild the NAV logic in Power BI (shows DAX depth)

If you would rather compute NAV from the raw feeds instead of the Excel model,
import `data\daily_financials.csv` as `Fin` and `data\fund_master.csv`, then add
these calculated columns on `Fin`.

```DAX
TotalAssets   = Fin[InvestmentsMV] + Fin[Cash] + Fin[Receivables]
TotalLiabs    = Fin[Payables] + Fin[AccruedMgmtFee]
NetAssets     = Fin[TotalAssets] - Fin[TotalLiabs]
NAVPerShare   = DIVIDE ( Fin[NetAssets], Fin[SharesOutstanding] )

PriorNAV =
VAR pd =
    CALCULATE (
        MAX ( Fin[Date] ),
        FILTER ( Fin, Fin[FundCode] = EARLIER ( Fin[FundCode] )
                   && Fin[Date] < EARLIER ( Fin[Date] ) )
    )
RETURN
    CALCULATE (
        MAX ( Fin[NAVPerShare] ),
        FILTER ( Fin, Fin[FundCode] = EARLIER ( Fin[FundCode] )
                   && Fin[Date] = pd )
    )

NAVChangePct = DIVIDE ( Fin[NAVPerShare] - Fin[PriorNAV], Fin[PriorNAV] )

ToleranceBps =
LOOKUPVALUE ( 'Fund Master'[NAVToleranceBps],
              'Fund Master'[FundCode], Fin[FundCode] )

BreachFlag =
IF ( ISBLANK ( Fin[PriorNAV] ), "",
     IF ( ABS ( Fin[NAVChangePct] ) * 10000 > Fin[ToleranceBps],
          "NAV MOVE BREACH", "" ) )
```

The results match the Excel model exactly. Use whichever source you present; do
not load both into the same visuals.

---

## Talking points for the interview

- The dashboard mirrors the daily NAV oversight cycle: recompute each fund's NAV
  from accounting inputs, compare the day-over-day move to a per-fund tolerance,
  and surface only the strikes that need a human before release.
- Tolerances are per fund and expressed in basis points, tighter for the money
  market fund (10 bps) than for equity funds (500 bps), which is why the same
  percentage move is an exception for one fund and normal for another.
- Every exception has a type, so the reviewer knows whether it is a price issue,
  a data quality issue, or a large flow, not just that something moved.
- The four checks (NAV move breach, stale price, negative cash, large flow) are
  the kinds of controls a fund accounting or NAV oversight team runs before sign
  off.

All data is synthetic and built for demonstration.
