# Data Models and Query Languages

Data Models
- effect
  - how the software is written
  - how we **think about the problem** that we are solving
- software as layered data models
  - example
    - L1 — **model** the real world in terms of objects or data structures and APIs that manipulate data structures
      - e.g., money flows, sensors, ...
    - L2 — need to **store** those data structures, express them in terms of a general-purpose data model
      - such as JSON, XML documents, tables in a relational database, or vertices and edges in a graph
    - L3 — database software that decides a way of **representing** that document, relational, or graph **data** in terms of **bytes** in memory, on disk, or on a network
      - allow the data to be queried, searched, manipulated, and processed in various ways
    - L4 — hardware engineers the figure out how to **represent bytes** in terms of electrical current, pulses of light, magnetic fields, and more
  - complex application can have many more intermediary levels, but with the same **basic idea**
    - each layer hides the complexity of the layers below it by providing a clean data model
- data models
  - relational model, document model, graph-based data models, event sourcing, and DataFrames
  - some types of data and some queries are easy to express in one model and awkward in another

**Declarative Query Languages**
- many query languages are declarative
  - specify the pattern of the data you want
    - what conditions the results must meet 
    - how you want the data to be transformed (e.g., sorted, grouped, and aggregated)
    - but not **how** you achieve that goal
- in contrast, with most programming languages, you would have to write an algorithm telling the computer which operations to perform in which order

## Relational vs. Document Models

Brief evolution
- best-kown data model is **SQL**
  - data is organized into **relations** (called tables in SQL), where each realtion is an unordered collection of **tuples** (rows in SQL)
- each subsequent competitor to the relational model generated a lot of hype in its time, but none lasted
  - instead, SQL has grown to incorporate other types of data
  - e.g., adding support for XML, JSON, and graph data
- **NoSQL**
  - a single technology
    - a loose set of ideas around new data models, schema flexibility, scalability, and a move toward open source licensing models
  - one lasting effect of NoSQL is the popularity of the **document model**, which usually represents data as **JSON**
    - originally popularized by specialized document databases such as MongoDB and Couchbase
    - although most relational databases have now also added JSON support

---
### The Object-Relational Mismatch

Criticism of SQL data model
- much application development is done in **object-oriented programming languages**
- there must be an awkward translation layer between the objects in the application code and the database model of tables, rows, and columns

Object-relational mapping (**ORM**)
- frameworks that reduce the amount of biolerplate code required for the awkward transition layer
- common cited problems
  - ORMs are complex and can't completely hide the differences between the two models
  - generally used for **OLTP** app development
    - for analytics purposes, the design of the relational schema still matters when using an ORM
  - many work only with relational OLTP databases
    - systems like search engines, graph databases, and NoSQL systems might find ORM **support lacking**
  - make it easy to accidentally write inefficient queries
- advantages
  - for data well suited to a relational model
    - some kind of translation between the persistent relational and in-memory object representation is inevitable
    - ORM reduce the amount of boilerplate code required for this translation
  - some help with caching the results of database queries, which can help reduce the load on the database
  - help with managing schema migrations and other administrative activities

---
### Document data model for one-to-many relationships

Motivation
- not all data lends itself well to a relational representation
  - e.g., consider a LinkedIn profile
    - `first_name`, `last_name` appear exactly once per user, so they can be modeled as columns on the user table
    - but most people have had more than one job in their career, and people may have varying numbers of periods of education

Two approaches to represent one-to-many relationship
- one way is to put positions, education, and contact information in separate tables, each with a foreign-key reference to the `users` table
- another way is as a JSON document
  - it is perhaps more natural and more closely object structure in application code

Benefits of JSON representation
- locality
  - fetching a profile in relational example involves 
    - performing multiple queries (each table by `user_id`)
    - performing a messy multiway join between the users table and its subordinate tables
  - JSON representation have all relevant information in one place, making the query both faster and simpler
- representation of tree structure
  - one-to-many relationships from the user profile to the user's positions, education history, and contact info imply a tree structure 
  - JSON representation makes this tree structure explicit
  - **example** tree structure  
    ![alt text](images/0301.png)

---
### Normalization, Denormalization, and Joins

ID vs. Text String
- for example, `region_id` vs. `Washington, DC, United States`
- advantages to have standardized list of geographic regions and let users choose from a drop-down list or autocomplete
  - consistent style and spelling
  - avoid ambiguity if several places have the same name
    - if the string were just Washington, DC, would it refer to DC or to the state?
  - ease of updating — the name is stored in only one place, it is easy to update across the board
  - localication support — when the site is translated into other languages, the standardized lists can be localized
    - so the region can be displayed in the viewer's language
  - better search functionality
    - a search for pepole on the US East Coast can match this profile, because the list of regions can encode the fact that Washington is located on the East Cost

Normalization
- whether to store an ID or a text string is a question of **normalization**
  - using an ID is more normalized: the information that is meaningful to humans is stored in only one place, and everything that refers to it uses an ID
  - when storing text directly, you are duplicating the human-meaningful information in every record that uses it
    - this representation is **denormalized**
- advantage of using an ID
  - it never needs to change
    - the ID can remain the same even if the information it identifies changes
    - anything meaningful to humans may need to change sometime in the future — and if the information is duplicated, all the redundant copies will need to be updated
- downside of normalized representation
  - every time you want to display a record containing an ID, you have to do an additional lookup to resolve the ID into something human-readable
  - i.e., **join**s
- **document databases** can store both normalized and denormalized data
  - but they are often **associated with denormalization** 
    - partly because the JSON data model makes it easy to store additional denormalized fields
    - partly because the weak support for joins in many document databases makes normalization inconvenient

Trade-offs of normalization
- motivation
  - in the LinkedIn profile example
    - `region_id` field is a reference to a standardized set of regions
    - `organizations` and `school_name` are just strings
      - these are denormalized: many people may have worked at the same company, but there is no ID linking them
  - it is worth considering whether the organization and school name should be entities instead, and the profile should reference their IDs
    - the same arguments for referencing the ID of a region also apply here
- **general principle**
  - normalized data is
    - faster to write (since there is only one copy)
    - slower to query (since it requires joins)
  - denormalized data is usually
    - faster to read (fewer joins)
    - more expensive to write (more copies to update, more disk space used)
  - additional consideration
    - need to consider the consistency of the database if a process crashes halfway through making its updates
- normalization form vs. type of system
  - normalization tends to be better for **OLTP** systems
    - where both reads and updates need to be fast
  - **analytical system** often fare better with denormalized data
    - since they perform updates in bulk
    - and the performance of read-only queries is the dominant concern
  - **small to moderate scale**
    - a normalized data model is often best because 
      - you don't have to worry about keeping multiple copies of the data consistent with one another
      - cost of performing joins is acceptable
  - **very large-scale systems**
    - cost of joins can become problematic

---
### Denormalization in the social networking case study

Normalized representation vs. denormalized one
- compare the two approaches to assemble the post timeline
  - original joins between posts and follows were too expensive, and the materialized timeline is a cache of the result of the joins
  - fan-out process that inserts a new post into followers' timelines was our way of keeping the denormalized representation consistent
- actual implementation
  - in the fan-out method, Twitter does not store the actual text of each post
  - each entry stores only 
    - the post ID
    - the ID of the user who posted it
    - a little bit of extra information to identify reposts and replies
  - this means
  - whenever the timeline is read, the service still needs to perform two joins
    - it looks up the post ID to fetch the actal post content (as well as statistics such as the number of likes and replies)
    - it looks up the sender's profile by ID (to get their username, profile picture, and other details)
- **hydrating** — the process of looking up the human-readable information by ID

Architectural choice
- reason for storing only IDs in the precomputed timeline is that the data they refer to is fast-changing
  - number of likes and replies may change multiple times per second on a popular post
  - some users regularly change their username or profile photo
- denormalizing this information into the materialized timeline would not make sense
  - since the timeline should show the latest like count and profile picture when it is viewed
  - and storage cost would be increased significantly by such denormalization
- hydrating post and user IDs is actually a fairly easy **operation to scale**
  - since it parallelizes well
  - and the cost doesn't depend on the number of accounts you are following or the number of followers you have

---
### Many-to-One and Many-to-Many Relationships

In the profile example
- **positions** and **education** tables are examples of **one-to-many** relationship
  - i.e., one resume has several positions, but each position belongs only to one resume
- `region_id` field is an example of **many-to-one** relationship
  - i.e., many people live in the same region, but we assume that each person lives in only one region at any one time
- `organizations` is an example of `many-to-many` relationships
  - i.e., one person may have worked for several organizations, and an organization has several past or present employee

Data structure and querying
- many-to-one and many-to-many relationships do not easily fit within one self-contained JSON document, they lend themselves more to a **normalized** representation
- many-to-many relationships often need to be queried in **both directions**
  - e.g., finding all the organizations that a particular person has worked for, and finding all the people who have worked at a particular organization
  - one way of enabling such queries is to store ID references on both sides
    - such that 
      - a resume includes the ID of each organization where the person has worked
      - the organization document includes the IDs of the resumes that mention that organization
    - this presentation is **denormalized**, since the relationship is stored in two places, which could become inconsistent with each other
- a normalized representation stores the relationship in only one place and relies on **secondary indexes**
  - allow the relationship to be efficiently queried in both directions

---
### Stars and Snowflakes: Schemas for Analytics

Widely used conventions for structure of tables in a data warehouse
- star schema
- snowflake schema
- dimensional modeling
- one big table (OBT)

Star schema
- structure
  - at the center of the schema is **fact table**
    - each row of the fact table represents an event that occurred at a particular time
      - it allows maximum flexibility of analysis later
      - but it can become extremely large
  - some columns in the fact table are **attributes**, such as the price at which the product was sold and the cost of buying it from the supplier
  - other columns in the fact table are foreign-key references to other tables, called **dimension tables**
    - dimensions represent the *who, what, where, when, how,* and *why* of the event
    - queries often involve multiple joins to multiple dimension tables
    - even date and time are often represented using dimension tables
      - allows additional information about dates (e.g., public holiday) to be encoded
      - enabling queries to differentiate between sales on holidays and non-holidays

Snowflake schema
- when dimensions of star schema are further broken into subdimensions
  - e.g., there could be separate tables for brands and product categories, 
  - and each row in the `dim_product` table could reference the brand and category as foreign keys, rather than strings in the `dim_product` table
- is more normalized than star schema, but star schemas are often preferred because they are simpler for analysts to work with

One big table (OBT)
- motivation
  - star or snowflake schema consists mostly of many-to-one relationships
    - e.g., many sales occur for one particular product, in one particular store
  - in principle, other relationship types could exist, but they are often denormalized to simplify queries
    - e.g., if a customer buys several different products at once, that multi-item transaction is not represented explicitly
    - instead, the fact table has a separate row for each product purchased, and those facts all just happen to have the same customer ID, store ID, and timestamp
- some data warehouse schemas take denormalization even further and leave out the dimension tables entirely
  - folding the information in the dimensions into denormalized columns in the fact table instead
  - essentially, precomputing the join between the fact table and the dimension tables
  - this approach is known as **one big table** (OBT)
- trade-off
  - it requres more storage space, it sometimes enables faster queries
  - in **analytics**, denormalization is unproblematic, since the data typically represents a log of historical data that is not going to change
  - the issue of consistency and write overheads that occur with denormalization in **OLTP** systems are not as pressing in analytics

---
### When to Use Which Model

Quick Overview
- document data model
  - schema flexibility
  - better performance due to locality
  - closer to the object model for some applications
- relational model
  - better support for joins and many-to-one and many-to-many relationships

Document model
- preferred when data in the application has a document-like structure
  - i.e., a try of one-to-many relationships
  - where typically the entire tree is loaded at once
- relational technique of **shredding** can lead to cumbersome schemas and unnecessarily complicated application code
  - shredding: splitting a document-like structure into multiple tables
- limitations
  - cannot refer directly to a nested item within a document
    - instead, you need to say something like, "the second item in the list of positions for user 251"
    - if you need to reference nested items, a relational approach works better, since you can refer to any item directly by its ID
- additional advantage
  - some applications allow the user to choose the order of items (e.g., to-do list)
  - document model supports such application well
    - because the items (or their IDs) can simply be stored in a JSON array to determine their order
  - in relational databases, there isn't a standard way of representing such reorderable lists various tricks are used, such as 
    - sorting by an integer column (requiring renumbering when you insert into the middle)
    - maintaining a linked list of IDs
    - using fractional indexing


---
### Schema flexibility in the document model

shcema-on-read vs. schema-on-write
- motivation
  - most document databases, and the JSON support in relational databases, do not enforce any schema on the data in documents
  - no schema means that
    - arbitrary keys and values can be added to a document
    - when reading, clients have no guarantees as to what fields the documents may contain
- **schema-on-read**
  - the structure of the data is implicit and interpreted only when the data is read
  - code that reads the data from document databases usually assumes some kind of structure
    - that is, there is an **implicit schema**, but it is not enforced by the database
- **schema-on-write**
  - the traditional approach of relational databases
  - the schema is explicit and the database ensures that all data conforms to it when the data is written

Difference between approaches
- the difference is particularly noticeable when an application wants to change the format of its data
- e.g., before: storing each uer's full name in one field; after: store the first name and last name separately
  - document database
    - just start writing new documents with the new fields and have code in the application that handles the case when old documents are read, for example
        ```java
        if (user && user.name && !user.first_name) {
          // documents written before Dec 8, 2023 don't have first_name
          user.first_name = user.name.split(" ")[0];
        }
        ```
    - **downside**
      - every part of application that reads from the database now needs to deal with documents in old formats
  - relational databases
    - would typically perform a **migration** along the lines of
        ```sql
        ALTER TABLE users ADD COLUMN first_name text DEFAULT NULL;
        UPDATE users SET first_name = split_part(name, '', 1);  --PostgreSQL
        UPDATE users SET first_name = substring_index(name, '', 1); --MySQL
        ```
    - adding a column with a default value is fast and unproblematic, even on large tables
    - running the UPDATE statement is likely to be slow on a large table
      - since every row needs to be rewritten
      - and other schema operations (such as changing the datatype of a column) also typically require the entire table to be copied

---
### Data locality for reads and writes

Trade-offs
- fact: a document is usually stored as a single continuous string, encoded as JSON, XML, or a binary variant (e.g., MongoDB's BSON)
- **locality advantageous** if application often needs to **access the entire document** (e.g., to render it on a web page)
  - if data is **split across multiple tables** 
    - multiple index lookups are required to retrieve it all
    - may require more disk seeks and take more time
  - locality advantage only applies if large parts of the document are needed at the same time
    - i.e., database needs to load the entire document
    - wasteful if only need to access a small part of a large document
- **downside**: on **updates** to a document, the entire document usually needs to be rewritten
  - it is generally recommended that you keep documents fairly small and avoid frequent small updates

Storing related data together for locality outside of document model
- Google's Spanner database offers the same locality properties in a relational data model
  - by allowing the schema to declare that a table's rows should be interleaved (nested) within a parent table
- Oracle allows the same thing, using a feature called **multi-table index cluster tables**
- **wide-column** data model popularized by Google's Bigtable and used, for example, in HBase and Accumulo has **column families**
  - which have a similar purpose of managing locality


---
### Query languages for documents

Most relational databases are queried using **SQL**, but document databases are more varied
- some only allow key-value access by primary key
- others also offer secondary indexes to query for values inside documents
- some provides rich query language

Examples of query languages
- XML databases are often queried using XQuery and XPath
  - allows complex queries, including joins across multiple documents, and format results as XML
- JSON Pointer and JSONPath provide an equivalent XPath for JSON
- MongoDB's aggregation pipeline is an example of a query language for collections of JSON documents

Example of query
- scenario
  - marine biologist adds an observation record to database every time sees animals in the ocean
  - now wants to generate a report saying **how many sharks that have been sighted per month**
- PostgreSQL
    ```SQL
    SELECT date_trunc('month', observation_timestamp) AS observation_month,
      sum(num_animals) AS total_animals
    FROM observations
    WHERE family = 'Sharks'
    GROUP BY observation_month;
    ```
- MongoDB's aggregation pipeline
    ```js
    db.observations.aggregate([
      {$match: {family:"Sharks"}},
      {$group: {
        _id: {
          year: {$year: "$observationTimestamp"},
          month: {$month: "$observationTimestamp"}
        },
        totalAnimals: {$sum: "numAnimals"}
      }}
    ]);
    ```

---
### Convergence of document and relational databases

Document databases and relational databases started out as very different approaches to data management, but they have grown more similar over time
- **relational databases** 
  - added support for JSON types and query operators, 
  - and the ability to index properties inside documents
- some **document databases** 
  - added support for joins, secondary indexes, and declarative query languages
- relational-document hybrids are a powerful combination
  - many document databases need relational-style references to other documents
  - many relational databases have sections where schema flexibility is beneficial
