Microsoft Fabric enables partners to integrate ISV solutions in three different ways
* Interop Scenario - In this scenario, ISV's integrate their analytical solutions with Microsoft Fabric workloads by ingesting curated/prepared data into Warehouse, Lakehouse or by setting up external shortcuts into Lakehouse and consume data by Power BI direct lake mode. Customers can use Microsoft Fabric compute or external applications such as Azure Synapse, ADF, Databricks to ingest data into Microsoft Fabric. Please review warehouse and lakehouse guidance documentation for more information. 

* Develop with Fabric Scenario - ISV's can embed their IP using Microsoft Fabric by using REST API endpoints provided by plaform and individual workloads. There are two scenarios that ISV's should think about
    - ISV owns Data: ISV's responsible to collect, process and curate data in ISV Microsoft Fabric tenant and each customer gets their own workspace to build on top of the data produced that t
    - Customer Owns Data: ISV's deploying solutions on Customer Microsoft Fabric tenants. In this scenario, ISV's will deploy their solutions on customer Azure tenant and run their solutions on Microsoft Fabric in customer tenant. In this case, ISV is responsible to setup and configure infrastructure required to run ISV's solution but customer is responsible to manage, operate and own the process and data.