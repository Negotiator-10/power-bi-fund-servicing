# Power BI Build Guide: Reconciliation Dashboard

This guide builds a Power BI report on the Cash and Position Reconciliation model.
The Excel model runs the matching engine and classifies every break, so Power BI
connects to it and presents the result. A DAX section lets you rebuild the match
logic natively if you want to show that depth.

Recommended path: build Option A first, then add Option B measures.

---

## Data source

- `Reconciliation_Model.xlsx`, sheet `Reconciliation` (one row per position with
  break type, aging, and status already computed)
- Supporting sheets `Break Register` and `Parameters` if you want them visible

---

## Option A: load the computed CSV (fast, correct)

Important: load `data\reconciliation.csv`, NOT the Excel sheet. The Excel model's
matching engine is built from formulas whose results are not cached, so Power BI
would import them as blanks. `reconciliation.csv` has every position, break type,
and aging value already calculated. (To use the Excel file instead, open it in
Excel once and save it first.)

1. Home, Get Data, Text/CSV, select `data\reconciliation.csv`.
2. Click Transform Data. Confirm types:
   - `FirstIdentified` Date, `DaysOpen` and `IsBreak` Whole Number,
     `AbsMVDiff` and the MV columns Decimal, `PriceDiffBps` Decimal.
   - Rename the query to `Recon`.
3. Close and Apply.

### Core measures

```DAX
Positions Reconciled = COUNTROWS ( Recon )

Open Breaks = SUM ( Recon[IsBreak] )

Match Rate =
DIVIDE ( [Positions Reconciled] - [Open Breaks], [Positions Reconciled] )

Break MV Exposure =
CALCULATE ( SUM ( Recon[AbsMVDiff] ), Recon[IsBreak] = 1 )

Oldest Break (days) =
CALCULATE ( MAX ( Recon[DaysOpen] ), Recon[IsBreak] = 1 )
```

---

## Report layout

One page. Match rate and break exposure are the headline numbers a reconciliation
lead looks at first. Use the fund navy `#1F3864` accent and reserve red for breaks
only.

### Top KPI row (four cards)

| Card | Measure | Format |
|---|---|---|
| Match rate | Match Rate | percentage, one decimal |
| Open breaks | Open Breaks | whole number |
| Break MV exposure | Break MV Exposure | currency, no decimals |
| Oldest break | Oldest Break (days) | whole number, suffix "days" |

Put a target line or a colour rule on Match Rate: green above 95 percent, amber
90 to 95, red below.

### Middle row

- **Breaks by type**: bar chart. Axis `Recon[BreakType]` filtered to remove
  MATCHED, value `Open Breaks`. This is the primary triage visual.
- **Break aging profile**: column chart. Axis `Recon[AgeBucket]`, value
  `Open Breaks`. Set the axis sort order manually to 0-2, 3-5, 6-10, 11-30, 30+.

### Bottom row

- **Break detail table**: table visual with Fund, SecurityID, SecurityName,
  AssetClass, BreakType, QtyDiff, MVDiff, DaysOpen, Status. Visual-level filter
  `IsBreak is 1`. Sort by DaysOpen descending so the oldest exposures are on top.
  Conditional formatting: red data bar on AbsMVDiff, red text where BreakType is
  MISSING IN CUSTODIAN or MISSING IN INTERNAL.

### Slicers

- `Recon[Fund]` as buttons
- `Recon[BreakType]` as a dropdown
- `Recon[Status]` as buttons (Open, Investigating)

---

## Option B: rebuild the matching engine in Power BI (shows DAX depth)

Import `data\internal_book.csv` as `Internal` and `data\custodian_statement.csv`
as `Custodian`. Both carry a `Key` column (Fund and SecurityID). Build a single
`Recon` table from the union of keys, then classify with DAX.

Simplest approach that stays inside the tool:

1. In Power Query, reference `Internal`, keep `Key`, append the `Key` column from
   `Custodian`, then Remove Duplicates. Name the result `Keys`.
2. Add calculated columns on `Keys`:

```DAX
IntQty  = LOOKUPVALUE ( Internal[Quantity],    Internal[Key],  Keys[Key], 0 )
IntMV   = LOOKUPVALUE ( Internal[MarketValue], Internal[Key],  Keys[Key], 0 )
CustQty = LOOKUPVALUE ( Custodian[Quantity],   Custodian[Key], Keys[Key], 0 )
CustMV  = LOOKUPVALUE ( Custodian[MarketValue],Custodian[Key], Keys[Key], 0 )

InInternal   = IF ( CONTAINS ( Internal,  Internal[Key],  Keys[Key] ), 1, 0 )
InCustodian  = IF ( CONTAINS ( Custodian, Custodian[Key], Keys[Key] ), 1, 0 )

MVDiff = Keys[IntMV] - Keys[CustMV]

BreakType =
SWITCH (
    TRUE (),
    Keys[InInternal] = 0, "MISSING IN INTERNAL",
    Keys[InCustodian] = 0, "MISSING IN CUSTODIAN",
    Keys[IntQty] <> Keys[CustQty], "QUANTITY BREAK",
    ABS ( Keys[MVDiff] ) > 1, "MV BREAK",
    "MATCHED"
)
```

This reproduces the core classification. The Excel model adds the price bps
tolerance and the cash break label, which you can layer in with the same pattern.
Present one source only.

---

## Talking points for the interview

- The dashboard reconciles the internal book of record against the custodian
  statement position by position and cash by cash, which is the core control in
  fund administration and custody operations.
- Every mismatch is classified, not just counted: missing on one side, quantity,
  price, cash, or residual market value. The type tells the analyst what to chase.
- Breaks are aged from a first-identified date, so the oldest exposures rise to
  the top. Aging is what turns a break list into a workflow.
- The match rate and break market-value exposure are the two numbers a
  reconciliation lead watches: how clean is the book, and how much money is
  currently in question.
- Status (Open, Investigating) mirrors the exception workflow an operations team
  runs day to day.

All data is synthetic and built for demonstration.
