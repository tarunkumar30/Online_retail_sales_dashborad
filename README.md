# Online Retail II – Sales Analysis (Power BI)

A one-page Power BI dashboard built on the **Online Retail II** dataset, a real two-year transaction log from a UK online gift-ware retailer. The report tracks sales, returns, orders, customers, products, countries and time trends, and sits on a star-schema data model with 15 DAX measures.

**Author:** Tarun Kumar T.
**Tools:** Power BI Desktop, Power Query, DAX, Excel (source data)

---

## Business problem

A retailer with many wholesale customers and strong seasonal swings needs a quick way to answer:

- How much are we selling, and how much comes back as returns?
- Which products, countries and customer groups drive revenue?
- When do sales peak, and how does this year compare with last year?

The dashboard answers these in one view, with Year, Quarter and Country slicers.

## Headline numbers

All values come from the dashboard and were re-calculated from the model data (they match).

| KPI | Value |
|---|---|
| Total Sales (excludes returns) | 19.64M |
| Net Revenue (sales minus returns) | 18.93M |
| Total Orders | 39,516 |
| Average Order Value | 497.11 |
| Active Customers (registered) | 5,852 |
| Return Rate (by value) | 3.65% |

Period covered: 1 Dec 2009 to 9 Dec 2011. Currency is sterling (GBP) according to the dataset documentation; the report itself shows no currency symbol.

## Dataset

- **Name:** Online Retail II
- **Source:** UCI Machine Learning Repository, https://archive.ics.uci.edu/dataset/502/online+retail+ii
- **Citation:** Chen, D. (2019). *Online Retail II* [Dataset]. UCI Machine Learning Repository. Licensed CC BY 4.0.
- **File used:** `online_retail_II.xlsx`, two sheets (2009-2010 and 2010-2011), 1,067,371 rows combined, 8 columns:
  `Invoice, StockCode, Description, Quantity, InvoiceDate, Price, Customer ID, Country`
- **Not included in this repo** because of its size (about 45 MB). Download it from the link above.

## What I did

1. **Combined and profiled** the two Excel sheets.
2. **Cleaned the data** and split it into one fact table and three dimension tables (exported as CSV).
3. **Loaded the tables** into Power BI with Power Query and set the data types.
4. **Built the model**: one fact table, three dimensions, three relationships.
5. **Wrote 15 DAX measures** for sales, returns, orders, customers and time comparisons.
6. **Designed the dashboard** and checked the KPIs against the data.

### Data cleaning

| Step | Rows affected | Why |
|---|---|---|
| Remove exact duplicate rows | 34,335 | Repeated lines, many from the December 2010 overlap between the two sheets |
| Remove zero or negative prices | 6,019 | Stock write-offs and missing-description rows, not sales |
| Remove non-product stock codes | 5,763 | Postage, bank charges, manual adjustments, Amazon fees, test codes and similar |
| **Rows kept in `fact_sales`** | **1,021,254** | |

Other decisions:

- **Cancellations** (invoices starting with `C`) are kept and flagged `IsReturn = TRUE`, with negative revenue.
- **Missing Customer ID** (about 23% of raw rows) is kept and set to `0`, labelled **Guest**. Guests account for 13.11% of sales.
- **Lowercase stock codes** (for example `85123a`) are converted to uppercase so each product has one code.

> **Note on how these numbers were obtained:** the Power BI file loads ready-made CSVs, so the cleaning happened before Power BI and the script is not in the project file. I reconstructed the steps above by comparing the raw workbook with `fact_sales`. They reproduce the model's row count, gross sales and returns exactly, but the original cleaning method should be confirmed before you publish this.

## Data model

| Table | Role | Grain | Key |
|---|---|---|---|
| `fact_sales` | Fact | One invoice line | `Invoice` + `StockCode` (no single key column) |
| `dim_date` | Dimension | One day (761 days, Dec 2009 to Dec 2011) | `Date` |
| `dim_product` | Dimension | One product (4,738) | `StockCode` |
| `dim_customer` | Dimension | One customer (5,876, incl. Guest) | `CustomerID` |
| `Table_mesure`, `Time Intelligence` | Empty tables that hold measures | n/a | n/a |

All three relationships are **many-to-one, single direction, active**: `fact_sales[Date]` to `dim_date[Date]`, `fact_sales[StockCode]` to `dim_product[StockCode]`, `fact_sales[CustomerID]` to `dim_customer[CustomerID]`. There is no separate country table; `Country` is a column in `fact_sales` and `dim_customer`.

## Key DAX measures

| Measure | Formula |
|---|---|
| Total Sales | `CALCULATE(SUM(fact_sales[Revenue]), fact_sales[IsReturn] = FALSE())` |
| Total Returns | `ABS(CALCULATE(SUM(fact_sales[Revenue]), fact_sales[IsReturn] = TRUE()))` |
| Net Revenue | `[Total Sales] - [Total Returns]` |
| Return Rate % | `DIVIDE([Total Returns], [Total Sales])` |
| Total Orders | `CALCULATE(DISTINCTCOUNT(fact_sales[Invoice]), fact_sales[IsReturn] = FALSE())` |
| Avg Order Value | `DIVIDE([Total Sales], [Total Orders])` |
| Active Customers | `CALCULATE(DISTINCTCOUNT(fact_sales[CustomerID]), fact_sales[IsReturn] = FALSE(), fact_sales[CustomerID] <> 0)` |
| Sales LY | `CALCULATE([Total Sales], SAMEPERIODLASTYEAR(dim_date[Date]))` |
| Sales PM | `CALCULATE([Total Sales], DATEADD(dim_date[Date], -1, MONTH))` |
| MoM Growth % | `DIVIDE([Total Sales] - [Sales PM], [Sales PM])` |

Also in the model: `Units Sold`, `Revenue per Customer`, `YoY Growth %`, `Sales YTD`, `Sales Rolling 3M`. These are defined but not used on the dashboard page.

Note that **Total Sales is gross** (returns are excluded, not subtracted). Use Net Revenue when you want sales after returns.

## What the data shows

- **Sales are seasonal.** September to November made up 36.2% of 2010 sales and 37.8% of 2011 sales (to 9 Dec). November is the peak: about 1.43M in 2010 and 1.45M in 2011.
- **Growth is modest.** Comparing the same window (Jan to Nov), 2011 was 3.0% above 2010 (8.86M vs 8.60M).
- **The UK dominates.** The UK is 85.5% of sales (16.8M). Next are EIRE (0.6M), Netherlands (0.5M), Germany (0.4M) and France (0.3M).
- **Product sales are spread out.** The top ten products add up to about 8% of sales. The top one, Regency Cakestand 3 Tier, is 0.33M.
- **Thursday and Tuesday are the busiest days** (about 4.0M and 3.9M). Saturday has almost no sales (one Saturday with trades in the whole dataset) and Sunday is about 9% of sales.
- **Guest checkouts are 13.11% of sales** (2,922 orders) and registered customers are 86.89%.

## Known issues and things to check

Being upfront about these is part of the project:

1. **"MoM Growth % by Year and Quarter" is not true month-over-month growth.** At quarter level the measure compares each quarter with a three-month window shifted back one month, so the two windows overlap. I reproduced all eight bar values this way. A monthly axis would be a clearer fit for this measure.
2. **December 2011 is incomplete** (data ends 9 Dec). This drives the sharp drop at the end of the trend line and most of the -11.55% in Q4 2011.
3. **Two of the top-ten products are driven by single cancelled orders.** *Paper Craft, Little Birdie* (one order of 80,995 units, 168,470) and *Medium Ceramic Top Storage Jar* (one order of 74,215 units, 77,184) were both cancelled in full the same day. The product chart uses Total Sales, which ignores returns, so they still rank. Net revenue for these two is close to zero.
4. **Return rate depends on the cleaning choice.** Removing non-product codes (including large manual and fee lines) lowers the return rate to 3.65%. With those rows kept it would be about 7.1% on my reconstruction of the raw data.
5. **Guest is not one customer.** Customer ID `0` stands in for all missing IDs.
6. **Four identical-looking row pairs remain** in `fact_sales`. In the raw data they differ by description text or by one minute, so I treated them as genuine lines.
7. **Not verified:** whether `dim_date` is marked as the model's date table, and whether Auto date/time is switched off. Hidden `LocalDateTable_*` tables suggest Auto date/time is on.
8. **Source path:** the report reads CSV files from a local folder on the author's machine. After cloning, update the paths in Power Query (Home > Transform data > Data source settings).

## Ideas for next steps

- Switch the MoM visual to a monthly axis.
- Add a Net Revenue product ranking next to the gross one.
- Turn off Auto date/time and mark `dim_date` as the date table.
- Add a return-analysis page and a customer-segmentation page (for example RFM).
- Rename `Table_mesure` to `Measures` and put measures in display folders.

## Repository contents (suggested)

```
├── README.md
└── Online_Retail_Project.pbix
```

## Skills shown

Data cleaning and reconciliation, star-schema modelling, Power Query typing, DAX (filter context, `CALCULATE`, time intelligence), dashboard design, and checking KPIs against source data.
