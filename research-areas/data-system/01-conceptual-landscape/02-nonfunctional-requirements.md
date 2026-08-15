# Defining Nonfunctional Requirements

Functional vs. Nonfunctional Requirements
- functional requirements — functionality that the application must offer
  - what screens and what buttons you need
  - what each operation is supposed to do in order to fulfill the purpose of your software
- nonfunctional requirements
  - e.g., the app should be fast, reliable, secure, legally compliant, and easy to maintain
  - specifically, this chapter covers
    - defining and measuring the **performance** of the a system
    - **reliability** — continuing to work correctly, even when things go wrong
    - **scalability** — efficient ways of adding computing capacity
    - easier to **maintain**

## Case Study: Social Network (e.g., Twitter) — Home Timelines

- Assumptions
  - users make a total of 500 million posts per day, or 5,800 posts per second on average
    - the rate can spike to as high as 150,000 posts per second occasionally
  - average user follows 200 people and has 200 followers

---
### Representing Users, Posts, and Follows

Keep all data in a relational database as follows:
- follower table
    |follower_id|followee_id|
    |---|---|
    |1705506|12|
- post table
    |id|sender_id|text|timestamp|
    |---|---|---|---|
    |20|12|just setting up my twttr|1142974214|
- user table
    |id|screen_name|profile_image|
    |---|---|---|
    |12|jack|1234567.jpg|

Assume
  - main read operation must be supported is **home timeline**
    - displays recent posts by people the user is following

SQL query

```sql
SELECT posts.*, users.*
    FROM follower
    JOIN post ON follower.followee_id = post.sender_id
    JOIN user ON post.sender_id = user.id
    WHERE user = current_user
    ORDER BY post.timestamp DESC
    LIMIT 1000
```

Objective — after somebody makes a post, want followers to be able to see it within five seconds
- ***polling*** — user's client repeat the query every five seconds while the user is online
  - if assume 10 million users are online
    - running the query 2 million times per second, this is a lot
  - query also needs to fetch a list of recent posts by each of those 200 people the user is following and merge those lists
    - 2 million timeline queries per second times 200 followed accounts makes **400 million** lookups per second

---
### Materializing and Updating Timelines

Polling is expensive, how can we do better?
- first, server could actively push new posts to any followers who are online
- second, precompute the results of the query so user's request for their home timeline can be served from a cache

**materialization** — speeds up reads, but in turn do more work on writes
- timeline cache is an example of a **materialized view**
  - mechanics
    - store a data structure containing user's home timeline (the recent posts by people they follow)
    - everytime a user makes a post, look up all their followers and insert that post into the home timeline of each follower
    - now precomputed home timeline can be served directly
  - downside
    - more work every time a user makes a post, home timelines are derived data that needs to be updated
      - when one initial request results in several downstream requests being carried out, **fan-out** is used to describe the factor by which the number of requests increases
    - comparative load
      - at a rate of 5,800 posts per second, with fan-out factor of 200 (200 followers)
      - just over 1 million home timline writes per second
      - better than 400 million per-sender post lookups per second
    - spikes
      - enqueue them and accept that it will temporarily take a bit longer for posts to show up in followers' timelines
      - but timelines remain fast to load, since simply served from a cache
- extreme cases
  - a user following a very large number of accounts, and those accounts post a lot
    - user has high rate of writes to their materialized timeline
    - user not likely to read all the posts → drop some of their timeline writes
    - show only a sample of the posts from those accounts
  - a celebrity account with a very large number of followers makes a post
    - insert post to millions of followers → dropping some writes is not OK
    - store celebrity posts separately and merging them with the materialized timeline when it is read
    - handling celebrities on a social network can require a lot of infrastructure

## Performance

Two main types of metric
- **response time**
  - the elapsed time from the moment when a user makes a request until they receive the requested answer
- **throughput**
  - the number of requests per second, or the data volume per second, that the system is processing
- relationship
  - service has low response time when request throughput is low
  - response time increases as load increases
  - because of **queueing**
    - request arrives on a highly loaded system → CPU is likely already in the process of handling an earlier request → incoming request needs to wait
    - as throughput approaches the maximum that the hardware can handle, queueing delays increase sharply (nonlinearly)

Side note — when an overloaded system won't recover
- **retry storm**
  - a long queue of requests is waiting to be handled
  - response times increase so much that clients time out and resent their request
  - causes the rate of requests to increase even further
  - making the problem worse — a **retry storm**
- **metastable failure**
  - even when the load is reduced again
  - such a system may remain in an overloaded state until it is rebooted or otherwise reset
  - it can cause serious outages in production systems
- solutions
  - **exponential backoff** — increase and randomize the time between successive retries on the client side
  - **circuit breaker** — temporarily stop sending requests to a service that has returned errors or timed out recently
  - **load shedding** — server detect when it is approaching overload and start proactively rejecting requests
  - **backpressure** — send back responses asking clients to slow down

---
### Latency and Response Time

Terminologies
- **response time**
  - what the client sees
  - includes all delays incurred anywhere in the system
- **service time**
  - duration for which the service is actively processing the client's request
- **queueing delays** — occurs at several points
  - after a request is received, need to wait until CPU is available
  - response packet need to be buffered before it is sent over the network if other tasks on the same machine are sending a lot of data via the outbound network
- **latency**
  - catchall term for time during which a request is not being actively processed
  - in particular, **network latency** or **network delay** refers to the time that a request and response spend travelling through the network

**Example flow**  
![alt text](images/0201.png)

Response time can vary significantly
- even the same request is sent over and over again
- factors that add random delays
  - a context switch to a background process
  - loss of a network packet and TCP retransmission
  - a garbage collection pause
  - a page fault forcing a read from disk
  - a mechanical vibrations in the server rack
- **queueing delays** often account for a **large part** of the variability in response times
  - **head-of-line blocking** — it takes only a small number of slow requests to hold up the processing of subsequent requests

---
### Average, Median, and Percentiles

- **arithmetic mean**
  - useful for estimating throughput limites
  - not a very good metric for mesauring "typical" response time
  - because it doesn't tell you how many users actually experienced that delay

- **percentiles**
  - to figure out how bad outliers are, look at higher percentiles: the 95th, 99th, and 99.9th percentile are common
    - if 95th percentile response time is 1.5 seconds, that means 95 out of 100 requests take less than 1.5 seconds, and 5 out of 100 requests take 1.5 seconds or more
  - high response-time percentiles, also known as **tail latencies**, are important because they directly affect users' experience of the service
    - reducing response times at very high percentiles is difficult because they are easily affected by random events outside of your control, and the benefits are dimishing
    - e.g., for amazon, customers with the slowest requests are often those who have the most data on their accounts → they are the most valuable customers

---
### Use of Response Time Metrics

- high percentiles are especially important in backend services that are called multiple times as part of serving a single end-user request
- even if you make the calls in parallel, the request still need to wait for the slowest of the parallel calls to complete

**SLO** and **SLA**
- percentiles are often used in **service level objectives** (SLOs) and **service level agreements** (SLAs) as ways of defining the expected performance and availability of a service
  - SLO may set a target for a service to 
    - have a median response time of less than 200 ms
    - have a 99th percentile under 1 second
    - have at least 99.9% of valid requests result in non-error responses
  - SLA is a contract that specifies what happens if the SLO is not met
    - e.g., customers may be entitled to a refund

## Reliability and Fault Tolerance

Typical expectations of reliability
- application performs the function that the user expected
- tolerate the user making mistakes or using the software in unexpected ways
- performance is good enough for the required use case, under the expected load and data volume
- system prevents any unauthorized access and abuse

**faults** vs. **failures**
- fault
  - when a **particular part** of a system stops working correctly
    - a single hard drive malfunctions
    - a single machine crashes
    - an external service (that the system depends on) has an outage
- failure
  - system **as a whole** stops providing the required service to the user
    - when it does not meet the **SLO**

---
### Fault Tolerance

Terminologies
- **fault-tolerant**
  - the system continues providing the required service to user in spite of certain faults occurring
- **single point of failure**
  - the system cannot tolerate a certain part becoming faulty, then that part is a SPOF

Example: tweeter service
- fault might happen during the fan-out process, a machine involved in updating the materialized timelines crashes or become unavailable
- to make it fault-tolerant
  - ensure that another machine can take over this task without missing any posts that should have been delivered, and without duplicating any posts

Prevention — **fault injection**
- it can make sense to increase the rate of faults by triggering them deliberately
  - e.g., by randomly killing individual processes without warning
- many critical bugs are actually due to poor error handling


---
### Hardware Faults

Facts
- approximately 2%-5% of **magnetic hard drives** fail per year
- approximately 0.5%-1% of **SSDs** fail per year
  - small numbers of bit errors are corrected automatically, 
  - but uncorrectable errors occur approximately once per year per drive
    - this error rate is higher than that of magnetic hard drives
- other hardware components also fail, although less frequently than hard derives
  - e.g., power supplies, RAID, controllers, memory modules
- approximately 1 in 1,000 machines has a **CPU core** that occasionally computes the wrong result, likely because of manufacturing defects
  - erroneous computation can lead to crash, or simply returning the wrong result
- data in **RAM** can be corrupted, either because of random events such as cosmic rays or because of permanent physical defects
  - even with error-correcting codes (ECC), more than 1% of machines encounter an uncorrectable error in a given year
    - typically leads to crash of the machine and the affected memory module needing to be replaced
  - certain pathological memory access patterns can flip bits with high probability
- an entire **datacenter** might become unavailable or even be permanently destroyed
  - e.g., because of a power outage or network misconfiguration
- **large-scale systems**
  - hardware faults happen often enough that they become part of normal system operation

Tolerating hardware faults through **redundancy**
- redundancy to the individual hardware components
  - examples
    - disks — RAID configuration
    - servers — dual power supplies and hot-swappable CPUs
    - datacenters — batteries and diesel generators for backup power
- effective when component faults are **independent**
  - occurrence of one fault does not change the likelihood that another fault will occur
  - experience has shown significant **correlations** between component failures
- **cloud systems** focus less on reliability of individual machines and aim to make services highly available by **tolerating faulty nodes** at the **software level**
  - cloud providers use **availability zones** to identify which resources are physically co-located
  - resources in the same place are more likely to fail at the same time than geographically separated resources
- **rolling upgrade** — **operational advantage** of systems that can tolerate the loss of entire machines
  - single-server system requires planned downtime if machine needs to be rebooted
  - multi-node fault-tolerant system can be patched by restarting one node at a time, without affecting the service for users

---
### Software Faults

Characteristics
- software faults are often very **highly correlated**
  - because it is common for many nodes to run the same software and thus have the same bugs
  - such faults are harder to anticipate, 
  - and they tend to cause many more failures than uncorrelated hardware faults
- faults often lie dormant for a long time until they are triggered by an unusual set of circumstances
  - it's revealed that software is making some kind of **assumption** about its environment
  - and while that assumption is usually true, it eventually stops being true for some reason
- no quick solution, but something helps
  - carefully thinking about assumptions and interactions in the system
  - through testing
  - ensuring process isolation
  - allowing processes to crash and restart
  - avoiding feedback loops such as retry storms
  - measuring, monitoring, and analyzing system behavior in production

Examples
- a software bug that causes every node to fail at the same time in particular circumstances
  - e.g., because of a firmware bug, all SSDs of certain models suddenly fail after precisely 32,768 hours of operation (less than four years)
  - rendering the data on them unrecoverable
- a runaway process that uses up a shared, limited resource, such as CPU time, memory, disk space, network bandwidth, or threads
  - e.g., a bug in a client library could cause a much higher request volume than anticipated
- a service that the system depends on slows down, becomes unresponsive, or starts returning corrupted responses
- an interaction between different systems results in emergent behavior that does not occur when each system is tested in isolation
- cascading failures, where a problem in one component causes another component to become overloaded and slow down, which in turn brings down another component

---
### Humans and Reliability

Minimizing the impact of human mistakes
- testing (both handwritten tests and property testing on lots of random inputs)
- rollback mechanisms for quickly reverting configuration changes
- gradual rollouts of new code
- detailed and clear monitoring, observability tools for diagnosing production issues
- well-designed interfaces that encourage "the right thing" and discourage "the wrong thing"


## Scalability


Motivation
- even if a system is working reliably today, that doesn't mean it will necessarily work reliably in the future
  - one common reason for degradation is **increased load**
  - **scalability** is the term used to describe a system's ability to cope with increased load
- for a new product with only a small number of users
  - the overriding engineering goal is usually to keep the system as simple and flexible as possible so that you can easily modify and adapt the features of product
  - it is counterproductive to worry about hypothetical scale that might be needed in the future
    - in the best case, investments in scalability are wasted effort and premature optimization
    - in the worst case, they lock you into an inflexible design and make it harder to evolve your application

Scalability involve questions like
- if the system grows in a particular way, what are our options for coping with the growth?
- how can we add computing resources to handle the additional load?
- based on current growth projections, when will we hit the limit of our current architecture?


---
### Understanding Load

Examples
- often the load can be measures of throughput, e.g., 
  - number of requests per second to a service
  - number of GB of new data arriving per day
- often other statistical characteristics of the load affect the access patterns and hence the scalability requirements
  - e.g., ratio of reads to writes in a database, the hit rate on a cache, or the number of data items per user


Investigate what happens when the load increases
- two ways 
  - when increase the load in a certain way and keep the system resources (CPUs, memory, network bandwidth, etc.) unchaged, how is the performance of system affected?
  - when increase the load in a certain way, how much resources need to be increased if wanting to keep performance unchanged?
- if doubling resources enables handling twice the load while keeping performance the same, we say that you have **linear scalability**
  - however, much more likely is that the **cost grows faster** than linearly
  - there are many reasons for  the inefficiency, e.g.,
    - with a lot of data, processing a single write request may involve more work than if having a small amount of data, even if the size of the request is the same

---
### Shared-Memory, Shared-Disk, and Shared-Nothing Architectures

Methods of scaling
- **vertical scaling** or **scaling up**
  - the simplest way of increasing the hardware resources of a service is to move it to a more powerful machine
    - a machine with more CPU cores, more RAM, and more disk space
  - **shared-memory architecture**
    - achieve parallelism on a single machine by using multiple processes or threads
    - all threads belong to the same process can access thes ame RAM
    - **problem**
      - cost grows faster than linearly
      - a high-end machine with twice the hardware resources of lower-spec machine typically costs significantly more than twice as much
      - and because of bottlenecks, that machine is unlikely to actually be able to handle twice the load
- **shared-disk architecture**
  - uses several machines with independent CPUs and RAM
    - stores data on an array of disks that is shared among the machines
    - which are connected via a fast network: **network-attached storage** (NAS) or a **storage area network** (SAN)
  - traditionally used for **on-premises data warehousing** workloads
    - but contention and the overhead of locking **limit the scalability** of the shared-disk approach
- **shared-nothing architecture** or **horizontal scaling** or **scaling out**
  - distributed system with multiple nodes, each of which has its own CPUs, RAM and disks
  - coordination between nodes is done at the software level, via a conventional network
  - advantage — potential to **scale linearly**
    - more easily adjust its hardware resources as load increases or decreases
    - greater fault tolerance by distributing the system across multiple datacenters or regions
  - downside
    - requires explicit **sharding** (C7)
    - incurs all the complexity of distributed systems (C9)

Side note
- cloud's native database systems that separate storage and compute
  - similar to a shared-disk architecture
  - but avoids scalability problems of older systems
    - instead of providing a filesystem (NAS) or block device (SAN) abstraction
    - the storage service offers a specialized API that is designed for the specific needs of the database

---
### Principles for Scalability

Specific and specialized
- large scale system architecture is usually highly specific to the application
  - the following two scenarios looks very different even with same data throughput (100 MB/second)
    - 100k requests per second, each at 1 kB in size
    - 3 requests per minute, each 2 GB
- an architecture that is appropriate for one level of load is unlikely to cope with 10 times that load
  - have to rethink architecture on every order of magnitude load increase
  - but not worth planning future scaling needs more than one order of magnitude in advance

Principle
- 1 — break a system into smaller components that can operate largely independently from on another
  - same principle behind
    - microservices
    - sharding (C7)
    - stream processing (C12)
    - shared-nothing architecture
  - challenge
    - line between things that should be together and things that should be apart
    - design guidelines for microservices can be found in `Sam Newman. Building Microservices, 2nd edition. O’Reilly Media, 2021. ISBN: 9781492034025`
- 2 — not to make things more complicated than necessary
  - autoscaling systems are cool, but if load is fairly predictable, a manually scaled system may have fewer operational surprises

## Maintainability

Characteristics
- requirements for an application frequently evolve, environment that software runs in change (e.g., dependencies and platform), and it may have bugs that need fixing
- majority of the cost of software is not in its initial development but in its ongoing maintenance
  - fixing bugs
  - keeping its systems operational
  - investigating failures
  - adapting it to new platforms
  - modifying it for new use cases
  - repaying technical debt
  - adding new features

Legacy systems
- system that successfully running for a long time may use outdated technologies that not many engineers understand today
- institutional knowledge of how and why the system was designed in a certain way may have been lost as people have left the organization
- fixing other people's mistakes might also be necessary

Principles
- **Operability**
  - make it easy for the organization to keep the system running smoothly
- **Simplicity**
  - make it easy for new engineers to understand the system 
    - by implementing it using well-understood, consistent patterns and structures 
    - avoid unnecessary complexity
- **Evolvability**
  - make it easy for engineers to make changes to the system in the future, adapting it and extending it for unanticipated use cases as requirements change

---
### Operability: Making Life Easy for Operations

**Automation**
- two-edged sword
  - plus side
    - for large-scale systems consisting of many thousands of machines, manual maintenance would be unreasonably expensive
    - automation is essential
  - downside
    - there will always be edge cases (such as rare failure scenarios) that require manual intervention
    - greater automation requires a more skilled operations team that can resolve those issues
    - automated system that goes wrong is harder to troubleshoot than a system that relies on an operator to perform some actions manually
- more automation is not always better
  - some amount of automation is important
  - need to find the sweet spot that suits the application and organization

Data systems automation targets
- allowing monitoring tools to check the system's key metrics and supporting observability tools
- avoiding dependency on individual machines
  - allowing machines to be taken down for maintenance while the system as a whole continues running uninterrupted
- providing good documentation and an easy-to-understand operational model
  - "if I do X, Y will happen"
- providing good default behavior, but also giving administrators the freedom to override defaults when needed
- self-healing where appropriate, but also giving administrators manual control over the system state when needed
- exhibiting predictable behavior, minimizing surprises


---
### Simplicity: Managing Complexity

**Complexity**
- characteristics
  - small software can have delightfully simple and expressive code
  - as projects get larger, they often become very complex and difficult to understand
- downsides of complexity
  - slow down everyone who needs to work on the system, further increasing the cost of maintenance
  - greater risk of introducing bugs when making a change
  - harder for developers to understand and reason about the system
  - more easily overlooking
    - hidden assumptions
    - unintended consequences
    - unexpected interactions
- reducing compexity greatly improves the maintainability of software

Reason about complexity
- 2 categories
  - essential complexity is inherent in the problem domain of the application
  - accidental complexity arises only because of limitations of our tooling
- note that boundaries between the essential and the accidental shift as tooling evolves

**Abstraction** — one of the best tools for managing complexity
- a good abstraction can hide a great deal of implementation detail behind a clean, simple-to-understand facade
- **reuse**
  - this reuse is more efficient than reimplementing a similar thing multiple times
  - also leads to higher-quality software
- can be created using methodologies such as 
  - `Erich Gamma, Richard Helm, Ralph Johnson, and John Vlissides. Design Patterns: Elements of Reusable Object-Oriented Software. Addison-Wesley Professional, 1994. ISBN: 9780201633610`
  - `Eric Evans. Domain-Driven Design: Tackling Complexity in the Heart of Software. Addison-Wesley Professional, 2003. ISBN: 9780321125217`

---
### Evolvability: Making Change Easy

Characteristics
- system's requirements constantly flux
  - learn new facts
  - previously unanticipated use cases emerge
  - business priorities change
  - users request new features
  - new platforms replace old platforms
  - legal or regulatory requirements change
  - growth of the system forces architectural changes
- **Agile**
  - for organizational processes, *Agile* working patterns provide a framework for adapting to change
  - e.g., test-driven development (TDD) and refactoring

Evolvability
- the ease of modifying a data system and adapting it to changing requirements is closely linked to its **simplicity** and its **abstractions**
- loosely coupled, simple systems are usually easier to modify than tightly coupled, complex ones

**Irreversibility**
- one major factor that makes change difficult in large systems is irreversibility
  - e.g., migrating from one database to another, but cannot switch back to the old system in case of problems with the new one
- irreversible actions need to be taken very carefully, minimizing irreversibility improves flexibility