# Introduction

This project explores customer and product performance using the AdventureWorks dataset, with the goal of answering a practical business question: **who are our best customers, and which products actually drive the business?** Using DuckDB to query the raw AdventureWorks CSV files directly, I built two analytical datasets — one on customer behavior, one on product performance — and brought both into Power BI to turn the numbers into a dashboard I could actually explore.

# Background

This project was driven by the following core questions:

1. How does the business perform overall — total revenue, total orders, quantity sold, and product count?
2. Which product categories generate the most revenue, orders, and customer interest?
3. How does revenue vary by product color?
4. Which individual products drive the most revenue and orders?
5. How are customers segmented, and how many customers fall into each segment?
6. Which regions have the most customers, orders, and revenue?
7. Who are the top individual customers by revenue and orders?
8. How does customer segment relate to order volume?

### Data source
The data behind this analysis comes from the AdventureWorks sample database (AdventureWorksDW/OLTP CSV exports), covering product, category, sales order, and customer tables.

# Tools Used

To build this analysis, I relied on:

- **DuckDB** — the core of my analysis, used to query and join the raw AdventureWorks CSV files directly with SQL, without needing a full database server
- **SQL** — for cleaning, joining, aggregating, and window-function calculations (revenue per region, revenue per category, customer segmentation logic)
- **Power BI** — for building the interactive dashboards from the exported analytics CSVs
- **Git & GitHub** — for version control and sharing the project

# The Analysis

## 1. Product Analytics

### How does the business perform overall, and which categories and colors drive revenue?

To answer this, I built a `product_details` → `product_sales` → `product_metrics` pipeline in DuckDB: joining product, subcategory, and category tables, then joining in sales order detail and header data to calculate revenue, order count, quantity sold, average selling price, and number of unique customers per product. A parallel `category_metrics` CTE rolled the same measures up to the category level, so each product row could be compared against its category total.

### Query

```sql
WITH product_details AS (
    SELECT
        p.ProductID, p.ProductNumber, p.Name AS ProductName,
        c.Name AS CategoryName, s.Name AS SubcategoryName,
        p.Color, p.ListPrice
    FROM read_csv_auto('Production Product.csv') p
    LEFT JOIN read_csv_auto('Production ProductSubcategory.csv') s
        ON p.ProductSubcategoryID = s.ProductSubcategoryID
    LEFT JOIN read_csv_auto('Production ProductCategory.csv') c
        ON s.ProductCategoryID = c.ProductCategoryID
),
product_sales AS (
    SELECT d.ProductID, h.SalesOrderID, h.CustomerID, d.OrderQty, d.UnitPrice, d.LineTotal
    FROM read_csv_auto('Sales SalesOrderDetail.csv') d
    LEFT JOIN read_csv_auto('Sales SalesOrderHeader.csv') h
        ON d.SalesOrderID = h.SalesOrderID
)
-- product-level and category-level metrics joined and exported to Product_Analytics.csv
```

*(Full query, including the `category_metrics` rollup and final join, is in `/sql` in this repo.)*

### Result

![Product Analytics Dashboard](Image
/Product_Analysis_Dashboard.png)

*Power BI dashboard showing total revenue, orders, quantity sold, product count, and breakdowns by category, product, and color.*

### Insights

- The business generated **$1.27M in total revenue** across **41K orders**, moving **62K units** across a catalog of just **29 products** — a small, concentrated product line rather than a long tail.
- **Bikes** is the dominant category by a wide margin — highest in customers-per-category, quantity sold, and share of total orders — with Accessories, Clothing, and Components trailing well behind.
- Revenue by color shows a clear concentration at the top: **black-colored products lead revenue generation**, with a steep drop-off after the top few colors, suggesting color/finish plays a real role in purchase decisions for this catalog.
- The revenue-vs-orders scatter by product highlights that a handful of products account for disproportionately high order volume and revenue relative to the rest of the catalog — classic 80/20 behavior worth digging into for restocking/marketing priority.
- Category share of orders (pie chart) confirms Bikes as the volume driver, with Accessories as a secondary contributor and Clothing/Components making up the smallest slices.

## 2. Customer Analytics

### How are customers segmented, and which customers/regions matter most?

For this, I built a `customer_sales` → `customer_metrics` → `customer_lifespan` pipeline: joining customer, person, sales order, and territory tables to get revenue, order count, average order value, quantity purchased, and first/last purchase dates per customer. I then calculated `CustomerLifespanMonths` and average monthly spend, and used a `CASE` statement to bucket customers into **New, Regular, Loyal, and VIP** segments based on lifespan and total revenue thresholds.

### Query

```sql
WITH customer_sales AS (
    SELECT
        c.CustomerID, c.PersonID, p.FirstName, p.LastName,
        h.SalesOrderID, CAST(CAST(OrderDate AS TIMESTAMP) AS DATE) AS OrderDate,
        d.ProductID, d.OrderQty, d.LineTotal, t.Name AS Region
    FROM read_csv_auto('Sales Customer.csv') c
    LEFT JOIN read_csv_auto('Person Person.csv') p ON c.PersonID = p.BusinessEntityID
    LEFT JOIN read_csv_auto('Sales SalesOrderHeader.csv') h ON c.CustomerID = h.CustomerID
    LEFT JOIN read_csv_auto('Sales SalesOrderDetail.csv') d ON h.SalesOrderID = d.SalesOrderID
    LEFT JOIN read_csv_auto('Sales SalesTerritory.csv') t ON c.TerritoryID = t.TerritoryID
)
-- customer_metrics + customer_lifespan CTEs calculate revenue, orders, lifespan,
-- then a CASE statement assigns each customer to New / Regular / Loyal / VIP
```

*(Full query, including the segmentation `CASE` logic, is in `/sql` in this repo.)*

### Result

![Customer Analytics Dashboard](Dashboard/image/Customer_Analytics_Dashboard.png)

*Power BI dashboard showing customer segments, regional distribution, top customers, and revenue/orders by region.*

### Insights

- Out of **19.82K total customers**, the **New** segment is by far the largest group, followed by Regular, VIP, and Loyal — suggesting either strong recent customer acquisition or a segmentation threshold that classifies most of the base as "New" by default.
- Despite being the largest segment by headcount, **New customers also generate the highest total order volume (11.9K)**, ahead of VIP (8.7K), Regular (6.8K), and Loyal (4.9K) — worth checking whether this reflects genuinely high early engagement or simply the size of the segment.
- The **Southwest region leads on both customer count and order volume**, with Northwest and Australia as the next-largest markets — the business shows a clear geographic concentration rather than an even spread.
- The top-10 customers by revenue and orders bar chart shows a fairly gradual decline rather than one or two extreme outliers — revenue is spread across a solid group of high-value customers rather than dependent on a single account.
- Revenue by region broadly follows the same ranking as order count by region, reinforcing that the Southwest/Northwest markets aren't just ordering more often but are also the largest revenue contributors.

# What I Learned

Working through this project pushed my skills forward in a few concrete ways:

- **SQL-first analytics with DuckDB** — writing multi-CTE pipelines (raw joins → metrics → rollups) directly against CSV files, without needing to stand up a full database server first.
- **Window functions in practice** — using `SUM() OVER (PARTITION BY ...)` to get category/region totals alongside row-level detail in a single query, rather than a separate aggregation step.
- **Designing a segmentation model** — translating a business idea ("who are our best customers?") into concrete, defensible `CASE` logic based on lifespan and revenue thresholds.
- **Dashboard design in Power BI** — turning exported analytical CSVs into a dashboard that surfaces KPIs (revenue, orders, quantity, product count) alongside drillable breakdowns by category, color, region, and segment.

# Insights

Pulling the two analyses together:

- **The business is concentrated, not spread thin** — a small catalog (29 products) dominated by one category (Bikes), and revenue concentrated in a handful of regions (Southwest, Northwest, Australia) rather than evenly distributed globally.
- **Customer value and customer count don't automatically align** — the New segment is both the largest group and the highest-order-volume segment, which raises a natural follow-up question about how "New" customers convert into Loyal/VIP over time.
- **Product and customer data tell a consistent story** — the same handful of categories, colors, and regions show up as the strongest performers across both dashboards, suggesting the business has a fairly well-defined core market rather than diffuse demand.

# Conclusion

This project took raw AdventureWorks CSVs, ran them through a DuckDB-based SQL pipeline to build two purpose-built analytical datasets, and turned them into an interactive Power BI view of the business. Beyond the specific findings — Bikes as the dominant category, Southwest as the leading region, New customers as the largest and most active segment — this project was as
much practice in **building a repeatable SQL-to-BI workflow** as it was in the analysis itself: from raw CSV, to modeled metrics in DuckDB, to a dashboard someone could actually use to make decisions.
