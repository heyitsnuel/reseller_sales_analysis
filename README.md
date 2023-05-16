##  README

This repository contains a technical analytics assignment that I got from one of my job applications as Data Analyst.
This demonstrates the use of **SQL** in **Google BigQuery** for **ETL**, **data quality analysis**, and **exploratory analysis**.

## Database Connection

The datasets are loaded into BigQuery with these details:
- Project id: `civic-genius-328315`
- Dataset id: `remote_assignment`

The datasets comprise of 4 dimension tables and 1 fact table:
- `FactResellerSales`
- `DimCustomer`
- `DimProduct`
- `DimSalesTerritory`
- `DimInvoices`

**NOTE**: The dataset and tables have not been opened for public, please [contact me](mailto:immanuel.ambhara@gmail.com) if you wish to get access.

You can query the tables from the [BigQuery console](https://console.cloud.google.com/bigquery). If you have not used BigQuery before, please follow the [Quick Starts documentation](https://cloud.google.com/bigquery/docs/quickstarts) beforehand. After you create an account and a billing project, make sure you add the `civic-genius-328315` project in order to be able to explore the tables.

Sample query:  
```
SELECT * FROM `civic-genius-328315.remote_assignment.DimSalesTerritory` LIMIT 10
```

## Exploratory Analysis with SQL

This section contains the findings regarding data quality for each table.

### DimCustomer
This table contains information about the customers.

#### Uniqueness
- The `CustomerKey`, `CustomerAlternateKey` and `EmailAddress` columns are unique. This shows that most of the key indentifiers are not duplicated.
- There are duplications in the `Phone` column. The uniqueness score is 49% (only 49% of the values in this column are unique). While phone numbers can be recycled, this score is too severe. It is unlikely that phone numbers are recycled hundreds of times already. For example, the number `1 (11) 500 555-0118` is duplicated 205 times.
- There seems to be duplications in `AddressLine1`, the uniqueness score is 69.3%. There are cases where customers have the same address. For example, being part of the same family, renting different rooms in the same building, or simply have moved out to a different address. However, since the uniqueness score seems quite low, this is worth a further investigation.

Query to evaluate uniqueness
```
SELECT
  COUNT(*) AS row_count,
  COUNT(DISTINCT customerkey) AS unique_customer_key_count,
  COUNT(DISTINCT customeralternatekey) AS unique_customer_alt_key_count,
  COUNT(DISTINCT emailaddress) AS unique_email_count,
  COUNT(DISTINCT phone) AS unique_phone_count,
  COUNT(DISTINCT addressline1) AS unique_address1_count
FROM `civic-genius-328315.remote_assignment.DimCustomer`
```

Query to evaluate `Phone` uniqueness specifically
```
SELECT
  phone,
  COUNT(*)
FROM `civic-genius-328315.remote_assignment.DimCustomer`
GROUP BY 1
HAVING COUNT(*) > 1
ORDER BY 2 DESC
```

#### Completeness
- Columns such as `Title`, `Middle Name`, `Suffix`, and `AddressLine2` have missing values denoted with "NULL" strings. Though these  columns are not critical, missing values should be left empty instead, to avoid any misprocessing by users and other applications. While a human may understand the intention, machines may mistake it for a value, leading to inaccurate results.

#### Consistency
- The date format in the `BirthDate` column is not standardized. It is supposed to be in `dd/mm/yyyy` format, but some of the month values have leading zeros (e.g. 06 for June instead of 6). This can lead to errors when loaded into a table or when analyzed. I loaded this column as String for now, but would need to be corrected if I intend to use it.
- The `Phone` column also appears to be in a non-standard format. Some values include country code and area code, while others do not.
- The `DateFirstPurchase` column's format, while standardized to be in `dd/mm/yy` format, it has leading zeroes which could be a problem in certain databases. Such format is not automatically recognized by BigQuery, we would need to eliminate the leading zeroes or find a way to specify a correct date format later on.

### DimProduct
This table contains information about the products that the company sells.

#### Patterns, Relationships, & Outliers
- There are apparent relationships between `WeightUnitMeasureCode` and `Weight`, as well as between `SizeUnitMeasureCode` and `Size`, where the former is a measurement unit for the latter.
- For non-null values in `StandardCost`, I identified the outliers using IQR, and found 39 outliers (~11%). This is the same with `ListPrice`, and `DealerPrice`.

Query to find outliers
```
WITH iqr AS (
  SELECT
    AVG(CAST(DealerPrice AS FLOAT64)) - 1.5 * STDDEV(CAST(DealerPrice AS FLOAT64)) AS lower_bound,
    AVG(CAST(DealerPrice AS FLOAT64)) + 1.5 * STDDEV(CAST(DealerPrice AS FLOAT64)) AS upper_bound
  FROM `civic-genius-328315.remote_assignment.DimProduct`
  WHERE DealerPrice <> 'NULL'
)

SELECT
  COUNT(*),
  COUNT(CASE WHEN CAST(DealerPrice AS FLOAT64) NOT BETWEEN lower_bound AND upper_bound THEN 1 END) AS outlier_count
FROM `civic-genius-328315.remote_assignment.DimProduct`, iqr
WHERE DealerPrice <> 'NULL'
```

#### Uniqueness
- The `ProductKey` column, as the primary identifier is unique.
- `ProductName` is duplicated, as one `ProductName` can have more than one `ProductKey`, usually having different `StandardCost` and can be validated using `StartDate` and `EndDate`.
- There are some duplications in `ProductAlternateKey` (Uniqueness score 85.7%).

Query to evaluate uniqueness
```
SELECT
  COUNT(*) AS row_count,
  COUNT(DISTINCT productkey) AS unique_product_key_count,
  1 - COUNT(DISTINCT productalternatekey) / COUNT(*) AS product_alt_key_uniq_score
FROM `civic-genius-328315.remote_assignment.DimProduct`
```

#### Completeness
- Columns such as `WeightUnitMeasureCode`, `SizeUnitMeasureCode`, `StandardCost`, `ListPrice`, `Size`, `Weight`, `ProductLine`, `DealerPrice`, `Class`, `Style`, `ModelName`, `Status`, and `EndDate` have missing values denoted with "NULL" strings. When the information is missing on these columns, leaving it empty is better to avoid misleading interpretations.
- The completeness of `WeightUnitMeasureCode` and `SizeUnitMeasureCode` are 43.1% and 38.1% respectively. This could be a fatal issue when trying to get reports about product weight or size.
- The completeness of `Status` is 58.9%, obtained by counting both blanks and "NULL" values.
- The completeness of `StandardCost`, `ListPrice`, and `DealerPrice` are 63%. The cause of missing values seems the same since these columns are highly correlated. This incompleteness could be an issue for cost-related analyses.
- 49.7% of the values in `StartDate` column is either `00:00.0` or "NULL", which poses as an incompleteness problem as well.
- 58.9% of the values in `EndDate` column is "NULL". Combined with blanks and `00:00.0`, it's missing values accumulate to 90% of the whole table.

Query to evaluate completeness
```
SELECT
  COUNT(*) AS row_count,
  1 - COUNT(CASE WHEN status = 'NULL' OR status IS NULL THEN 1 END) / COUNT(*) AS status_compl_score,
  COUNT(CASE WHEN startdate IN ('00:00.0', 'NULL') OR startdate IS NULL THEN 1 END) / COUNT(*) AS startdate_missing_score,
  COUNT(CASE WHEN enddate IN ('00:00.0', 'NULL') OR enddate IS NULL THEN 1 END) / COUNT(*) AS enddate_missing_score
FROM `civic-genius-328315.remote_assignment.DimProduct`
```

#### Consistency
- `StartDate` in most cases should be earlier than `EndDate`, however ~55.9% of the rows have `EndDate` earlier than `StartDate`. This excludes `EndDate` = "NULL" and blanks, since it can be assumed that the corresponding product is still valid or active in such cases.

Query to evaluate consistency
```
SELECT
  COUNT(*) AS row_count,
  COUNT(CASE WHEN StartDate > EndDate THEN 1 END) AS invalid_count
FROM `civic-genius-328315.remote_assignment.DimProduct`
WHERE EndDate <> "NULL" AND EndDate IS NOT NULL
```


### DimSalesTerritory

#### Completeness
- There seems to be only one concern, that there is missing information for one particular row corresponding to `SalesTerritoryKey` 11.

### DimInvoices
This table contains information about the invoices. 

#### Patterns, Relationships, & Outliers
- The `discount` column defines the `discount_amount_usd`, which makes the calculation for `amount_usd`. When there is no discount, `discount_amount_usd` equals to `amount_usd`. When there is discount, `discount_amount_usd` = `amount_usd` - `discount`.
- There are 60 outliers out of 500 values in `amount_usd`. These are the values greater than the upper bound (`mean + 1.5 * std dev`) of 2,620.

Query to find outliers
```
SELECT
  COUNT(*) AS row_count,
  COUNT(CASE WHEN startdate IN ('00:00.0', 'NULL') OR startdate IS NULL THEN 1 END) / COUNT(*) AS startdate_missing_score,
  COUNT(CASE WHEN enddate IN ('00:00.0', 'NULL') OR enddate IS NULL THEN 1 END) / COUNT(*) AS enddate_missing_score
FROM `civic-genius-328315.remote_assignment.DimProduct`
```

#### Uniqueness
- The `invoice_id` column, as the primary identifier is unique.

#### Completeness
- The `product_type` column has a few missing values, though its completeness score is quite high (92.6%), this could be improved.
- 92.8% of the `discount` column is blank. Assuming this means no discount, it is better to impute this with 0.

Query to evaluate completeness
```
SELECT
  COUNT(*) AS row_count,
  COUNT(DISTINCT invoice_id) AS invoice_count,
  COUNT(CASE WHEN product_type IS NOT NULL THEN 1 END) AS product_type_compl_score,
  COUNT(CASE WHEN discount IS NULL THEN 1 END) / COUNT(*) AS discount_missing_score
FROM `civic-genius-328315.remote_assignment.DimInvoices`
```

### FactResellerSales
This fact table stores detailed information about sales transactions.

#### Patterns, Relationships, & Outliers
- Since `SalesOrderNumber` is repeated, we can combine it with `SalesOrderLineNumber` to make it as the primary identifier
- `OrderQuantity` * `UnitPrice` = `ExtendedAmount`
- `UnitPriceDiscount` * `ExtendedAmount` = `DiscountAmount`
- `ExtendedAmount` - `DiscountAmount` = `SalesAmount`
- `OrderQuantity` * `ProductStandardCost` = `TotalProductCost`
- There are a few outliers in some of the fields, such as 7,370 rows (12%) in `ProductStandardCost`, 3,671 (6%) in `OrderQuantity`, 3,952 (6.5%) in `SalesAmount`

Query to validate the unique identifier
```
SELECT
  COUNT(*) as row_count,
  COUNT(DISTINCT CONCAT(salesordernumber, '-', salesorderlinenumber)) AS unique_id_count
FROM `civic-genius-328315.remote_assignment.FactResellerSales`
```

Query to find outliers
```
WITH iqr AS (
  SELECT
    AVG(ProductStandardCost) - 1.5 * STDDEV(ProductStandardCost) AS lower_bound,
    AVG(ProductStandardCost) + 1.5 * STDDEV(ProductStandardCost) AS upper_bound
  FROM `civic-genius-328315.remote_assignment.FactResellerSales`
)

SELECT
  COUNT(*),
  COUNT(CASE WHEN ProductStandardCost NOT BETWEEN lower_bound AND upper_bound THEN 1 END) AS outlier_count
FROM `civic-genius-328315.remote_assignment.FactResellerSales`, iqr
```

## SQL Questions

### Question 1: What is the highest transaction of each month in 2012 for the product Sport-100 Helmet, Red?

```
WITH prod_filter AS (
  SELECT ProductKey
  FROM `civic-genius-328315.remote_assignment.DimProduct`
  WHERE ProductName = 'Sport-100 Helmet, Red'
)

SELECT
  FORMAT_DATE('%m-%Y', OrderDate) AS Month,
  ROUND(SalesAmount, 1) AS SalesAmount,
  DATE(OrderDate) AS OrderDate
FROM `civic-genius-328315.remote_assignment.FactResellerSales`
WHERE EXTRACT(year FROM OrderDate) = 2012
  AND ProductKey IN (SELECT ProductKey FROM prod)
QUALIFY ROW_NUMBER() OVER (PARTITION BY FORMAT_DATE('%m-%Y', OrderDate) ORDER BY SalesAmount DESC) = 1
ORDER BY 1
```

#### Explanation
- In the `prod_filter` CTE, we shortlist the `ProductKey` in question.
- Since the order of the filters matters in BigQuery, we first filter the fact table to only return sales from 2012.
- Then we filter the ProductKey. This order of filters will optimize BigQuery to only process the intended subset of data instead of scanning the whole table.
- To get the highest transaction, we rank it using the `ROW_NUMBER()` function, passing the Month as the partition and sorting the `SalesAmount` in descending order.
- Finally, we filter the highest transaction from the resulting rank using the `QUALIFY` clause.
- To get the desired date format for the output, we use the `FORMAT_DATE()` function.

### Question 2: Find all the products and their total sales amount by month. Only consider products that had at least one sale in 2012.

```
WITH prod_filter AS (
  SELECT
    ProductKey,
    COUNT(*) AS TotalSales2012
  FROM `civic-genius-328315.remote_assignment.FactResellerSales`
  WHERE EXTRACT(year FROM OrderDate) = 2012
  GROUP BY 1
)

SELECT
  ProductName,
  ROUND(SUM(SalesAmount), 1) AS SalesAmount,
  FORMAT_DATE('%m-%Y', OrderDate) AS Month
FROM (
  SELECT ProductKey, ProductName FROM `civic-genius-328315.remote_assignment.DimProduct`
  WHERE ProductKey IN (SELECT ProductKey FROM prod_filter WHERE TotalSales2012 >= 1)
) p
LEFT JOIN `civic-genius-328315.remote_assignment.FactResellerSales` s
  USING (ProductKey)
GROUP BY 1, 3 ORDER BY 1, 3
```

#### Explanation
- In the `prod_filter` CTE, we shortlist the `ProductKey` that have at least one sale in 2012.
- Afterwards, we filter the `DimProduct` table to only include the `ProductKey` from `prod_filter` CTE. This way, we minimize the number of data being processed in the join operation.
- Then we perform a left join with the fact table using `ProductKey` as the join key.
- Finally, we sum the `SalesAmount` grouped by `ProductName` and Month, rounded to one decimal point.

### Question 3: What are the age groups of customers categorised by marital status and gender?

```
WITH age_ref AS (
  SELECT
    CustomerKey,
    DATE_DIFF(CURRENT_DATE, PARSE_DATE('%d/%m/%Y', BirthDate), DAY) / 365 AS AgeInYear
  FROM `civic-genius-328315.remote_assignment.DimCustomer`
)

SELECT
  MaritalStatus,
  Gender,
  COUNT(CASE WHEN AgeInYear < 35 THEN CustomerKey END) AS AgeBelow35,
  COUNT(CASE WHEN AgeInYear BETWEEN 35 AND 50 THEN CustomerKey END) AS AgeBetween35and50,
  COUNT(CASE WHEN AgeInYear > 35 THEN CustomerKey END) AS AgeAbove50
FROM `civic-genius-328315.remote_assignment.DimCustomer` c
JOIN age_ref a
  USING (CustomerKey)
GROUP BY 1, 2
ORDER BY 1, 2
```

#### Explanation
- The `age_ref` CTE aims to calculate customers' age by subtracting the number of days between current date and the `BirthDate`, and divide the result by 365 to get the year. Using the `DATEDIFF()` function this way will get us more accurate age, compared to only getting the year difference directly.
- From here on, we can simply use the `COUNT()` function combined with `CASE` statements to categorize the age ranges.

### Question 4: What is the combination of product & country with the lowest amount of sales per month in 2012?

```
SELECT
  FORMAT_DATE('%m-%Y', OrderDate) AS Month,
  SalesTerritoryCountry,
  ProductName,
  ROUND(SalesAmount, 1) AS SalesAmount
FROM `civic-genius-328315.remote_assignment.FactResellerSales` s
LEFT JOIN `civic-genius-328315.remote_assignment.DimSalesTerritory` t
  USING (SalesTerritoryKey)
LEFT JOIN `civic-genius-328315.remote_assignment.DimProduct` p
  USING (ProductKey)
WHERE EXTRACT(year FROM OrderDate) = 2012
QUALIFY ROW_NUMBER() OVER (PARTITION BY FORMAT_DATE('%m-%Y', OrderDate), SalesTerritoryCountry, ProductName ORDER BY SalesAmount) = 1
ORDER BY 1, 2, 3
```

#### Explanation
- First, we limit the fact table to only process the data of sales in 2012.
- Then we perform two left joins: one with `DimSalesTerritory` using `SalesTerritoryKey` to get `SalesTerritoryCountry`, and another with `DimProduct` using `ProductKey` to get `ProductName`.
- To get the lowest transaction, we do the `ROW_NUMBER()` function partitioned by Month, `SalesTerritoryCountry`, and `ProductName`, sorted by `SalesAmount` in ascending order.

### Question 5: What is 2022 monthly basis Annual Recurring Revenue (ARR) of products named EOR Monthly and EOR Yearly?

```

```

#### Explanation
