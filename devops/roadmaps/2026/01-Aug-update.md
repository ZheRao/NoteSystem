# Full Theoretical Map

Absolutely. I think this should become a **parallel roadmap beside your three research areas**, not another research area itself.

I would call it something like:

> **`engineering-foundations` — the systematic development of software-engineering, computer-systems, and operational competence required to build, deploy, and maintain reliable production systems.**

And importantly, I would **order it by dependency**, rather than making Docker, Kubernetes, networking, algorithms, databases, security, etc. into a flat list of things you're supposedly deficient in.

Your research areas remain:

**`data-systems` · `distributed-systems` · `neural-networks`**.

This roadmap supplies the engineering foundation underneath and around them.

---

## Engineering Foundations & DevOps Roadmap

### 0. Engineering environment & tools — NOW

This one is small enough that I would actually do it immediately alongside your current work.

#### Learn

```text
Linux fundamentals
    ↓
shell / processes / filesystem
    ↓
Git beyond add/commit/push
    ↓
debugging
    ↓
profiling
    ↓
Python environments & packaging
    ↓
logging
    ↓
basic automation/scripts
```

You already use Linux and Git, but the objective is to stop treating the development environment itself as magic.

#### Primary resource

**MIT — The Missing Semester of Your CS Education**

[MIT Missing Semester](https://missing.csail.mit.edu/?utm_source=chatgpt.com)

This is almost absurdly well matched to the gap we're discussing. The 2026 curriculum covers shell, command-line environment, development tools, debugging/profiling, Git, packaging/shipping code, and code quality. ([Missing Semester][1])

**Priority: HIGH / inexpensive**

Do this fairly soon. It's only a small course, and you'll use the skills continuously.

---

## 1. Data structures & algorithms — START EARLY, CONTINUE SLOWLY

This is the one major foundation that Growlytics is unlikely to systematically generate.

#### Stage 1 — Complexity

Understand:

```text
time complexity
space complexity
Big-O / Θ / Ω
amortized analysis
```

Not just “this is O(n).”

Ask:

> What operation actually causes the work?

#### Stage 2 — Fundamental structures

```text
arrays / dynamic arrays
linked lists
stacks
queues
hash tables
sets
```

For each:

**representation → operations → complexity → trade-offs → implementation**

#### Stage 3 — Ordered structures

```text
binary search
binary search trees
balanced trees
heaps / priority queues
```

#### Stage 4 — Graphs

```text
graph representations
BFS
DFS
topological sorting
shortest paths
```

#### Stage 5 — Algorithmic paradigms

```text
divide & conquer
greedy algorithms
recursion
dynamic programming
```

#### Primary course

[MIT 6.006 — Introduction to Algorithms](https://ocw.mit.edu/courses/6-006-introduction-to-algorithms-spring-2020/?utm_source=chatgpt.com)

MIT's curriculum covers dynamic arrays, heaps, balanced BSTs, hash tables, sorting, graph search, and dynamic programming, while emphasizing correctness and performance analysis. ([MIT OpenCourseWare][2])

#### Reference

**CLRS — Introduction to Algorithms**

Don't read CLRS cover-to-cover initially. Use it when you want a rigorous reference for something you're studying.

#### Practice

After understanding each structure:

```text
learn hash table
     ↓
implement simple hash table
     ↓
understand complexity
     ↓
solve 5–10 relevant problems
     ↓
move on
```

LeetCode belongs **here**, as exercises—not as the curriculum.

**Priority: HIGH / continuous**

I would start this relatively early and run it at low intensity for a long time.

---

## 2. Computer systems — HIGHEST foundational priority

This might produce the greatest improvement in your *overall technical fluency*.

#### Learn

Start at:

```text
program
  ↓
compiler/interpreter
  ↓
machine representation
  ↓
CPU
  ↓
memory
  ↓
operating system
```

Then understand:

#### Representation

```text
bits / bytes
integers
floating point
memory representation
```

#### Program execution

```text
machine instructions
stack
heap
function calls
linking
```

#### Memory hierarchy

```text
registers
cache
RAM
disk
locality
```

#### Processes

```text
program vs process
process creation
context switching
signals
system calls
```

#### Memory

```text
address spaces
virtual memory
paging
memory allocation
```

#### I/O

```text
files
file descriptors
devices
system-level I/O
```

#### Concurrency

```text
threads
shared memory
race conditions
locks
semaphores
condition variables
deadlocks
```

#### Primary resource

**Computer Systems: A Programmer's Perspective — Bryant & O'Hallaron**

[CMU CS:APP](https://csapp.cs.cmu.edu/?utm_source=chatgpt.com)

CMU explicitly designed it around understanding systems **from a programmer's perspective**, covering hardware, OS, compiler, networking, virtual memory, system I/O and concurrent programming. ([CS:APP][3])

That's exactly what you need.

**Priority: VERY HIGH**

I'd put this ahead of serious distributed-systems study.

---

## 3. Operating systems — after / overlapping CS:APP

Once processes, memory and machine execution stop being completely foreign, go deeper.

#### Learn

##### Virtualization

```text
processes
CPU scheduling
address spaces
virtual memory
paging
```

##### Concurrency

```text
threads
locks
condition variables
semaphores
concurrent data structures
deadlock
```

##### Persistence

```text
disks
I/O
filesystems
journaling
crash consistency
```

#### Primary resource

**Operating Systems: Three Easy Pieces — Remzi & Andrea Arpaci-Dusseau**

[Operating Systems: Three Easy Pieces](https://pages.cs.wisc.edu/~remzi/OSTEP/?utm_source=chatgpt.com)

And don't merely read it. OSTEP provides simulators and programming exercises specifically intended to make these mechanisms concrete. ([UW Computer Sciences][4])

This is where questions like your recent:

> What actually is a thread?
> Why can SQLite connections cross threads?
> What does the OS schedule?
> What does blocking actually mean?

start belonging to a coherent model rather than isolated FastAPI facts.

**Priority: VERY HIGH**

---

## 4. Networking & the web stack

Now make this pipeline non-magical:

```text
React
 ↓
HTTP request
 ↓
DNS
 ↓
TCP connection
 ↓
IP
 ↓
server socket
 ↓
OS
 ↓
FastAPI
```

#### Learn

```text
IP
TCP / UDP
ports
sockets
DNS
HTTP
HTTPS
TLS
connection lifecycle
timeouts
retries
proxies
reverse proxies
load balancers
```

Then application-level concerns:

```text
REST
HTTP methods
status codes
headers
cookies
sessions
CORS
connection pooling
```

#### Primary resource

**Kurose & Ross — Computer Networking: A Top-Down Approach**

[Computer Networking: A Top-Down Approach resources](https://gaia.cs.umass.edu/kurose_ross/?utm_source=chatgpt.com)

**Priority: HIGH**

You don't need network-engineer depth. You need enough that deploying a web service no longer feels like sending requests into the ether.

---

## 5. Software design & codebase architecture

At this point, start systematically developing judgment about **how software should be structured**.

#### Learn

```text
abstraction
encapsulation
interfaces
modules
cohesion
coupling
dependency direction
composition
separation of concerns
```

Then:

```text
API boundaries
domain boundaries
error handling
configuration
dependency injection
backward compatibility
refactoring
technical debt
```

#### Primary resource

**John Ousterhout — A Philosophy of Software Design**

[A Philosophy of Software Design](https://web.stanford.edu/~ouster/cgi-bin/book.php?utm_source=chatgpt.com)

This should interact constantly with Project Strata.

When the book discusses an abstraction, don't just think:

> Interesting.

Ask:

> **Where does this exist in Strata? Is my boundary actually good?**

**Priority: HIGH**

This is less a course to “finish” than a lens you continually apply.

---

## 6. Testing & correctness engineering

I would put this **before serious DevOps**.

Because automating deployment of software you can't confidently verify isn't much of an accomplishment.

#### Learn

```text
unit tests
integration tests
end-to-end tests
```

Then:

```text
fixtures
mocks / fakes
test isolation
determinism
property/invariant testing
regression tests
failure-path testing
```

And critically:

```text
What belongs in:
    type system?
    validation layer?
    database constraint?
    runtime assertion?
    automated test?
```

This connects directly to what you're already discovering about invariants.

#### Resources

Use [pytest documentation](https://docs.pytest.org/?utm_source=chatgpt.com) alongside actual Project Strata tests.

For APIs, use [FastAPI documentation](https://fastapi.tiangolo.com/?utm_source=chatgpt.com) as the implementation reference rather than generic tutorials.

**Priority: VERY HIGH for current work**

You can start this immediately even while the foundational tracks continue.

---

## 7. Databases — now go substantially deeper

This overlaps directly with your **data-systems research area**.

By this stage you'll have:

```text
algorithms
computer systems
OS
concurrency
software design
```

underneath you.

Now databases become much richer.

#### Layer 1 — Relational foundations

```text
relational model
schemas
keys
constraints
normalization
SQL
```

#### Layer 2 — Storage

```text
pages
records
indexes
B-trees
buffer pools
```

#### Layer 3 — Query processing

```text
scans
joins
query plans
cost models
optimization
```

#### Layer 4 — Transactions

```text
ACID
atomicity
isolation
locking
MVCC
deadlocks
isolation anomalies
```

#### Layer 5 — Recovery

```text
WAL
logging
checkpoints
crash recovery
```

#### Primary course

**CMU 15-445/645 — Database Systems**

[CMU Database Systems](https://15445.courses.cs.cmu.edu/?utm_source=chatgpt.com)

And use [PostgreSQL documentation](https://www.postgresql.org/docs/?utm_source=chatgpt.com) as your real-system reference once you migrate beyond SQLite.

**Priority: VERY HIGH**

This should eventually become one of the strongest parts of your competence distribution because it overlaps so strongly with your work and research interests.

---

## 8. Security engineering — BEFORE public production deployment

Not optional.

But also don't disappear into cryptography textbooks yet.

Start with application security.

#### Learn

```text
authentication
authorization
sessions/tokens
password handling
secrets
least privilege
tenant isolation
```

Then attacks:

```text
SQL injection
XSS
CSRF
broken access control
insecure configuration
dependency/supply-chain risks
```

Then:

```text
TLS
encryption at rest
audit logging
secret rotation
dependency scanning
```

#### Primary resource

[OWASP Top 10:2025](https://owasp.org/Top10/?utm_source=chatgpt.com)

The current OWASP Top 10 includes broken access control, security misconfiguration, supply-chain failures, cryptographic failures, injection, authentication failures, integrity failures, and logging/alerting failures. ([OWASP Foundation][5])

Project Strata eventually contains **financial information from multiple clients**.

So especially:

> authentication ≠ authorization ≠ tenant isolation

Those distinctions need to become extremely clear before multi-client production deployment.

**Priority: CRITICAL before production**

---

## 9. Containers — Docker

**Only now** would I make Docker a deliberate study topic.

Because you'll understand what it's packaging:

```text
application
dependencies
runtime
process
networking
filesystem
environment
```

#### Learn

```text
images
containers
Dockerfiles
layers
build cache
volumes
networks
registries
environment variables
```

Then:

```text
Docker Compose
frontend container
API container
PostgreSQL container
```

#### Primary resource

[Docker Get Started](https://docs.docker.com/get-started/?utm_source=chatgpt.com)

Docker's own introductory material progresses from containers to images and registries, while its Compose material adds multi-service applications, persistence, health checks and debugging. ([Docker Documentation][6])

A beautiful milestone would be:

```text
docker compose up

→ PostgreSQL
→ FastAPI
→ React
→ Strata works
```

Now Docker isn't something you “learned.”

It's how your actual system runs.

**Priority: HIGH once deployment begins**

---

## 10. CI — continuous integration

Now automate the checks you're already doing.

Conceptually:

```text
git push
   ↓
CI starts
   ↓
install dependencies
   ↓
lint / static checks
   ↓
unit tests
   ↓
integration tests
   ↓
build
   ↓
PASS / FAIL
```

#### Learn

```text
CI pipelines
build artifacts
test automation
dependency caching
environment variables
secrets
branch protection
```

Since you're already using GitHub, **GitHub Actions** is a reasonable implementation choice.

[GitHub Actions documentation](https://docs.github.com/actions?utm_source=chatgpt.com)

Microsoft's DevOps material also explicitly places Git, unit testing, CI, CD, infrastructure-as-code and monitoring into the software lifecycle. ([Microsoft Learn][7])

**Priority: HIGH**

---

## 11. Cloud deployment — Azure

Now move the actual system.

Not:

> “Take Azure certification → therefore understand cloud.”

Instead:

```text
local Strata
     ↓
containerized Strata
     ↓
CI-tested Strata
     ↓
Azure-hosted Strata
```

#### Learn

```text
compute
managed databases
object storage
networking
DNS
TLS certificates
identity
secrets
configuration
logging
backups
```

Then:

```text
dev
test/staging
production
```

Understand **why environments are separated**, not merely that professional projects have three directories.

#### Resources

[Microsoft Learn — Azure](https://learn.microsoft.com/azure/?utm_source=chatgpt.com)

and:

[Microsoft DevOps Resource Center](https://learn.microsoft.com/en-us/devops/?utm_source=chatgpt.com)

Microsoft organizes its current DevOps material around developing, delivering, and operating systems, including CI, CD, infrastructure-as-code, monitoring, safe deployment, and production reliability. ([Microsoft Learn][7])

**Priority: HIGH once Strata is ready**

---

## 12. CD & deployment engineering

Only after you've manually deployed enough times to understand what you're automating.

```text
merge
 ↓
CI
 ↓
build image
 ↓
registry
 ↓
deploy staging
 ↓
migration
 ↓
health check
 ↓
production
```

Learn:

```text
continuous delivery vs deployment
deployment strategies
rollbacks
health checks
schema migrations
backward-compatible changes
feature flags
zero/minimal-downtime deployment
```

Microsoft's architecture guidance defines CI around automated build/test integration and CD around keeping software automatically built, tested and ready for deployment. ([Microsoft Learn][8])

**Priority: MEDIUM-HIGH after first real deployment**

---

## 13. Observability & production operations

This is where you learn that:

> **running successfully ≠ operating successfully.**

#### The three basic signals

```text
logs
metrics
traces
```

Then:

```text
latency
throughput
error rates
resource utilization
health checks
alerts
dashboards
```

And operationally:

```text
What failed?
When?
For which client?
Which request?
Which revision?
Which service?
What changed immediately beforehand?
```

Eventually investigate:

[OpenTelemetry](https://opentelemetry.io/docs/?utm_source=chatgpt.com)

But initially even excellent structured logs and useful health metrics are enough.

**Priority: HIGH once something actually runs in production**

---

## 14. Reliability & failure engineering

Now deliberately stop assuming success.

Ask:

```text
What if API request fails?
What if QBO returns malformed data?
What if process dies halfway through?
What if DB transaction fails?
What if user retries?
What if request arrives twice?
What if deployment fails?
What if database disappears?
What if backup is corrupt?
```

Learn:

```text
idempotency
timeouts
retries
exponential backoff
circuit breaking
graceful degradation
health checks
backup
restore
disaster recovery
```

And critically:

> **actually restore a backup.**

A backup you've never restored is an assumption.

**Priority: HIGH in production systems**

---

## 15. Data-intensive architecture

Now I would introduce **Designing Data-Intensive Applications**.

Not at the beginning.

At this point you'll have enough experience for the book to hit differently.

#### Resource

[Designing Data-Intensive Applications — Martin Kleppmann](https://dataintensive.net/?utm_source=chatgpt.com)

This becomes the bridge between:

```text
database systems
       +
networking
       +
reliability
       +
distributed systems
```

Topics like:

```text
data models
storage engines
replication
partitioning
transactions
distributed systems
batch processing
stream processing
```

will attach to mechanisms you already understand.

**Priority: LATER / extremely valuable**

---

## 16. Distributed systems — NOW your research area can go deep

Only here would I seriously attack the formal distributed-systems curriculum.

Now you're equipped to understand why these problems exist.

```text
multiple machines
      ↓
partial failure
      ↓
unreliable networks
      ↓
concurrent state
      ↓
replication
      ↓
consistency
      ↓
consensus
```

#### Learn

```text
failure models
logical clocks
replication
partitioning
consistency models
quorums
leader election
consensus
distributed transactions
fault tolerance
```

#### Primary course

**MIT 6.5840 — Distributed Systems**

[MIT 6.5840 Distributed Systems](https://pdos.csail.mit.edu/6.5840/?utm_source=chatgpt.com)

MIT explicitly lists fault tolerance, replication and consistency as core topics and expects substantial programming plus prior systems knowledge. ([PDOS][9])

That prerequisite is telling us something.

**Distributed systems isn't supposed to be the beginning.**

**Priority: LATER / deep research area**

---

## 17. Kubernetes — deliberately late

I want Kubernetes almost at the bottom.

You should first understand:

```text
Linux
processes
networking
containers
multiple services
cloud deployment
CI/CD
observability
failure
scaling
```

Then eventually you can ask:

> My deployment architecture has become difficult to operate. What problem does Kubernetes solve?

Now concepts such as:

```text
pods
deployments
services
ingress
config maps
secrets
health probes
autoscaling
rolling deployments
```

have reasons to exist.

If Project Strata never reaches a scale/operational structure that justifies Kubernetes, **you don't need to put it into Project Strata simply to prove you know Kubernetes.**

Learn it eventually because it's relevant industry infrastructure, but don't cargo-cult it.

**Priority: LOW initially / later professional breadth**

---

## 18. External engineering calibration — CONTINUOUS

And this one doesn't have prerequisites.

Run it quietly alongside everything.

#### Read production code

When you encounter a problem:

> How do mature projects solve this?

Actually inspect their implementation.

#### Read engineering postmortems

Especially:

```text
database failures
outages
bad deployments
race conditions
data corruption
security incidents
```

#### Contribute to open source

Eventually make small contributions where someone else reviews your code.

The goal isn't GitHub green squares.

The goal is:

> **Someone better than me disagreed with my implementation. Why?**

#### Seek technical peers

Communities, conferences, OSS maintainers, engineers you meet, eventually coworkers.

This is the one resource I cannot replace with textbooks.

**Priority: PERMANENT**

---

## The complete dependency order

So if I collapsed everything into the roadmap I'd actually put into your NoteSystem:

```text
ENGINEERING FOUNDATIONS
│
├── 0. Developer environment & tools
│     Linux · Shell · Git · Debugging · Profiling · Packaging
│
├── 1. Data Structures & Algorithms ───────────────┐
│     complexity · structures · algorithms         │ continuous
│                                                  │
├── 2. Computer Systems                            │
│     machine · memory · processes · I/O           │
│             ↓                                    │
├── 3. Operating Systems                           │
│     virtualization · concurrency · persistence   │
│             ↓                                    │
├── 4. Networking                                  │
│     TCP/IP · DNS · HTTP · TLS · sockets          │
│             ↓                                    │
├── 5. Software Design                             │
│     abstractions · boundaries · dependencies     │
│             ↓                                    │
├── 6. Testing & Correctness                       │
│     unit · integration · invariants · E2E        │
│             ↓                                    │
├── 7. Database Engineering ←→ DATA-SYSTEMS        │
│     storage · indexes · queries · transactions   │
│             ↓                                    │
├── 8. Security                                    │
│     authn · authz · isolation · OWASP            │
│             ↓                                    │
├── 9. Containers                                  │
│     Docker · Compose                             │
│             ↓                                    │
├── 10. Continuous Integration                     │
│      automated test/build                        │
│             ↓                                    │
├── 11. Cloud / Azure                              │
│      compute · DB · networking · identity        │
│             ↓                                    │
├── 12. Continuous Delivery                        │
│      deploy · migrations · rollback              │
│             ↓                                    │
├── 13. Observability                              │
│      logs · metrics · traces                     │
│             ↓                                    │
├── 14. Reliability                                │
│      retry · idempotency · recovery · backup     │
│             ↓                                    │
├── 15. Data-Intensive Architecture / DDIA         │
│             ↓                                    │
├── 16. Distributed Systems ←→ DISTRIBUTED-SYSTEMS │
│      replication · consistency · consensus       │
│             ↓                                    │
└── 17. Kubernetes                                 │
                                                   │
      EXTERNAL ENGINEERING CALIBRATION ────────────┘
      code review · OSS · peers · postmortems
```

There is one modification I'd make when **actually executing** this roadmap, though.

### Do not execute it sequentially

Dependency order does **not** mean:

> Finish CS:APP → finish OSTEP → finish networking → finish software design → two years later return to Project Strata.

That would be disastrous for the learning process that's working for you.

Instead, I'd maintain **three concurrent lanes**:

| Lane           | Current activity               | Pace                          |
| -------------- | ------------------------------ | ----------------------------- |
| **Build**      | Project Strata                 | Primary                       |
| **Foundation** | DS&A + computer systems        | Slow, continuous              |
| **Research**   | Neural networks / data systems | Slow, curiosity & work-driven |

And allow Project Strata to **pull future engineering topics forward when necessary**.

For example, if you need to deploy Strata six months before you've finished OSTEP:

**deploy it.**

Learn enough Docker, Azure and security to do the job responsibly.

The dependency graph means *“these concepts make later concepts substantially easier to understand,”* not *“you are forbidden to touch Docker until chapter 27.”*

---

### And that gives you a much more stable answer to the fear from yesterday

Six months from now you're going to discover another thing.

Maybe:

> “Wait, what the hell is a write barrier?”

Or:

> “Wait, databases have predicate locks?”

Or:

> “Wait, TLS certificates actually work HOW?”

Or:

> “What the hell is backpressure?”

Previously that discovery could mean:

> **I'm a fraud. There's another entire domain I didn't even know existed.**

With this map, the reaction becomes:

> Backpressure? Networking/distributed systems. Haven't reached it deeply yet.

> Predicate locks? Data systems/database concurrency. It's on the path.

> Certificates? Networking/security. Later.

> Memory ordering? Computer systems/concurrency. Ah.

That's why I like this roadmap much more than a giant list of technologies.

It doesn't promise that you've identified **everything you'll ever need to know**.

It gives almost every important thing you're likely to discover **a home**.

And meanwhile, your three research areas answer a different question. They aren't there to make you employable. They're the areas in which, over many years, you may eventually go substantially deeper than ordinary engineering competence.

So the long-term picture isn't:

> **Become equally good at everything.**

It's:

> **Build a broad, structurally sound engineering floor underneath three unusually deep intellectual columns.**

That is what I would consider a genuinely healthy competence distribution for the trajectory you're trying to build.

[1]: https://missing.csail.mit.edu/?utm_source=chatgpt.com "The Missing Semester of Your CS Education"
[2]: https://ocw.mit.edu/courses/6-006-introduction-to-algorithms-spring-2020/pages/syllabus/?utm_source=chatgpt.com "Syllabus | Introduction to Algorithms | Electrical Engineering and Computer Science | MIT OpenCourseWare"
[3]: https://csapp.cs.cmu.edu/3e/perspective.html?utm_source=chatgpt.com "CS:APP3e, Bryant and O'Hallaron"
[4]: https://pages.cs.wisc.edu/~remzi/OSTEP/Homework/homework.html?utm_source=chatgpt.com "Homework"
[5]: https://owasp.org/Top10/?utm_source=chatgpt.com "OWASP Top 10:2025"
[6]: https://docs.docker.com/get-started/?utm_source=chatgpt.com "Get started | Docker Docs"
[7]: https://learn.microsoft.com/en-us/devops/?utm_source=chatgpt.com "DevOps resource center - Azure DevOps | Microsoft Learn"
[8]: https://learn.microsoft.com/en-us/azure/architecture/guide/devops/devops-get-started?utm_source=chatgpt.com "Get Started with DevOps Architecture Design - Azure Architecture Center | Microsoft Learn"
[9]: https://pdos.csail.mit.edu/6.824/index.html?utm_source=chatgpt.com "6.5840 Home Page: Spring 2026"


# Ajusted Level 1 Map

Yes. I would adjust it.

Right now you have **three cognitively expensive things competing for the same kind of attention**:

* Project Strata: deep, applied engineering
* DDIA/data-systems: deep conceptual systems study
* autograd/neural-networks: deep first-principles study

Adding “now read CS:APP carefully too” would give you a fourth deep track. I don't think that's the right move.

What you need is a distinction between **deep study** and **operational literacy**.

## The model I recommend

Keep only **two deep intellectual tracks active outside work** at a time:

> **Deep track A:** data systems / DDIA
> **Deep track B:** neural networks / autograd

Project Strata itself is already your third major learning engine.

Everything else should initially be learned in **Level-1 packets**: one or two hours, perhaps an evening or weekend session, sufficient to answer:

> What is this thing?
> Why does it exist?
> What are the main failure modes?
> What do I need to do correctly in Strata?
> What do I explicitly *not* understand yet?

Then you move on.

I would **not** start reading CS:APP sequentially right now.

Use CS:APP as a **slow background/reference book**, and pull sections from it when your work creates demand.

---

## Your Level-1 curriculum

I would actually make yourself a small `engineering-working-competence` curriculum separate from your research notes.

The objective is **not mastery**. The exit criterion is:

> “I understand enough to use this responsibly and recognize when I need to learn more.”

### 1. Computer systems — 2–4 hours initially

You only need this mental model:

```text
program
  ↓
process
  ├── memory
  ├── threads
  ├── file descriptors / I/O
  └── OS resources

OS
  ↓
CPU scheduling
memory management
filesystem
network
```

Know the distinction between:

* program / process / thread
* concurrency / parallelism
* CPU-bound / I/O-bound
* user code / system call
* stack / heap at a conceptual level
* shared process resources vs thread execution
* blocking I/O
* race conditions and locks

For Python specifically, the official docs group threads, processes and `asyncio` under concurrent execution and explicitly distinguish approaches based on CPU-bound vs I/O-bound workloads. ([Python documentation][1])

Then read the FastAPI async page because it maps those concepts directly onto the technology you're using. FastAPI's documentation explicitly explains concurrency, `async def`, and when synchronous work is involved. ([FastAPI][2])

**Stop there.**

Don't start learning assembly.

Put CS:APP on your shelf and occasionally read the relevant chapter when something interests you.

---

## 2. Operating systems — 2–3 hours initially

For Level 1, learn only:

```text
processes
threads
scheduler
context switching
virtual memory — concept only
files
locks
deadlock — concept
signals / shutdown
```

You mainly need to be able to reason about statements like:

> Uvicorn starts a process.

> FastAPI may execute synchronous work on worker threads.

> Multiple threads can access shared process state.

> Two processes do not normally share Python memory.

> The OS can suspend one thread and run another.

Then **stop**.

OSTEP can remain a long-term background resource. You do not need to study filesystem implementation, page replacement algorithms, kernel scheduling policies, etc. right now.

---

## 3. Networking + web — maybe 4–6 hours

This one I would learn relatively soon because you're building an actual web application.

Your Level-1 map:

```text
browser
   ↓
DNS resolves hostname
   ↓
TCP connection
   ↓
TLS if HTTPS
   ↓
HTTP request
   ↓
server
   ↓
FastAPI route
   ↓
HTTP response
```

Understand:

* IP address
* port
* hostname
* DNS
* TCP
* HTTP
* HTTPS/TLS conceptually
* request/response
* headers
* methods
* status codes
* cookies/token conceptually
* CORS
* timeout
* retry
* reverse proxy

Don't study BGP.

Don't study TCP congestion-control algorithms.

Don't study Ethernet framing.

When you eventually deploy Strata, deepen the pieces deployment exposes.

---

## 4. Software design — don't make it a course right now

This one should mostly happen **through code review of Strata**.

For Level 1, retain perhaps seven principles:

```text
1. Separate domain logic from I/O.
2. Give modules one coherent responsibility.
3. Prefer explicit interfaces.
4. Avoid duplicating business rules.
5. Dependencies should point toward stable domain logic.
6. Make invalid states difficult to represent.
7. Refactor when evidence appears, not because abstraction looks elegant.
```

Read **A Philosophy of Software Design** slowly when convenient.

One chapter every couple of weeks would be plenty.

The value will come from asking:

> “Does Project Strata currently violate this?”

not from finishing the book.

---

## 5. Testing — learn this NOW, through implementation

This is a good example of something where you don't need a large preparatory course.

Use the official pytest documentation as a reference and start testing Strata. ([FastAPI][3])

Your Level-1 concepts are:

```text
unit test
integration test
end-to-end test

happy path
failure path
edge case
regression test
fixture
```

But for your system I care especially about **invariants**.

Examples:

```text
allocation percentages sum to 1
duplicate keys cannot exist
cash-flow components reconcile
revision lookup returns correct predecessor
invalid month cannot enter system
```

Every time you discover a bug:

> Can this become a regression test?

That's most of the Level-1 education you need.

---

## 6. Database engineering — learn NOW and deeply where work demands it

Do **not** wait until you've finished CS:APP.

For Level 1, make sure you understand:

```text
tables
rows
primary keys
foreign keys
unique constraints
NOT NULL
indexes

transactions
commit
rollback
atomicity

basic isolation
concurrent readers/writers

schema migration
```

The PostgreSQL docs are excellent reference material for constraints, indexes and transaction isolation. ([PostgreSQL][4])

And this is exactly where DDIA/data-systems remains your deep track.

So your database learning can have two layers:

```text
WORK:
How do I correctly enforce uniqueness today?

RESEARCH:
How do databases implement isolation and concurrency control?
```

Those reinforce one another instead of competing.

---

## 7. Security — one focused day gives you enormous Level-1 value

You don't need to become a security engineer.

Initially learn:

```text
authentication
authorization
least privilege
secrets
tenant isolation

SQL injection
XSS
CSRF
broken access control
security misconfiguration

TLS
password hashing concept
dependency vulnerabilities
audit logs
```

Read the current **OWASP Top 10:2025** overview. OWASP explicitly positions it as an awareness document for developers rather than a complete security curriculum. ([OWASP Foundation][5])

Then relate every category to Strata:

> Could client A ever request client B's ID?

> Where are credentials stored?

> Can user-controlled input become SQL?

> What happens if authorization checks exist only in the frontend?

That gets you much farther than abstract security study.

---

## 8. Docker — literally use the 45-minute official workshop

This is exactly the kind of **quick-resource Level 1** you're asking for.

Docker's official Get Started section has a guided **45-minute workshop**. ([Docker Documentation][6])

Do that.

Then containerize one tiny FastAPI application.

Then containerize Strata when appropriate.

Your Level-1 vocabulary:

```text
image
container
Dockerfile
build
run
port mapping
volume
environment variable
registry
Compose
```

Docker's official overview also explains the client/daemon/image/container/registry model clearly. ([Docker Documentation][7])

You do **not** need a 30-hour Docker course.

---

## 9. CI — another 1–2 hour Level-1 topic

Use the official **GitHub Actions Quickstart**. ([GitHub Docs][8])

Then make one workflow:

```text
push / PR
   ↓
checkout
   ↓
install Python
   ↓
install dependencies
   ↓
pytest
```

Congratulations: you now understand CI well enough to start using it.

GitHub describes Actions as a CI/CD platform for automating build, test and deployment pipelines. ([GitHub Docs][9])

Everything else can come when you need it.

---

## 10. Azure deployment — learn it by deploying

When Strata reaches that point, don't first take an Azure fundamentals certification course.

Microsoft currently has a FastAPI-compatible Python App Service quickstart that walks from app to deployed service. ([Microsoft Learn][10])

Use that.

Level-1 Azure:

```text
resource
resource group
App Service
managed database
environment/configuration
identity
secret
network endpoint
logging
```

Later you can understand Azure architecture properly.

Microsoft explicitly describes App Service as a managed PaaS for hosting Python and other web applications, which is precisely why it can be a good first deployment target: you don't need to manage the underlying servers yourself. ([Microsoft Learn][11])

---

## 11. Observability — start embarrassingly simple

Don't begin with OpenTelemetry architecture.

Start with:

```text
structured logs
request ID
timestamp
level
client ID
operation
error
duration
```

Then answer:

> If this request fails tomorrow while I'm asleep, can I reconstruct what happened?

Once you need metrics/traces, OpenTelemetry has a developer-oriented getting-started path. ([OpenTelemetry][12])

Level-1 observability is not “master distributed tracing.”

It's:

> **I can tell what my application is doing after I stop looking at the terminal.**

---

## 12. Reliability — use a checklist, not a textbook initially

For every important operation ask:

```text
What if it fails before starting?
What if it fails halfway through?
What if it succeeds but the response is lost?
What if caller retries?
What if the same request happens twice?
What if database is unavailable?
What if process crashes?
```

Learn these terms as they appear:

```text
transaction
idempotency
timeout
retry
backoff
backup
restore
health check
```

That is enough for Level 1.

---

## And I would make one pretty significant change to your current study allocation

I **wouldn't add CS:APP as another regular deep reading commitment right now.**

You already have:

```text
WORK
Project Strata

DEEP STUDY 1
DDIA / data systems

DEEP STUDY 2
autograd / neural networks
```

That's plenty.

Instead:

```text
                    CURRENT INTELLECTUAL BUDGET

                 ┌───────────────────────────┐
                 │       PROJECT STRATA      │
                 │        PRIMARY            │
                 │                           │
                 │ coding / architecture     │
                 │ database / React / API    │
                 │ testing / deployment      │
                 └───────────────────────────┘

          ┌────────────────┐    ┌────────────────┐
          │  DATA SYSTEMS  │    │ NEURAL NETWORKS│
          │      DDIA      │    │    AUTOGRAD    │
          │     DEEP       │    │      DEEP      │
          └────────────────┘    └────────────────┘

                  ┌─────────────────────┐
                  │ ENGINEERING LEVEL 1 │
                  │                     │
                  │ 1–3 hour packets    │
                  │ only when relevant  │
                  └─────────────────────┘

                  ┌─────────────────────┐
                  │ SYSTEMS BACKGROUND  │
                  │                     │
                  │ CS:APP / OSTEP      │
                  │ very slow / optional│
                  └─────────────────────┘
```

And **CS:APP becomes dessert rather than another obligation**.

If one evening you're fascinated by what memory actually looks like, read the relevant section.

If FastAPI raises another threading issue, read concurrency.

If you start debugging performance, read memory hierarchy.

Over a couple of years, you'll have worked through a surprising amount of it without maintaining yet another curriculum.

### In fact, I think you should allow DDIA to slow down too

This might be the biggest thing.

Your research areas are supposed to be **multi-year intellectual programs**, not courses you need to finish quickly.

There is no prize for finishing DDIA this autumn.

If one DDIA chapter takes three weeks because you're simultaneously implementing database constraints, testing your API, and understanding SQLite transactions—

**that's excellent.**

The point is that these ideas become part of your thinking.

Likewise, you don't have to rush micrograd/autograd because neural networks still exist in 2027.

Your current development could therefore feel more like:

```text
MON–FRI
Project Strata creates questions.

WHEN BLOCKED
Spend 30–120 minutes acquiring Level-1 knowledge.

DEEP FREE TIME
Choose DDIA OR autograd depending on energy/interest.

LOW-ENERGY / CURIOSITY TIME
CS:APP / OSTEP / engineering article.

WEEKEND OCCASIONALLY
One Level-1 practical:
Docker Strata.
Set up CI.
Deploy test service.
Add logging.
Restore database backup.
```

That's much more sustainable than maintaining six curricula.

---

And I think there's a deeper principle here that fits how you've been learning lately:

> **Breadth should be acquired just-in-time. Depth should compound just-in-case.**

You need broad **working literacy** quickly because you're responsible for a whole system.

But you only have enough life to develop genuine depth in a few things simultaneously.

So:

**Testing?** Learn enough this week to test correctly.

**Docker?** 45-minute workshop when deployment approaches.

**CI?** GitHub quickstart + make your tests run.

**Security?** OWASP + audit Strata's boundaries.

**Azure?** Deploy a real FastAPI application.

**Networking?** Understand the request path you're operating.

**OS?** Understand the process/thread/I/O model you're relying on.

Meanwhile:

**DDIA/data systems** → years.

**distributed systems** → years.

**neural networks** → years.

**computer systems** → quietly compounds for years.

That is, I think, the version where you can continue growing extremely seriously **without requiring your entire non-working life to become a remedial CS degree.**

[1]: https://docs.python.org/3/library/concurrency.html?utm_source=chatgpt.com "Concurrent Execution — Python 3.14.6 documentation"
[2]: https://fastapi.tiangolo.com/async/?utm_source=chatgpt.com "Concurrency and async / await"
[3]: https://fastapi.tiangolo.com/tutorial/?utm_source=chatgpt.com "Tutorial - User Guide"
[4]: https://www.postgresql.org/docs/current/ddl-constraints.html?utm_source=chatgpt.com "Documentation: 18: 5.5. Constraints"
[5]: https://owasp.org/Top10/2025/?utm_source=chatgpt.com "OWASP Top 10:2025"
[6]: https://docs.docker.com/get-started/?utm_source=chatgpt.com "Get started"
[7]: https://docs.docker.com/get-started/docker-overview/?utm_source=chatgpt.com "What is Docker?"
[8]: https://docs.github.com/actions/get-started/quickstart?utm_source=chatgpt.com "Quickstart for GitHub Actions"
[9]: https://docs.github.com/articles/getting-started-with-github-actions?utm_source=chatgpt.com "Understanding GitHub Actions"
[10]: https://learn.microsoft.com/en-us/azure/app-service/quickstart-python?utm_source=chatgpt.com "Deploy a Python (Django, Flask, or FastAPI) web app to ..."
[11]: https://learn.microsoft.com/en-us/azure/app-service/?utm_source=chatgpt.com "Azure App Service documentation"
[12]: https://opentelemetry.io/docs/getting-started/dev/?utm_source=chatgpt.com "Getting started for Developers"
