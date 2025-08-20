# Ecommerce-Data-Analysis
Every click, every cart, every order tells a story 
# 🛒 E-Commerce Sales Analysis (SQL Project)

## 📌 Project Overview

I wanted to analyze customer behavior, product performance, and delivery efficiency for an **online retail business**.
Using **SQL Server**, I explored:

* Customer purchase patterns (repeat buyers, seasonal trends)
* High-selling products and profitability
* Delivery timelines and shipping delays
* Discounts vs. profit margins

From raw transactions ➝ to **trends** ➝ to **actionable insights for growth**.

---

## 📊 Data Exploration & Cleaning

### Check Schema & Records

```sql
EXEC sp_help ecommerce_sales;
SELECT COUNT(*) FROM ecommerce_sales;
SELECT TOP 10 * FROM ecommerce_sales;
```

### Fix Dates & Consistency

```sql
-- Convert to DATE format
UPDATE ecommerce_sales
SET [Ship Date] = TRY_CONVERT(DATE, [Ship Date], 101)
WHERE TRY_CONVERT(DATE, [Ship Date], 101) IS NOT NULL;

UPDATE ecommerce_sales
SET [Order Date] = TRY_CONVERT(DATE, [Order Date], 101)
WHERE TRY_CONVERT(DATE, [Order Date], 101) IS NOT NULL;
```

### Null & Invalid Values

```sql
-- Dynamic null check
DECLARE @TableName NVARCHAR(128) = 'ecommerce_sales';
DECLARE @SQL NVARCHAR(MAX) = '';

SELECT @SQL = @SQL +
    'SELECT ''' + COLUMN_NAME + ''' AS ColumnName, COUNT(*) AS NullCount FROM ' + @TableName +
    ' WHERE [' + COLUMN_NAME + '] IS NULL UNION ALL '
FROM INFORMATION_SCHEMA.COLUMNS
WHERE TABLE_NAME = @TableName;

SET @SQL = LEFT(@SQL, LEN(@SQL) - 10);
EXEC sp_executesql @SQL;
```

---

## 📦 Delivery Analysis

### Shipping Delays

```sql
SELECT [Order ID], [Order Date], [Ship Date],
DATEDIFF(DAY, [Order Date],[Ship Date]) AS Shipping_Delay
FROM ecommerce_sales
WHERE DATEDIFF(DAY, [Order Date],[Ship Date]) = 0;
```

### Same-Day Deliveries

```sql
SELECT [Category], [Sub-Category], State, COUNT(*) AS SameDayOrders
FROM ecommerce_sales
WHERE DATEDIFF(DAY, [Order Date], [Ship Date]) = 0
GROUP BY [Category], [Sub-Category], State
ORDER BY SameDayOrders DESC;
```

### Delivery Trends by State

```sql
SELECT State,
    MIN(DATEDIFF(DAY, [Order Date], [Ship Date])) AS MinDaysToShip,
    MAX(DATEDIFF(DAY, [Order Date], [Ship Date])) AS MaxDaysToShip,
    AVG(DATEDIFF(DAY, [Order Date], [Ship Date])) AS AvgDaysToShip,
    COUNT(*) AS TotalOrders
FROM ecommerce_sales
GROUP BY State
ORDER BY AvgDaysToShip DESC;
```

---

## 📅 Order Trends

### Yearly Orders & Growth

```sql
WITH YearlyOrders AS (
    SELECT YEAR([Order Date]) AS OrderYear, COUNT([Order ID]) AS TotalOrders
    FROM ecommerce_sales
    GROUP BY YEAR([Order Date])
)
SELECT OrderYear, TotalOrders,
    LAG(TotalOrders) OVER (ORDER BY OrderYear) AS PreviousYearOrders,
    ROUND((CAST(TotalOrders AS FLOAT) - LAG(TotalOrders) OVER (ORDER BY OrderYear))
          / LAG(TotalOrders) OVER (ORDER BY OrderYear) * 100, 2) AS YoYGrowthPct
FROM YearlyOrders
ORDER BY OrderYear;
```

### Monthly & Quarterly Trends

```sql
SELECT YEAR([Order Date]) AS OrderYear,
       DATEPART(QUARTER, [Order Date]) AS OrderQuarter,
       COUNT(*) AS TotalOrders
FROM ecommerce_sales
GROUP BY YEAR([Order Date]), DATEPART(QUARTER, [Order Date])
ORDER BY OrderYear, OrderQuarter;

SELECT YEAR([Order Date]) AS OrderYear,
       MONTH([Order Date]) AS OrderMonth,
       DATENAME(MONTH, [Order Date]) AS MonthName,
       COUNT(*) AS TotalOrders
FROM ecommerce_sales
GROUP BY YEAR([Order Date]), MONTH([Order Date]), DATENAME(MONTH, [Order Date])
ORDER BY OrderYear, OrderMonth;
```

---

## 💰 Sales & Profitability

### By Category

```sql
SELECT Category, SUM(Sales) AS TotalSales, SUM(Profit) AS TotalProfit, COUNT(*) AS TotalOrders
FROM ecommerce_sales
GROUP BY Category
ORDER BY TotalSales DESC;
```

### By Sub-Category

```sql
SELECT [Sub-Category],
       SUM(Sales) AS TotalSales,
       SUM(Profit) AS TotalProfit,
       SUM(Profit) * 100.0 / NULLIF(SUM(Sales), 0) AS ProfitMarginPct,
       AVG(Discount) AS AvgDiscount,
       COUNT(*) AS TotalOrders
FROM ecommerce_sales
GROUP BY [Sub-Category]
ORDER BY ProfitMarginPct ASC;
```

➡️ Example Insight:
📉 **Tables**: High sales but **negative profit margins** in East, South, Central due to heavy discounts.

---

## 🧑‍🤝‍🧑 Customer Insights

### Total Customers

```sql
SELECT COUNT(DISTINCT [Customer ID]) AS TotalCustomers
FROM ecommerce_sales;
```

### Repeat Customers

```sql
WITH CustomerYear AS (
    SELECT [Customer ID], [Customer Name], YEAR([Order Date]) AS OrderYear
    FROM ecommerce_sales
    GROUP BY [Customer ID], [Customer Name], YEAR([Order Date])
)
SELECT [Customer ID], [Customer Name], COUNT(*) AS YearsActive
FROM CustomerYear
GROUP BY [Customer ID], [Customer Name]
HAVING COUNT(*) > 1
ORDER BY YearsActive DESC;
```

### Top Customers

```sql
SELECT TOP 10 [Customer ID], [Customer Name],
       SUM(Sales) AS TotalSales, SUM(Profit) AS TotalProfit, COUNT(*) AS TotalOrders
FROM ecommerce_sales
GROUP BY [Customer ID], [Customer Name]
ORDER BY TotalProfit DESC;
```
### KPI

```sql
WITH YearlyOrders AS (
    SELECT 
        YEAR([Order Date]) AS OrderYear,
        COUNT(*) AS TotalOrders
    FROM ecommerce_sales
    GROUP BY YEAR([Order Date])
),
YoYGrowth AS (
    SELECT 
        OrderYear,
        TotalOrders,
        LAG(TotalOrders) OVER (ORDER BY OrderYear) AS PrevYearOrders,
        ROUND(
            (TotalOrders - LAG(TotalOrders) OVER (ORDER BY OrderYear)) * 100.0 /
            NULLIF(LAG(TotalOrders) OVER (ORDER BY OrderYear),0), 2
        ) AS YoYGrowthPct
    FROM YearlyOrders
),
LatestYear AS (
    SELECT MAX(OrderYear) AS LatestOrderYear FROM YearlyOrders
)
SELECT
    -- Sales KPIs
    (SELECT SUM(Sales) FROM ecommerce_sales) AS TotalSales,
    (SELECT AVG(Sales) FROM ecommerce_sales) AS AvgSalesPerOrder,
    (SELECT SUM(Quantity) FROM ecommerce_sales) AS TotalProductsSold,

    -- Profit KPIs
    (SELECT SUM(Profit) FROM ecommerce_sales) AS TotalProfit,
    (SELECT SUM(Profit)*100.0/NULLIF(SUM(Sales),0) FROM ecommerce_sales) AS ProfitMarginPct,

    -- Customer KPIs
    (SELECT COUNT(DISTINCT [Customer ID]) FROM ecommerce_sales) AS TotalCustomers,
    (SELECT COUNT(*) FROM (
        SELECT [Customer ID], COUNT(DISTINCT YEAR([Order Date])) AS YearsActive
        FROM ecommerce_sales
        GROUP BY [Customer ID]
        HAVING COUNT(DISTINCT YEAR([Order Date])) > 1
    ) AS RepeatCustomers) AS RepeatCustomersCount,

    -- Order & Delivery KPIs
    (SELECT AVG(DATEDIFF(DAY, [Order Date], [Ship Date])) FROM ecommerce_sales) AS AvgShippingDelay,
    (SELECT COUNT(*) FROM ecommerce_sales WHERE DATEDIFF(DAY, [Order Date], [Ship Date])=0) AS SameDayOrders,
    (SELECT MIN(DATEDIFF(DAY, [Order Date], [Ship Date])) FROM ecommerce_sales) AS MinShippingDays,
    (SELECT MAX(DATEDIFF(DAY, [Order Date], [Ship Date])) FROM ecommerce_sales) AS MaxShippingDays,

    -- Trend KPIs using latest available year
    (SELECT TotalOrders FROM YoYGrowth WHERE OrderYear = (SELECT LatestOrderYear FROM LatestYear)) AS OrdersLatestYear,
    (SELECT YoYGrowthPct FROM YoYGrowth WHERE OrderYear = (SELECT LatestOrderYear FROM LatestYear)) AS YoYGrowthLatestYear
;
```
---

## 🏆 Key Findings

* 🚨 **Tables** sold at a loss in most regions due to deep discounts.
* 📦 **New York City** had the fastest deliveries, including same-day.
* 🎄 **Q4 sales spikes** showed strong holiday seasonality.
* 🔄 **Repeat customers** formed a significant base year over year.

---

