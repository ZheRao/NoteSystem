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