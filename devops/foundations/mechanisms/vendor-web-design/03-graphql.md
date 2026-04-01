# GraphQL — Comprehensive Guide for Data Extraction & API Reverse Engineering

## 1. What GraphQL Is

GraphQL is a **query language and API runtime** created by Facebook in 2015.

It allows clients (web apps, scripts, dashboards) to **request exactly the data they need** from a server rather than receiving fixed responses.

Traditional APIs return predefined payloads. GraphQL allows the **client to shape the response**.

Example conceptually:

Client asks:

```
{ user { id name } }
```

Server responds with only:

```
{ "user": { "id": 1, "name": "Alice" } }
```

This reduces over-fetching and under-fetching.

## 2. Why Software Companies Use GraphQL

Modern web apps (like Harvest Profit) use GraphQL because:

### 2.1. Flexible Data Retrieval

Frontends request exactly what they need.

### 2.2. Fewer API Endpoints

Instead of dozens of REST endpoints:

```
/api/users
/api/users/123
/api/orders
/api/orders/123
```

GraphQL uses **one endpoint**:

```
/graphql
```

### 2.3. Efficient for Complex Data

Nested queries allow retrieving relationships in one request.

Example:

```
{
  farm {
    name
    fields {
      name
      acres
      crops {
        name
        yield
      }
    }
  }
}
```

### 2.4. Ideal for Web Applications

React / modern frontends integrate naturally with GraphQL.

## 3. Core Concepts

### Schema

The schema defines **all available data and operations**.

Example:

```
type Farm {
  id: ID
  name: String
}
```

The schema acts as a **contract between client and server**.

### Query

Used to **retrieve data**.

Example:

```
query {
  farms {
    id
    name
  }
}
```

### Mutation

Used to **modify data**.

Example:

```
mutation {
  createFarm(name:"North Field") {
    id
  }
}
```

### Variables

Allows dynamic inputs.

Example:

```
query GetFarm($id: ID!) {
  farm(id: $id) {
    name
  }
}
```

Request payload:

```
{
  "variables": {"id": 42}
}
```

### Resolver

Resolvers are server functions that actually fetch data from:

• databases  
• other APIs  
• services  

Example conceptual flow:

```
Client Query
   ↓
GraphQL Server
   ↓
Resolver
   ↓
Database
```

## 4. What You Saw in Harvest Profit

You observed something like:

```
POST /graphql

{
  operationName: "LegacyGrainInventoryLoadsExportMutation",
  query: "mutation LegacyGrainInventoryLoadsExportMutation...",
  variables: {...}
}
```

This means:

1. The frontend triggers a **GraphQL mutation**.
2. The server schedules an **export job**.
3. The server returns a **job ID**.
4. The client polls for completion.
5. When ready → download CSV.

Typical flow:

```
User clicks Export
     ↓
Mutation request
     ↓
Server creates export job
     ↓
Job runs in background
     ↓
Client polls status
     ↓
Download link appears
```

## 5. GraphQL Request Structure

Typical HTTP request:

```
POST /graphql
Content-Type: application/json
```

Body:

```
{
  "operationName": "QueryName",
  "query": "query QueryName { ... }",
  "variables": {}
}
```

Important parts:

| Field         | Purpose              |
| ------------- | -------------------- |
| operationName | identifies operation |
| query         | GraphQL query text   |
| variables     | dynamic parameters   |

## 6. How to Interact With GraphQL (Automation)

Python example:

```python
import requests

url = "https://example.com/graphql"

payload = {
    "operationName": "GetFarms",
    "query": "query GetFarms { farms { id name } }",
    "variables": {}
}

response = requests.post(url, json=payload)
print(response.json())
```

Authentication typically uses:

• session cookies  
• JWT tokens  
• API keys

## 7. Reverse Engineering GraphQL APIs

Typical workflow:

1. Open browser DevTools
2. Go to Network tab
3. Filter for **graphql** requests
4. Inspect request payload
5. Copy query + variables
6. Replicate with Python requests

Key items to capture:

• headers  
• cookies  
• query  
• variables  
• operationName

## 8. Introspection (GraphQL Superpower)

Many GraphQL APIs expose **schema introspection**.

This allows discovering the entire API.

Example query:

```
{
  __schema {
    types {
      name
    }
  }
}
```

Tools:

• GraphiQL  
• Apollo Studio  
• Insomnia

However many production APIs disable this.

## 9. GraphQL vs REST

REST:

```
GET /users
GET /users/1
GET /users/1/orders
```

GraphQL:

```
{
  user(id:1) {
    name
    orders {
      total
    }
  }
}
```

Advantages:

• flexible  
• efficient  
• single endpoint

Disadvantages:

• harder caching  
• complex backend

## 10. How Companies Build Data Export With GraphQL

Typical architecture:

```
Web App
   ↓
GraphQL API
   ↓
Resolver
   ↓
Background Job
   ↓
CSV generator
   ↓
Cloud storage
   ↓
Download link
```

Reasons:

• avoid blocking requests  
• handle large datasets  
• allow async processing

## 11. Common GraphQL Server Implementations

NodeJS

• Apollo Server  
• GraphQL Yoga  
• Express GraphQL

Python

• Graphene  
• Strawberry

Ruby

• graphql-ruby

Java

• graphql-java

## 12. Common Alternatives

### REST API

Most traditional systems.

```
GET /orders
```

### gRPC

High-performance internal APIs.

Uses protobuf.

Common in microservices.

### OData

Used by Microsoft ecosystems.

Example:

```
/orders?$filter=status eq 'open'
```

### Direct Data Exports

Some systems simply provide:

• CSV download endpoints  
• S3 links

## 13. Typical Data Extraction Strategy

Order of preference:

1️⃣ Official API

2️⃣ GraphQL requests

3️⃣ Hidden REST endpoints

4️⃣ CSV export endpoints

5️⃣ Browser automation


## 14. Signals You Are Dealing With GraphQL

Look for:

```
/graphql
```

Payload contains:

```
query
mutation
variables
```

Response structure:

```
{
  "data": {...}
}
```

Errors appear as:

```
{
  "errors": [...]
}
```

## 15. Key Takeaways

GraphQL is:

• a query language  
• an API runtime  
• a flexible data retrieval system

For data extraction engineers it means:

**you can replicate frontend requests directly**.

The browser is usually the best documentation.


## 16. Practical Mindset for Reverse Engineering

Always ask:

1. What query triggered the export?

2. What variables control the dataset?

3. Does it return data directly or create a job?

4. Where is the download link generated?

5. What authentication is required?

