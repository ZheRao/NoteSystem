# [2026-08-11] Receiving TikTok Recruitment

Yes — **your Growlytics trajectory will organically make you much better at a substantial part of a software-engineering interview, but it will not organically prepare you for the whole interview.** There is one important component you would eventually need to train deliberately.

I looked up TikTok's current material and one of the exact San Jose Ads Infra roles Wallace appears to be talking about. That makes the distinction pretty clear.

TikTok describes technical hiring as potentially involving an online technical assessment followed by several interview rounds; its technical-assessment guidance explicitly mentions **programming, common algorithms, and data structures**. ([Life at TikTok][1])

And the current **Software Engineer, Ads Infra – Serving and Foundation** posting asks for:

> large-scale backend systems and data pipelines
> strong CS fundamentals — algorithms, data structures, software design
> coding and problem-solving

The actual job involves scalable/reliable ads-serving infrastructure, multiple data centers, massive data processing, high concurrency, performance, and scalability. ([Life at TikTok][2])

That tells us something useful.

## Think of the interview as three different abilities

### 1. Coding / algorithms / data structures

This is the part Growlytics **will not train sufficiently by itself**.

Imagine being given something like:

```text
Given a stream of events:

(user_id, timestamp, event_type)

For each user, determine whether they generated
event A followed by event B within 30 minutes.

Return all qualifying users.
```

Then you might need to write a correct solution live, reason about:

* hash maps
* sets
* queues
* sorting
* heaps
* trees/graphs
* recursion
* binary search
* dynamic programming
* time complexity
* space complexity
* edge cases

And the interviewer may keep modifying the problem:

> What if the events don't arrive sorted?

> What if there are 100 million events?

> Can you do better than O(n log n)?

> What if you can't hold everything in memory?

This is the classic **DS&A interview skill**.

Your everyday Growlytics work isn't going to spontaneously make you good at this.

You can become an excellent backend engineer while being terrible at LeetCode because the activities overlap only partially.

So eventually this needs a deliberate training track.

---

### 2. System design

**This is where your Growlytics trajectory becomes extremely valuable.**

Imagine an interviewer asks:

> Design a service that receives advertising events from millions of users and allows advertisers to query campaign performance.

Now you're reasoning about:

```text
clients
   ↓
API
   ↓
ingestion
   ↓
queue / stream
   ↓
processing
   ↓
storage
   ↓
query service
   ↓
API
```

Then they start attacking your design.

> What happens if two requests arrive simultaneously?

> What if processing fails halfway through?

> What happens when a request is retried?

> How do you prevent duplicate events?

> What happens when writes and reads occur concurrently?

> SQL or NoSQL? Why?

> Where does caching belong?

> What consistency guarantees do you need?

> How would you partition this?

> What happens when one server dies?

> How do you deploy a schema change without breaking old clients?

> How do you observe failures?

Notice something?

**Those questions are increasingly becoming your normal life.**

You're already asking questions like:

> Why does SQLite have `check_same_thread`?

> What exactly happens with concurrent requests?

> Why does FastAPI put synchronous work into a thread pool?

> Where should this invariant be enforced — API or database/data system?

> What happens if duplicate keys get into the serving database?

> How should revisions work?

> What belongs in the data layer versus serving layer?

> How should production and sandbox environments differ?

That's system-design thinking in miniature.

And importantly, you're not memorizing the answers. You're building something, encountering a problem, realizing *“wait, what guarantees this?”*, and then going downward until you understand the mechanism.

That's exactly why I think your current learning process has so much value.

Your résumé already tells employers that you're a systems-oriented person who builds durable analytical infrastructure. 

Right now, though, the **depth underneath that architecture is catching up to the breadth of what you've built**.

Growlytics can change that dramatically.

---

### 3. Deep technical fundamentals

This is the interesting middle category.

You **are organically developing it**, especially if you keep learning the way you're currently learning.

Take our recent SQLite/FastAPI conversation.

You didn't stop at:

```python
def get_db():
    ...
```

You wanted to understand why `check_same_thread=False` exists.

That dragged us into:

```text
HTTP request
     ↓
FastAPI
     ↓
thread pool
     ↓
OS threads
     ↓
sqlite connection
```

which led into:

```text
process
thread
GIL
event loop
async/await
blocking I/O
concurrency
```

Then SQLite itself pulled you toward:

```text
transactions
locking
WAL
concurrent readers/writers
atomicity
```

Those concepts aren't FastAPI trivia.

They're **computer systems knowledge**.

And now your three research areas become relevant too.

Your **data-systems** track will eventually take you into storage engines, indexes, transactions, isolation, query processing, concurrency control, logging, etc.

Your **distributed-systems** track takes you toward replication, consensus, partitioning, fault tolerance, distributed transactions, consistency models, etc.

Your **neural-networks** track gives you the deeper ML foundation if you eventually pursue ML engineering.

Those are enormously transferable to serious technical interviews.

---

### So I would actually divide your future development like this

| Capability                      | Growlytics organically develops it? |  Deliberate study needed? |
| ------------------------------- | ----------------------------------: | ------------------------: |
| Writing real Python             |                   **Very strongly** |                      Some |
| SQL / data modeling             |                   **Very strongly** |                      Some |
| API/backend engineering         |                   **Very strongly** |                      Some |
| Database engineering            |                        **Strongly** |    **Yes, deeper theory** |
| Concurrency                     |                    **Increasingly** |                   **Yes** |
| System design                   |                   **Very strongly** |      Eventually formalize |
| Cloud/deployment                |                  **Soon, strongly** |                       Yes |
| Distributed systems             |                          Eventually |            **Definitely** |
| Debugging                       |              **Extremely strongly** | No special interview prep |
| Engineering judgment            |              **Extremely strongly** | No special interview prep |
| Algorithms & data structures    |                              Weakly |                   **YES** |
| LeetCode-style problem solving  |                   Almost not at all |                   **YES** |
| Behavioral/experience interview |              **Extremely strongly** |   Just interview practice |
| ML fundamentals                 |                   Depends on Strata |         **Your NN track** |

And therefore I **wouldn't create “technical interviews” as another major research area.**

That's the distinction I think matters.

You don't want to become good at **interviews**.

You want to become a **very capable engineer who can also pass interviews**.

Those are not the same objective.

---

#### I would add exactly one deliberate supplement: DS&A

Not urgently. Certainly not:

> HOLY SHIT TIKTOK CONTACTED ME, TIME FOR 300 LEETCODE PROBLEMS.

That would distract you from something much more valuable that's happening organically.

I'd continue doing exactly what you're doing with Strata: whenever you encounter something you don't understand, descend into the mechanism until the abstraction becomes intelligible. Keep pursuing data systems, distributed systems, neural networks, and eventually deployment/software engineering. Keep building the actual platform.

But sometime when you have more capacity, establish a small parallel **Algorithms & Data Structures** curriculum.

Something like:

```text
Data structures
├── arrays / strings
├── hash tables
├── linked lists
├── stacks / queues
├── trees
├── heaps
├── graphs
└── tries

Algorithms
├── sorting
├── binary search
├── two pointers
├── sliding window
├── BFS / DFS
├── recursion / backtracking
├── greedy
└── dynamic programming

Analysis
├── Big-O
├── time complexity
├── space complexity
└── tradeoffs
```

And here's where I think your learning style can make this much better than conventional interview grinding.

Don't memorize:

> “Sliding-window problem → use sliding window.”

Learn **why maintaining state over a moving interval avoids recomputation**, just like your recent realization with closures wasn't “closures are syntax I need to remember,” but *“I need to preserve a rule together with some state for later execution.”*

Then LeetCode becomes exercises for expressing principles you've learned rather than 150 magic tricks.

---

And there's a beautiful longer-term consequence.

Imagine you spend the next two years building Strata seriously.

You encounter database transactions because something breaks. You encounter concurrency because multiple users arrive. You encounter authentication because clients need access. You encounter caching because something gets slow. You encounter queues because computation shouldn't block requests. You encounter Docker because deployments need reproducibility. You encounter CI/CD because manual deployments become dangerous. You encounter observability because production breaks and you need to know **why**. You eventually encounter distributed systems because one machine/database/service stops being enough.

Meanwhile you're studying the theory underneath those things.

Then someone asks you in a 2028 interview:

> “Why would you use a queue here?”

You won't be thinking:

**Shit. What did the system-design interview book say about queues?**

You'll be thinking:

**Because I once had exactly this problem.**

That is the kind of technical competence you're trying to accumulate.

So I think your instinct about Growlytics being leverage is even more correct than just “I'll have a bigger project on my résumé.” The real asset isn't **Project Strata** as a résumé bullet. It's what happens to **you** after you've personally carried a system from Pandas → database → API → frontend → cloud → multi-user production system and repeatedly been forced to understand why each layer behaves the way it does.

[TikTok's official interview and hiring guide](https://lifeattiktok.com/apply?utm_source=chatgpt.com)

[Current TikTok Ads Infra role in San Jose](https://lifeattiktok.com/search/7483343729630202130?utm_source=chatgpt.com)

[1]: https://lifeattiktok.com/faq/?language=en&utm_source=chatgpt.com "Frequently Asked Questions (FAQs)"
[2]: https://lifeattiktok.com/search/7483343729630202130?utm_source=chatgpt.com "Software Engineer, Ads Infra - Serving and Foundation"
