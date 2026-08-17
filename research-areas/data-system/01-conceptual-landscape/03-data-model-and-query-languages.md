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