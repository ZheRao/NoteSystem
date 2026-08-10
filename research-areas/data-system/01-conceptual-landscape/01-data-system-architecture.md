
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

## Operational (Transactional) vs. Analytical Systems

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


### Data Warehousing vs. Data Lake

Data Warehouse
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

Data Lake
- Motive:
  - advanced data science processes often requires turning the rows and columns of a database table into a vector or matrix of numerical value called **features**
  - the need to make data available in a form that is suitable for use by data scientists, the anwer is ***data lake***
- Compared to **data warehouse**
  - a data lake simply contains files, without imposing any particular file format, data model, or schema
  - besides being flexible, a data lake is often cheaper than relational data storage, since it can use commoditized file storage such as object stores
- Inside a data lake
  - files might be collections of database records, encoded using a file format such as Avro or Parquet
  - can contain text, images, videos, sensor readings, sparse matrices, feature vectors, genome sequences, or any other kind of data
- Part of **ETL**
  - some cases, the data lake has become an intermediate stop on the path from the **operational systems** to the **data warehouse**
  - data lake contains "raw" data produced by the operational systems, without transforming into a relational data warehouse schema
  - **advantage** — each consumer of the data can transform the raw data into the form that best suits their needs

Beyond the Data Lake
- **Stream of events**
  - file-based data analytics reruns the analysis periodically to respond to changes in the data
  - stream processing allows analytical systems to respond to events much faster, on the order of seconds
- **reverse ETL**
  - outputs of analytical systems are made available to operational systems
  - e.g., ML models can be deployed to operational systems by using specialized tools such as TFX, Kuberflow, or MLflow

### Systems of Record vs. Derived Data

System of Record
- known as ***source of truth*** — holds authoritative or canonical version of data
  - e.g., new data first comes in is written here
  - each fact is represented exactly once
- purpose
  - primary databases to which data is first written


Derived Data Systems
- taking existing data from another system and transforming or processing it in some way
  - if you lose derived data, you can re-create it from the original source
- **redundant**
  - because it duplicates existing info
  - however **essential** for getting **good performance** on read queries
  - can derive several databases from a single source, enabling looking at the data from different points of view
- purpose
  - indexes and caches that speed up common read operations, especially for queries that the **system of records** cannot answer efficiently
- key note
  - by being clear about which data is derived from which other data, clarity can be brought to an otherwise confusing system architecture
- **data integration**
  - by bringing data from another system, you need a process for updating the derived data when the original in the system of record changes
  - **unfortunately** many databases are designed so that the target application will always need to use only that one database
  - it's not easy to integrate multiple systems in order to propagate such updates.

## Cloud Vs. Self-Hosting

In-house vs. Outsourced
- rule of thumb
  - in-house — core competency or competitive advantage for the organization
  - outsource — non-core, routine, or commonplace should left to a vendor
- software
  - extreme #1 — write and run software in-house
  - extreme #2 — cloud services or SaaS products
    - implemented and operated by an external vendor
    - access only through a web interface of API
  - middle ground — off-the-shelf software (open source or commercial) that **self-host**, or deploy yourself
    - e.g., download MySQL and install it on a server you control, with your own hardware **(on premise)**
- related question: how to deploy services, either in the cloud or on premises
  - e.g., whether use an orchestration framework such as Kubernetes
  - not in scope of this book

### Pros and Cons of Cloud Services

Which option is cheapter and easier?
- In-house if
  - experienced setting up and operating the systems
  - load is quite predictable (i.e., the number of machines needed doesnot fluctuate wildly)
  - want to configure and tune perforamce on particular workload
- Cloud service if
  - need a system you don't know how to deploy and operate
    - hiring and training staff specifically to maintain and operate the system can get very expensive
    - you still need an operations team when you're using the cloud
    - outsurcing the basic system administration can free up your team to focus on higher-level concerns
  - want potential better service from privoder's operational expertise from many customers
  - load on your system varies a lot over time
    - if machines provisioned to handle peak load, then most of the time computing resources are idle 
    - cloud services have the advantage to scale computing resources up or down in response to changes in demand
  - dataset is so large that querying it quickly requires significant computing resources
    - cloud can save money, since you can return unused resources to the provider rather than leaving them idle

Downsides of cloud services — **lack of control**
- lack of feature — have to ask the vendor to add
- service goes down — wait for recovery
- triggers a bug or causes performance problems — diagnosing is difficult
- service shut down, become unacceptably expensive, changes their product in a way you don't like — you are at their mercy 
- trust for data security?

### Cloud Native System Architecture

Cloud native
- rise of the cloud has a profound effect on how data systems are implemented on a technical level
- ***cloud native*** describes an architecture that is designed to take advantage of cloud services
- advantages
  - better performance on the same hardware
  - faster recovery from failures
  - ability to quickly scale computing resources 
  - supporting larger datasets
- examples of self-hosted and cloud native data systems

  |category|self-hosted systems|cloud native systems|
  |---|---|---|
  | Operational/OLTP | MySQL, PostgreSQL, MongoDB | AWS Aurora, Azure SQL DB Hyperscale, Google Cloud Spanner |
  | Analytical/OLAP | Teradata, ClickHouse, Spark | Snowflake, Google BigQuery, Azure Synapse Analytics | 

Layers of cloud services
- self-hosted — run on conventional operating system such as Linux or Windows
  - store data as files on the filesystem
  - communicate via standard network protocals such as TCP/IP
- cloud — run on an IaaS environment, using one or more VMs
  - with a certain allocation of CPUs, memory, disk, and network bandwidth
    - can be provisioned faster and come in a greater variety of sizes
    - otherwise they are similar to traditional computers
- layers
  - key idea of cloud also involves building upon lower-level cloud services to create higher-level services
  - object storage services such as Amazon S3, Azure Blob Storage, and Cloudflare R2 store large files
    - they provide more limited APIs than a typical filesystem (basic file reads and writes)
    - their advantage is hiding the underlying physical machine
  - many other services are built upon object storage and other cloud services
    - e.g., Snowflake is a cloud-based analytical database (data warehouse) that relies on S3 for data storage
    - and some other services are built upon Snowflake

Separation of storage and compute
- storage
  - **RAID** (redundant array of independent disks)
    - used to maintain copies of the data on several disks attached to the same machine
    - can be implemented either in hardware or in software by the operating system
    - is transparent to the applications accessing the filesystem
- storage options
  - **local disks** (VM)
    - treated more like an ephemeral cache and less like long-term storage
    - because lcoal disk becomes inaccessible if the associated instance fails, or being replaced
  - **virtual disk storage**
    - can be detached from one instance and attched to a different one
      - e.g., Amazon EBS, Azure managed disks, persistant disks in Google Cloud
    - not a physical disk, but a cloud service provided by a separate set of machines that emulates the behavior of a disk — **block device**
      - typically 4 KiB in size
    - flaws
      - introduces overheads that can be avoided in systems that are designed from the ground up for the cloud
      - applications sensitive to network glitches, since every I/O operation on the virtual block device is a network call
  - **storage services**
    - avoid using virtual disks, and offer dedicated storage services that are optimized for particular workloads
    - object storage services such as S3 are designed for long-term storage of fairly large files
      - ranging from hundreds of kilobytes to several gigabytes in size
    - cloud databases typically manager smaller values in a separate service and store larger data blocks (containing many individual values) in an object store
- cloud-native storage characteristics
  - **separation of storage**
    - in tranditional system architecture, same computer is responsible for both storage (disk) and computation (CPU and RAM)
    - cloud-native systems separate these two responsibilities
      - e.g., S3 only stores files, analytical code must run somewhere outside of S3
      - implies transferring the data over the network, which will discuss later
  - **multitenant**
    - rather having a separate machine for each customer, data and computation from several customers are handled on the same shared hardware by the same services
    - requires careful engineering to ensure that one customer's activity does not affect the performance or security of the system for other customers

### Operations in the Cloud Era

- before
  - people managing an organization's server-side data infrastructure were known as ***database administrators (DBAs)*** or ***system administrators (sysadmins)***
- now
  - organizations tried to integrate the roles of software development and operations into teams with a shared responsibility for both backend services and data infrastructure
    - the ***DevOps*** philogophy has guided this trend
    - ***site reliability engineers (SREs)*** are Google's implementation of this idea
- roles of operation
  - **goal** 
    - reliably deliver services to users (including configuring infra and deploying applications)
    - ensure a stable production environment (including monitoring and diagnosing any problems that may affect reliability)
  - **self-hosted systems**
    - significant amount of work at the level of individual machines, such as capacity planning 
    - e.g.,
      - monitoring available disk space and adding more disks before running out of space
      - provisioning new machines
      - installing OS patches
  - **cloud services**
    - present API that hides the individual machines implementing the service
      - can store data without planning capacity needs in advance
      - services remain highly available, even when individual machines have failed

Operations with cloud services
- some examples
  - choosing the most appropriate service for a given task
  - integrating services with each other
  - migrating from one service to another
- caution
  - important to know what resources are being used for which purposes so you don't waste money on resources that are not needed
  - cloud services have resource limits or **quotas**
    - such as maximum number of processes you can run concurrently
    - should be planned around before run into them
  - other operational aspects
    - maintain security of application and libraries it uses
    - manage interactions between your own services
    - monitor load on your services
    - track down cause of problems such as performance degradations or outages

## Distributed vs. Single-Node Systems

- ***distributed system*** — system that involves several machines communicating via a network
- ***node*** — each of the processes participating in a distributed system

### Problems with Distributed Systems

- key constraints
  - every request and API call that traverses the network needs to deal with the possibility of failure
  - making a call to another service could be vastly slower than calling a function in the same process even though datacenter networks are fast
    - e.g., for large volumes of data, it can be faster to bring the computation to the machine that already has the data
  - more nodes are not always faster 
    - in some cases, a simple single-threaded program on one computer can perform significantly better than a cluster with over 100 CPU cores
  - troubleshooting a distributed system is often difficult
    - if the system is slow to respond, how do you figure out where the problem lies?
    - techniques are developed under `observability` chapter
- key problem it solves
  - ensure consistency
    - databases provide various mechanisms for ensuring data consistency, but each service has its own database
    - distributed transactions are a possible technique for ensuring consistency
    - but rarely used in **microservices context** because they counter the goal of making services independent from each other
- single machine vs. cluster
  - performing a task on a single machine is often simpler and cheaper than setting up a distributed system
  - CPUs, memory, and disks have grown larger, faster, and more reliable
  - when combined with single-node databases such as DuckDB, SQLite, and KuzuDB, many workloads can now run on a single node

### Microservices and Severless

Microservices
- key mechanics
  - most common way of distributing a system across multiple machines is to divide them into **clients and servers** and let clients make requests to servers
    - HTTP is most commonly used for this communication
  - the same process may be both
    - a server (handling incoming requests)
    - a client (making outbound requests to other services)
- evolution
  - **service-oriented architecture (SOA)**
    - traditional way of building applications
  - **microservices architecture**
    - recent refinement
    - a service has one weel-defined purposes
    - each service exposes an API that can be called by clients via the network
    - complex application can be decomposed into multiple interacting services
- advantages
  - each service 
    - can be updated independently
    - can be assigned the hardware resources it needs
  - hiding the implementation details behind an API
    - service owners are free to change the implementation without affecting clients
  - independent data storage
    - each service can have its own databases and not to share databases between services
    - shared database
      - make database structure part of the service's API → difficult to change
      - cause one service's queries to negatively impact the performance of other services
- complexity
  - in testing a service
    - have to run all the other services that it depends on
  - in deployment
    - each service requires 
      - infrastructure for depoying new releases
      - adjusting the allocated hardware resources to match the load
      - collecting logs
      - monitoring service health
      - alerting an on-call engineer in the case of a problem
    - Kubernetes have become a popular way of deploying services, since they provide a foundation for this infrastructure
- challenge to evolve
  - scenario
    - clients expect API to have certain fields
    - add or remove fields in an API as business needs change
    - cause clients to fail, and not discovered until later in the development cycle
  - API description standards such as OpenAPI and gRPC help manage the relationship between client and server APIs
- caution
  - using microservices is likely to be unnecessary overhead in small companies

Serverless (*function as a service*)
- another approach to deploy services — management of infrastructure is outsourced to a cloud vendor
  - VMs — explicitly choose when to start up or shut down an instance
  - serverless model — cloud provider automatically allocates and frees hardware resources as needed, based on the incoming requests to your service
    - pay only for the time your application code is running rather than having to provision resources in advance
- caution
  - providers impose a time limit on function execution and limit runtime envrionments
  - services might suffer from slow start times when a function is first invoked

### Cloud Computing vs. Supercomputing

- Cloud computing is not the only way of building large-scale computing systems 
- **high-performance computing (HPC)**, also known as **supercomputing** is another way with different priorities and uses different techniques

Differences
- purpose
  - supercomputers — computationally intensive scientific computing tasks
    - weather forecast
    - molecular dynamics
    - complex optimization problems, and solving PDEs
  - cloud computing — serve user requests with high availability
    - online services
    - business data systems
- jobs
  - supercomputer — large batch jobs that checkpoint the state of their computation to disk from time to time
    - node fails — stop entire cluster workload —repair the faulty node — restart computation from last checkpoint
  - cloud service — prioritize continually serve users with minimal interruptions
    - stopping entire cluster is not desirable
- resources sharing
  - supercomputer — typically communicate through shared memory and RDMA
    - supports high bandwidth and low latency
    - assume a high level of trust
  - cloud computing — network and machines are shared by mutually untrusting organizations
    - require stronger security mechanisms such as resource isolation
- physical machines
  - supercomputer — generally have nodes close together
  - cloud computing — allows nodes to be distributed across multiple geographic regions