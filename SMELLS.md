# reservation-service: Smells and One Fix

Fill in each section. One section per milestone. Keep it short and specific. Point at files
and methods, not adjectives.

---

## Milestone 1: Three smells

Three smells, each in a different part of the module. For each one, fill in all five parts.

### Smell 1

**The smell.** Duplication over reuse: pricing exists twice 

**Classic or agent-specific.** agent-specific, caused by missing context

**Where in the code.** ReservationManager.calculatePrice/applyDiscounts and ReportGenerator.priceOf implement the same pricing. They even have parallel constant sets under different names: PREMIUM_MULTIPLIER vs PREMIUM_RATE_MULTIPLIER, LONG_BOOKING_MINUTES vs LONG_BOOKING_CUTOFF, and so on.

**The principle it violates.** a single source of truth. Pricing knowledge should live in one place.

**What it makes expensive.** changing the evening discount means editing both copies. If someone edits only one, revenue() silently stops matching what customers were charged. No test catches that today, because the reporting test only checks that revenue equals the stored priceCents under the current rules.

### Smell 2

**The smell.** God class

**Classic or agent-specific.** classic

**Where in the code.** ReservationManager. Its own docstring admits it does everything: room registry, booking lifecycle, conflict checking, pricing, notification dispatch, receipt formatting, and daily summaries.
 
**The principle it violates.** cohesion (one reason to change).

**What it makes expensive.** a receipt wording change, a pricing change, and a notification change all land in the same 229-line file. That is divergent change.

### Smell 3

**The smell.** Speculative generality / phantom complexity

**Classic or agent-specific.** agent-specific, caused by an underspecified request plus free volume

**Where in the code.** src/cache/

**The principle it violates.** listBookingsForRoom reads from QueryCache, but nothing anywhere calls cache.set, so the cache branch never fires. It is complexity with no function. The notifier registry is a second example: notifierFactory.ts is a plugin registry where ChannelName can only ever be 'email'.

**What it makes expensive.** if someone "turns the cache on" by adding a set, cancelBooking never invalidates it, so formatDailySummary would show cancelled bookings as confirmed for 30 seconds. The cache also hides a dependency on Date.now().

---

## Milestone 2: One small fix

One fix, behavior preserved, suite green, zero test edits.

**Which smell you attacked.** I'd fix smell 1: it is the easiest to fix and fixing it now will save a lot of debugging time moving forward

**What changed.** extract a pure priceBooking(room, start, end) into a new src/pricing.ts with one set of constants, and have both classes call it.

**What you deliberately did not touch.** don't change whether revenue() recomputes prices or uses the stored booking.priceCents. That is a real design question (should a rate change rewrite past revenue?), and answering it would change behavior. Also leave calculatePrice as a public method on the manager, delegating to the new function, because the tests call through the manager.

**How you know behavior is preserved.** the four pricing tests in booking.test.ts pin base, long, premium, and evening prices, and the revenue tests cross-check against them.

---

## Milestone 3: Two proposals and one false positive

One proposal for each milestone 1 smell you did not fix.

### Proposal A (not coded)

**The problem.** Name it.

**The decomposition.** What are the pieces, what does each own, and where do the rules live?

**One cost.** Something this actually costs. "No real downside" is not a cost.

### Proposal B (not coded)

**The problem.**

**The decomposition.**

**One cost.**

### The thing that looks smelly but is fine

**What it is.** File and method.

**Why it is fine.** Defend it with properties of the code, not with its line count.

**What would flip your verdict.** Name the change that would turn this into a real problem.
