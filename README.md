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

The dataset and tables have not been opened for public, please [contact me](mailto:immanuel.ambhara@gmail.com) if you wish to access it.

You can query the tables from the [BigQuery console](https://console.cloud.google.com/bigquery). If you have not used BigQuery before, please follow the [Quick Starts documentation](https://cloud.google.com/bigquery/docs/quickstarts) beforehand. After you create an account and a billing project, make sure you add the `civic-genius-328315` project in order to be able to explore the tables.

Sample query:  
```SELECT * FROM `civic-genius-328315.remote_assignment.DimSalesTerritory` LIMIT 10```

## Exploratory Analysis with SQL


## SQL Questions
