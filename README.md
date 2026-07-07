# DeltaFlow-OnPrem-to-Azure-ETE-Pipeline

End-to-end Azure data engineering pipeline that incrementally migrates data from an on-premises SQL Server to Azure Data Lake Storage Gen2 using Azure Data Factory and watermark-based loading, then transforms the data in Azure Databricks using the Medallion architecture (Bronze, Silver, Gold), and Azure Synapse Analytics.

This project orchestrates the ingestion, transformation, and optimization of relational ERP data from an on-premises SQL Server environment into high-performance, analytical Star Schemas ready for business intelligence.

## Prerequisites
- Azure Subscription with an active Resource Group.
- Knowledge of PySpark( Autoloader, Batch streaming, Delta tables

## Architecture
![](Architecture.jpeg)

The data platform transitions through an automated pipeline designed for incremental updates, minimal processing costs, and maximum query performance:
Ingestion (Staging): Azure Data Factory securely copies transactional data from on-premises SQL Server instances and drops them as compressed Parquet files into a raw landing zone in ADLS Gen2.
![](ADF_pipeline.jpeg)

##### Bronze Layer (Raw Replication): 
An ingestion pipeline was created to dynamically auto load all source data from the source containers as they drop to the bronze layer. Databricks Auto Loader monitors cloud folders to discover new batches incrementally, appending raw entries seamlessly into schema-inferred Bronze Parquet tables.  

##### Silver Layer (Cleaning & Harmonization): 
The silver layer houses all transformations done to the tables. This startes with batch streaming of data, cleaning, standardizing of data types, reformating of column names, flattening of transactional constraints and finally saving as a delta table using the MERGE strategy to handle upserts smoothly, preventing data duplication for downstream analytics

##### Gold Layer (Analytical Star Schema): 
Gold layer serves as the final layer. Introduced denormalised tables by joining dim tables to produce flat tables ready for use downstream. To reconcile structural discrepancies within the AdventureWorks source data, total prices were re-calculated from sales detail rows. Finalized tables are then updated via idempotent multi-key Delta MERGE operations and Z-Ordered to guarantee fast, cost-efficient Synapse queries.

##### Final Pipeline was setup for all layers to run in parallel 
![](Final_Sales_Pipeline.jpeg)

##### Serving Layer: 
Azure Synapse Serverless SQL Pools expose the optimized Gold Delta files as relational views, serving sub-second, highly cost-efficient aggregations straight to Power BI.

## Getting Started - Setup

- Provision Azure Resources: Log in to the Azure Portal, create a Resource Group, and provision the following services:
  - Azure Storage Account
  - Azure Data Factory (ADF)
  - Azure Databricks Access Connector
  - Azure Databricks Workspace

- Configure the Data Lake: During the Storage Account setup, enable Hierarchical Namespace (HNS) to upgrade standard Blob storage into a true directory-based Data Lake (ADLS Gen2). Create your storage containers. To allow Unity Catalog to securely access the data lake, go to the Storage Account's Access Control (IAM) and assign the following roles to the Access Connector's Managed Identity:
  - Storage Blob Data Contributor
  - Storage Account Contributor
  - EventGrid EventSubscription Contributor
  - Storage Queue Data Contributor

- Configure Unity Catalog & Storage Credentials: Databricks strictly enforces a limit of one Unity Catalog metastore per region per account; all workspaces in the same region share this central metastore. Log into your Databricks workspace as a Metastore Admin or Account Admin. In the Catalog Explorer, create a new Storage Credential by pasting the Azure Resource ID of your Access Connector. Once connected, set up your external locations, schemas, and external tables pointing to the ADLS Gen2 data lake.

- Initialize Ingestion: Launch Azure Data Factory and configure the integration runtime/linked services to connect to your on-premises database (see resources below).

- Process Data: Once the ADF ingestion pipeline is successfully built and running, launch Azure Databricks to begin data transformation.


## Resources 
- https://learn.microsoft.com/en-us/azure/databricks/data-governance/unity-catalog/setup-uc
- https://learn.microsoft.com/en-us/azure/databricks/ingestion/cloud-object-storage/auto-loader/unity-catalog
- https://learn.microsoft.com/en-us/azure/databricks/tables/external


