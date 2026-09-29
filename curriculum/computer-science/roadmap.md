# Computer Science Curriculum

## Philosophy

This curriculum is organized around **subjects and understanding, not university courses**.

A subject is durable:

> operating systems, networking, algorithms, database systems, distributed systems

A particular resource is not:

> MIT 6.1810, Stanford CS144, a textbook, a lecture series, a website

Courses change. Lectures disappear. Universities restrict access. Textbooks get new editions. Better resources appear.

Therefore:

```text
SUBJECT
   │
   ├── What should I understand?
   ├── What should I be able to do?
   │
   └── Resources
         ├── books
         ├── open courses
         ├── lectures
         ├── exercises / labs
         └── papers / references
```

The **subject and learning objectives are the curriculum**.

The resources are interchangeable implementations of that curriculum.

As understanding develops, resources can be added, removed, or replaced without redesigning the roadmap.

---

## The overall structure

```text
LAYER 1 — COMPUTATIONAL FOUNDATIONS
        Programming & low-level mechanics
        Data structures & algorithms
                    │
                    ▼
LAYER 2 — THE MACHINE
        Computer organization
        Computer systems
                    │
                    ▼
LAYER 3 — SYSTEM SOFTWARE
        Operating systems
        Networking
                    │
                    ▼
LAYER 4 — BUILDING SOFTWARE
        Software construction & design
        Database systems
        Security
                    │
                    ▼
LAYER 5 — SYSTEMS AT SCALE
        Computer systems engineering
        Distributed systems
                    │
                    ▼
LAYER 6 — SPECIALIZATION
        Data systems
        Distributed systems
        Neural networks / ML systems
```

This is not intended to reproduce an entire undergraduate computer-science degree.

The curriculum is deliberately biased toward someone who wants to:

> **build serious data-intensive software systems while eventually developing deep expertise in data systems, distributed systems, and neural networks / ML systems.**

Subjects such as compilers, graphics, programming-language theory, robotics, HCI, computational geometry, advanced architecture, and formal methods remain legitimate future branches, but they are not currently part of the core path.

---

## Layer 1 — Computational Foundations

### 1A. Programming and Low-Level Mechanics

#### Purpose

Develop the low-level vocabulary and intuition that higher-level languages such as Python normally hide.

This is not about becoming a professional C programmer.

It is about understanding what increasingly disappears underneath abstractions.

#### Core topics

```text
source code → compiler → machine code

types
integer representation
floating-point representation
overflow

arrays
strings

memory
addresses
pointers

stack
heap
allocation
deallocation

basic debugging
segmentation faults
buffer overflows

basic compilation
linking
libraries
```

#### Desired understanding

By the end, concepts such as:

```text
pointer
address
stack
heap
allocation
compiler
linker
binary representation
```

should no longer feel like foreign vocabulary.

The goal is sufficient low-level exposure to support the later study of computer systems and operating systems.

#### Possible resources

**Courses**
- Harvard CS50x — selected Weeks 1–5
- Other introductory C/systems material as useful

**Books / references**
- *C Programming: A Modern Approach* — K. N. King
- *The C Programming Language* — Kernighan & Ritchie, as reference rather than necessarily cover-to-cover

**Practice**
- small C programs
- manual memory experiments
- debugger experiments
- implementing simple data structures

#### Depth target

**Medium.**

This is foundation-building, not specialization.

---

## 1B. Data Structures & Algorithms

### Purpose

Develop the ability to reason about computational problems independently of a particular framework or application.

The important transition is from:

> “I know how to make the program work.”

toward:

> “I can reason about the structure of the problem and the computational consequences of different solutions.”

#### Core topics

```text
asymptotic complexity

arrays
linked lists
stacks
queues

hash tables

sorting
searching

trees
binary search trees
heaps

graphs
BFS
DFS

shortest paths

recursion

dynamic programming
```

Later extensions may include:

```text
greedy algorithms
union-find
minimum spanning trees
advanced graph algorithms
amortized analysis
```

#### Desired understanding

Be able to:

- recognize common data structures and their trade-offs;
- reason using time and space complexity;
- select appropriate structures for problems;
- understand common graph representations and traversal;
- recognize when dynamic programming applies;
- implement important structures and algorithms rather than merely recognize their names.

#### Possible resources

**Primary open course**
- MIT 6.006 — Introduction to Algorithms

**Books**
- *Introduction to Algorithms* — Cormen, Leiserson, Rivest & Stein
- *Algorithms* — Sedgewick & Wayne
- *The Algorithm Design Manual* — Skiena

These are alternatives/references rather than three books to read cover-to-cover.

**Practice**
- implementations from scratch
- selected exercises
- LeetCode or similar problems used selectively for reinforcement, not as the curriculum itself

#### Depth target

**High.**

---

## Layer 2 — Computer Systems

### 2A. Computer Organization and Machine-Level Execution

#### Purpose

Answer the question:

> **What actually happens underneath a program when it runs?**

Build the bridge:

```text
Python / high-level language
          ↓
          C
          ↓
        memory
          ↓
machine representation
          ↓
       assembly
          ↓
         CPU
          ↓
memory hierarchy
```

#### Core topics

```text
binary representation
integers
floating point

machine instructions
assembly

registers
stack frames
function calls

compilation
linking
object files

memory hierarchy
CPU caches

locality
performance

exceptions / interrupts — introductory level
```

#### Desired understanding

Be able to reason conceptually about:

- how high-level code becomes machine instructions;
- how function calls appear at machine level;
- what registers and stack frames are;
- how data is represented in memory;
- why memory hierarchy exists;
- why cache locality affects performance;
- what compilation and linking actually do.

This layer should make the computer beneath Python substantially less mysterious.

#### Possible resources

**Primary book**
- *Computer Systems: A Programmer's Perspective (CS:APP)* — Bryant & O'Hallaron

**Courses / supplementary teaching**
- Stanford CS107 materials where publicly available
- Berkeley CS61C materials where publicly available
- Carnegie Mellon systems material where useful

**Practice**
- CS:APP labs where accessible
- assembly inspection
- C experiments
- debugger work
- memory-layout experiments

#### Depth target

**High.**

The book/resource does not define completion. Understanding the subject does.

---

## Layer 3 — System Software

## 3A. Operating Systems

### Purpose

Understand the layer that turns hardware into the environment in which ordinary programs can exist.

This subject should answer questions such as:

```text
What is a process?

What is a thread?

What does the operating system schedule?

What is a system call?

What happens when a program performs I/O?

Why does every process appear to have its own memory?

What is virtual memory?

How do multiple programs safely share a machine?

How do threads coordinate?

What actually is a file?
```

#### Core topics

##### Processes and execution

```text
processes
process creation
system calls
context switching
scheduling
user mode / kernel mode
```

##### Memory

```text
address spaces
virtual memory
paging
page tables
TLBs
memory allocation
```

##### Concurrency

```text
threads
race conditions
atomicity
locks
mutexes
condition variables
semaphores
deadlock
```

##### Persistence and I/O

```text
devices
interrupts
I/O
files
directories
filesystems
buffering
crash consistency
```

##### Communication

```text
pipes
IPC
shared memory
sockets — introductory connection to networking
```

#### Desired understanding

Eventually be able to mentally connect:

```text
FastAPI request
      ↓
Python process
      ↓
thread / event loop
      ↓
system calls
      ↓
operating system
      ↓
CPU / memory / network / disk
```

Questions about blocking, threads, processes, concurrency, shared state and I/O should increasingly be understood as systems questions rather than framework-specific mysteries.

#### Possible resources

**Primary conceptual book**
- *Operating Systems: Three Easy Pieces (OSTEP)* — Remzi & Andrea Arpaci-Dusseau

**Implementation companion**
- *xv6: a simple, Unix-like teaching operating system*

**Courses**
- MIT 6.1810 — Operating System Engineering, where public material is useful
- other open OS courses as appropriate

**Reference**
- *Operating System Concepts* — Silberschatz, Galvin & Gagne

#### Suggested learning relationship

```text
OSTEP
  │
  ├── understand the idea
  │
  ▼
xv6 / labs / experiments
  │
  └── see how the mechanism can actually be implemented
```

#### Depth target

**High.**

---

## 3B. Computer Networking

### Purpose

Understand how independent machines communicate.

Eventually this:

```text
browser → Internet → Azure → API → database
```

should expand mentally into layers of concrete mechanisms rather than appearing as one mysterious arrow.

#### Core topics

```text
network layering

application layer
HTTP
DNS

sockets

transport layer
UDP
TCP

reliability
flow control
congestion control

network layer
IP
routing

link layer — conceptual understanding

NAT
firewalls

TLS — relationship to security
```

#### Desired understanding

Be able to reason about:

- what happens when a browser requests an API endpoint;
- what sockets represent;
- why TCP exists;
- how reliability is constructed over an unreliable network;
- how IP moves packets between networks;
- what DNS actually does;
- where HTTP and TLS fit;
- how latency, packet loss and congestion affect applications.

#### Possible resources

**Primary textbook**
- *Computer Networking: A Top-Down Approach* — Kurose & Ross

**Alternative / deeper reference**
- *Computer Networks* — Tanenbaum et al.

**Courses**
- Stanford CS144 materials where publicly accessible
- other open networking courses

**Practical work**
- socket programming
- packet inspection with Wireshark
- simple HTTP server/client
- selected TCP implementation exercises
- eventually more substantial networking projects if useful

#### Depth target

**Medium-high.**

---

## Layer 4 — Building Mature Software

## 4A. Software Construction and Design

### Purpose

Move from:

> **I can make software work**

toward:

> **I can construct software that remains understandable, testable, reliable, and changeable.**

This subject runs partly in parallel with actual engineering work because many lessons only become meaningful after experiencing real software complexity.

#### Core topics

```text
abstraction
interfaces
modularity

specifications
contracts
invariants

testing
test design
integration testing

state
mutability

error handling

dependency management

API design

complexity management

code organization
refactoring

changeability
maintainability

concurrency as a software-design concern
```

#### Desired understanding

Develop judgment about:

```text
Where should this responsibility live?

What should this interface expose?

What assumptions does this component make?

What invariant must always remain true?

How will failure propagate?

How can I test this?

How difficult will this design be to change?

What complexity am I hiding versus merely moving?
```

#### Possible resources

**Books**
- *A Philosophy of Software Design* — John Ousterhout
- *Code Complete* — Steve McConnell, selective/reference
- *Refactoring* — Martin Fowler, selective/reference
- *Working Effectively with Legacy Code* — Michael Feathers, when relevant

**Courses**
- MIT 6.102 Software Construction materials where publicly accessible
  - [course website](https://web.mit.edu/6.102/www/sp26/)
- Stanford CS190 materials where useful

**Most important laboratory**
- real software systems

Project Strata and future projects provide continuous opportunities to apply this subject.

#### Depth target

**Very high and ongoing.**

This is less a course to “finish” than a discipline in which judgment compounds over years.

---

## 4B. Database Systems

### Purpose

Understand what exists underneath:

```text
SELECT ...
INSERT ...
UPDATE ...
COMMIT
```

and eventually reason rigorously about how database systems store, retrieve, coordinate and recover data.

#### Core topics

##### Storage

```text
pages
files
buffer pools
storage layouts
```

##### Indexes

```text
B-trees
B+ trees
hash indexes
```

##### Query execution

```text
relational algebra
query operators
joins
sorting
query execution
```

##### Query optimization

```text
statistics
cost estimation
access paths
join ordering
```

##### Transactions

```text
ACID
concurrency anomalies
isolation
serializability

locking
2PL
MVCC
optimistic concurrency control
```

##### Durability and recovery

```text
WAL
logging
checkpoints
crash recovery
```

##### Later extensions

```text
parallel databases
distributed databases
replication
partitioning
```

#### Desired understanding

Connect application-level experiences such as:

```text
SQLite transaction
PostgreSQL MVCC
lost update
snapshot isolation
index
query plan
WAL
```

to the mechanisms that implement them.

#### Possible resources

**Primary course**
- CMU 15-445/645 Database Systems and its publicly available material

**Books / references**
- *Database System Concepts* — Silberschatz, Korth & Sudarshan
- *Database Management Systems* — Ramakrishnan & Gehrke
- *Architecture of a Database System* — Hellerstein, Stonebraker & Hamilton

**Complementary systems perspective**
- *Designing Data-Intensive Applications (DDIA)* — Martin Kleppmann

DDIA and database internals serve different purposes:

```text
DDIA
"What trade-offs appear in data-intensive systems?"

              ↕

Database internals
"How are these mechanisms actually constructed?"
```

#### Practical work

- SQL experiments
- transaction/isolation experiments
- query-plan inspection
- indexing experiments
- CMU database implementation projects where practical
- building simplified database components

#### Depth target

**Very high.**

This is also one of the eventual specialization areas.

---

## 4C. Computer Security

Security runs vertically through the curriculum rather than waiting until everything else is complete.

There are two tracks.

### Practical Software Security — ongoing

#### Topics

```text
authentication
authorization
tenant isolation

input validation
SQL injection
XSS
CSRF

password storage

secrets management

TLS

least privilege

dependency security

logging / auditing

cloud permissions
```

#### Resources

- OWASP documentation and guides
- framework-specific security documentation
- cloud-provider security documentation
- threat modeling against real systems

Apply continuously to actual software.

---

### Computer Security Fundamentals — later

Once operating systems and networking are understood, study the mechanisms more deeply.

#### Topics

```text
memory vulnerabilities
control-flow attacks
sandboxing

cryptographic primitives
authentication protocols

web security

network security

access control

software isolation

system security
```

#### Possible resources

**Books**
- *Security Engineering* — Ross Anderson
- *Computer Security: Principles and Practice* — Stallings & Brown, as possible reference

**Courses**
- Stanford CS155 materials where available
- other high-quality open computer-security courses

#### Depth target

**Practical security now; deeper academic treatment later.**

---

## Layer 5 — Systems at Scale

## 5A. Computer Systems Engineering

### Purpose

Earlier subjects study pieces:

```text
CPU
OS
network
database
software
security
```

Systems engineering asks:

> **How do we combine these pieces into a system that continues to behave correctly under complexity, concurrency, load and failure?**

#### Core topics

```text
complexity management

client/server systems

naming

caching

concurrency

atomicity

fault tolerance

recovery

performance

security

reliability

availability

system boundaries

interfaces

end-to-end design

system evolution
```

#### Desired understanding

Learn to reason about an entire system rather than one component.

Questions become things like:

```text
Where should state live?

What happens if this component fails halfway through?

What assumptions cross this interface?

Which component should guarantee this property?

What can safely be retried?

What happens under concurrent requests?

Where can stale state exist?

What does recovery look like?

Which guarantees are end-to-end?
```

#### Possible resources

**Primary book**
- *Principles of Computer System Design: An Introduction* — Saltzer & Kaashoek

**Courses**
- MIT 6.1800 / historical 6.033 material where publicly available

**Complementary books**
- *Designing Data-Intensive Applications*
- *A Philosophy of Software Design*

**Case studies / papers**
- selected classic systems papers as understanding matures

#### Depth target

**Very high.**

This is where many previously independent areas begin to converge.

---

## 5B. Distributed Systems

### Purpose

Understand what changes when:

> **there is no longer one computer, one memory space, one clock, or one reliable point of truth.**

#### Core topics

```text
partial failure

RPC

time
clocks
ordering

replication

consistency models

linearizability

distributed transactions

consensus

leader election

Raft / Paxos

partitioning

fault tolerance

distributed state

eventual consistency
```

Later:

```text
stream processing
distributed databases
distributed storage
coordination systems
large-scale data processing
```

#### Desired understanding

Understand why distributed systems are fundamentally difficult rather than merely learning the names of distributed technologies.

Eventually be able to reason about:

```text
What if the request timed out but actually succeeded?

What if two nodes disagree?

What does "latest" mean without a global clock?

How do replicas converge?

How can a system choose one leader?

What consistency guarantee does the application actually require?

What happens during a network partition?
```

#### Possible resources

**Books**
- *Designing Data-Intensive Applications* — Kleppmann
- *Distributed Systems* — Maarten van Steen & Andrew Tanenbaum, possible comprehensive reference
- *Distributed Algorithms* — Nancy Lynch, much later if theoretical depth becomes useful

**Courses**
- MIT 6.5840 Distributed Systems materials
- other publicly accessible distributed-systems courses

**Implementation**
- distributed key/value store
- replication experiments
- Raft implementation
- fault-injection experiments

**Papers**

Eventually, original systems papers become increasingly important:

```text
MapReduce
GFS
Dynamo
Bigtable
Spanner
Raft
Kafka
etc.
```

Not as a reading checklist now, but as primary literature once the foundations exist.

#### Depth target

**Very high — eventual specialization.**

---

## Layer 6 — Specialization

The broad curriculum eventually feeds three areas in which much deeper study can occur.

```text
                 COMPUTER SCIENCE FOUNDATION
                            │
             ┌──────────────┼──────────────┐
             │              │              │
             ▼              ▼              ▼
        DATA SYSTEMS   DISTRIBUTED     NEURAL NETWORKS
                         SYSTEMS         / ML SYSTEMS
```

These are not merely final courses.

They are potentially multi-year research and engineering directions.

---

## 6A. Data Systems

Built primarily on:

```text
algorithms
    +
computer systems
    +
operating systems
    +
databases
    +
distributed systems
```

Potential deeper areas:

```text
storage engines
query processing
query optimization
transaction processing
distributed databases
stream processing
analytical databases
data-intensive architectures
```

Resources will increasingly shift from textbooks toward:

```text
advanced courses
systems papers
source code
experiments
actual database implementations
```

---

## 6B. Distributed Systems

Built primarily on:

```text
operating systems
      +
networking
      +
databases
      +
systems engineering
```

Potential deeper areas:

```text
consensus
replication
distributed storage
coordination
distributed transactions
fault-tolerant systems
large-scale computation
distributed databases
```

At sufficient depth, research papers become part of the normal curriculum.

---

## 6C. Neural Networks and ML Systems

Maintain this as a separate specialization roadmap rather than forcing it into the general CS sequence.

The rough progression remains:

```text
mathematical foundations
        │
        ▼
autograd / computation graphs
        │
        ▼
neural-network foundations
        │
        ▼
deep-learning architectures
        │
        ▼
training systems
        │
        ▼
ML systems
        │
        ▼
distributed training / inference
```

Possible resources can eventually include textbooks, papers and courses such as CS231n, CS336 or whatever the best accessible equivalents are at the time.

The resources should be chosen when each stage becomes relevant rather than permanently fixed years in advance.

---

## The Dependency Graph

The roadmap therefore becomes:

```text
                 PROGRAMMING / LOW-LEVEL MECHANICS
                              │
                ┌─────────────┴─────────────┐
                │                           │
                ▼                           ▼
           ALGORITHMS               COMPUTER SYSTEMS
                │                           │
                │                    ┌──────┴──────┐
                │                    │             │
                │                    ▼             ▼
                │             OPERATING        NETWORKING
                │              SYSTEMS
                │                    │             │
                │                    └──────┬──────┘
                │                           │
                ├──────────────┬────────────┘
                │              │
                ▼              ▼
          SOFTWARE         DATABASE
         CONSTRUCTION       SYSTEMS
                │              │
                └───────┬──────┘
                        │
                        ▼
                SYSTEMS ENGINEERING
                        │
                        ▼
                DISTRIBUTED SYSTEMS
                        │
                        ▼
              DEEPER SPECIALIZATION
```

Security runs vertically alongside the entire graph.

Real engineering work also runs alongside the entire graph.

---

## Resources are attached to nodes, not vice versa

This distinction is important enough to make explicit.

The roadmap should **not** say:

```text
Finish CS50
     ↓
Finish MIT 6.006
     ↓
Finish CS:APP
     ↓
Finish OSTEP
     ↓
Finish Kurose
```

Instead:

```text
Learn ALGORITHMS
    │
    ├── MIT 6.006
    ├── CLRS
    ├── Sedgewick
    └── exercises

Learn COMPUTER SYSTEMS
    │
    ├── CS:APP
    ├── CS107 material
    ├── CS61C material
    └── experiments

Learn OPERATING SYSTEMS
    │
    ├── OSTEP
    ├── xv6
    ├── MIT 6.1810 material
    └── labs
```

We can use one resource heavily, several lightly, or replace one entirely.

If a resource disappears:

> **the curriculum does not change.**

If a better resource appears:

> **the curriculum does not change.**

If one book explains virtual memory beautifully but explains concurrency poorly:

> use another resource for concurrency.

Resources serve the learning objective.

The learning objective does not serve the resource.

---

## How to study each subject

For every major subject, use approximately the same cycle:

```text
1. ORIENT
   What problem is this subject trying to solve?

        ↓

2. BUILD THE CONCEPTUAL MODEL
   Book / lectures / discussion

        ↓

3. MAKE IT CONCRETE
   Code / labs / experiments

        ↓

4. CONNECT IT
   How does this explain things I've already encountered?

        ↓

5. TEST UNDERSTANDING
   Explain it, predict behavior, solve problems

        ↓

6. GO DEEPER WHERE USEFUL
   alternative explanation / paper / implementation

        ↓

7. ADVANCE
   once the important learning objectives are satisfied
```

Completion therefore does **not** mean:

> “I watched every lecture.”

or:

> “I read every page.”

It means:

> **“I achieved the level of understanding this node was intended to give me.”**

---

## Depth is deliberately uneven

Not every subject deserves equal investment.

#### Medium

```text
introductory C / programming mechanics
```

Enough to expose the machinery hidden by higher-level programming.

#### Medium-high

```text
networking
general security foundations
```

Strong working understanding, with deeper specialization only if needed.

#### High

```text
algorithms
computer systems
operating systems
```

These establish the intellectual foundation underneath later work.

#### Very high

```text
software construction
database systems
systems engineering
distributed systems
```

These connect directly to the kind of systems expertise this curriculum is ultimately trying to develop.

#### Potential specialization depth

```text
data systems
distributed systems
neural networks / ML systems
```

There is effectively no predefined endpoint here.

---

## This is not twelve simultaneous obligations

The curriculum describes **direction**, not current workload.

At any moment there should normally be only one active general-CS subject.

For example:

```text
NOW
Programming foundations / selected CS50

        ↓

NEXT
Algorithms or Computer Systems

        ↓

LATER
Operating Systems / Networking

        ↓

MUCH LATER
Databases / Systems Engineering

        ↓

EVENTUALLY
Distributed Systems
```

Meanwhile, work and independent research continue on their own tracks.

A possible operating model remains:

```text
PRIMARY ENGINEERING WORK
        Project Strata / future work

DEEP RESEARCH
        currently DDIA / data systems

EXPLORATION
        neural networks / personal projects

GENERAL CS CURRICULUM
        one active subject
        slow progression
```

The CS curriculum can move at one or two study sessions per week.

Some weeks can contain none.

There is no deadline.

---

## Time horizon

Think in years.

Not:

```text
How quickly can I finish CS:APP?
```

but:

```text
How much more deeply do I understand computers
than I did one year ago?
```

A rough progression might look like:

```text
FOUNDATIONS
"What actually happens beneath Python?"

        ↓

MACHINE + OS + NETWORKING
"I understand how programs actually execute
and communicate."

        ↓

SOFTWARE + DATABASES
"I understand how substantial software and
data systems are constructed."

        ↓

SYSTEMS ENGINEERING
"I can reason across component boundaries,
failure modes, concurrency and complexity."

        ↓

DISTRIBUTED SYSTEMS
"I understand what changes when computation
and state span multiple machines."

        ↓

SPECIALIZATION
"I can investigate data systems, distributed
systems and ML systems at increasing depth."
```

These are stages of understanding, not calendar deadlines.

---

## The governing principle

The curriculum should remain stable while its resources evolve.

```text
                    STABLE

               SUBJECT / TOPICS
                     │
               LEARNING GOALS
                     │
               EXIT CRITERIA
                     │
                     ▼

                   FLEXIBLE

          ┌──────────┼──────────┐
          │          │          │
        BOOKS      COURSES     LABS
          │          │          │
          └──────────┼──────────┘
                     │
                   PAPERS
                     │
                 PROJECTS
```

We therefore never again need to say:

> “I can't take Stanford CS107, so there is a hole in my curriculum.”

There is no Stanford-CS107-shaped hole.

There is a **Computer Systems** subject.

Today CS:APP may be the best primary resource for it. Stanford material may supplement it. In two years we may discover something better and replace both.

The destination stays exactly where it was.

---

## The long-term objective

The objective is not to reproduce the transcript of someone who completed an undergraduate CS degree.

Nor is it to accumulate famous course names.

The objective is to spend years deliberately constructing:

> **a computer scientist's understanding underneath an engineer's accumulated experience.**

The curriculum gives that development structure without pretending we can determine today exactly which textbook, professor, lecture series, project, or paper will teach every future concept best.

We choose those when we reach them.

**Subjects are permanent. Resources are provisional. Understanding is the goal.**



# Ordering

I would **not** recommend blindly following the roadmap top-to-bottom.

The dependency graph is real, but **“prerequisite” does not mean “you must master this entire subject before touching anything above it.”** For you, a strict bottom-up curriculum could actually produce worse learning outcomes because you'd spend months studying machine representation and low-level systems before reaching the questions that currently make those mechanisms meaningful.

The original roadmap itself was already trying to avoid that trap: it said the subjects establish prerequisites but that you don't have to execute everything sequentially. :chatgpt-content-reference{index="0"} I think we should make that principle much stronger.

## I would use a spiral, not a staircase

A staircase says:

```text
C
↓
Algorithms
↓
Computer Organization
↓
Operating Systems
↓
Networking
↓
Software Construction
↓
Databases
↓
Systems Engineering
↓
Distributed Systems

FINALLY, after several years:
interesting large-scale systems!
```

I don't think that's right for you.

Instead:

```text
                 ┌───────────────────────────┐
                 │     SYSTEMS QUESTIONS     │
                 │ DDIA / Strata / projects │
                 └─────────────┬─────────────┘
                               │
              "I don't understand why..."
                               │
                               ▼
                    FOUNDATION BELOW IT
                               │
                               ▼
                     return to the system
                  with a better mental model
```

You're already doing this naturally.

You encountered **concurrent database access** in Strata → suddenly transactions, isolation, MVCC and locking mattered.

You read about PostgreSQL transaction IDs in DDIA → suddenly the fact that a 32-bit integer has roughly 4 billion possible bit patterns connected to something real.

You encountered async FastAPI → suddenly *process, thread, event loop, blocking, scheduling* weren't textbook vocabulary anymore.

That's an extremely powerful way for you to learn.

## So I'd actually recommend this order

### Phase 0 — Finish your current CS50 selection

You're already here. Keep going through approximately:

```text
C
Arrays
Algorithms
Memory
Data Structures
```

Don't turn CS50 into a giant project.

Its job is orientation.

You should emerge knowing enough C and low-level vocabulary that later material can say `pointer`, `heap`, `linked list`, `hash table`, `stack frame`, etc. without constantly stopping you.

Then I would **not immediately march into a 1,000-page computer-architecture book.**

---

## Phase 1 — Software Construction + Systems Orientation

This is where I'd deviate significantly from the pure prerequisite graph.

Your immediate professional problem is building increasingly serious software.

So after CS50, I'd put:

**Software Construction / Design**

alongside continued DDIA.

Something like:

```text
              CS50 selected
                    │
                    ▼
          SOFTWARE CONSTRUCTION
                    │
         ┌──────────┴──────────┐
         │                     │
      STRATA                  DDIA
         │                     │
         └──────────┬──────────┘
                    │
          questions emerge
```

You already have enough programming experience for concepts like abstraction, contracts, invariants, testing, state, APIs, error propagation and modularity to attach to something.

And the feedback loop is immediate:

> learn something Tuesday → notice it in Strata Wednesday.

That's ideal.

I'd probably start slowly reading **A Philosophy of Software Design** here rather than treating it as something you're academically forbidden from touching until you've finished operating systems.

---

## Phase 2 — Operating Systems + Networking

This would be my first major **foundational systems push**.

And interestingly, I'd probably put it **before a deep CS:APP read** for you.

Why?

Look at the questions you've actually been asking:

```text
What exactly is a process?

What exactly is a thread?

What does blocking mean?

What does the event loop do?

Why does sqlite3 block?

What happens when FastAPI gets
multiple requests?

What's shared between threads?

How does the browser communicate
with my API?

What happens when I deploy this
to Azure?
```

The original roadmap explicitly recognized that these aren't fundamentally FastAPI questions; they're computer-systems questions. :chatgpt-content-reference{index="1"}

But most of what you're missing **right now** sits around:

```text
              application
                  │
            ┌─────┴─────┐
            ▼           ▼
       OPERATING      NETWORK
        SYSTEM
```

not:

```text
transistor
   ↓
logic gates
   ↓
CPU datapath
   ↓
assembly
```

So I'd learn **OSTEP fairly early**.

And I suspect you'll find it considerably more exciting than you imagine because suddenly:

> “Oh. THAT'S what FastAPI/Python/Linux is sitting on.”

Networking can follow or partially overlap.

Then:

```text
React browser
     ↓
HTTP
     ↓
TCP
     ↓
IP
     ↓
network
     ↓
Azure machine
     ↓
OS
     ↓
Python process
     ↓
FastAPI
```

starts becoming concrete.

That's enormously relevant to Strata.

---

## Phase 3 — Computer Systems / Machine-Level Execution

**Now** I'd go downward.

This is where I'd bring in CS:APP selectively.

And I think this ordering solves exactly the motivational problem you're worried about.

Imagine studying virtual memory having never cared about processes.

It can feel like:

> page tables, addresses, TLBs... why am I learning this?

Instead, you've already studied OS virtual memory and thought:

> Wait. The OS gives every process an address space. **How the hell does the hardware actually make that illusion work?**

Excellent.

Now descend:

```text
process virtual memory
        │
        ▼
    page tables
        │
        ▼
       TLB
        │
        ▼
physical memory
        │
        ▼
CPU / hardware
```

Likewise with function execution:

```text
Python calls function
       ↓
C implementation
       ↓
machine instructions
       ↓
registers
       ↓
stack frames
       ↓
CPU
```

Now machine-level execution answers questions you already possess.

**That's when I want you reading CS:APP.**

Not because Chapter N appears before Operating Systems on somebody's university curriculum.

---

## Phase 4 — Algorithms

This one is unusually independent.

You don't need operating systems to understand BFS.

You don't need networking to understand dynamic programming.

You don't need assembly to understand a heap.

So I wouldn't obsess over exactly where algorithms sits.

I'd probably place serious algorithms study around this point simply because **it is important but isn't currently your largest knowledge bottleneck**.

You've already got enough algorithmic literacy to work productively.

Eventually, though:

```text
complexity
hashing
trees
heaps
graphs
BFS / DFS
shortest paths
dynamic programming
```

deserve proper treatment. Those were the core objectives we originally identified for the algorithms node. :chatgpt-content-reference{index="2"}

You may also find it particularly useful before getting serious about your Xiangqi project.

---

## Phase 5 — Database Systems

This is where things start getting **very exciting for you**.

Because by then you have:

```text
algorithms
     +
computer systems
     +
operating systems
     +
concurrency
     +
DDIA
     +
real database experience
```

And now you descend into:

```text
                    DATABASE
                       │
      ┌────────────────┼────────────────┐
      ▼                ▼                ▼
   STORAGE          INDEXES        EXECUTION
      │                │                │
      ▼                ▼                ▼
   pages           B+ trees           joins
buffer pool        hashing       query optimizer
      │
      └───────────────┬────────────────┘
                      ▼
                 TRANSACTIONS
                      │
          ┌───────────┼───────────┐
          ▼           ▼           ▼
         MVCC       locking       WAL
```

Your existing DDIA study makes this especially valuable.

DDIA has already introduced you to things like transactions, isolation, replication, partitioning and distributed trade-offs. Database internals lets you descend beneath those abstractions.

The original roadmap made exactly this distinction:

> DDIA: *Here are the trade-offs.*

> Database internals: *Let's build the mechanism.*

:chatgpt-content-reference{index="3"}

That remains one of my favorite parts of this curriculum.

---

## Phase 6 — Systems Engineering

**Now we zoom back out.**

This is important.

Notice the motion:

```text
                APPLICATIONS
                     ↓

                OS / NETWORK

                     ↓

               MACHINE LEVEL

                     ↓

             DATABASE INTERNALS

                     ↓

              ZOOOOOM BACK OUT
                     ↓

             SYSTEMS ENGINEERING
```

Now concepts like:

```text
failure
recovery
atomicity
concurrency
caching
naming
reliability
performance
security
interfaces
```

aren't abstractions.

You've seen their mechanisms.

This is where something like *Principles of Computer System Design* becomes much more powerful.

The original curriculum deliberately placed systems engineering late for precisely this reason: it asks you to combine what you know about software, hardware, networking, concurrency, failure and storage into reasoning about complete systems. :chatgpt-content-reference{index="4"}

---

## Phase 7 — Distributed Systems

And **then I would go hard into distributed systems**.

Not because you're prohibited from learning distributed systems now.

Quite the opposite:

**keep reading DDIA.**

Keep asking distributed-systems questions.

Learn what replication means. Learn consistency. Learn partitioning. Learn consensus conceptually.

What I would postpone is the *deep implementation-level curriculum*.

Because consider Raft.

At first it looks like:

> “Here's this weird algorithm where servers vote for leaders.”

After the foundations, you see:

```text
              DISTRIBUTED CONSENSUS

       networking              concurrency
           │                       │
      messages fail           state races
      messages delay          synchronization
      messages reorder        atomicity
           │                       │
           └──────────┬────────────┘
                      │
                  replication
                      │
                persistent state
                      │
                      ▼
              crash / recovery
                      │
                      ▼
                  consensus
```

Now you're equipped to understand **why every ugly piece exists**.

That's why the original roadmap deliberately resisted doing deep distributed systems too early. :chatgpt-content-reference{index="5"}

---

## So my actual recommended sequence for you is

I'd modify our roadmap into:

```text
0. PROGRAMMING / LOW-LEVEL ORIENTATION
   └── finish selected CS50
              │
              ▼
1. SOFTWARE CONSTRUCTION
   └── immediately applicable to Strata
              │
              ▼
2. OPERATING SYSTEMS
   └── processes / threads / concurrency / I/O / memory
              │
              ▼
3. NETWORKING
   └── sockets / HTTP / TCP / IP / DNS
              │
              ▼
4. COMPUTER SYSTEMS
   └── C / assembly / memory / linking / caches / machine
              │
              ▼
5. ALGORITHMS
   └── serious DS&A treatment
              │
              ▼
6. DATABASE SYSTEMS
   └── storage / indexes / queries / transactions / recovery
              │
              ▼
7. SYSTEMS ENGINEERING
   └── whole-system reasoning
              │
              ▼
8. DISTRIBUTED SYSTEMS
   └── deep treatment
              │
              ▼
9. SPECIALIZATION
   ├── data systems
   ├── distributed systems
   └── ML systems
```

But there's a second track running simultaneously:

```text
                       CURIOSITY TRACK
                             │
       ┌─────────────────────┼────────────────────┐
       ▼                     ▼                    ▼
     DDIA               STRATA QUESTIONS      PROJECTS
       │                     │                    │
       └─────────────────────┼────────────────────┘
                             │
                   GO ANYWHERE YOU WANT
```

**That track has no prerequisites.**

If tomorrow DDIA introduces Byzantine fault tolerance and you're fascinated, read about it.

If Strata makes you wonder how TCP detects packet loss, investigate it.

If your Xiangqi engine makes you wonder how CPU caches affect tree search, go look.

The ordered curriculum tells you **where to systematically build depth**.

It should never tell you **what you're allowed to be curious about.**

And that distinction resolves your concern about grinding tedious material. I don't want you thinking:

> “Apparently I'm on Chapter 6 of CS:APP for the next four Saturdays because the roadmap says so.”

Instead:

> “I'm currently building my computer-systems foundation. CS:APP is helping. This chapter is important because it explains X. I understand the learning objective; let's study it.”

And if some 80-page section turns out to provide depth irrelevant to our objective, **we skip it**.

That's exactly why changing the roadmap from *courses/books* to *subjects* was important.

The ordered path gives us the **dependency structure**. Your work, DDIA, personal projects, and curiosity give us the **motivation and forward glimpses**. I think combining those two—rather than choosing either rigid bottom-up study or chaotic just-in-time learning—is probably the strongest curriculum for you.