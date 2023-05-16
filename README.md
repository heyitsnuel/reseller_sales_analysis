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
select
  count(*) as row_count,
  count(distinct customerkey) as unique_customer_key_count,
  count(distinct customeralternatekey) as unique_customer_alt_key_count,
  count(distinct emailaddress) as unique_email_count,
  count(distinct phone) as unique_phone_count,
  count(distinct addressline1) as unique_address1_count
from `civic-genius-328315.remote_assignment.DimCustomer`
```

Query to evaluate `Phone` uniqueness specifically
```
select
  phone,
  count(*)
from `civic-genius-328315.remote_assignment.DimCustomer`
group by 1
having count(*) > 1
order by 2 desc
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
with iqr as (
  select
    avg(cast(DealerPrice as float64)) - 1.5 * stddev(cast(DealerPrice as float64)) as lower_bound,
    avg(cast(DealerPrice as float64)) + 1.5 * stddev(cast(DealerPrice as float64)) as upper_bound
  from `civic-genius-328315.remote_assignment.DimProduct`
  where DealerPrice <> 'NULL'
)

select
  count(*),
  count(case when cast(DealerPrice as float64) not between lower_bound and upper_bound then 1 end) as outlier_count
from `civic-genius-328315.remote_assignment.DimProduct`, iqr
where DealerPrice <> 'NULL'
```

#### Uniqueness
- The `ProductKey` column, as the primary identifier is unique.
- There are some duplications in `ProductAlternateKey` (Uniqueness score 85.7%).

Query to evaluate uniqueness
```
select
  count(*) as row_count,
  count(distinct productkey) as unique_product_key_count,
  1 - count(distinct productalternatekey)/count(*) as product_alt_key_uniq_score
from `civic-genius-328315.remote_assignment.DimProduct`
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
select
  count(*) as row_count,
  1 - count(case when status = 'NULL' or status is null then 1 end) / count(*) as status_compl_score,
  count(case when startdate IN ('00:00.0', 'NULL') or startdate is null then 1 end) / count(*) as startdate_missing_score,
  count(case when enddate IN ('00:00.0', 'NULL') or enddate is null then 1 end) / count(*) as enddate_missing_score
from `civic-genius-328315.remote_assignment.DimProduct`
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
select
  count(*) as row_count,
  count(case when startdate IN ('00:00.0', 'NULL') or startdate is null then 1 end) / count(*) as startdate_missing_score,
  count(case when enddate IN ('00:00.0', 'NULL') or enddate is null then 1 end) / count(*) as enddate_missing_score
from `civic-genius-328315.remote_assignment.DimProduct`
```

#### Uniqueness
- The `invoice_id` column, as the primary identifier is unique.

#### Completeness
- The `product_type` column has a few missing values, though its completeness score is quite high (92.6%), this could be improved.
- 92.8% of the `discount` column is blank. Assuming this means no discount, it is better to impute this with 0.

Query to evaluate completeness
```
select
  count(*) as row_count,
  count(distinct invoice_id) as invoice_count,
  count(case when product_type is not null then 1 end) as product_type_compl_score,
  count(case when discount is null then 1 end) / count(*) as discount_missing_score
from `civic-genius-328315.remote_assignment.DimInvoices`
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
select
  count(*) as row_count,
  count(distinct concat(salesordernumber, '-', salesorderlinenumber))  as unique_id_count
from `civic-genius-328315.remote_assignment.FactResellerSales`
```

Query to find outliers
```
with iqr as (
  select
    avg(ProductStandardCost) - 1.5 * stddev(ProductStandardCost) as lower_bound,
    avg(ProductStandardCost) + 1.5 * stddev(ProductStandardCost) as upper_bound
  from `civic-genius-328315.remote_assignment.FactResellerSales`
)

select
  count(*),
  count(case when ProductStandardCost not between lower_bound and upper_bound then 1 end) as outlier_count
from `civic-genius-328315.remote_assignment.FactResellerSales`, iqr
```

## SQL Questions
