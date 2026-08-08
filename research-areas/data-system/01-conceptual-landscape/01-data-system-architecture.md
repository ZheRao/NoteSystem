
# Trade-offs in Data Systems Architecture

## Motivation
- Need of application grows in complexity → necessary to combine multiple storage and processing systems that can provide different capabilities
- Things to worry about includes
  - storing and processing large data volumes
  - managing changes to data
  - ensure consistency in failures and concurrency
  - ensure services are highly available
- As application/user base grows, how to choose which database systems with different characteristics are suitable?
  - one team will have one set of priorities, while another have entirely different goals
  - even though they might be working under the same dataset

## Operational vs. Analytical Systems

Frontend vs. Backend
- Backend: 
  - reachable via HTTP, usually consists of application code that **reads and writes data** in one or more databases
  - sometimes interface with additional data systems, such as caches or message queues
  - application code is often ***stateless***
    - meaning when it finishes handling one HTTP request, it forgets everything about the request
    - any info that needs to persist from one request to another needs to be stored either on the client or in the server-side data infrastructure

Different Professions
- **business analysts** — generates reports about the activities of the organization to help management make better decisions
- **data scientists** — look for novel insights in data or create user-facing product features
- **data engineers** — integrate the operational and analytical systems and take responsibility for the organization's data infrastructure more widely
- **analytics engineers** — model and transform data to make it more useful for the business analysts and data scientist in an organization





Operational Systems
- consist of the backend services and data infrastructure where data is created
- typical work
  - look up a small number of records by a key (*point query*)
  - records are inserted, updated, or deleted based on the user's input (***interactive***)
- known as 
  - ***online transaction processing (OLTP)***
- database
  - usually run fixed sets of queries that are baked into the application code
  - one-off custom queries used only occasionally for maintenance or troubleshooting


Analytical Systems
- serve the needs of business analysts and data scientist
  - contain a read-only copy of the data from the operational systems
  - optimized for the types of data processing that are needed for analytics
- typical work
  - an analytical query scans over a huge number of records and calculate aggregate statistics
  - not returning the individual records to the user
- known as 
  - ***online analytical processing (OLAP)***
- datanase
  - usually give usersthe freedom to write arbitrary SQL queries by hand, 
  - or to generate queries automatically using a data visualization or dashboard tool (e.g., Power BI)

Product Analytics (real-time analytics) Systems
- designed for analytical workloads (queries that aggregate over many records)
  - but embedded into user-facing products
- ingest data in **real time** 
  - and are optimized for low-latency query responses
- in contrast
  - typical **OLAP** systems ingest data in batches and are optimized for high-throughput query processing
- e.g., Pinot, Druid, and ClickHouse


Data Warehousing vs. Data Lake

- Data Warehouse
  - Motive:
    - many mostly independent **OLTP** systems with different complexity, that require different teams to maintain them
    - undesirable for business analysts and data scientists to **directly query** these OLTP systems
      - **data silo** — data spread across multiple operation systems, difficult to combine
      - unsuitable and incompatible **schemas and data layouts**
      - expensive analytical queries, impact performace for other users
  - Solution
    - stop using OLTP systems for analytics purposes
    - run the analytics on a seperate database system (***data warehouse***)
  - In essense
    - A ***data warehouse*** is a separate database that analysts can query to their hearts' content, without affecting OLTP operations
      - it stores data very differently from OLTP databases
    - The data warehouse contains a read-only copy of the data from all the various OLTP systems in the company
      - data is extracted from OLTP databases (using either a **periodic data dump** or a **continuous stream of updates**)
      - then data is transformed into an analysis-friendly schema, cleaned up, and then loaded into the data warehouse
      - this is the **ETL**
  - Sometimes it is necessary
    - you do not have direct access to the original database, since it is accessible only via the software vendor's API
    - bring data into your own data warehouse enbales analyses that are not possible via the SaaS API
    - ETL for SaaS APIs is often implemented by specialist data connector services such as Fivetran, Singer, or Airbyte
  - Necessity of **separation**
    - it is good practice for each operational system to have its own database, leading to potentially hundreds of separate operational databases
      - on the other hand, an enterprise usually has a single data warehouse
    - general-purpose systems can handle small data volumes comfortably
      - but the greater the scale, the more specialized systems tend to become

- Data Lake
  - Motive:
    - advanced data science processes often requires turning the rows and columns of a database table into a vector or matrix of numerical value called **features**
    - the need to make data available in a form that is suitable for use by data scientists, the anwer is ***data lake***
  - Compared to **data warehouse**
    - a data lake simply contains files, without imposing any particular file format, data model, or schema