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

### Data Quality Analysis

Below are the findings regarding data quality, categorized by table name.

#### DimCustomer
This table contains information about the customers.  

The `CustomerKey`, `CustomerAlternateKey` and `EmailAddress` columns are unique. This shows that most of the key indentifiers are not duplicated.

**Concerns**:
- Columns such as `Title`, `Middle Name`, `Suffix`, and `AddressLine2` have missing values denoted with "NULL" strings. Though these  columns are not critical, missing values should be left empty instead, to avoid any misprocessing by users and other applications. While a human may understand the intention, machines may mistake it for a value, leading to inaccurate results.
- The date format in the `BirthDate` column is not standardized. It is supposed to be in `dd/mm/yyyy` format, but some of the month values have leading zeros (e.g. 06 for June instead of 6). This can lead to errors when loaded into a table or when analyzed. I loaded this column as String for now, but would need to be corrected if I intend to use it.
- The `Phone` column also appears to be in a non-standard format. Some values include country code and area code, while others do not.
- The `DateFirstPurchase` column's format, while standardized to be in `dd/mm/yy` format, it has leading zeroes which could be a problem in certain databases. Such format is not automatically recognized by BigQuery, we would need to eliminate the leading zeroes or find a way to specify a correct date format later on.
- There are duplications in the `Phone` column. The uniqueness score is 49%, meaning only 49% of the values in this column are unique. I understand that phone numbers can be recycled, but this looks too severe. It is unlikely that phone numbers are recycled hundreds of times already. For example, the number `1 (11) 500 555-0118` is duplicated 205 times.
- There seems to be duplications in `AddressLine1`, the uniqueness score is 69.3%. There are cases where customers have the same address. For example, being part of the same family, renting different rooms in the same building, or simply have moved out to a different address. However, since the uniqueness score seems quite low, this is worth a further investigation.

#### DimProduct
This table contains information about the products that the company sells.

The `ProductKey` column, as the primary identifier is unique.  
There are apparent relationships between `WeightUnitMeasureCode` and `Weight`, as well as between `SizeUnitMeasureCode` and `Size`, where the former is a measurement unit for the latter.

**Concerns**:
- Columns such as `WeightUnitMeasureCode`, `SizeUnitMeasureCode`, `StandardCost`, `ListPrice`, `Size`, `Weight`, `ProductLine`, `DealerPrice`, `Class`, `Style`, `ModelName`, `Status`, and `EndDate` have missing values denoted with "NULL" strings. When the information is missing on these columns, leaving it empty is better to avoid misleading interpretations.
- The completeness of `WeightUnitMeasureCode` and `SizeUnitMeasureCode` are 43.1% and 38.1% respectively. This could be a fatal issue when trying to get reports about product weight or size.
- The completeness of `Status` is 58.9%, obtained by counting both blanks and "NULL" values.
- The completeness of `StandardCost`, `ListPrice`, and `DealerPrice` are 63%. The cause of missing values seems the same since these columns are highly correlated. This incompleteness could be an issue for cost-related analyses.
- 49.7% of the values in `StartDate` column is either `00:00.0` or "NULL", which poses as an incompleteness problem as well.
- 58.9% of the values in `EndDate` column is "NULL". Combined with blanks and `00:00.0`, it's missing values accumulate to 90% of the whole table.
- There are some duplications in `ProductAlternateKey` (Uniqueness score 85.7%).
- For non-null values in `StandardCost`, I identified the outliers using IQR, and found 39 outliers (~11%). This is the same with `ListPrice`, and `DealerPrice`.

#### DimSalesTerritory

There seems to be only one concern, that there is missing information for one particular row corresponding to `SalesTerritoryKey` 11.

#### DimInvoices
This table contains information about the invoices.

The `invoice_id` column, as the primary identifier is unique.  
The `discount` column defines the `discount_amount_usd`, which makes the calculation for `amount_usd`.  
When there is no discount, `discount_amount_usd` equals to `amount_usd`.  
When there is discount, `discount_amount_usd` = `amount_usd` - `discount`.

**Concerns**:
- The `product_type` column has a few missing values, but its completeness score is 92.6% nevertheless.
- 92.8% of the `discount` column is blank. Assuming this means no discount, it is better to impute this with 0.
- There are 60 outliers out of 500 values in `amount_usd`. These are the values greater than the upper bound (`mean + 1.5 * std dev`) of 2,620.

## SQL Questions
