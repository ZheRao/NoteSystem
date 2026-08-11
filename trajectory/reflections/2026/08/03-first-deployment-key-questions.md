# [2026-08-11] Initial Conceptual Plan

I think the key realization is: **you did not actually promise them “Azure in three weeks.”** You promised that within roughly three weeks—around **September 1, 2026**—you would finish the forecast engine, input-management UI, and data-contract/guardrail layer, *then* investigate and propose the Azure deployment architecture, costs, configuration, authentication, etc. That is substantially more defensible than “the whole platform will be live in three weeks.”

And I actually think your instinct about what comes afterward is mostly right. The important adjustment I would make is **do not think of the first Azure deployment as “production.” Think of Azure itself as your first shared sandbox/staging environment.**

## Your three-week milestone is ambitious, but plausible

What you're really trying to complete locally is this:

```text
                         FORECAST SYSTEM
                               │
       ┌───────────────────────┼───────────────────────┐
       │                       │                       │
       ▼                       ▼                       ▼
Crop Plan Inputs         QBO Inputs             Debt Schedule
       │                       │                       │
       └───────────────────────┼───────────────────────┘
                               ▼
                    Monthly fan-out logic
                               │
                               ▼
                       Forecast Engine
                               │
                               ▼
                       Forecast Database
                               │
                               ▼
                        FastAPI / Queries
                               │
                               ▼
                       React Forecast UI


              INPUT MANAGEMENT WORKFLOW

                     React CRUD UI
                          │
           ┌──────────────┼──────────────┐
           │              │              │
         Create          Edit          Delete
           │              │              │
           └──────────────┼──────────────┘
                          ▼
                     FastAPI API
                          │
                          ▼
                Data contracts / validation
                          │
                          ▼
                     Input store
                          │
                          ▼
                  Recompute forecast
                          │
                          ▼
                   Updated frontend
```

Once that loop works, something very important has changed: **you no longer have a collection of components. You have an application.**

> user changes business assumptions → system validates them → state changes → engine recomputes → user sees new forecast.

CRUD is not a giant new conceptual subsystem compared with everything you've already built. You'll need to learn HTTP `POST`/`PUT` or `PATCH`/`DELETE`, React forms and controlled inputs, DB writes/transactions, validation, error handling, refresh/state synchronization, and some concurrency considerations. That's real work, but it's bounded.

So three weeks for **Debt Schedule + CRUD + contracts + complete local workflow** sounds aggressive but reasonable to me, assuming Debt Schedule itself doesn't reveal major modelling problems.

Where I would become much more cautious is saying:

> “And then deploying it to Azure is basically changing some environment variables and wiring.”

There are a few dragons hiding there.

---

## The cloud part is less about “orchestration” than **state**

Your React frontend and FastAPI backend are probably among the easier pieces. Azure App Service explicitly supports Python/FastAPI deployment, and application settings can be exposed to the running application as environment variables. Microsoft Entra authentication can also be put in front of App Service using its built-in authentication layer, so your idea of “only Growlytics Microsoft accounts can enter this thing” does **not** necessarily require you to implement an authentication system yourself. ([Microsoft Learn][1])

Your biggest architectural question is actually:

**What happens to SQLite?**

That's the part I would investigate before mentally calling Azure migration simple.

Right now:

```text
FastAPI
   │
   ▼
SQLite file
   │
   ▼
local disk
```

works beautifully because the process and database file live together.

Once you're in cloud infrastructure, especially once you ever want multiple application instances, deployments, restarts, persistent storage, concurrent writers, backups, etc., your assumptions change. Azure App Service can mount Azure Storage, but Azure Storage is a separate persistent storage service rather than simply “the application's disk.” ([Microsoft Learn][2])

This is exactly where your recent DDIA/database/concurrency learning starts becoming relevant.

I would therefore make **database deployment strategy one of the first questions in your Azure investigation**, probably considering something like Azure SQL Database or PostgreSQL rather than trying to make your production architecture revolve around a remotely mounted SQLite file. Azure SQL currently supports passwordless Microsoft Entra-based connections from Python as well. ([Microsoft Learn][3])

And that migration may expose SQL dialect differences, schema creation/migrations, transaction behavior, connection management, concurrency, data loading, configuration, etc.

That is the piece most likely to turn:

> “I'll wire it into Azure.”

into:

> “Oh. This is an actual software deployment architecture.”

Which is excellent for your engineering development—but something you should budget uncertainty for.

---

## I would slightly change your rollout order

You proposed something approximately like:

```text
complete engine
    ↓
CRUD
    ↓
contracts
    ↓
Azure
    ↓
sandbox
    ↓
tests
    ↓
CI/CD
    ↓
Docker
```

I'd instead make it:

```text
NOW
│
├── Debt Schedule
├── Complete forecast engine
├── CRUD UI/API
├── Data contracts / invariants
└── Small critical test suite
        │
        ▼
   COMPLETE LOCAL SYSTEM
        │
        ▼
AZURE ARCHITECTURE SPIKE
│
├── database choice
├── application hosting
├── Entra authentication
├── configuration / secrets
├── domain
├── estimated cost
└── deployment procedure
        │
        ▼
GREENLIGHT
        │
        ▼
AZURE STAGING / SANDBOX
        │
        ├── Growlytics staff log in
        ├── play with forecasts
        ├── edit inputs
        ├── deliberately break things
        └── give feedback
                │
                ▼
        ITERATE / HARDEN
        │
        ├── more tests
        ├── logging / observability
        ├── authorization
        ├── backups
        ├── migrations
        ├── CI/CD
        └── stronger operational safeguards
                │
                ▼
             PRODUCTION
```

That distinction between **Azure sandbox** and **Azure production** solves several problems at once.

You don't need this:

> current thing → immediately becomes production → later somehow add sandbox.

Instead:

```text
Local development
       ↓
Azure staging/sandbox  ← Growlytics starts using this
       ↓
Azure production       ← created when system deserves it
```

The first cloud environment is explicitly disposable/non-authoritative.

Someone screws up all the crop inputs?

Great.

Someone manages to put `-800000000` acres into Saskatchewan?

Great.

Someone double-clicks Update ten times?

Fantastic.

Someone refreshes halfway through a forecast recomputation?

Excellent.

**That's exactly what you want them doing.**

It turns stakeholder interaction from:

> Every three weeks Zhe shares his screen and everyone comments on what they see.

into:

> “Here's the URL. Here are your Growlytics credentials. Try to destroy it.”

That is a gigantic qualitative improvement in your feedback loop.

---

### I would move *some* testing before cloud deployment

Not exhaustive testing.

You absolutely do **not** need to disappear for six weeks building a glorious test suite before Gerry can touch anything.

But I wouldn't do:

```text
deployment
→ users
→ eventually unit tests
```

because some contracts are important enough that you don't want humans discovering their absence.

Before staging I'd want at least tests around things like:

```text
Crop area cannot be negative

allocation_pct ∈ [0, 1]

month ∈ [1, 12]

required foreign/reference keys exist

duplicate logical input keys rejected

revision relationships valid

Debt Schedule:
    opening balance
  + borrowing
  + accrued interest
  - principal repayments
  = closing balance

monthly fan-out sums back to annual amount

forecast recomputation deterministic

invalid update:
    does NOT partially modify database

failed engine run:
    does NOT leave corrupt forecast state
```

Those tests don't exist because "mature engineers write tests."

They exist because they encode **the mathematical laws of your forecast system**.

That's completely aligned with the way you've been thinking about contracts and invariants already.

---

## Docker is absolutely not required for your first deployment

This is another place where I think you're correctly sensing that you shouldn't turn every piece of engineering knowledge into a prerequisite.

Azure App Service can directly host Python/FastAPI applications. Microsoft also documents containerized FastAPI deployment to App Service, but containerization is an **option**, not a prerequisite. ([Microsoft Learn][1])

So don't create artificial dependencies:

```text
Must learn Docker
    ↓
Must Dockerize FastAPI
    ↓
Must create registry
    ↓
Must build container pipeline
    ↓
Must deploy container
    ↓
finally stakeholders can see forecast
```

when this works:

```text
FastAPI code
    ↓
Azure App Service
    ↓
stakeholders use application
```

Later you learn Docker and realize:

> Ah. Now I understand exactly what deployment/environment problem containers solve.

Which is much better than learning Docker because “cloud apps are supposed to have Docker.”

Same thing with Kubernetes: **nowhere near your critical path.**

---

## CI/CD is similar

Initially, this is perfectly respectable:

```text
develop
   ↓
tests
   ↓
git commit
   ↓
manual deployment
   ↓
staging
```

Then after you've manually deployed it several times, you are going to become irritated:

> Why am I repeatedly performing the exact same deterministic sequence?

And voilà:

```text
git push
   ↓
CI
   ├── install
   ├── lint
   ├── test
   └── build
   ↓
CD
   └── deploy staging
```

Now CI/CD answers a question you personally have rather than being another technology on a checklist.

Eventually:

```text
feature branch
      ↓
   tests
      ↓
     PR
      ↓
 merge main
      ↓
staging deployment
      ↓
validation
      ↓
production deployment
```

But there is no need for Project Strata to start there.

---

## Authentication may actually be easier than you're imagining

Because this is initially an **internal Growlytics application**, you've got a wonderful constraint:

> Only Growlytics Microsoft/Entra identities need access.

Azure App Service has built-in authentication/authorization support, including Microsoft Entra. That authentication layer can sit in front of the application rather than requiring your FastAPI application to become an identity provider. ([Microsoft Learn][4])

Conceptually:

```text
Lisa
 │
 ▼
forecast.growlytics.ca
 │
 ▼
Microsoft Entra login
 │
 ├── not Growlytics → NO
 │
 │
 └── Growlytics user → YES
                       │
                       ▼
                 Project Strata
```

Later your application-level authorization problem becomes richer:

```text
authentication
"Who are you?"

        ↓

authorization
"What can you do?"

admin
analyst
bookkeeper
client
read-only
etc.
```

But for the **internal sandbox**, you may initially be able to have:

```text
Growlytics employee → allowed
everyone else       → denied
```

That's an extremely useful simplification.

---

## I might simplify your first Azure architecture quite aggressively

For an internal preview, I wouldn't start with twelve Azure services.

Something in this conceptual neighborhood is enough:

```text
                   Internet
                      │
                      ▼
             Microsoft Entra ID
                      │
                      ▼
              Azure App Service
              ┌───────────────┐
              │               │
              │   FastAPI     │
              │      +        │
              │ React build   │
              │               │
              └───────┬───────┘
                      │
                      ▼
                Managed DB
            Azure SQL/Postgres
```

You could even have FastAPI serve the compiled React assets initially, giving you one origin and one deployed application.

That avoids prematurely creating:

```text
Static Web App
       │
       │ CORS
       │ tokens
       │ auth forwarding
       ▼
FastAPI App Service
       │
       ▼
database
```

A split frontend/backend architecture may eventually be desirable, but it introduces additional deployment and authentication boundaries. For the goal **“let Lisa/Gerry/Georges use Strata rather than watch my monitor,”** minimizing boundaries has considerable value.

That's an architectural recommendation I'd validate during your Azure spike rather than treating as predetermined.

---

## So was your timeline too optimistic?

I would divide your forecast into confidence bands.

**By ~September 1:**

```text
Debt Schedule                   high/moderate confidence
Complete forecast engine        high confidence
Basic CRUD                      high confidence
Contracts/validation            high confidence
Local end-to-end workflow       high confidence
Critical tests                  high confidence
```

That strikes me as a reasonable three-week target.

Then:

```text
Azure architecture research
costing
service selection
database strategy
authentication design
domain/configuration
migration plan
```

I would **not promise a duration yet**, which is exactly what you told them. You're going to investigate and return with the plan.

Excellent.

After approval, I'd establish a second milestone:

> **“Internal Strata Preview”**

not:

> **“Production Launch.”**

The acceptance criteria become something like:

```text
✓ Accessible through a Growlytics-controlled URL
✓ Growlytics Microsoft authentication
✓ Persistent cloud database
✓ Crop/QBO/Debt inputs editable
✓ Engine recomputes
✓ Forecast displays
✓ Invalid inputs rejected
✓ Multiple staff can access it
✓ failures logged sufficiently for Zhe to diagnose
✓ test data / sandbox data only
```

And importantly:

```text
NOT REQUIRED YET

× customer access
× perfect UI
× complete RBAC
× CI/CD sophistication
× Docker
× Kubernetes
× autoscaling
× multi-region anything
× perfect observability
× production SLA
× elaborate infrastructure-as-code
× complete test coverage
```

That boundary is what makes your plan realistic.

And there's something about your meeting statement that I actually like a lot: **you didn't commit to an Azure date when you don't yet understand the Azure problem well enough to estimate it.**

You said, essentially:

> I know the application work well enough to commit to three weeks.
> I don't yet know the deployment architecture well enough to commit.
> I will investigate services, requirements, configuration and costs, return with a proposal, and proceed after approval.

That's exactly the right epistemic boundary.

The only thing I would correct in your own mental model is that **the remaining work after the local application isn't merely cloud wiring**. Database persistence/migration, identity, configuration, deployment semantics, failure recovery, and environment separation will teach you quite a bit.

But that's also why this next phase is so valuable: Project Strata is about to force the transition from *“I can build software that works on my machine”* to *“I can operate a software system that other humans depend on.”* Those are different levels of engineering, and you now have a very natural reason to cross that boundary.

[1]: https://learn.microsoft.com/en-us/azure/app-service/quickstart-python?utm_source=chatgpt.com "Deploy a Python (Django, Flask, or FastAPI) web app to ..."
[2]: https://learn.microsoft.com/en-us/azure/app-service/configure-connect-to-azure-storage?utm_source=chatgpt.com "Mount Azure Storage as a local share in App Service"
[3]: https://learn.microsoft.com/en-us/azure/azure-sql/database/azure-sql-python-quickstart?view=azuresql&utm_source=chatgpt.com "Connect to and Query Azure SQL Database Using Python ..."
[4]: https://learn.microsoft.com/en-us/azure/app-service/tutorial-auth-aad?utm_source=chatgpt.com "Tutorial: Authenticate Users End-to-End - Azure App Service"

# Validation, Unit Tests, Authentication

I think the key realization is: **you did not actually promise them “Azure in three weeks.”** You promised that within roughly three weeks—around **September 1, 2026**—you would finish the forecast engine, input-management UI, and data-contract/guardrail layer, *then* investigate and propose the Azure deployment architecture, costs, configuration, authentication, etc. That is substantially more defensible than “the whole platform will be live in three weeks.”

And I actually think your instinct about what comes afterward is mostly right. The important adjustment I would make is **do not think of the first Azure deployment as “production.” Think of Azure itself as your first shared sandbox/staging environment.**

## Your three-week milestone is ambitious, but plausible

What you're really trying to complete locally is this:

```text
                         FORECAST SYSTEM
                               │
       ┌───────────────────────┼───────────────────────┐
       │                       │                       │
       ▼                       ▼                       ▼
Crop Plan Inputs         QBO Inputs             Debt Schedule
       │                       │                       │
       └───────────────────────┼───────────────────────┘
                               ▼
                    Monthly fan-out logic
                               │
                               ▼
                       Forecast Engine
                               │
                               ▼
                       Forecast Database
                               │
                               ▼
                        FastAPI / Queries
                               │
                               ▼
                       React Forecast UI


              INPUT MANAGEMENT WORKFLOW

                     React CRUD UI
                          │
           ┌──────────────┼──────────────┐
           │              │              │
         Create          Edit          Delete
           │              │              │
           └──────────────┼──────────────┘
                          ▼
                     FastAPI API
                          │
                          ▼
                Data contracts / validation
                          │
                          ▼
                     Input store
                          │
                          ▼
                  Recompute forecast
                          │
                          ▼
                   Updated frontend
```

Once that loop works, something very important has changed: **you no longer have a collection of components. You have an application.**

> user changes business assumptions → system validates them → state changes → engine recomputes → user sees new forecast.

CRUD is not a giant new conceptual subsystem compared with everything you've already built. You'll need to learn HTTP `POST`/`PUT` or `PATCH`/`DELETE`, React forms and controlled inputs, DB writes/transactions, validation, error handling, refresh/state synchronization, and some concurrency considerations. That's real work, but it's bounded.

So three weeks for **Debt Schedule + CRUD + contracts + complete local workflow** sounds aggressive but reasonable to me, assuming Debt Schedule itself doesn't reveal major modelling problems.

Where I would become much more cautious is saying:

> “And then deploying it to Azure is basically changing some environment variables and wiring.”

There are a few dragons hiding there.

---

## The cloud part is less about “orchestration” than **state**

Your React frontend and FastAPI backend are probably among the easier pieces. Azure App Service explicitly supports Python/FastAPI deployment, and application settings can be exposed to the running application as environment variables. Microsoft Entra authentication can also be put in front of App Service using its built-in authentication layer, so your idea of “only Growlytics Microsoft accounts can enter this thing” does **not** necessarily require you to implement an authentication system yourself. ([Microsoft Learn][1])

Your biggest architectural question is actually:

**What happens to SQLite?**

That's the part I would investigate before mentally calling Azure migration simple.

Right now:

```text
FastAPI
   │
   ▼
SQLite file
   │
   ▼
local disk
```

works beautifully because the process and database file live together.

Once you're in cloud infrastructure, especially once you ever want multiple application instances, deployments, restarts, persistent storage, concurrent writers, backups, etc., your assumptions change. Azure App Service can mount Azure Storage, but Azure Storage is a separate persistent storage service rather than simply “the application's disk.” ([Microsoft Learn][2])

This is exactly where your recent DDIA/database/concurrency learning starts becoming relevant.

I would therefore make **database deployment strategy one of the first questions in your Azure investigation**, probably considering something like Azure SQL Database or PostgreSQL rather than trying to make your production architecture revolve around a remotely mounted SQLite file. Azure SQL currently supports passwordless Microsoft Entra-based connections from Python as well. ([Microsoft Learn][3])

And that migration may expose SQL dialect differences, schema creation/migrations, transaction behavior, connection management, concurrency, data loading, configuration, etc.

That is the piece most likely to turn:

> “I'll wire it into Azure.”

into:

> “Oh. This is an actual software deployment architecture.”

Which is excellent for your engineering development—but something you should budget uncertainty for.

---

## I would slightly change your rollout order

You proposed something approximately like:

```text
complete engine
    ↓
CRUD
    ↓
contracts
    ↓
Azure
    ↓
sandbox
    ↓
tests
    ↓
CI/CD
    ↓
Docker
```

I'd instead make it:

```text
NOW
│
├── Debt Schedule
├── Complete forecast engine
├── CRUD UI/API
├── Data contracts / invariants
└── Small critical test suite
        │
        ▼
   COMPLETE LOCAL SYSTEM
        │
        ▼
AZURE ARCHITECTURE SPIKE
│
├── database choice
├── application hosting
├── Entra authentication
├── configuration / secrets
├── domain
├── estimated cost
└── deployment procedure
        │
        ▼
GREENLIGHT
        │
        ▼
AZURE STAGING / SANDBOX
        │
        ├── Growlytics staff log in
        ├── play with forecasts
        ├── edit inputs
        ├── deliberately break things
        └── give feedback
                │
                ▼
        ITERATE / HARDEN
        │
        ├── more tests
        ├── logging / observability
        ├── authorization
        ├── backups
        ├── migrations
        ├── CI/CD
        └── stronger operational safeguards
                │
                ▼
             PRODUCTION
```

That distinction between **Azure sandbox** and **Azure production** solves several problems at once.

You don't need this:

> current thing → immediately becomes production → later somehow add sandbox.

Instead:

```text
Local development
       ↓
Azure staging/sandbox  ← Growlytics starts using this
       ↓
Azure production       ← created when system deserves it
```

The first cloud environment is explicitly disposable/non-authoritative.

Someone screws up all the crop inputs?

Great.

Someone manages to put `-800000000` acres into Saskatchewan?

Great.

Someone double-clicks Update ten times?

Fantastic.

Someone refreshes halfway through a forecast recomputation?

Excellent.

**That's exactly what you want them doing.**

It turns stakeholder interaction from:

> Every three weeks Zhe shares his screen and everyone comments on what they see.

into:

> “Here's the URL. Here are your Growlytics credentials. Try to destroy it.”

That is a gigantic qualitative improvement in your feedback loop.

---

### I would move *some* testing before cloud deployment

Not exhaustive testing.

You absolutely do **not** need to disappear for six weeks building a glorious test suite before Gerry can touch anything.

But I wouldn't do:

```text
deployment
→ users
→ eventually unit tests
```

because some contracts are important enough that you don't want humans discovering their absence.

Before staging I'd want at least tests around things like:

```text
Crop area cannot be negative

allocation_pct ∈ [0, 1]

month ∈ [1, 12]

required foreign/reference keys exist

duplicate logical input keys rejected

revision relationships valid

Debt Schedule:
    opening balance
  + borrowing
  + accrued interest
  - principal repayments
  = closing balance

monthly fan-out sums back to annual amount

forecast recomputation deterministic

invalid update:
    does NOT partially modify database

failed engine run:
    does NOT leave corrupt forecast state
```

Those tests don't exist because "mature engineers write tests."

They exist because they encode **the mathematical laws of your forecast system**.

That's completely aligned with the way you've been thinking about contracts and invariants already.

---

## Docker is absolutely not required for your first deployment

This is another place where I think you're correctly sensing that you shouldn't turn every piece of engineering knowledge into a prerequisite.

Azure App Service can directly host Python/FastAPI applications. Microsoft also documents containerized FastAPI deployment to App Service, but containerization is an **option**, not a prerequisite. ([Microsoft Learn][1])

So don't create artificial dependencies:

```text
Must learn Docker
    ↓
Must Dockerize FastAPI
    ↓
Must create registry
    ↓
Must build container pipeline
    ↓
Must deploy container
    ↓
finally stakeholders can see forecast
```

when this works:

```text
FastAPI code
    ↓
Azure App Service
    ↓
stakeholders use application
```

Later you learn Docker and realize:

> Ah. Now I understand exactly what deployment/environment problem containers solve.

Which is much better than learning Docker because “cloud apps are supposed to have Docker.”

Same thing with Kubernetes: **nowhere near your critical path.**

---

## CI/CD is similar

Initially, this is perfectly respectable:

```text
develop
   ↓
tests
   ↓
git commit
   ↓
manual deployment
   ↓
staging
```

Then after you've manually deployed it several times, you are going to become irritated:

> Why am I repeatedly performing the exact same deterministic sequence?

And voilà:

```text
git push
   ↓
CI
   ├── install
   ├── lint
   ├── test
   └── build
   ↓
CD
   └── deploy staging
```

Now CI/CD answers a question you personally have rather than being another technology on a checklist.

Eventually:

```text
feature branch
      ↓
   tests
      ↓
     PR
      ↓
 merge main
      ↓
staging deployment
      ↓
validation
      ↓
production deployment
```

But there is no need for Project Strata to start there.

---

## Authentication may actually be easier than you're imagining

Because this is initially an **internal Growlytics application**, you've got a wonderful constraint:

> Only Growlytics Microsoft/Entra identities need access.

Azure App Service has built-in authentication/authorization support, including Microsoft Entra. That authentication layer can sit in front of the application rather than requiring your FastAPI application to become an identity provider. ([Microsoft Learn][4])

Conceptually:

```text
Lisa
 │
 ▼
forecast.growlytics.ca
 │
 ▼
Microsoft Entra login
 │
 ├── not Growlytics → NO
 │
 │
 └── Growlytics user → YES
                       │
                       ▼
                 Project Strata
```

Later your application-level authorization problem becomes richer:

```text
authentication
"Who are you?"

        ↓

authorization
"What can you do?"

admin
analyst
bookkeeper
client
read-only
etc.
```

But for the **internal sandbox**, you may initially be able to have:

```text
Growlytics employee → allowed
everyone else       → denied
```

That's an extremely useful simplification.

---

## I might simplify your first Azure architecture quite aggressively

For an internal preview, I wouldn't start with twelve Azure services.

Something in this conceptual neighborhood is enough:

```text
                   Internet
                      │
                      ▼
             Microsoft Entra ID
                      │
                      ▼
              Azure App Service
              ┌───────────────┐
              │               │
              │   FastAPI     │
              │      +        │
              │ React build   │
              │               │
              └───────┬───────┘
                      │
                      ▼
                Managed DB
            Azure SQL/Postgres
```

You could even have FastAPI serve the compiled React assets initially, giving you one origin and one deployed application.

That avoids prematurely creating:

```text
Static Web App
       │
       │ CORS
       │ tokens
       │ auth forwarding
       ▼
FastAPI App Service
       │
       ▼
database
```

A split frontend/backend architecture may eventually be desirable, but it introduces additional deployment and authentication boundaries. For the goal **“let Lisa/Gerry/Georges use Strata rather than watch my monitor,”** minimizing boundaries has considerable value.

That's an architectural recommendation I'd validate during your Azure spike rather than treating as predetermined.

---

## So was your timeline too optimistic?

I would divide your forecast into confidence bands.

**By ~September 1:**

```text
Debt Schedule                   high/moderate confidence
Complete forecast engine        high confidence
Basic CRUD                      high confidence
Contracts/validation            high confidence
Local end-to-end workflow       high confidence
Critical tests                  high confidence
```

That strikes me as a reasonable three-week target.

Then:

```text
Azure architecture research
costing
service selection
database strategy
authentication design
domain/configuration
migration plan
```

I would **not promise a duration yet**, which is exactly what you told them. You're going to investigate and return with the plan.

Excellent.

After approval, I'd establish a second milestone:

> **“Internal Strata Preview”**

not:

> **“Production Launch.”**

The acceptance criteria become something like:

```text
✓ Accessible through a Growlytics-controlled URL
✓ Growlytics Microsoft authentication
✓ Persistent cloud database
✓ Crop/QBO/Debt inputs editable
✓ Engine recomputes
✓ Forecast displays
✓ Invalid inputs rejected
✓ Multiple staff can access it
✓ failures logged sufficiently for Zhe to diagnose
✓ test data / sandbox data only
```

And importantly:

```text
NOT REQUIRED YET

× customer access
× perfect UI
× complete RBAC
× CI/CD sophistication
× Docker
× Kubernetes
× autoscaling
× multi-region anything
× perfect observability
× production SLA
× elaborate infrastructure-as-code
× complete test coverage
```

That boundary is what makes your plan realistic.

And there's something about your meeting statement that I actually like a lot: **you didn't commit to an Azure date when you don't yet understand the Azure problem well enough to estimate it.**

You said, essentially:

> I know the application work well enough to commit to three weeks.
> I don't yet know the deployment architecture well enough to commit.
> I will investigate services, requirements, configuration and costs, return with a proposal, and proceed after approval.

That's exactly the right epistemic boundary.

The only thing I would correct in your own mental model is that **the remaining work after the local application isn't merely cloud wiring**. Database persistence/migration, identity, configuration, deployment semantics, failure recovery, and environment separation will teach you quite a bit.

But that's also why this next phase is so valuable: Project Strata is about to force the transition from *“I can build software that works on my machine”* to *“I can operate a software system that other humans depend on.”* Those are different levels of engineering, and you now have a very natural reason to cross that boundary.

[1]: https://learn.microsoft.com/en-us/azure/app-service/quickstart-python?utm_source=chatgpt.com "Deploy a Python (Django, Flask, or FastAPI) web app to ..."
[2]: https://learn.microsoft.com/en-us/azure/app-service/configure-connect-to-azure-storage?utm_source=chatgpt.com "Mount Azure Storage as a local share in App Service"
[3]: https://learn.microsoft.com/en-us/azure/azure-sql/database/azure-sql-python-quickstart?view=azuresql&utm_source=chatgpt.com "Connect to and Query Azure SQL Database Using Python ..."
[4]: https://learn.microsoft.com/en-us/azure/app-service/tutorial-auth-aad?utm_source=chatgpt.com "Tutorial: Authenticate Users End-to-End - Azure App Service"


# Git, Serverless, Logging

Yes—your mental model is close, but you’re mixing **deployment workflow** with **runtime workflow**. Git is mostly about getting code *onto* Azure; it usually is not what orchestrates every user edit.

When you push code to GitHub, a deployment pipeline can build and deploy that version of your FastAPI/React application to Azure. Then Azure runs that deployed code in a Python runtime/container. App Service supports Python web apps directly, and Azure Functions supports Python too. ([Microsoft Learn][1])

The runtime flow you described would be something more like:

```text
                DEPLOYMENT FLOW

you change code
      ↓
git commit / push
      ↓
GitHub
      ↓
(optional later: CI tests)
      ↓
Azure deployment
      ↓
new application version running


                 RUNTIME FLOW

Lisa opens Strata
      ↓
React frontend loads
      ↓
Lisa edits input
      ↓
POST/PATCH /forecast-input
      ↓
FastAPI receives request
      ↓
validate proposed input
      ↓
transaction
      ↓
update input DB
      ↓
run forecast engine
      ↓
validate forecast output
      ↓
write new forecast
      ↓
COMMIT
      ↓
API responds "success"
      ↓
React fetches new forecast
      ↓
new numbers appear
```

So you don't necessarily configure Azure with a sequence like:

> “When database changes, execute `validate.py`; then run `forecast.py`; then execute `build_db.py`; then refresh React.”

**Your Python application itself can own that orchestration.**

For example, conceptually:

```python
def update_forecast_input(update):
    validate_input(update)

    with transaction():
        save_input(update)

        forecast = build_forecast()

        validate_forecast(forecast)

        save_forecast(forecast)

    return forecast
```

Your API endpoint might call that orchestration function:

```python
@app.patch("/inputs/{input_key}")
def update_input(...):
    result = forecast_service.update_input(...)
    return result
```

That's actually preferable architecturally because your **business workflow belongs to your application**, not to some collection of Azure checkboxes.

Azure supplies the machine/runtime/infrastructure. Your code defines what “update a forecast” means.

---

## But I would change one part of that workflow

You said:

> update database → validation → forecast engine → build outputs → validate → build database → refresh frontend

I'd prefer:

```text
user proposes edit
      ↓
validate BEFORE changing authoritative state
      ↓
start transaction
      ↓
apply update
      ↓
compute forecast
      ↓
validate output
      ↓
persist complete valid result
      ↓
commit
      ↓
tell frontend success
```

If anything fails:

```text
ROLLBACK
      ↓
return meaningful error
      ↓
frontend keeps showing last valid forecast
```

That's the contract philosophy we were just discussing.

---

## Your second question is more interesting: should Strata even run 24/7?

Possibly **not the computational engine**, but I don't think the entire Strata application can sleep whenever nobody is updating inputs.

Because think about this interaction:

```text
Gerry opens:
strata.growlytics.ca
```

Something needs to respond.

Even if nobody is changing the forecast, he still needs to:

```text
authenticate
load frontend
fetch current forecast
browse charts
inspect lineage
look at inputs
```

So there are actually two workloads:

```text
                         STRATA

           ┌───────────────┴────────────────┐
           │                                │
           ▼                                ▼
      SERVING WORKLOAD               COMPUTE WORKLOAD

"show me things"                  "change the forecast"

React/static files                validation
FastAPI GET requests              engine
database queries                  recalculation
authentication                    output generation

frequent / lightweight            occasional / heavier
```

And **those do not necessarily need the same execution model**.

That distinction is going to become important.

---

## Serverless could indeed fit the computation side

Azure Functions can be triggered by HTTP requests, queues, timers and other events, and Python is supported. Some Functions hosting plans dynamically scale and can scale toward zero when idle, which is where cold starts arise. Microsoft currently recommends Flex Consumption as its newer serverless consumption option; cold-start behavior is still something to account for. ([Microsoft Learn][2])

So conceptually you could eventually have:

```text
React
  │
  │ user clicks Update
  ▼
FastAPI
  │
  │ save/validate request
  ▼
Queue
  │
  ▼
Azure Function wakes up
  │
  ├── load inputs
  ├── run forecast
  ├── validate
  └── write outputs
  │
  ▼
sleep
```

That is genuine serverless/event-driven computation.

And yes, **Python works perfectly naturally in that model**. ([Microsoft Learn][2])

SQL is slightly different conceptually.

The SQL database isn't “Python code running serverlessly.” It is its own managed service:

```text
Python Function
      │
      │ SQL queries
      ▼
Azure SQL / PostgreSQL
```

The function wakes up, establishes a DB connection, reads/writes data, then terminates.

---

### But I would NOT start Strata this way

This is where I'd protect you from prematurely becoming clever.

You could build:

```text
React
 ↓
API gateway
 ↓
FastAPI
 ↓
queue
 ↓
Function
 ↓
database
 ↓
status tracking
 ↓
frontend polling
```

And then discover that you've created six new concepts just so Lisa can edit fertilizer costs.

For the first Azure sandbox, I strongly prefer:

```text
                App Service
           ┌──────────────────┐
           │                  │
           │     FastAPI      │
           │        +         │
           │  React frontend  │
           │        +         │
           │ forecast engine  │
           │                  │
           └────────┬─────────┘
                    │
                    ▼
               managed DB
```

And:

```text
PATCH /input
      ↓
FastAPI
      ↓
validate
      ↓
engine()
      ↓
validate
      ↓
database
      ↓
response
```

That's boring.

**Boring is fantastic right now.**

You understand almost every box already.

---

## Also: don't optimize too early around “24/7”

I understand why you're thinking:

> If people modify forecasts twice a day, why am I paying for a server for 24 hours?

That's a completely legitimate infrastructure question.

But first establish what the actual cost difference is.

For an internal application with a handful of Growlytics users, a small always-running App Service may be inexpensive enough that saving perhaps some infrastructure cost is not remotely worth adding event-driven architecture complexity.

That is exactly one of the questions for your Azure architecture/cost spike.

Compare:

```text
OPTION A
Small always-running web app
simple architecture
predictable latency
higher idle resource usage

OPTION B
Scale-to-zero/serverless
lower idle compute
cold starts
more execution boundaries
possibly more complicated orchestration
```

Don't assume B is superior just because the workload is intermittent.

---

## There's another important issue: how long does the engine take?

Suppose forecast recomputation takes:

```text
2 seconds
```

Then this is beautiful:

```text
PATCH
 ↓
calculate
 ↓
200 OK
```

Suppose eventually it takes:

```text
45 seconds
```

Still perhaps manageable, though unpleasant.

Suppose QBO + historical processing + benchmarking eventually means:

```text
8 minutes
```

Then **you should no longer tie computation to the HTTP request.**

App Service web requests are subject to request timeout limits; Microsoft documents roughly four-minute load-balancer limits for long-running requests, so lengthy computation is a strong reason to move it into an asynchronous/background job. ([Microsoft Learn][3])

Then you graduate to:

```text
user clicks Update
      ↓
POST /forecast-jobs
      ↓
202 Accepted

job_id = 9821
      ↓
queue
      ↓
worker/function computes
      ↓
database:
job 9821 = complete
      ↓
frontend asks:
GET /forecast-jobs/9821
      ↓
complete
      ↓
load new forecast
```

And the UI can say:

```text
Recalculating forecast...

✓ Forecast updated
```

That's a real reason to introduce background/event-driven architecture.

Not:

> “Serverless sounds cloud-like.”

---

## Cold start is exactly what you think it is

With scale-to-zero infrastructure:

```text
no requests
      ↓
0 running instances

...

request arrives
      ↓
Azure allocates worker
      ↓
Python runtime starts
      ↓
dependencies import
      ↓
your app/function initializes
      ↓
request executes
```

That startup delay is the **cold start**.

Azure Functions Flex Consumption has mechanisms aimed at reducing cold starts and can optionally keep always-ready instances, while Premium provides prewarmed instances specifically to avoid cold starts. ([Microsoft Learn][4])

For:

> “Lisa changed the crop plan and is waiting for the forecast.”

a few seconds of cold start might be perfectly acceptable.

For:

> “Gerry clicked between two dashboard pages.”

you probably don't want every page interaction waking the backend from death.

Another reason serving and computation may eventually diverge.

---

## Your third question: YES. Logging now becomes important.

And this is actually a very natural next engineering step for you.

What you currently have:

```python
print("building current production")
print("finished input costs")
print("writing orchestration")
```

is essentially primitive logging.

The Python `logging` module gives you structure around that.

Instead of:

```python
print("forecast complete")
```

you eventually want something conceptually like:

```text
2026-08-11T16:23:41
INFO
forecast_run_started
client_id=abc
revision=7
run_id=4fa81...

2026-08-11T16:23:42
INFO
input_validation_complete
records=142

2026-08-11T16:23:43
INFO
debt_schedule_complete
loans=6

2026-08-11T16:23:44
INFO
output_validation_complete
records=928

2026-08-11T16:23:44
INFO
forecast_run_committed
duration_ms=3241
```

Then if something explodes:

```text
2026-08-11T16:23:44
ERROR
forecast_invariant_failed

client_id=abc
revision=7
run_id=4fa81...

invariant=monthly_fanout_reconciliation
output_key=input-costs-wheat-7
annual_value=843120.00
monthly_total=843119.97
difference=-0.03
```

Suddenly debugging production becomes dramatically different.

---

### One concept I especially want you to learn here: correlation/run IDs

Imagine Lisa says:

> “I updated Client Smith around 2:13 and it failed.”

Your logs might contain 500 messages around 2:13.

If every forecast execution generates:

```python
run_id = uuid.uuid4()
```

and every log associated with that computation contains that ID:

```text
run_id=8fd932...
```

then you can query:

```text
show me everything involving run_id=8fd932
```

and reconstruct the entire execution:

```text
13:42:10 request received
13:42:10 validation started
13:42:10 validation passed
13:42:11 engine started
13:42:12 crop plan complete
13:42:13 QBO component complete
13:42:14 debt schedule FAILED
13:42:14 transaction rolled back
13:42:14 API returned 422
```

That's immensely valuable.

---

## And there are different logging levels for a reason

A useful initial mental model:

```text
DEBUG
"Tell me everything useful while developing."

INFO
"Normal important system events."

WARNING
"Something unusual happened,
but system still works."

ERROR
"This operation failed."

CRITICAL
"System itself may not be able to function."
```

Examples:

```python
logger.debug("Loaded %s input rows", len(inputs))

logger.info(
    "Forecast run started client=%s revision=%s",
    client_id,
    revision,
)

logger.warning(
    "Missing optional memo for input=%s",
    input_key,
)

logger.error(
    "Forecast validation failed",
    exc_info=True,
)
```

And absolutely avoid logging sensitive things such as:

```text
passwords
auth tokens
database credentials
secret keys
full private financial information unnecessarily
```

Logging itself becomes part of your security model.

---

## Azure can then collect these logs for you

Once deployed, you don't want to SSH into a machine and read a text file.

Azure Monitor/Application Insights can collect application telemetry. Microsoft's current Python guidance centers on OpenTelemetry through its Azure Monitor OpenTelemetry distribution, which can collect logs, traces, metrics, and request-related telemetry. ([Microsoft Learn][5])

Eventually the observability picture becomes:

```text
                    STRATA
                      │
             ┌────────┼─────────┐
             │        │         │
            logs    traces    metrics
             │        │         │
             └────────┼─────────┘
                      ▼
              Application Insights
                      │
          ┌───────────┼───────────┐
          ▼           ▼           ▼
       errors      latency    requests
       search      trends      failures
```

Then you can answer questions like:

```text
How many forecast computations failed today?

Which endpoint is slow?

What exception occurred?

What happened during run_id X?

How long does Debt Schedule take?

Are forecast runs getting slower?

Which invariant fails most frequently?
```

That's what **observability** starts meaning.

Logging is one component of it.

---

## I think there is a very clean evolution path for you

**Stage 1 — now**

```text
Python logging
      ↓
console

structured messages
timestamps
levels
run_id
client/revision identifiers
exceptions
```

Don't overengineer it.

Then **Stage 2 — initial Azure sandbox**:

```text
Python logging
      ↓
Azure Application Insights / Monitor
```

Microsoft provides OpenTelemetry-based instrumentation for Python applications, so you don't need to invent a logging storage system yourself. ([Microsoft Learn][6])

Then much later:

```text
logs
traces
metrics
alerts
dashboards
```

as operational needs appear.

---

### And I think the largest conceptual correction from your three questions is this

You currently picture the cloud roughly as:

```text
Azure
   │
   └── complicated machinery that orchestrates
       my Python scripts
```

I'd change that mental model to:

```text
                     AZURE

       gives your software somewhere to live
       gives it compute
       gives it identity
       gives it networking
       gives it persistent services
       gives it observability

                       │
                       ▼

                 YOUR APPLICATION

       decides what the business process means:

       update input
            ↓
       validate
            ↓
       compute
            ↓
       validate
            ↓
       persist
            ↓
       return result
```

And Git sits largely **above that**:

```text
                    DEVELOPMENT

you
 │
 ▼
Git
 │
 ▼
deployment mechanism
 │
 ▼
Azure
 │
 ▼
running Strata


                     RUNTIME

user
 │
 ▼
running Strata
 │
 ▼
database / engine
```

Those are two completely different lifecycles.

Once you see that separation, cloud deployment becomes much less mystical. You're not transforming your Python application into “cloud code.” You're deciding **what process runs your code, what wakes it up, what persistent state it talks to, how requests reach it, and how you observe what happened.**

For your first internal Strata deployment, I would keep that answer deliberately boring: **one small FastAPI application, one managed database, React served with it or alongside it, Entra authentication, Python structured logging + Application Insights, synchronous forecast recomputation as long as recomputation remains comfortably short.** Only introduce Functions/queues/background jobs when the engine runtime or cost profile gives you an actual reason to. ([Microsoft Learn][7])

That architecture is simple enough for you to understand completely—which, at this stage, is a much more valuable property than being maximally “cloud-native.”

[1]: https://learn.microsoft.com/en-us/azure/app-service/configure-language-python?utm_source=chatgpt.com "Configure Linux Python Apps - Azure App Service"
[2]: https://learn.microsoft.com/en-us/azure/azure-functions/functions-scale?utm_source=chatgpt.com "Azure Functions Scale and Hosting"
[3]: https://learn.microsoft.com/en-us/troubleshoot/azure/app-service/web-request-times-out-app-service?utm_source=chatgpt.com "Web Request Times Out In App Service - Azure"
[4]: https://learn.microsoft.com/en-us/azure/azure-functions/flex-consumption-plan?utm_source=chatgpt.com "Azure Functions Flex Consumption plan hosting"
[5]: https://learn.microsoft.com/en-us/azure/azure-monitor/app/opentelemetry-enable?utm_source=chatgpt.com "Enable OpenTelemetry in Application Insights - Azure"
[6]: https://learn.microsoft.com/en-us/azure/azure-monitor/app/opentelemetry-configuration?utm_source=chatgpt.com "Configure Azure Monitor OpenTelemetry"
[7]: https://learn.microsoft.com/en-us/azure/azure-monitor/app/app-insights-overview?utm_source=chatgpt.com "Application Insights OpenTelemetry observability overview"
