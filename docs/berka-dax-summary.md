# Berka DAX measure reference
### Retail Banking Analytics, 1993 to 1998

Every definition below was read from the live Power BI model, not from notes. 77 measures across 10 display folders, all in a dedicated `_Measures` table.

---

## Contents

| Folder | Count | Covers |
|---|---|---|
| 01 Core | 8 | Transaction and entity counts |
| 02 Balances | 11 | Semi-additive balances, distribution, segmentation |
| 03 Credit | 12 | Loan risk, right-censoring correction |
| 04 Products | 9 | Penetration and behaviour |
| 05 District | 6 | Socio-economics and correlation |
| 06 Distress | 3 | Watchlist |
| 07 Role-playing dates | 3 | `USERELATIONSHIP` patterns |
| 08 Clients via bridge | 6 | Many-to-many resolution |
| 09 Scenario | 5 | What-if parameters |
| 10 Helpers | 8 | Dynamic titles and colours |
| 99 Deprecated | 3 | Redundant, pending deletion |

---

## 01 Core

```dax
Transaction Count = COUNTROWS ( fact_transaction )
```

```dax
Total Inflow =
CALCULATE ( SUM ( fact_transaction[amount] ), dim_transaction_type[is_credit] = TRUE )
```

```dax
Total Outflow =
CALCULATE ( SUM ( fact_transaction[amount] ), dim_transaction_type[is_credit] = FALSE )
```

```dax
Net Flow = [Total Inflow] - [Total Outflow]
```

```dax
Transaction Value = [Total Inflow] + [Total Outflow]
```

```dax
Avg Transaction Value = AVERAGE ( fact_transaction[amount] )
```

```dax
Avg Daily Transaction Value =
DIVIDE ( [Transaction Value], DISTINCTCOUNT ( fact_transaction[date_key] ) )
```

Uses a daily average rather than a sum so February is not penalised for being short. Feeds the seasonality heatmap.

```dax
Account Count = DISTINCTCOUNT ( dim_account[account_key] )
```

```dax
Client Count = DISTINCTCOUNT ( dim_client[client_key] )
```

**Note on both counts:** they iterate dimensions, which sit on the one side of every relationship. Filters travel dimension to fact, never back, so neither responds to a date slicer. That is correct model behaviour, not a bug. Use `[Cumulative Accounts Opened]` where a time-aware account stock is needed.

---

## 02 Balances

The semi-additive group. Balances cannot be summed across time: twelve monthly snapshots added together return roughly twelve times the real figure, and the error looks plausible enough to ship.

```dax
Closing Balance =
LASTNONBLANKVALUE ( dim_date[full_date], SUM ( fact_balance_snapshot[closing_balance] ) )
```

Reads the last date with data in the current filter context. Only the **end** of a date range affects it, which is correct for a point-in-time balance.

```dax
Total Deposits =
VAR _LastKey = MAX ( fact_balance_snapshot[month_start_key] )
RETURN
    CALCULATE (
        SUM ( fact_balance_snapshot[closing_balance] ),
        fact_balance_snapshot[month_start_key] = _LastKey
    )
```

Same semi-additive result without a time-intelligence function. Pins to the latest snapshot month present in context.

```dax
Avg Monthly Balance = AVERAGE ( fact_balance_snapshot[closing_balance] )
```

Mean across account-months, distinct from the mean at a point in time.

```dax
Avg Balance per Account = DIVIDE ( [Closing Balance], [Account Count] )
```

```dax
Median Balance =
VAR _LastKey = MAX ( fact_balance_snapshot[month_start_key] )
RETURN
    CALCULATE (
        MEDIANX (
            VALUES ( fact_balance_snapshot[account_key] ),
            CALCULATE ( SUM ( fact_balance_snapshot[closing_balance] ) )
        ),
        fact_balance_snapshot[month_start_key] = _LastKey
    )
```

**Rebuilt during the dashboard build.** The original iterated `VALUES ( dim_account[account_key] )`, all 4,500 accounts regardless of filter. In 1993 only 1,139 had snapshot rows, so the measure returned blank for three of six years and a wrong value for a fourth. Iterating the fact instead fixes it.

```dax
Median vs Mean Gap % =
DIVIDE ( [Avg Balance per Account] - [Median Balance], [Avg Balance per Account] )
```

A steady 10 to 12% gap across all years: right-skewed, but not extremely so.

```dax
Deposits YoY % =
VAR _Current = [Closing Balance]
VAR _Prior = CALCULATE ( [Closing Balance], DATEADD ( dim_date[full_date], -1, YEAR ) )
RETURN
    DIVIDE ( _Current - _Prior, _Prior )
```

The `_Current` underscore prefix is not stylistic. `Current` is reserved in DAX.

```dax
Avg Balance YoY % =
VAR _Curr = [Avg Balance per Account]
VAR _Prior = CALCULATE ( [Avg Balance per Account], DATEADD ( dim_date[full_date], -1, YEAR ) )
RETURN
    DIVIDE ( _Curr - _Prior, _Prior )
```

```dax
Accounts in Overdraft =
CALCULATE (
    DISTINCTCOUNT ( fact_balance_snapshot[account_key] ),
    fact_balance_snapshot[closing_balance] < 0
)
```

```dax
Low Balance Accounts =
VAR _LastKey = MAX ( fact_balance_snapshot[month_start_key] )
RETURN
    CALCULATE (
        DISTINCTCOUNT ( fact_balance_snapshot[account_key] ),
        fact_balance_snapshot[month_start_key] = _LastKey,
        fact_balance_snapshot[closing_balance] < 5000
    )
```

The `month_start_key` pin matters. Without it, an account that dipped below 5,000 once in 1994 counts even if it has held 80,000 since. The 5,000 threshold is a chosen number, unlike `[Accounts in Overdraft]` where zero is a real boundary.

```dax
Low Balance Share % =
VAR _LastKey = MAX ( fact_balance_snapshot[month_start_key] )
VAR _AllAccts =
    CALCULATE (
        DISTINCTCOUNT ( fact_balance_snapshot[account_key] ),
        fact_balance_snapshot[month_start_key] = _LastKey
    )
RETURN
    DIVIDE ( [Low Balance Accounts], _AllAccts )
```

Divides by accounts present in the snapshot, not by `[Account Count]`. In 1993 that is 1,139 accounts, not 4,500. Using the full count would have reported 1.6% instead of the true 6.5%.

### Dynamic segmentation

```dax
Accounts by Balance Band =
VAR _MinB = MIN ( 'Balance Band'[Min] )
VAR _MaxB = MIN ( 'Balance Band'[Max] )
RETURN
    CALCULATE (
        DISTINCTCOUNT ( fact_balance_snapshot[account_key] ),
        FILTER (
            VALUES ( fact_balance_snapshot[account_key] ),
            VAR _B = CALCULATE ( AVERAGE ( fact_balance_snapshot[closing_balance] ) )
            RETURN _B > _MinB && _B <= _MaxB
        )
    )
```

Classifies accounts into bands at query time against a disconnected `Balance Band` table. Bands are editable without touching the pipeline, and the measure recalculates per filter context.

**The limitation worth knowing:** a disconnected table only affects measures that explicitly read it. For a slicer that filters the whole page, a real `dim_account[balance_band]` calculated column was added separately. The two coexist: the disconnected version drives the histogram axis, the column drives the slicer.

---

## 03 Credit

```dax
Loan Count = COUNTROWS ( fact_loan_cohort )
```

```dax
Loan Book Value = SUM ( fact_loan_cohort[loan_amount] )
```

Cumulative **disbursed**, never outstanding. The source has no repayment schedule, so loans never come off the book. Worth stating wherever this appears.

```dax
Avg Loan Amount = DIVIDE ( [Loan Book Value], [Loan Count] )
```

```dax
Defaulted Loans = CALCULATE ( [Loan Count], fact_loan_cohort[is_default] = TRUE )
```

Default is status B (finished, unpaid) or D (running, in debt).

```dax
Default Rate =
IF (
    NOT ISBLANK ( [Loan Count] ),
    DIVIDE ( COALESCE ( [Defaulted Loans], 0 ), [Loan Count] )
)
```

**The blank-versus-zero fix, and the most consequential bug found in the build.**

`[Defaulted Loans]` returns blank when a district has no defaults, and `DIVIDE ( BLANK, 13 )` is blank rather than 0%. Blank rows are silently excluded from charts, sorts and averages, so every district with a clean record was dropping out of the league table, the band chart, and the correlation. The report was only ever showing districts that had at least one default, biasing everything upward.

Clearest case: Louny, 13 loans, zero defaults, 8.23% unemployment, rendered empty.

The measure now returns 0% where loans exist and blank only where there are none, so a clean district is visibly distinct from an empty one.

```dax
Default Rate (Matured Only) =
CALCULATE ( [Default Rate], fact_loan_cohort[is_fully_matured] = TRUE )
```

**The right-censoring correction.** Raw default falls from roughly 20% in the 1993 cohort to 2.5% in 1998, which looks like improving credit quality. It is not. The 1998 loans have had months to fail rather than years.

The dashboard plots both series together and lets the gap be the point. Portfolio-wide: 11.1% raw against 13.2% matured only.

```dax
Default Rate at 12 Months =
CALCULATE ( [Default Rate], fact_loan_cohort[months_on_book] >= 12 )
```

A fixed-window alternative to the matured-only cut.

```dax
Amount at Risk =
CALCULATE ( SUM ( fact_loan_cohort[loan_amount] ), fact_loan_cohort[is_default] = TRUE )
```

```dax
Amount at Risk Share % = DIVIDE ( [Amount at Risk], [Loan Book Value] )
```

```dax
Avg Affordability Ratio = AVERAGE ( fact_loan_cohort[payment_to_district_salary] )
```

Monthly instalment divided by district average salary, computed in the pipeline. A better burden proxy than anything reconstructable in DAX, since the dataset has no income variable.

```dax
Avg Balance at Origination = AVERAGE ( fact_loan_cohort[balance_at_origination] )
```

Minimum is negative: some accounts were already overdrawn when the loan was written.

```dax
Loan Book Cumulative =
CALCULATE (
    SUM ( fact_loan_cohort[loan_amount] ),
    FILTER ( ALL ( dim_date ), dim_date[full_date] <= MAX ( dim_date[full_date] ) )
)
```

Converts a flow into a stock so it can be plotted against deposits on one axis. Without it, monthly lending at a few million against deposits at 197 million collapses to a flat line.

---

## 04 Products

```dax
Cards Held = COUNTROWS ( dim_card )
Card Penetration % = DIVIDE ( [Cards Held], [Account Count] )
```

```dax
Accounts with Loan =
CALCULATE (
    DISTINCTCOUNT ( fact_account_behaviour[account_key] ),
    fact_account_behaviour[has_loan] = TRUE
)
Loan Penetration % = DIVIDE ( [Accounts with Loan], [Account Count] )
```

```dax
Standing Order Count = COUNTROWS ( fact_order )
Standing Order Value = SUM ( fact_order[amount] )
```

`fact_order` has no date column in the source, so both are static under any date filter. Label them accordingly.

```dax
Avg Transactions per Month = AVERAGE ( fact_account_behaviour[txns_per_month] )
Avg Recency (Days) = AVERAGE ( fact_account_behaviour[recency_days] )
```

`fact_account_behaviour` holds as-at-1998-12-31 aggregates with no date relationship, deliberately. Both are static.

```dax
Accounts Holding Product =
SWITCH (
    SELECTEDVALUE ( 'Product'[Product] ),
    "Accounts", [Account Count],
    "Standing orders", COUNTROWS ( FILTER ( fact_account_behaviour, fact_account_behaviour[n_standing_orders] > 0 ) ),
    "Cards", COUNTROWS ( FILTER ( fact_account_behaviour, fact_account_behaviour[n_cards] > 0 ) ),
    "Loans", [Accounts with Loan],
    "Joint access", COUNTROWS ( FILTER ( fact_account_behaviour, fact_account_behaviour[has_disponent] = TRUE ) )
)
```

A second disconnected-table pattern, built so a product-holding chart has one consistent denominator. **Every branch counts accounts, not items.** The obvious alternative, `[Standing Order Count]`, counts 6,471 orders against 4,500 accounts, and five bars where one has a different denominator is a chart that quietly lies.

Results: Accounts 4,500 / Standing orders 3,758 / Cards 892 / Loans 682 / Joint access 869.

---

## 05 District

```dax
Avg Unemployment 96 = AVERAGE ( dim_district[unemployment_96] )
Avg Salary = AVERAGE ( dim_district[avg_salary] )
Avg Urban Share = AVERAGE ( dim_district[pct_urban] )
Avg Entrepreneurs per 1000 = AVERAGE ( dim_district[entrepreneurs_per_1000] )
```

```dax
Crime Rate per 1000 =
DIVIDE ( SUM ( dim_district[crimes_96] ), SUM ( dim_district[n_inhabitants] ) ) * 1000
```

`crimes_96` is a raw count. Charted unconverted it just shows where the people live.

```dax
Corr Unemployment Default =
VAR _MinLoans = 5
VAR _T =
    FILTER (
        ADDCOLUMNS (
            VALUES ( dim_district[district_name] ),
            "@N", [Loan Count],
            "@X", [Avg Unemployment 96],
            "@Y", COALESCE ( [Default Rate], 0 )
        ),
        [@N] >= _MinLoans && NOT ISBLANK ( [@X] )
    )
VAR _N = COUNTROWS ( _T )
VAR _MX = AVERAGEX ( _T, [@X] )
VAR _MY = AVERAGEX ( _T, [@Y] )
VAR _Cov = SUMX ( _T, ( [@X] - _MX ) * ( [@Y] - _MY ) )
VAR _SX = SQRT ( SUMX ( _T, ( [@X] - _MX ) ^ 2 ) )
VAR _SY = SQRT ( SUMX ( _T, ( [@Y] - _MY ) ^ 2 ) )
RETURN
    IF ( _N >= 5, DIVIDE ( _Cov, _SX * _SY ) )
```

Pearson correlation computed in DAX, with three deliberate guards:

- **`COALESCE` on the Y value.** Without it, districts with zero defaults are excluded and the entire low end of the axis disappears.
- **A minimum loan count.** Median loans per district is 7, so unfiltered the measure correlates noise.
- **`IF ( _N >= 5 )`.** Returns blank rather than a meaningless coefficient on a tiny sample.

**Result: r ≈ 0.04.** Verified at three thresholds: -0.03 across all 77 districts, +0.04 at minimum 5 loans (n = 65), +0.10 at minimum 10 (n = 17). Near zero however it is cut.

This is a legitimate null finding. The caveat that travels with it: district rates built on a median of 7 loans are too noisy to detect a relationship either way, so "no correlation found" is not "no relationship exists".

---

## 06 Distress

```dax
Accounts in Distress = DISTINCTCOUNT ( account_distress[account_key] )
```

```dax
Distress Share % = DIVIDE ( [Accounts in Distress], [Account Count] )
```

Both static. `account_distress` reaches `dim_date` only through `dim_account`, which has no active date path.

```dax
Distress Flags Raised =
CALCULATE (
    [Accounts in Distress],
    USERELATIONSHIP ( account_distress[first_distress_date_key], dim_date[date_key] )
)
```

The period-aware version, using a relationship added during the dashboard build and kept **inactive** on purpose: `account_distress` already reaches `dim_date` through `dim_account`, and a second active path would be ambiguous.

**Interpretive discipline.** Sanction interest is charged *because* an account is already failing. It is a symptom, not an origination-time predictor. Fine on a timeline, invalid in any forward-looking risk claim.

---

## 07 Role-playing dates

Three date columns need `dim_date` but cannot all hold active relationships to it. The inactive ones are reached on demand.

```dax
Accounts Opened =
CALCULATE (
    COUNTROWS ( dim_account ),
    USERELATIONSHIP ( dim_account[opened_date_key], dim_date[date_key] )
)
```

```dax
Cards Issued =
CALCULATE (
    COUNTROWS ( dim_card ),
    USERELATIONSHIP ( dim_card[issued_date_key], dim_date[date_key] )
)
```

```dax
Cumulative Accounts Opened =
CALCULATE (
    [Accounts Opened],
    FILTER ( ALL ( dim_date ), dim_date[full_date] <= MAX ( dim_date[full_date] ) )
)
```

Flow versus stock. `Accounts Opened` gives new accounts per year (1,139 / 439 / 661 / 1,363 / 898 / 0); `Cumulative` gives accounts on the books (1,139 / 1,578 / 2,239 / 3,602 / 4,500 / 4,500). Both return 4,500 unfiltered, which is correct and not a duplication.

---

## 08 Clients via bridge

One account can have an owner and a disponent; one client can hold several accounts. `bridge_disposition` resolves the many-to-many. Reaching client attributes from transaction facts needs the bridge filtering in both directions, enabled per measure rather than permanently.

```dax
Transactions by Client Attribute =
CALCULATE (
    [Transaction Count],
    CROSSFILTER ( bridge_disposition[client_key], dim_client[client_key], BOTH )
)
```

```dax
Balance by Client Attribute =
CALCULATE (
    [Closing Balance],
    CROSSFILTER ( bridge_disposition[client_key], dim_client[client_key], BOTH )
)
```

`CROSSFILTER` inside `CALCULATE` scopes the bidirectional filter to one evaluation. A permanently bidirectional relationship would introduce ambiguity across the whole model.

```dax
Owner Count =
CALCULATE ( DISTINCTCOUNT ( bridge_disposition[client_key] ), bridge_disposition[is_owner] = TRUE )

Disponent Count =
CALCULATE ( DISTINCTCOUNT ( bridge_disposition[client_key] ), bridge_disposition[is_owner] = FALSE )
```

4,500 owners and 869 disponents, summing to 5,369 clients. A disponent has access to someone else's account without owning it.

**The decision this forces:** card penetration is 892 over 5,369 (16.6%) or 892 over 4,500 (19.8%). Both are true, three points apart, and a reader assumes whichever one you did not pick unless you say. All 892 cards went to owners, so including disponents inflates the denominator without adding a single cardholder.

```dax
Accounts per Client = DIVIDE ( [Account Count], [Client Count] )
```

```dax
Clients by District (Client Home) =
CALCULATE (
    [Client Count],
    USERELATIONSHIP ( dim_client[district_key], dim_district[district_key] )
)
```

Client home district rather than account district. The two differ, and every district visual in the report uses account district unless the title says otherwise.

---

## 09 Scenario

Two what-if parameter tables drive a simple margin model.

```dax
Interest Rate Value = SELECTEDVALUE ( 'Interest Rate'[Interest Rate] )
Loss Given Default Value = SELECTEDVALUE ( 'Loss Given Default'[Loss Given Default] )
```

```dax
Scenario Interest Income = [Loan Book Value] * 'Interest Rate'[Interest Rate Value]
Scenario Expected Loss = [Amount at Risk] * 'Loss Given Default'[Loss Given Default Value]
Scenario Net Margin = [Scenario Interest Income] - [Scenario Expected Loss]
```

Illustrative rather than a credit model. The dataset has no interest rates or recovery data, so both inputs are reader-supplied assumptions.

---

## 10 Helpers

```dax
Selected Period Label =
IF (
    HASONEVALUE ( dim_date[calendar_year] ),
    "Year " & SELECTEDVALUE ( dim_date[calendar_year] ),
    "All years, 1993-1998"
)
```

The most useful piece of chrome in the report. A reader who cannot tell whether they are looking at a filtered view will misread every total on the page.

```dax
Dynamic Title = "Deposits and Lending, " & [Selected Period Label]
```

```dax
Trajectory Title =
VAR _Acct = SELECTEDVALUE ( dim_account[account_key] )
RETURN
    IF (
        ISBLANK ( _Acct ),
        "Balance trajectory: select a row below",
        "Balance trajectory, account " & _Acct
    )
```

`IF ( ISBLANK ( ... ) )` rather than the second argument of `SELECTEDVALUE`, so the prompt text also reads sensibly when several rows are selected.

```dax
District Header = SELECTEDVALUE ( dim_district[district_name], "All districts" )
Account Header = "Account " & SELECTEDVALUE ( dim_account[account_key], "-" )
```

Tooltip page headings.

```dax
Default Rate Colour =
SWITCH (
    TRUE (),
    [Default Rate] > 0.15, "#FD625E",
    [Default Rate] > 0.10, "#F2C80F",
    "#01B8AA"
)
```

```dax
Status Colour =
SWITCH (
    TRUE (),
    SELECTEDVALUE ( fact_loan_cohort[is_default] ) = TRUE (), "#FD625E",
    SELECTEDVALUE ( fact_loan_cohort[is_fully_matured] ) = TRUE (), "#01B8AA",
    "#118DFF"
)
```

Applied through Format style **Field value** in conditional formatting. Driving colour off `is_default` rather than a hard-coded status letter means the chart uses the same flag as `[Default Rate]` and cannot drift out of agreement with the KPI card above it.

Hex is unavoidable here. A measure cannot reference a theme swatch by name, so these are Power BI's standard sentiment values. Change a sentiment colour in the theme customiser and both measures need updating.

```dax
Data Coverage Note =
"Payment purpose is recorded for " &
FORMAT (
    DIVIDE (
        CALCULATE ( [Transaction Count], NOT ISBLANK ( dim_transaction_type[k_symbol_cz] ) ),
        [Transaction Count]
    ),
    "0.0%"
) & " of transactions."
```

Surfaces a data-quality limitation on the canvas instead of burying it in a README.

---

## 99 Deprecated

```dax
Loans Issued            -- identical to [Loan Count]
Loan Amount Issued      -- identical to [Loan Book Value]
Default Rate (Issued)   -- identical to [Default Rate]
```

All three wrap `USERELATIONSHIP` on `fact_loan_cohort[date_key]`, which turned out to be an **active** relationship. `USERELATIONSHIP` on an already-active relationship is a no-op, so they return the same values as the plain measures.

Hidden and foldered rather than deleted, because a deleted measure breaks any visual referencing it silently. Delete once page 1 is confirmed clean.

---

## Calculated columns supporting these measures

| Column | Table | Purpose |
|---|---|---|
| `amount_band` (+ hidden sort) | `fact_loan_cohort` | under 100K / 100-250K / over 250K |
| `duration_band` (+ hidden sort) | `fact_loan_cohort` | 12-24m / 36m / 48-60m |
| `unemployment_band` (+ hidden sort) | `dim_district` | under 2% / 2-4% / 4-6% / 6% and over |
| `balance_band` (+ hidden sort) | `dim_account` | Slicer counterpart to the disconnected band table |
| `Month label` | `dim_date` | `FORMAT ( month_start, "MMM yy" )`, sorted by `year_month` |

**On `dim_account[balance_band]`:** it bands each account's average month-end balance across all months, computed at refresh. It does not respond to the Period slicer. That is the price of a slicer that filters the whole page, and it belongs in the slicer's tooltip so nobody mistakes it for a live calculation.

---

## Cross-validation

Cumulative net transaction flow: **197,151,289 CZK**
Total closing balance from the snapshot table: **197,140,234 CZK**
Difference: **0.006%**

Two independent computation paths, one summing over a million transaction rows and one reading month-end snapshots, reconciling to within eleven thousand on 197 million.

---

## Conventions

- All measures live in a dedicated `_Measures` table, foldered `01` through `10`
- Format strings set at measure level; display units set per visual, so the same measure can show millions on a card and full precision in a tooltip
- Variables prefixed `_` throughout. `Current` is reserved in DAX, which is where the convention started
- `DIVIDE` everywhere rather than `/`, for safe division
- Every derived measure references other measures rather than repeating base logic, so a fix propagates once
