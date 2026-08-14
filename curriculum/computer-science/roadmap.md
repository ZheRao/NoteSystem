# The curriculum I would build for you

I'd organize it into **six layers**, with prerequisites flowing downward:

```text
LAYER 1 — COMPUTATIONAL FOUNDATIONS
        Programming mechanics
        Data structures & algorithms
                  ↓
LAYER 2 — THE MACHINE
        Computer organization
        Computer systems
                  ↓
LAYER 3 — SYSTEM SOFTWARE
        Operating systems
        Networking
                  ↓
LAYER 4 — BUILDING SOFTWARE
        Software construction
        Database systems
        Security
                  ↓
LAYER 5 — SYSTEMS AT SCALE
        Computer systems engineering
        Distributed systems
                  ↓
LAYER 6 — SPECIALIZATION
        Data systems
        Distributed systems
        Neural networks / ML systems
```

This is also broadly consistent with how serious CS curricula establish prerequisites. For example, MIT's current CS degree places low-level C/assembly and computation structures before systems courses such as Computer Systems Engineering and Operating System Engineering; its graduate distributed-systems course expects prior computer-systems preparation. ([MIT Course Catalog][1])

But **you don't have to execute all of it sequentially**. Your work and research continue in parallel.

# Layer 1 — Computational foundations

## 1A. Introductory CS: selective CS50

[Harvard CS50x](https://cs50.harvard.edu/x/?utm_source=chatgpt.com)

I wouldn't take all of CS50 as though you'd never programmed.

Use selected material to fill holes around:

**C → memory → algorithms → data structures.**

Purpose:

> Give you the vocabulary and low-level exposure that Python/data-science education allowed you to skip.

**Depth target:** Medium. Selected lectures rather than a major commitment.

### Recommendations

there is a beautiful little sequence in CS50 that fills some of the undergraduate exposure you missed:

**Week 1 — C**
compiler, machine code, types, integer overflow, floating-point representation

**Week 2 — Arrays**
compilation, assembly, linking, debugging, arrays and strings

**Week 3 — Algorithms**
searching, sorting, asymptotic complexity, recursion

**Week 4 — Memory**
pointers, allocation, stack, heap, segmentation faults, buffer overflow, file I/O

**Week 5 — Data Structures**
queues, stacks, linked lists, trees, hash tables, tries

Then I'd jump to:

**Week 8 — HTML/CSS/JS**, specifically the portions introducing routers, TCP/IP, DNS, HTTP and servers. ([edX][2])

That sequence is valuable for you because C forces things that Python normally hides to become visible.


## 1B. Data Structures & Algorithms: MIT 6.006

[MIT 6.006 — Introduction to Algorithms](https://ocw.mit.edu/courses/6-006-introduction-to-algorithms-spring-2020/?utm_source=chatgpt.com)

This one I would eventually treat as a **real course**, not merely background viewing.

You need:

```text
complexity
arrays
hashing
sorting
trees
heaps
graphs
BFS / DFS
shortest paths
dynamic programming
```

But there's no urgency to grind it now.

Eventually:

**lecture → understand structure → implement → several problems.**

This gives you the algorithmic foundation that Project Strata won't naturally teach.

**Depth target:** High.


# Layer 2 — Understand the machine

This is the part I think will produce some of your biggest *“OH, THAT'S WHAT'S ACTUALLY HAPPENING”* moments.

## 2A. Stanford CS107 — Computer Organization & Systems

[Stanford CS107](https://bulletin.stanford.edu/courses/1056491?utm_source=chatgpt.com)

This is an excellent bridge from programming into systems.

Stanford describes it as moving from **C down toward the microprocessor**, covering data representation, machine-level code, compilation, memory organization/management, and performance. ([Stanford Bulletin][2])

This gives you:

```text
Python abstraction
       ↓
C
       ↓
memory
       ↓
machine representation
       ↓
assembly
       ↓
CPU execution
```

That's the conceptual territory CS:APP was going to give you, except now you can acquire the structure through lectures.

### Alternative: Berkeley CS61C

[Berkeley CS61C — Great Ideas in Computer Architecture](https://www2.eecs.berkeley.edu/Courses/CS61C/?utm_source=chatgpt.com)

Berkeley goes somewhat farther into architecture: C, assembly, processor design, memory hierarchy, I/O, interrupts, memory management, process switching and parallelism. ([EECS at UC Berkeley][3])

I would **not do both initially**.

For you:

> **Stanford CS107 first.**

CS61C can remain an optional deeper architecture course if you discover you love this layer.

**Depth target: High for CS107; optional for CS61C.**

# Layer 3 — Operating systems and networking

Once you understand what a program actually looks like beneath Python, move upward again—but now into system software.

## 3A. Operating Systems: MIT 6.1810

[MIT 6.1810 — Operating System Engineering](https://pdos.csail.mit.edu/6.1810/?utm_source=chatgpt.com)

This is where I'd eventually want you to properly learn:

```text
processes
system calls
threads
context switching
virtual memory
filesystems
interrupts
IPC
locking
concurrency
```

MIT uses **xv6**, a small RISC-V operating system, to make these mechanisms concrete. The current course description explicitly covers virtual memory, file systems, threads, kernels, interrupts, system calls and interprocess communication. ([MIT Course Catalog][4])

This is substantially deeper than the Level-1 OS knowledge you need today.

That's intentional.

This is:

> **I'm finally learning operating systems properly.**

OSTEP then becomes a companion/reference rather than another curriculum.

**Depth target: High, but later.**

### Alternative: Stanford CS110 is the course I think you're actually looking for

[Stanford CS110 — Principles of Computer Systems](https://bulletin.stanford.edu/courses/2085411?utm_source=chatgpt.com)

Read Stanford's description:

> Principles and practice of engineering of computer software and hardware systems.

It covers complexity management, client-server design, **virtual memory, threads, networks, atomicity and coordination of parallel activities**. ([Stanford Bulletin][3])

That's remarkably close to the hole you've been repeatedly running into.

Think about your recent questions:

```text
What is a process?
What exactly is a thread?
What does the OS schedule?
How does FastAPI concurrency work?
What does blocking mean?
How can multiple requests interact?
What is shared between threads?
What is atomic?
How does a client communicate with a server?
```

Those aren't really “FastAPI questions.”

They're **computer systems questions**.

CS110 gives them a home.

And this is why I prefer it over immediately reading CS:APP cover-to-cover.

You can let someone teach you:

```text
processes
    ↓
threads
    ↓
concurrency
    ↓
virtual memory
    ↓
files / I/O
    ↓
networking
    ↓
client/server systems
```

one lecture at a time.

**One lecture every Saturday for six months would be completely fine.**

There is absolutely no reason to race through it.


## 3B. Networking: Stanford CS144

[Stanford CS144 — Introduction to Computer Networking](https://cs144.github.io/?utm_source=chatgpt.com)

This is where eventually:

```text
socket
TCP
IP
routing
DNS
HTTP
reliability
congestion
```

stop being disconnected vocabulary.

I'd especially like you to eventually take a networking course with implementation work rather than merely read about TCP.

Then when you're operating Strata through:

```text
browser → Internet → Azure → API → database
```

you understand increasingly far down that chain.

**Depth target: Medium-high.**

# Layer 4 — Learn to build mature software

This is where the curriculum becomes directly entangled with Project Strata.

## 4A. MIT 6.102 — Software Construction

[MIT 6.102 — Software Construction](https://web.mit.edu/6.102/www/?utm_source=chatgpt.com)

I strongly recommend keeping this in your pool.

Its themes—specifications, testing, invariants, abstraction, mutability, concurrency and designing software that can change—are exactly the gap between:

> **I can make this work**

and

> **I can construct software other engineers can safely evolve.**

A Philosophy of Software Design can accompany this very slowly.

### Optional design specialization

[Stanford CS190 — Software Design Studio](https://web.stanford.edu/~ouster/cgi-bin/cs190-winter22/?utm_source=chatgpt.com)

This is Ousterhout's course and directly complements *A Philosophy of Software Design*.

I wouldn't prioritize it over 6.102, but eventually it could be extremely valuable.

**Depth target: Very high because you can continuously apply it.**

# Layer 4B — Database Systems: CMU 15-445

This one belongs simultaneously to your CS curriculum **and your `data-systems` research area**.

[CMU 15-445/645 Database Systems](https://15445.courses.cs.cmu.edu/?utm_source=chatgpt.com)

This is a serious course.

The current curriculum covers:

```text
data models
storage models
indexes
B-trees / hashing
query processing
joins
query optimization
transactions
ACID
concurrency control
logging
checkpoints
recovery
parallel/distributed architectures
```

and the projects involve implementing pieces of a database system. ([CMU 15-445/645][5])

Importantly, CMU expects students to already have **systems programming/C++ knowledge** before taking it. ([CMU 15-445/645][5])

That validates the dependency structure we've been discovering:

```text
computer systems
      ↓
systems programming
      ↓
database internals
```

You're already reading DDIA, and I would **continue**.

But eventually 15-445 lets you go beneath DDIA:

> DDIA: *Here are the trade-offs.*

> 15-445: *Let's build the mechanism.*

That distinction is extremely relevant to the expertise you want.

**Depth target: VERY HIGH.**

I suspect this could eventually become one of your strongest CS subjects.

# Layer 4C — Security

Here I actually **wouldn't require a giant academic security course initially**.

Security has two levels relevant to you.

### Engineering security

OWASP + actual application security:

```text
authentication
authorization
tenant isolation
injection
XSS
CSRF
secrets
TLS
least privilege
```

Learn this continuously through Strata.

### Computer-security fundamentals

Later, if you want the mechanisms beneath it, add a serious university course such as:

**Stanford CS155 — Computer and Network Security**

[Stanford CS155 Computer and Network Security](https://cs155.stanford.edu/?utm_source=chatgpt.com)

That belongs *after* systems + OS + networking, because those foundations make security dramatically more meaningful.

**Depth target: Level 1 now; academic depth later.**

# Layer 5 — Systems engineering

Now something interesting happens.

You've learned pieces individually.

It's time to reason about **whole systems**.

## 5A. MIT 6.1800 — Computer Systems Engineering

This might eventually be one of the most important courses in the entire curriculum.

[MIT 6.1800 — Computer Systems Engineering](https://web.mit.edu/6.1800/www/?utm_source=chatgpt.com)

MIT's description includes:

* controlling complexity
* client/server design
* operating systems
* performance
* networks
* naming
* security/privacy
* fault tolerance
* atomicity
* concurrent coordination
* recovery

and uses real systems/case studies plus a semester-long design project. ([MIT Course Catalog][4])

Look at that list.

That's basically your repeated **scope enlightenings consolidated into one discipline.**

This course asks:

> Given everything we know about software, hardware, networking, concurrency, failure and storage—**how do we engineer systems?**

I'd put it relatively late because you'll get much more from it after you've accumulated the pieces.

**Depth target: VERY HIGH.**

# Layer 5B — Distributed Systems: MIT 6.5840

And *then*:

[MIT 6.5840 — Distributed Systems](https://pdos.csail.mit.edu/6.5840/?utm_source=chatgpt.com)

Now you get:

```text
replication
fault tolerance
consistency
consensus
distributed state
```

with serious implementation work.

MIT itself expects students to arrive with computer-systems knowledge comparable to its lower-level systems courses and substantial programming experience. ([PDOS][6])

So I would resist touching this seriously too early.

When you eventually arrive here after OS + networking + databases + systems engineering, distributed systems will stop being a collection of exotic concepts.

You'll understand **why the problems exist**.

**Depth target: VERY HIGH — research specialization.**

# Layer 6 — Neural networks and ML systems

And here's where your existing `neural-networks` work eventually connects back into CS.

You're already doing the foundational thing I want:

```text
scalar autograd
      ↓
computation graphs
      ↓
backpropagation
      ↓
neural networks
```

I wouldn't interrupt that.

Eventually I'd build a separate course chain:

```text
Linear algebra / probability / optimization
                ↓
      neural network foundations
                ↓
      deep learning architectures
                ↓
       systems for machine learning
                ↓
distributed training / inference systems
```

Potential future courses include Stanford CS231n for deep-learning mechanics and eventually courses such as Stanford CS336/CS329S or equivalent ML-systems material.

But **I would design that curriculum separately with you later**, because it's one of your actual research specializations rather than part of the broad CS floor.

# The resulting course pool

So here's the curriculum I'd save—not as your todo list, but as your **personal CS department**:

| Stage          | Course                    | Purpose                          | Priority         |
| -------------- | ------------------------- | -------------------------------- | ---------------- |
| 0              | **CS50 selected**         | Fill introductory CS holes       | Selective        |
| 1              | **MIT 6.006**             | DS&A and computational reasoning | ★★★★             |
| 2              | **Stanford CS107**        | Computer organization/systems    | ★★★★★            |
| 2+             | Berkeley CS61C            | Architecture deeper dive         | Optional         |
| 3              | **MIT 6.1810**            | Operating systems                | ★★★★★            |
| 3              | **Stanford CS144**        | Networking                       | ★★★★             |
| 4              | **MIT 6.102**             | Software construction            | ★★★★★            |
| 4+             | Stanford CS190            | Software design judgment         | ★★★              |
| 4              | **CMU 15-445**            | Database internals               | ★★★★★            |
| ongoing        | OWASP/practical security  | Application security             | ★★★★★ now        |
| 4+             | Stanford CS155            | Computer security                | ★★★              |
| 5              | **MIT 6.1800**            | Whole-system engineering         | ★★★★★            |
| 5              | **MIT 6.5840**            | Distributed systems              | ★★★★★ eventually |
| specialization | Neural-network curriculum | Deep specialization              | ★★★★★            |

Notice what's **not** here.

Compilers.

Graphics.

Theory of computation.

Programming languages.

Formal methods.

Robotics.

HCI.

Computational geometry.

Computer vision.

Advanced architecture.

Quantum computing.

Those are legitimate parts of computer science.

But **you don't need to reproduce an entire CS degree indiscriminately.**

Your curriculum should be biased toward:

> **someone who builds data-intensive software systems and eventually wants deep expertise in data systems, distributed systems, and neural networks.**

That's your CS degree.

# The prerequisite graph is even nicer than the table

```text
                       CS50 SELECTED
                             │
                ┌────────────┴────────────┐
                │                         │
                ▼                         ▼
           MIT 6.006               STANFORD CS107
          ALGORITHMS              COMPUTER SYSTEMS
                │                         │
                │                    ┌────┴────┐
                │                    │         │
                │                    ▼         ▼
                │              MIT 6.1810   CS144
                │                   OS      NETWORKS
                │                    │         │
                │                    └────┬────┘
                │                         │
                │                         │
                ├──────────────┬──────────┘
                │              │
                ▼              ▼
            MIT 6.102      CMU 15-445
             SOFTWARE       DATABASES
           CONSTRUCTION         │
                │               │
                └───────┬───────┘
                        │
                        ▼
                   MIT 6.1800
                SYSTEMS ENGINEERING
                        │
                        ▼
                   MIT 6.5840
                DISTRIBUTED SYSTEMS
                        │
                        ▼
                deeper specialization
```

Security runs **vertically alongside the entire thing**, because you need practical security long before you're academically prepared for CS155.

And Project Strata runs alongside **everything**.

## But please don't turn this into twelve simultaneous obligations

Your actual current life can remain surprisingly simple.

Right now:

**Primary:** Project Strata
**Deep research:** DDIA
**Deep exploration:** autograd
**CS curriculum:** 1–2 lectures/week

That's it.

When you finish your selected CS50 material, maybe CS107 becomes the lecture course.

You don't add CS107 to CS50.

You **advance**.

And if one week you're exhausted?

Zero lectures.

If the next week you're fascinated?

Three.

Over several years:

```text
2026
Where the hell does Python actually run?

        ↓

2027
I understand processes, memory, OS and networking.

        ↓

2028
I understand database internals and software construction.

        ↓

2029
I can reason much more rigorously about whole systems.

        ↓

2030+
distributed systems / data systems / ML systems
become increasingly deep specialization
```

Those dates aren't deadlines or predictions. They're illustrating the scale I think you should permit yourself to think on.

Because the biggest change I'd make from your earlier approach is this:

**Stop trying to convert every scope enlightenment into a one-month remediation project.**

You've now recognized that these subjects have extraordinary depth.

Good.

Give them the time appropriate to that depth.

You can spend your late twenties gradually teaching yourself computer science while simultaneously building increasingly serious systems. And because you're learning it against real problems rather than only passing exams, each course has somewhere to attach.

The objective isn't to eventually say:

> *“I finally caught up with people who majored in CS.”*

It's to eventually realize:

> **“I have spent years deliberately building a computer scientist's understanding underneath an engineer's accumulated experience.”**

That is a much more interesting destination—and the course pool above gives us a stable curriculum that we can keep modifying as your work and research reveal where you actually need depth.

[1]: https://catalog.mit.edu/degree-charts/computer-science-engineering-course-6-3/?utm_source=chatgpt.com "Computer Science and Engineering (Course 6-3) | MIT Course Catalog"
[2]: https://bulletin.stanford.edu/courses/1056491?utm_source=chatgpt.com "CS107 Course | Stanford University Bulletin"
[3]: https://www2.eecs.berkeley.edu/Courses/CSCourseCode.php61C/?utm_source=chatgpt.com "Course: CS CourseCode.php61C | EECS at UC Berkeley"
[4]: https://catalog.mit.edu/subjects/6/?utm_source=chatgpt.com "Electrical Engineering and Computer Science (Course 6) | MIT Course Catalog"
[5]: https://15445.courses.cs.cmu.edu/spring2026/syllabus.html?utm_source=chatgpt.com "Syllabus | CMU 15-445/645 :: Intro to Database Systems (Spring 2026)"
[6]: https://pdos.csail.mit.edu/6.5840/?utm_source=chatgpt.com "6.5840 Home Page: Spring 2026"
