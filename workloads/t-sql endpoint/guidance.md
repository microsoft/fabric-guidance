T-SQL endpoint is an endpoint that applications use to connect both Synapse Data Warehouse & Lakehouse in Microsoft Fabric. Using t-sql endpoint, applications can ingest & read data from warehouse artifact and applications can only read data from lakehouse artifact.

## Connectivity

* T-SQL endpoint uses Tabular Data Stream(TDS) protocol to connect to a Synapse Data Warehouse in Microsoft Fabric.
* t-sql endpoint uses proxy connection to establish connectivity. Here is a connectivity view of Fabric DW. ![sample connectivity](tsql-connectivity.png)
* Proxy connections can lead to connection timeout or transient connection failures. When executing long running queries, it is recommended to embed retry logic to ETL/ELT orchestration layer.
* Microsoft Fabric supports [various drivers](https://learn.microsoft.com/en-us/sql/connect/sql-connection-libraries?view=sql-server-ver16#drivers-for-relational-access) for relational access by different clients. Some of common drivers used to connect t-sql endpoint are 
    - Microsoft ODBC Driver for SQL Server
    - ADO.NET
    - Microsoft OLEDB Driver for SQL Server
    - Microsoft JDBC Driver for SQL Server

* ```Note: Direct Lake mode connectivity to Synapse Data Warehouse in Microsoft Fabric will be available post GA```

## Supported Authentication Modes by t-sql endpoint

Microsoft Fabric supports only Azure Active Directory (Azure Entra ID) authentication.

* For interactive workloads, Microsoft recommend customers to use pass through authentication.
* For ETL/ELT, automation & ALM, Microsoft recommend customers to user service principal based authentication. 
```Note: Allow Service Principals to use Power BI APIs must be turned on in the admin portal to add a service principal explicitly to a workspace to access Fabric DW```
</br>
![Tenant Level Setting](/workloads/t-sql%20endpoint/sp-tenant-setting.png)

```If your workloads are to be shared between tenants, then the service principal created in a tenant should be registered in other tenant(s) and should have admin consent to access resources in new tenants. Please note that the admin consent is mandatory to use the same service principal in multiple tentants. For example: If a ISV host data in ISV tenant and needs access to data in customer tenant, then the service principal that is created in ISV tenant must be registered in customer tenant using admin consent and then provide permissions on the data sources```

* SQL Authentication is not supported by t-sql endpoint.
* Workspace Managed Identity (equivalent to Managed Service Identity) will be available post GA. </br>
```Note: For automation purposes, Microsoft recommends customers\ISV's to use Service Principal based authentication.```

## Patterns to Ingest Data into Synapse Data Warehouse in Microsoft Fabric

Following are few options available to ingest data into Synapse Data Warehouse in Microsoft Fabric.

* COPY INTO from storage account.
    - storage accounts protected by VNET/PE will be bypassed using "trusted access" feature in Microsoft Fabric. Available post ga
* Use Azure Data Factory or Synapse Pipelines to ingest data into Fabric Data Warehouse. Note that both these services using COPY command to ingest data into DW. Once Ingested, use stored procedures to transform data for performance.
    - Azure Data Factory and Synapse Analytics has ability to connect to firewall protected data sources. This option aligns if customers are using either of these services to ingest data into Fabric DW.
* Ingest data with Microsoft Fabric Data Factory using Data Flow Gen2, Fabric Pipelines. Remember that these options might not be performant as the product is in still in preview.
    - While Data Flow Gen2 supports gateway to connect to data sources protected by firewall, Fabric pipelines do not.
* Fabric DW is not recommended for real-time ingestion. For real-time ingestion workloads, use event streams or spark streaming with Lakehouse. The data ingested as table structures in Lakehouse can be consumed using t-sql endpoint.

## Data Access Patterns for Consumption

t-sql endpoint supports two data access patterns

* Direct Query - This access pattern is commonly used to submit user queries using t-sql endpoint. Every client including 3rd party tools such as dbt, qlik compose must to use t-sql endpoint to ingest/query data in Microsoft Fabric DW.
* One Lake Access - Data ingested by DW in Microsoft Fabric is available to external readers such as Fabric Spark, Azure Synapse Spark, Azure Databricks and Power BI direct lake for consumption purposes without using sql engine.

> Points to consider:
>
> - One lake access is read only access pattern. Fabric DW does not have a staging files section unlike Lakehouse artifact.
>
> - Fabric DW will support data ingestion using COPY INTO from Lakehouse files by GA.
>
> - Cross Database queries are supported within workspace for now but cross database queries across workspaces will be supported by GA+.

## Performance & Capacity Considerations to use t-sql endpoint

<details>
<summary>SQL Frontend</summary>
* All Databases (DW/LH) hosted in a single workspace are registered to a single endpoint and capacity. Please consider following
    - Metadata operations on a single FE can be slow if number of objects are in the order of tens of thousands.
    - A single FE can cause slowness in returning large amounts of data due to network, cross region or client irrespective of capacity & compute size.
    - Cross database queries will use the capacity of the t-sql endpoint that runs the sql queries.
    - Maximum number of concurrent connections allowed per workspace SQL FE are 32767 and the maximum number of concurrent query executions depend upon the capacity units/capacity resources.
</details>

<details>
<summary>Locking & Isolation</summary>
T-sql endpoint (DW/LH) in Microsoft Fabric defaults to Snapshot Isolation (SI), different from its predecessors include Azure Synapse Gen2 Dedicated SQL Pool. Please take care of following considerations when implementing ETL/ELT using DML statements.

* ```Locks ensure concurrency correctness among transactions. T-sql endpoint can issue 3 different locks. Sch-S(Schema Stability), IX (insert/update/delete) & Sch-M(Schema Modification). Sch-S and IX are incompatable with Sch-M lock. If one of the many concurrent transactions on a table request Sch-M lock, then that trasaction will wait until Sch-S or IX locks are released on a table.```
* ```Sch-S and IX locks are compatible. You can read and write data at the same time with SI. Dirty reads are prevented with SI.```
* ``` Deadlocks are applied as is.```
* ```T-SQL endpoint silently ignores SET TRANSACTION ISOLATION LEVEL command.```

* T-sql endpoint uses optimistic concurrency control to detect conflicts during writes. The conflicts among transactions are detected at the time of commiting transactions. All concurrent writes (inserts/deletes/updates) can happen in parallel.
    - Write conflict occur between update and delete statements (not insert-insert situation).
    - The transaction that commits first will win between two or more concurrent transactions and the other transactions will fail during a write conflict.

* >**```Note: The Isolation and conflict strategy applied by t-sql endpoint is different from other Microsoft SQL technologies including Azure Synapse Dedicated SQL Pool. We recommend you to modify your ETL\ELT DML logic by adding retry logic whereever applicable, typically stored procedures that are consumed by different ETL applications.```**

* >**```Note: SI lets concurrent updates/deletes can consume additional consume resources and still the transaction can fail. In the future, t-sql endpoint will detect and handle write conflicts at a file level instead of transaction. Example: If two transactions are updating/deleting records in different files, then both update/delete transactions will be committed. If both transactions update/delete records from a same file, then the transaction that commits later will fail because both transactions attempted to update/delete data from a same file. With file level write conflicts, the scope of conflict is smaller compared to an entire transaction. ```**
</details>

* ```TODO```
    - Resource Considerations - TODO
    - Optimizations & Performance Considerations.```

## Security Considerations

TODO

## Development, Automation & ALM

* Customers can use CRUD Api's to create and manage databases but continuous integration and deployment is managed via git integration, database projects and sql packages.
    - Application lifecycle of database can be managed in two ways
        - Fabric GUI, UI driven approach
            - Git Integration can be enabled on a workspace in Fabric GUI. As a customer/ISV, you can version control database code using Fabric GUI.
            - Fabric GUI + deployment pipelines will enable you to push changes from dev -> test -> prod. The deployment pipelines allows users to manage database state in an incremental fashion.
        - Database Projects
            - Fabric DW supports database projects via Azure Data Studio. In near future, database projects will be supported by SSDT as well. Customers can build database code in Azure Data Studio, SSDT and version control using these tools.
            - Once developed, the database code can be deployed using sql package incrementally.

* We recommend you to use GUI approach for small databases. For medium/large complex workloads, use SSDT/ADS + SQl Package to manage ALM of your database. 


    |  Feature | Release Timeline   |
    |:---|---:|
    | Database CRUD API  |  GA |
    | Database Git Integration in Fabric GUI | Post GA, CY 2023 Q4 |
    | Database Projects in VS Code & ADS  | GA |
    | SQL Package | GA  |
    | Deployment Pipelines in Fabric GUI | GA |

## Migration

Microsoft is providing tooling to migration Azure Synapse Gen2 Dedicated SQL Pool to Synapse Data Warehouse in Microsoft Fabric. Migration tooling provided by Microsoft will will cover following.

* Database Code - In Scope
* Data - In Scope
* ELT/ETL tools - Out of Scope

Migration tooling is provided in a phased approach.

| Approach | Capabilities | Release Timeline |
|:---|:---: |---:|
| PowerShell Scripts + Documentation | DDL Extract, DML Assessment, Data Extraction from Gen2 & Data Import to Fabric DW   |  GA |
| DML Assessment & T-SQL Converter | Validate & Convert Gen 2 T-SQL to Fabric DW | Post GA, CY 2023 Q4 |
| LLM | Convert T-SQL scripts using Fabric GUI | GA |
| End-End migration tooling | Gen2 DW Assessment, Code/DML Conversion, Data Import from Gen2 & Export to Fabric DW | CY 2024, Q3 |

## FAQ's


