## 1. Product definition layer

Before engineering, you need clarity on:
- who the users are
- what each user type can see
- what actions they take
- what decisions they make
- what must be real-time vs delayed
- what is static vs interactive
- what is informational vs transactional

**Roadblock**

If this is fuzzy, engineering thrashes.

Example:
- “dashboard” is vague
- “scenario explorer” is vague
- “customized client experience” is vague

These must become:
- exact page purposes
- exact user actions
- exact outputs

## 2. Identity, auth, and client isolation

This is one of the biggest hidden mountains.

You need to answer:
- how users log in
- whether you reuse existing website auth
- how the system identifies which client they belong to
- how to prevent one client from seeing another client’s data
- what internal users can see vs external users
- how sessions expire
- password reset / MFA / account lifecycle
- audit trail for access

**Roadblock**
- It is easy to make a beautiful UI.
- It is hard to make it safe.

This is where many “just build the website” ideas fall apart.

## 3. Data ingestion / ETL / normalization layer

This is your native zone, and it is still a giant.

You need to solve:
- source systems
- file formats
- Excel quirks
- manual uploads vs automated feeds
- schema drift
- business logic extraction
- benchmarking definitions
- deduplication
- refresh frequency
- error handling
- reprocessing and replay
- lineage and traceability

**Roadblock**

The frontend experience depends on stable, trustworthy, structured data.  
If this layer is weak, the UI becomes a lie.

## 4. Business logic / metrics layer

This is separate from ETL.

You need a place where logic lives, such as:
- ratios
- percentile rankings
- scenario formulas
- benchmark comparisons
- insurance calculations
- inventory metrics
- derived KPIs

**Roadblock**

If logic is buried in:
- Excel formulas
- frontend code
- BI visuals

then the system becomes unmaintainable.

You need a **reusable logic layer**.

## 5. Application backend / API layer

Now we get to the serving layer.

The UI cannot usually talk directly to raw tables.  
You need a backend that answers:
- what pages need what data
- how filters are applied
- how scenario inputs are processed
- how permissions are enforced
- how data is shaped for the frontend
- what is cached vs computed on demand

This could expose:
- REST APIs
- GraphQL APIs
- server-rendered endpoints
- embedded BI tokens
- custom scenario computation services

**Roadblock**

This is where the product becomes a real application, not just dashboards.

And yes, GraphQL is possible, but it is not automatically the answer. It helps when:
- the frontend needs flexible data shapes
- many different pages need overlapping data
- you want frontend-friendly querying

But it also adds complexity:
- schema design
- resolver performance
- authorization rules
- caching challenges

So GraphQL is a tool, not a solution.

## 6. Frontend application layer

Now the actual website experience.

You need to build:
- navigation
- layout
- state management
- page routing
- filters
- drill-down interactions
- scenario controls
- loading/error states
- responsive behavior
- chart rendering
- user feedback

**Roadblock**

Figma gives you static clarity.  
Frontend engineering must handle:
- changing data
- latency
- permissions
- edge cases
- broken states

This is where “Apple-like simplicity” is expensive:
> simple-looking products often require more engineering discipline, not less.

## 7. Visualization layer

This is bigger than people think.

You need to decide:
- custom charts in frontend
- embedded Power BI
- hybrid approach
- static KPI cards vs interactive visuals
- drill interactions
- export behavior
- mobile handling
- performance with large data

**Options at a high level**

You could:
- build charts directly in the frontend with chart libraries
- embed BI artifacts
- mix both

**Roadblock**

Each choice has tradeoffs:

Custom charts
- maximum UX flexibility
- more engineering cost

Embedded BI
- faster analytics delivery
- less UX control
- auth/embedding complexity

This is a huge scoping axis.

## 8. Scenario engine / simulation layer

This is the flashy part, but also one of the hardest.

If users change:
- yield
- price
- cost assumptions
- insurance choices

then the system must:
- recompute outputs
- preserve logic consistency
- show updated KPIs
- possibly explain changes

**Roadblock**

This is not just sliders.  
This is a **calculation engine**.

Questions:
- where does scenario logic run?
- client-side?
- backend service?
- precomputed assumptions?
- what is the latency target?
- how do you prevent inconsistent states?

This is a major subsystem by itself.

## 9. Hosting / deployment / infrastructure

Yes, this matters.

You need a destination for:
- frontend app hosting
- backend/API hosting
- databases / warehouses
- auth integration
- secrets management
- scheduled jobs / ETL runners
- file storage
- monitoring
- backups

**Roadblock**

People say “put it on Azure” or similar as if that solves it.  
It does not.

Hosting is not one decision.  
It is a bundle of operational decisions.

## 10. Reliability / observability / operations

Once real clients use it, you need:
- monitoring
- logs
- alerting
- refresh failure visibility
- data quality checks
- broken pipeline handling
- support process
- rollback process
- SLA expectations

**Roadblock**

Without this, the system works only when you are personally babysitting it.

That is not a product.  
That is a fragile demo.

## 11. Multi-tenant architecture

This is a giant hidden problem if multiple clients use the product.

You need to decide:
- single shared database vs tenant partitioning
- shared app with row isolation vs stronger separation
- client-specific configurations
- branding differences
- package/feature entitlements
- onboarding new clients
- offboarding clients

**Roadblock**

The moment each client has:
- different data
- different products
- different access
- different views

you are no longer building one app.  
You are building a **multi-tenant configurable platform**.

That is much deeper.

## 12. Content / explanation / trust layer

If the product explains results, you need:
- metric definitions
- “how this is calculated”
- assumptions
- data freshness labels
- warnings about estimate quality
- benchmark methodology explanations

**Roadblock**

Without this, clients may see numbers but not trust them.

And your own intuition already knows:
> product without trust collapses.

## 13. Change management / iteration strategy

Even if the engineering is possible, you still need:
- phased roadmap
- MVP definition
- what gets deferred
- feedback loop from users
- UX validation
- feature requests process

**Roadblock**

If everything is treated as must-have from day one, the project collapses under scope.