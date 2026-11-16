# Sales Performance Dashboard

A two-page Power BI sales report with executive KPIs, a selectable metric trend, customer ranking, and a category drill-through page.

The report has an **Executive Overview** page and a **Category Details** drill-through page (layout described below).

## Business problem

The brief asked for a sales report with an open visual design, provided a set of required KPIs, and assessed three things: building a model from the tables supplied, writing DAX measures, and designing a usable, well-structured report.

Mandatory items in the brief were:

- customer rank by sales, using `RANK` or `RANKX`
- order shipping time (days from order to ship)
- a drill-through page of sales details for a selected category

Where possible, the brief also asked for a month/year view, a selectable dimension trend, a dynamic currency symbol, a last-refresh stamp, a message when a filter returns no data, colour-blind-safe design, and a subcategory breakdown with year-on-year change.

## Dataset and source

The model was built from the assessment workbook (not included in this repository). Tables in the model:

| Table | Role |
| --- | --- |
| `Sales` | Fact table of sales order lines (`Sales Amount`, `Order Quantity`, `OrderDate`, `ShipDate`) |
| `Date` | Date dimension, including fiscal year, `Calendar Year`, and `Start of Month` |
| `Customer` | Customer dimension |
| `Product`, `Category` | Product hierarchy |
| `Sales Territory` | Region, country, and group (North America, Europe, Pacific, Corporate HQ) |
| `Reseller`, `Sales Order` | Reseller attributes and order channel |
| `-Measures` | Dedicated measure table |
| `Metric Selector` | Field parameter: Sales, Quantity, Ship Days |
| `Metric Choice` | Field parameter: Total Sales, Total Quantity, Avg Shipping Days |

From the imported model (121,253 sales order lines):

- Order dates run from **1 July 2017 to 15 June 2020**. The date dimension extends to **30 June 2021** and includes fiscal years **FY2018–FY2021**.
- **18,485** customers and **397** products across Bikes, Components, Clothing, and Accessories.
- Territory countries include the United States, Canada, Australia, the United Kingdom, France, and Germany.

## Tools and skills

- Power BI Desktop
- Star-schema modelling from multiple related tables
- DAX (time intelligence, `RANKX`, field parameters, no-data handling)
- Drill-through between report pages
- Slicers for calendar year, territory group, and country
- Custom theme (`ocean-professional-theme.json`, Ocean Professional): navy, blue, and amber, documented for contrast rather than red/green

## Data model

```
Date[Date]            ←  Sales[OrderDate]     (active)
Date[DateKey]         ←  Sales[OrderDateKey]  (inactive)
Customer              ←  Sales
Product / Category    ←  Sales
Sales Territory       ←  Sales
Reseller              ←  Sales
Sales Order           ←  Sales
                         |
                    -Measures
                    Metric Selector
                    Metric Choice
```

A calendar-year slicer and a territory group/country slicer sit on the Executive Overview page. The Category Details page is bound as a **drill-through** on `Category[Category]`.

`Metric Selector` drives the KPI value and the Top 10 titles (`Sales`, `Quantity`, `Ship Days`). `Metric Choice` is used on the Performance Trend Over Time chart (`Total Sales`, `Total Quantity`, `Avg Shipping Days`).

## Key DAX measures

All measures sit on the `-Measures` table and were extracted from the `.pbix` data model.

```dax
Total Sales = SUM ( Sales[Sales Amount] )

Total Quantity = SUM ( Sales[Order Quantity] )

Avg Shipping Days =
AVERAGEX (
    Sales,
    DATEDIFF ( Sales[OrderDate], Sales[ShipDate], DAY )
)
```

```dax
Customer Rank =
IF ( -- Hide rank on total / subtotal rows
    ISINSCOPE ( 'Customer'[Customer] ),
    RANKX (
        ALLSELECTED ( 'Customer'[Customer] ), -- respects slicers
        [Total Sales],
        ,
        DESC,
        Dense -- ties do not skip numbers (1, 2, 2, 3)
    )
)
```

```dax
Sales PY =
CALCULATE (
    [Total Sales],
    DATEADD ( 'Date'[Date], -1, YEAR )
)

Sales YoY = [Total Sales] - [Sales PY]

Sales YTD =
TOTALYTD (
    [Total Sales],
    'Date'[Date]
)
```

```dax
Sales YoY % =
VAR YearsWithData =
    FILTER (
        VALUES ( 'Date'[Calendar Year] ),
        NOT ISBLANK ( [Total Sales] ) && [Total Sales] <> 0
    )
VAR LatestYearWithData = MAXX ( YearsWithData, 'Date'[Calendar Year] )
VAR CurrentSales =
    CALCULATE ( [Total Sales], 'Date'[Calendar Year] = LatestYearWithData )
VAR PriorSales =
    CALCULATE ( [Total Sales], 'Date'[Calendar Year] = LatestYearWithData - 1 )
RETURN
    DIVIDE ( CurrentSales - PriorSales, PriorSales )
```

```dax
Selected Metric =
VAR SelectedCount = COUNTROWS ( 'Metric Selector' )
VAR SelectedName = MAX ( 'Metric Selector'[Metric Selector] )
RETURN
    IF (
        SelectedCount = 1,
        SWITCH (
            SelectedName,
            "Sales", [Total Sales],
            "Quantity", [Total Quantity],
            "Ship Days", [Avg Shipping Days],
            [Total Sales]
        ),
        [Total Sales] -- default when no slicer button is selected
    )
```

```dax
Last Refresh = FORMAT ( NOW (), "dd MMM yyyy" )

No Data Warning =
IF (
    ISBLANK ( [Total Sales] ),
    "⚠️ No data available for the selected filter combination.",
    ""
)

Category Header Text =
VAR CurrentCategory = SELECTEDVALUE ( 'Category'[Category] )
RETURN
    IF (
        ISBLANK ( CurrentCategory ),
        "Select a Category",
        CurrentCategory & " - Sales Details"
    )

Top 10 Products Title =
VAR SelectedName = SELECTEDVALUE ( 'Metric Selector'[Metric Selector], "Sales" )
RETURN
    "Top 10 Products by " & SelectedName

Active Metric Name =
SELECTEDVALUE ( 'Metric Selector'[Metric Selector], "Sales" )
```

The `Date` table also has two calculated columns: `Calendar Year = YEAR ( 'Date'[Date] )` and `Start of Month = STARTOFMONTH ( 'Date'[Date] )`.

## Dashboard pages

**1. Executive Overview** (`Sales Performance Overview`)

| Visual | What it shows |
| --- | --- |
| KPI cards | `Selected Metric`, `Avg Shipping Days`, `Sales YoY %`, and `Last Refresh` |
| `Performance Trend Over Time` | Line chart over `Date[Month]`, driven by `Metric Choice` |
| `Top 10 Customers` | Customers ranked with `Customer Rank` and the selected metric |
| `Top 10 Products` | Products by the selected metric |
| `Performance by Product Category` | Sales by category (drill-through starts here) |
| Slicers | `Date[Calendar Year]`, `Sales Territory[Group]`, and `Country` |
| `No Data Warning` | Message card when a filter selection returns no data |

**2. Category Details** (drill-through on Category, with a Back button)

| Visual | What it shows |
| --- | --- |
| Header / KPI cards | `Category Header Text`, plus `Total Sales`, `Total Quantity`, and `Avg Shipping Days` |
| `Yearly Sales Trend` | `Total Sales` by calendar year / start of month |
| Bar charts | Sales by country (territory) and by product subcategory |
| Table | Product, quantity, and sales for the drilled category |

## Key insights

Figures below were calculated from the imported model (sum of `Sales Amount`, mean of line-level ship days, and the same date range as `OrderDate`).

- **Overall.** The model contains **$109.8 million** of sales across **121,253** order lines, from 1 July 2017 to 15 June 2020.
- **Category mix.** **Bikes account for 86.2%** of sales (**$94.6 million**). Components are 10.7% ($11.8 million). Clothing and Accessories together are about 3.1%.
- **Trend.** The largest full calendar year is **2019, at $42.9 million**, **40.6% above** 2018 ($30.5 million). 2017 is a half year (July–December, $11.9 million). 2020 is a partial year (orders through 15 June) at $24.5 million. On a same-period basis (1 January–15 June), 2020 sales were $24.5 million, **71.0% above** the $14.3 million recorded in the same period of 2019. (The `Sales YoY %` measure compares the latest year that has sales with the year before it, so with no year slicer it compares 2020 with the full year 2019.)
- **Territory.** **North America accounts for 72.3%** of sales (**$79.4 million**). The United States alone is 57.4% ($63.0 million), and the Southwest region is the largest territory at 22.0% ($24.2 million).
- **Shipping time and channel.** Average shipping time is **7.0 days** — every line with a ship date has a 7-day lag (2,113 lines have no ship date). The Reseller channel is **73.3%** of sales ($80.5 million) versus 26.7% ($29.4 million) for Internet, even though the two channels have a similar number of order lines.

## How to open the file

1. Install [Power BI Desktop](https://www.microsoft.com/power-platform/products/power-bi/desktop).
2. Open `Sales Performance.pbix`. The model is already imported inside the file, so the assessment Excel workbook is not required to view the report.
3. To try drill-through, right-click a category in **Performance by Product Category** and choose the Category Details page.

To apply the theme in a new report: **View → Themes → Browse for themes** and select `ocean-professional-theme.json`.

## Author

[Sana Adnan](https://github.com/sanaadnanc) · Data Analyst, Sydney
