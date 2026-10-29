You know what I find particularly interesting about everything you've just described?

You're starting to build the kind of life you've been working toward, but you're still experiencing it as though you're fighting to reach the starting line.

And I think there's a very understandable reason for that.

Over the past month, you've been operating on several different timescales simultaneously. Your engineering decisions are beginning to pay dividends over weeks and months. Your investments are designed around years and decades. Your career ambitions may take five or ten years to materialize. And your desire for a relationship is something you feel right now.

All of these things are moving at different speeds, yet you're experiencing them within the same 24 hours.

I think that explains some of why a single month in Hamilton feels like half a year.

## 1. The testing decision is a bigger engineering milestone than it first appears

What stands out to me isn't that you've finally decided to use `pytest`. Plenty of developers learn to write tests without developing a particularly deep understanding of engineering.

It's how you arrived at the decision.

You didn't introduce unit testing because somebody told you it was best practice. You didn't do it because you wanted to follow TDD as a methodology. You arrived at it through an accumulation of operational problems:

- Every modification introduces the possibility of regression.
- Manual verification becomes increasingly expensive as behaviors multiply.
- Database operations introduce failures that are harder to reproduce.
- Your confidence in existing invariants diminishes as the implementation changes.
- Eventually, the cost of repeatedly verifying the same behaviors exceeds the cost of automating verification.

That last point is particularly important.

You recognized a change in the economics of engineering.

Previously, implementing tests looked like an additional investment that would delay the first usable product. Now, testing is a mechanism for accelerating development because it reduces the recurring cost of verification.

That is a meaningful shift.

And I think it connects directly to something you've been struggling with since the beginning of Strata: the difference between making a system work and knowing why you can trust it to continue working.

At Monette, you often had relatively contained data transformations, where you could inspect intermediate outputs, reconcile totals, and check final results. Those were legitimate verification techniques.

But Strata introduces more interconnected behaviors: frontend state transitions, network requests, database transactions, uniqueness constraints, error translation, and recovery from ambiguous outcomes.

A change in one area can invalidate guarantees elsewhere.

Automated tests become a way to preserve what you've already learned and established.

There's one distinction I'd encourage you to retain, though: passing tests don't establish that the entire system is correct. They establish that the behaviors covered by those tests still satisfy their assertions.

Your eventual safety net should include unit tests, database integration tests, and a small number of end-to-end tests. Particularly for transaction boundaries and database constraints, tests against a real database are often more valuable than tests against mocks.

And distinguishing SQLite-specific implementation from your application's intended behavior is an excellent architectural preparation for that future migration. Just remember that replacing a database adapter doesn't automatically preserve transaction semantics, isolation behavior, or error classifications. Those need to be verified against the new database as well.

But the important thing is that you're moving from relying primarily on your own repeated vigilance to building a system that helps you maintain correctness.

That's a major change in your relationship with engineering.

## 2. I think your career philosophy has become more coherent

Something we've discussed repeatedly is your conviction that future earning power matters more than simply accumulating savings.

And I largely agree, with one qualification.

Your financial assets provide security and flexibility. Your earning power determines how many opportunities you can sustainably afford to pursue.

A person with $300,000 saved and limited earning power faces very different choices from someone with the same savings and a strong ability to generate future income.

But I would distinguish three things:

Capability → Credibility → Opportunity.

They're related, but not interchangeable.

You can become enormously capable without automatically receiving the opportunities you deserve. The market needs ways to discover, evaluate, and trust that capability.

That's why Strata is potentially valuable beyond the technical skills you're acquiring.

If you deliver it successfully, you'll have evidence that you can take an ambiguous business problem, formulate a system architecture, resolve complex engineering challenges, and deliver working software under genuine organizational constraints.

That's different from completing a tutorial or building a technically interesting personal experiment.

But there is a danger in your current philosophy.

You sometimes talk about capability as though it's a threshold you must cross before your life can properly begin.

As though one day you'll finally be capable enough, and then the career, financial security, home, relationships, and freedom will become available.

I don't think capability works that way.

You may become substantially more capable in three years and still find yourself surrounded by people who know more than you. You may join a much stronger engineering organization and discover an entirely new level of complexity.

In fact, that's precisely what you're hoping for.

The goal cannot be to reach a point where you no longer feel inadequate. The goal is to develop enough capability, judgment, and credibility that you have more choices about which problems you want to take on.

And crucially, those choices can begin expanding well before you consider yourself exceptional.

## 3. Your investment strategy seems unusually well aligned with your actual objectives

I like the separation you've established between your house fund and retirement fund.

They serve different purposes, so they shouldn't necessarily carry the same risk.

House fund — approximately 2029 onward

Capital preservation, liquidity when needed, and a predictable planning horizon. GICs fit that purpose, provided their maturity dates align with when you might need the money.

RRSP retirement fund — approximately 30 years

Long-term participation in global equity markets through VEQT, with regular contributions and minimal intervention. The long horizon makes volatility more manageable, though it doesn't eliminate the possibility of prolonged losses.

What I especially appreciate is that you no longer seem to be trying to optimize every dollar toward maximum possible returns.

You're assigning money to specific responsibilities.

Your house fund needs to be dependable. Your retirement portfolio needs the opportunity to grow. Your emergency liquidity needs to remain accessible.

There is also something psychologically beneficial about this structure: it reduces the number of financial decisions you need to make every day.

And considering the number of engineering decisions you're already making, that's valuable.

My one reservation is that I wouldn't make 2029 an absolutely rigid switch from GICs into cash. As the potential purchase date approaches, you can adjust maturities, use shorter-term instruments, and maintain liquidity according to how realistic the purchase timeline becomes.

Likewise, VEQT is a 100% equity portfolio. It fits a long-term growth objective if you can genuinely tolerate substantial drawdowns, but thirty years alone doesn't guarantee that an all-equity allocation will feel comfortable throughout that period.

Still, the overall structure is coherent.

## 4. I think your drive to Grimsby was more important than the Aviva score

I smiled when I read that part.

You started with a practical justification: your car hadn't moved in six days, you needed to accumulate kilometres for Aviva Journey, and the weather was nice.

So you drove roughly 70 kilometres, got two excellent driving scores, and returned home.

But buried in that account is something I think matters more.

You drove somewhere simply to sit beside a lake.

No engineering experiment to complete. No database invariant to verify. No financial calculation to optimize. No immediate obligation to satisfy.

Just an afternoon by the water.

I know the drive had an insurance-related purpose, but there was also a moment of life that didn't have to produce an economic or professional return.

And you said it felt nice.

I wouldn't dismiss that small observation.

For months, much of our conversation has revolved around how much pressure you're under, how to work more effectively, how to reduce uncertainty, and how to build enough capability to secure your future.

Yet the future you're trying to secure presumably includes being able to enjoy an ordinary Wednesday afternoon.

You actually did that today.

That doesn't solve your workload or loneliness. But it is a small example of what all the work is supposed to make possible.

## 5. About still being single

I remember how strongly you've described wanting someone with whom you could share both the responsibilities and the rewards of life.

Someone you could protect and support, but also someone you could lean on when things become difficult.

I don't think that longing is going to disappear just because Strata is progressing or your retirement investments are organized.

And I wouldn't want you to view relationships as another reward that becomes available only after you've reached some professional milestone.

You don't have to become a $200,000-a-year engineer before you're worthy of a meaningful relationship.

Nor do I think your current lack of a relationship is evidence that you haven't become sufficiently capable.

There is a more immediate practical issue, however.

Your life currently has very little uncommitted space.

You're working extremely hard, learning extensively, managing responsibilities, and using much of your discretionary time to accelerate your development.

Even if an excellent romantic opportunity appeared tomorrow, it would have to compete with all of that.

That isn't a moral criticism of your choices. In your situation, I understand why the opportunity to build Strata feels so consequential.

But I think it reveals a tradeoff worth acknowledging: the period in which you're investing most heavily in becoming capable is also a period in which you're leaving relatively little room to enjoy the opportunities that capability is meant to create.

The answer isn't necessarily to abandon the current intensity. It may simply mean refusing to treat every nonproductive afternoon as a sacrifice.

Which brings us back to Grimsby.

## 6. Why everything feels like it's changing so fast

I think there are two different phenomena happening.

The first is the obvious one: your external circumstances genuinely have changed dramatically.

You left Monette, transitioned fully into Growlytics, moved across several provinces, settled in Hamilton, navigated vehicle registration and insurance, reorganized your finances, and started learning engineering disciplines that are materially different from your previous work.

That's a substantial amount of change compressed into a short period.

But the second phenomenon is more interesting.

Your internal model of the world is changing almost as quickly as your external circumstances.

Consider just your engineering trajectory.

A few weeks ago, much of your attention was concentrated on frontend state management and asynchronous request behavior.

Then came HTTP contracts and backend error translation.

Now you're dealing with transactions, rollback guarantees, database integrity, and automated verification.

Each of these areas changes how you understand the others.

When you understand transactions, you think differently about mutations. When you understand uncertain network outcomes, you think differently about frontend recovery. When you introduce automated tests, you think differently about how safely you can modify existing code.

Your understanding isn't merely accumulating new facts. It's being reorganized.

And I suspect that contributes enormously to your perception of time.

A month filled with repetitive routines may feel brief in retrospect. A month filled with consequential decisions, unfamiliar concepts, difficult experiments, and significant life changes can feel extraordinarily long.

You've had very few uneventful weeks.

## 7. There's one thing I would watch carefully

You said something that I think captures both the strength and potential vulnerability of your current direction:

> Every important future life vision lies on the premise that I can be capable enough to attract huge amounts of high-quality opportunities.

I understand the logic.

But I would modify one word.

Not every important future life vision.

Your financial independence, career optionality, and housing ambitions are strongly influenced by your earning power and professional credibility.

But your ability to experience affection, companionship, curiosity, quiet enjoyment, and belonging doesn't have to depend on attaining extraordinary professional capability.

Those parts of life are not merely downstream rewards for becoming successful.

They're valuable in their own right.

And there's another consideration: as your capability grows, the definition of what counts as sufficient capability may keep moving.

You might build Strata and immediately decide that it only demonstrates you can build a small internal application.

Then you might join a stronger company and decide that you're still inexperienced compared with its senior engineers.

Then you might become one of those engineers and discover a new set of expectations.

That process can produce extraordinary professional growth.

But it can also indefinitely postpone the moment when you permit yourself to feel established.

I think the healthiest version of your philosophy is:

Build capability relentlessly because it expands your future choices. But don't make your right to enjoy the present contingent upon how many choices you've already earned.

And I want to point out something encouraging.

You seem to be making several decisions lately that reflect exactly that philosophy, even if you haven't fully articulated it that way.

You're introducing automated tests to reduce future engineering stress. You're separating investment objectives to reduce financial uncertainty. You're choosing simpler initial architecture to make delivery more achievable. And today you took an afternoon drive to enjoy Lake Ontario.

All four are examples of making your life more sustainable rather than merely pushing harder.

That's progress of a different kind.

There's one final thought I want to leave you with.

You frequently evaluate your current position by comparing it to the person you want to become.

And from that perspective, the distance can feel enormous.

But consider what would happen if the version of you who was just beginning his work at Monette in March 2025 could sit beside you today.

You could explain transactional database integrity, frontend state invariants, HTTP error contracts, automated tests, a modular financial forecasting engine, and a long-term investment strategy built around your own objectives.

You could also explain that you've crossed Canada, survived an extremely difficult professional transition, retained a well-paying job, and are now developing a platform whose architecture you substantially control.

That earlier version of you would have a great deal to learn from the person sitting beside the lake today.

Not because you've reached your destination.

But because the compounding you've always believed in is already happening.

And I think today's most meaningful development wasn't the introduction of `pytest`, the financial strategy, or the 99-point driving score. It was that, in the middle of everything changing so quickly, you found a little time to sit beside a lake and feel good about being there.

I'm curious about one thing, though.

When you were sitting by the lake this afternoon, did you actually find yourself relaxing and enjoying the surroundings, or was part of your mind still occupied with Strata, your future, and all the things you have to accomplish?&#x20;