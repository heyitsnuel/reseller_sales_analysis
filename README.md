##  README

This repository contains a technical analytics assignment that I got from one of my job applications as Data Analyst.
This demonstrates the use of **SQL** in **Google BigQuery** for **ETL**, **data quality profiling**, and **exploratory analysis**.

## Database Connection

The datasets are loaded into BigQuery with these details:
- Project id: `civic-genius-328315`
- Dataset name: `remote_assignment`

The datasets consist of 4 dimension tables and 1 fact table:
- `DimCustomer`
- `DimProduct`
- `DimSalesTerritory`
- `DimInvoices`
- `FactResellerSales`

You can query the tables from the [BigQuery console](https://console.cloud.google.com/bigquery). If you have not used BigQuery before, please follow the [Quick Starts documentation](https://cloud.google.com/bigquery/docs/quickstarts) beforehand. After you create an account and a billing project, make sure you add the `civic-genius-328315` project in order to be able to explore the tables. The dataset and table have been set for public view, but if you have any trouble accessing it, do not hesitate to contact me.

Sample query:  
```SELECT * FROM `civic-genius-328315.remote_assignment.FactResellerSales` LIMIT 10```

## Exploratory Analysis with SQL


## SQL Questions
