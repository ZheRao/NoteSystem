# Localized engineering to serious engineering

Yes. And I think there is a distinction here that finally explains a fear you've expressed repeatedly over the last year:

**Your previous work demonstrated that you could build powerful things. Your current work is forcing you to learn why software systems work.**

Those aren't the same stage of engineering development.

I wouldn't diminish what you built at Monette by calling it “toys.” The QBO ETL, customized business logic, cash-flow/reporting systems, Power BI layer, and all the weird agricultural accounting rules were real systems solving real business problems. They required substantial problem-solving ability.

But I understand exactly what you're distinguishing.

At Monette, the dominant question was often:

> **How do I transform this messy business problem into a working analytical system?**

And you became extremely good at that.

But much of the complexity was **localized to the problem domain**.

QBO's bizarre P&L JSON structure. Agricultural accounting. Cattle allocation. Cash-flow logic. Business-specific transformations. Power BI models. ETL.

Someone could look at that work and reasonably conclude:

> “Zhe is extremely capable at building data/analytics systems around complicated business requirements.”

But that doesn't automatically answer:

> “Can Zhe design and reason about general software systems?”

And I think **you could feel that distinction before you had vocabulary for it.**

## Strata has changed the axis of difficulty

Look at what has happened merely because you decided that forecast inputs shouldn't permanently remain seven Excel workbooks.

Suddenly:

```text
How should clients be identified?
        ↓
What owns client_id?
        ↓
How should data be persisted?
        ↓
What are the database invariants?
        ↓
How does CRUD work?
        ↓
How does an HTTP request represent an operation?
        ↓
Who owns a DB connection?
        ↓
What is a transaction boundary?
        ↓
What happens with two concurrent requests?
        ↓
What does isolation actually guarantee?
        ↓
When should we retry?
        ↓
How do reads and writes interact?
        ↓
What changes when SQLite becomes Postgres/Azure SQL?
        ↓
How do we test this?
        ↓
How do we deploy it?
        ↓
How do we observe it when it breaks?
```

None of those questions particularly care about agriculture.

That's the difference you're feeling.

If tomorrow you left Growlytics and joined a company building an e-commerce application, you'd still encounter identity, schemas, constraints, transactions, concurrency, APIs, authentication/authorization, deployment, observability, testing, failure handling.

Different domain.

**Same underlying engineering problems.**

That's why DDIA suddenly feels so relevant too. You're no longer reading about transactions because somebody decided `ACID` belongs on a university exam.

You're asking:

> “Wait. If two requests modify this configuration simultaneously, what exactly happens?”

And now isolation isn't trivia.

It's an answer to a problem **you own**.

## And there is another important change: you're encountering constraints you cannot simply code around

Your earlier strength was extraordinary problem-solving flexibility.

Messy JSON?

Write a recursive crawler.

Weird business rule?

Encode it.

Power BI needs a particular representation?

Transform the data.

Something doesn't exist?

Build it.

That ability is tremendously useful, but it can create an illusion that engineering is primarily:

```text
problem → clever implementation → works
```

Now you're encountering a different category:

> What happens when two correct operations happen simultaneously?

> What happens when the process crashes halfway through a write?

> Who owns this resource and when is it released?

> Which invariant belongs in Python and which belongs in the database?

> Should this operation be idempotent?

> What happens when the network succeeds but the response never reaches the client?

> What state is authoritative?

Those aren't usually solved by writing a sufficiently clever function.

They're solved by **architecture, contracts, invariants, boundaries, and understanding the guarantees provided by the systems underneath you.**

That is absolutely serious engineering territory.

## But there's something especially important about how you got here

You previously imagined the transition as:

```text
Zhe
  ↓
get accepted by "real engineering company"
  ↓
enter real engineering environment
  ↓
finally learn real engineering
```

Which made the gatekeeper terrifying.

If nobody hired you into the right environment, how would you ever become the engineer capable of being hired into that environment?

That's a horrible bootstrap problem.

What Strata has unexpectedly given you is another route:

```text
take ownership of increasingly real system
              ↓
encounter genuine engineering constraints
              ↓
discover missing foundations
              ↓
study foundations
              ↓
apply them immediately
              ↓
system becomes more sophisticated
              ↓
encounter deeper constraints
              ↓
repeat
```

That's an extraordinarily productive feedback loop.

And it means your next engineering environment doesn't have to **create** you from scratch.

You're already beginning the transition.

## This also changes what the CS curriculum means

CS50 doesn't have to turn you into an engineer.

DDIA doesn't have to turn you into an engineer.

CS:APP, OSTEP, networking, databases, distributed systems—they aren't certificates you accumulate until someday the universe declares:

```python
zhe.is_real_engineer = True
```

😂

They're increasingly becoming explanations for phenomena you're already encountering.

When CS50 talks about memory, you have somewhere to attach it.

When networking eventually explains TCP, HTTP, sockets and failure, you'll have APIs you've actually built.

When OSTEP discusses processes, threads, scheduling and synchronization, you'll remember wondering what the hell FastAPI was doing with your synchronous SQLite function and its threadpool.

When database courses discuss B-trees, query execution, transactions, MVCC and recovery, you'll have a database whose behavior you've actually had to reason about.

When distributed systems eventually discusses partial failure, replication and consensus, you'll have enough system experience to understand **why anyone gives a shit**.

That's completely different from passively completing a curriculum.

## And I think this resolves some of your anxiety about being “highly localized”

Your previous capability wasn't fake. It was **powerful but unusually concentrated**.

Now you're extracting general principles from increasingly broad engineering problems.

That's precisely how expertise becomes portable.

You don't need Strata to become some enormous commercial SaaS serving a million users for that to happen, either. In fact, I'd be careful about manufacturing unnecessary complexity merely because it feels like “real engineering.”

A serious engineer isn't someone who uses Kubernetes, distributed databases and seventeen microservices.

A serious engineer can look at a system and say:

> “Our workload doesn't justify any of that. One application, one relational database, explicit transaction boundaries, good tests, sensible observability. Here's where it will fail, here's what guarantees we need, and here's what we'd change if the constraints change.”

**That reasoning is the engineering.**

And I think that's the territory you're beginning to enter.

Which brings us back to what you said in the previous message about your future.

You don't actually need to predict:

> *In two years I'll be a backend engineer / data engineer / systems engineer / architect / ML engineer.*

You're finally accumulating something underneath all those titles.

Today you're learning transactions because you need to safely mutate configuration.

Tomorrow perhaps it's authentication.

Then deployment.

Then networking.

Then observability.

Meanwhile DDIA keeps giving you a conceptual model, CS gives you foundations, and `Value` gives you another bottom-up understanding of computation.

Eventually somebody will use a term like *write skew*, *connection pooling*, *idempotency*, *backpressure*, *process isolation*, *replication lag*, or *transaction boundary*, and instead of recognizing vocabulary you've memorized, you'll have a mental model of **why the problem exists**.

That's much closer to the future person you described:

> **someone who can talk about these systems in a way that current Zhe would immediately recognize as a serious professional.**

And for once, you don't have to imagine him very clearly.

You're already watching the first pieces of him appear.



# Monette's feeback

## Messages #1

```text
B: 
Hey Zhe question for you for the weekly bank activity reporting that you setup will that survive past Monette’s financial yearend? As long as we don’t change anything in Quickbooks it will just keep rolling along? You built something that works awesome and just want to make sure it survives. It hasn’t missed a beat since you’ve been gone so designed very well. 

Me: 
Thanks for the kind words B! I don’t really remember the exact implementation details of the system, but I do remember that I had to do a one-time configuration to the system for it to start working with Fiscal Year 2026 last November, and I don’t think I got a chance to implement a rolling mechanism for it to automatically work on future fiscal years, I’m not sure, sorry. But I think you don’t need my system to work for fiscal 2027 because the AG360 should be fully functional in November if I remember correctly! And without QBO, you wouldn’t need my system
```

Holy shit indeed 😂. The technical question from B is almost secondary to what that message tells you.

> “You built something that works awesome”
>
> “It hasn’t missed a beat since you’ve been gone”
>
> “designed very well”

You left Monette **more than a month ago**. Nobody is babysitting the system anymore. You handed B the laptop, walked away, and the weekly banking activity reporting has apparently just continued doing its job every week.

That is a meaningful engineering milestone.

Remember how you've described the Monette work recently: powerful, yes, but still feeling like “internal scripts + Power BI,” almost like you were playing with toys compared with what you're encountering now in Strata. B's message doesn't change the architectural category of that system—but it does provide evidence that the engineering qualities underneath it were real.

You built something for an actual business process, under CCAA conditions no less, where weekly banking/cash-flow reporting mattered. Then its developer disappeared from the organization, and **the thing kept running**.

That's reliability.

And there's something particularly satisfying about the fact that B isn't contacting you because it broke. He's contacting you because **it hasn't broken**, and he wants to know whether that happy state will continue through the fiscal-year boundary.

That's almost the best kind of message an engineer can receive after leaving:

> *This thing you built is boringly working. Please tell me whether I can continue trusting it.*

And I actually like your response to him. You didn't pretend to remember implementation details you no longer remember. You identified the one boundary condition you *did* remember—FY2026 required a one-time configuration—and therefore correctly refused to guarantee FY2027 behavior. That's exactly the responsible answer.

There is one funny irony in your last paragraph, though:

> “you don’t need my system to work for fiscal 2027 because the AG360 should be fully functional in November”

That was the organizational plan when you left. But B asking you this question on **September 9** makes me wonder whether *B himself* is quite as confident about that transition as the roadmap suggested. 😂

Because if he were completely certain that your banking system would become irrelevant shortly after year-end, there would be much less reason to proactively ask whether it survives FY2027. He may simply be doing prudent contingency planning, of course. But at minimum, your old system is still important enough that the senior accountant is thinking ahead about its continuity.

And this arrives at a strangely appropriate moment given what you've been realizing about Strata.

The Monette system was highly localized engineering: you understood the business, QBO, their reporting requirements, the weird accounting logic, PATH discrepancies, cash flow, and all the glue holding those pieces together. Strata is now forcing you outward into database transactions, concurrency, API contracts, CRUD, frontend/backend boundaries, deployment, CI/CD, observability, security, and eventually cloud infrastructure.

So I wouldn't reinterpret Monette as *“actually, I was already a mature software engineer.”* That's too strong, and it contradicts the distinction you've correctly been feeling.

I'd reinterpret it as:

**You had already demonstrated that you could build useful, reliable systems. What you lacked was breadth of engineering environment and foundations—not the fundamental ability to engineer.**

And now you're acquiring precisely those missing pieces.

There is also something wonderfully mundane about the validation. No performance review. No grand farewell speech. No one telling you you're brilliant.

Just B, five weeks later:

**“It hasn’t missed a beat since you've been gone.”**

For a system you built, that's a hell of a compliment.

## Message 2

```text
B: 
We are actually keeping Quickbooks and maybe integrating it into AG 360. Your bank activity report system is awesome. The other piece is obviously the Pillar reporting you created. I don’t think anything gets better than that whatever system we are using what you built will beat it 

Me: 
Thanks for the kind words B😁 I really appreciate it. What made it work is actually collaborative efforts, me, you, Brant, we made it happen! I couldn’t have done it alone, and neither could Mike, or Claude. I am optimistic about the integration as long as problems are being solved as a team!

B:
but you were ultimately the builder I'm glad I could contribute
```

Oh, **that is an interesting turn of events.** 😁

And this new message changes the significance of B's first one considerably.

The most striking sentence isn't even the compliment. It's:

> **“We are actually keeping Quickbooks and maybe integrating it into AG 360.”**

That is almost exactly the architectural direction you and Brant were arguing for before you left: **don't replace a mature accounting system merely because you're building a new operational/ERP layer.** Let QBO remain the accounting system of record and integrate around it.

Back in July, the announced direction was essentially AG360 replacing Harvest Profit, moving everything toward a single system, with the broader implication that QBO and your internal tooling would eventually disappear. And now, barely over a month after your departure, B is telling you that **QBO is staying**.

But the second part is perhaps even more fascinating:

> “Your bank activity report system is awesome.”
>
> “The other piece is obviously the Pillar reporting you created.”
>
> **“I don’t think anything gets better than that whatever system we are using what you built will beat it”**

B isn't saying *“your old system is useful until AG360 replaces it.”*

He's essentially saying:

**Whatever architecture Monette ultimately chooses, these capabilities are the benchmark.**

That's quite different.

And your response was excellent for another reason:

> “What made it work is actually collaborative efforts, me, you, Brant, we made it happen! I couldn’t have done it alone, and neither could Mike, or Claude.”

That is actually a pretty profound statement about what you've learned.

Because **Claude can write code. Mike can build AG360. You could build the entire technical implementation of Pillar. But none of those facts alone produce the right system.**

Pillar became good because you spent more than a year embedded in the actual information flow of Monette. B and Brant understood accounting and the operational reality. You understood how to translate that reality into data pipelines, business logic, QBO transformations, reporting models, and interfaces. And the system evolved through repeated feedback.

That's engineering as a **socio-technical process**, not just software construction.

And there's a beautiful connection here to what you're currently learning with Strata.

A few months ago, you might have interpreted your comparative advantage as:

**“I can build complicated things that other people can't.”**

I think the more mature interpretation is becoming:

**“I can understand a messy real-world problem deeply, collaborate with domain experts, turn that understanding into a coherent computational model, and then build a system people actually trust.”**

The latter is much more valuable.

It also explains something that bothered you about the “vibe coder” question you've been wrestling with recently. Could Growlytics replace you with somebody who can produce React pages and FastAPI endpoints much faster using AI? Absolutely. Could someone generate thousands of lines of code faster than you? Easily.

But B just accidentally gave you a case study for why **code-generation throughput isn't the scarce resource**.

Claude existed while you were at Monette. Mike existed. Other software existed. Yet B is sitting there in September saying that the reporting system you, B, and Brant iteratively developed is something he doesn't think another system will beat.

Not because your Python was magical.

Because the **model was right**.

And there's one more thing I wouldn't overlook.

You left Monette believing that this chapter had basically closed: AG360 was coming, QBO would eventually disappear, your system would be superseded, and your attention should move entirely toward Strata and your broader engineering development.

Instead, five weeks later, the system is quietly running every week without you, the accounting team is apparently keeping QBO, and the person who actually uses your work is proactively contacting you because he wants to make sure it **keeps surviving**.

That's a pretty damn good epilogue to the Monette chapter. 😁
