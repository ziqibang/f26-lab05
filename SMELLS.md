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

**The problem.** God class (smell 2): `ReservationManager` in `src/reservationManager.ts`. It has several unrelated reasons to change. It holds the room registry (`registerRoom`/`getRoom`/`listRooms`), runs the booking lifecycle (`createBooking`, `cancelBooking`), and owns the overlap rule (`hasConflict`). It also sends notifications (`dispatchNotification`, `recentNotifications`) and does presentation (`formatReceipt`, `formatDailySummary`, `formatClock`, `formatMoney`). On top of that, it builds its own notifier and cache inside the constructor instead of taking them as parameters.

**The decomposition.**
- `src/rooms.ts`: a `RoomRegistry` class that owns the `Map<string, Room>` and `register`/`get`/`list`.
- `src/availability.ts`: owns the overlap rule. `ReservationManager.hasConflict` and `ReportGenerator.overlapsWindow` both use half-open overlap, and `isSlotFree` is its negation. `hasConflict` moves here as `overlaps(aStart, aEnd, bStart, bEnd)`. (`overlapsWindow` is the same rule written a second way, and it could move here later too.)
- `src/formatting.ts`: pure `formatClock`, `formatMoney`, `formatReceipt(booking, roomName)`, and `formatDailySummary(roomName, bookings)`. They have no state and take no storage.
- `src/bookingNotifier.ts`: a `BookingNotifier` that is given a `NotificationChannel`. It owns `notificationLog`, and its `notify(booking, subject, roomName)` calls `formatReceipt`.
- `ReservationManager` stays as a thin facade that coordinates the use cases. `createBooking` does lookup room → `validateReservationRequest` → `overlaps` → `priceBooking` → `storage.save` → `notifier.notify`. The constructor becomes `(storage, notifier = new BookingNotifier(new EmailChannel()))`. Pricing already lives in `src/pricing.ts` since milestone 2.

Callers don't change. `tests/fixtures.ts` `newService()` still calls `new ReservationManager(storage)` and `registerRoom`, and the tests still call `createBooking`, `listBookingsForRoom`, `formatDailySummary`, and `listRooms` on the manager, which now delegate.

**One cost.** The extraction would move code the suite doesn't check. No test calls `formatReceipt` or `recentNotifications` or looks at what the notifier sends. The only formatting test is one `toContain` check on `formatDailySummary` in `booking.test.ts`. So moving receipt formatting and notification dispatch into new modules happens with no net. For example, a changed receipt line, or a notification sent before `storage.save` instead of after, would still leave the suite green. We would have to write characterization tests for the receipt text and the notification log first, which makes the refactor bigger than it looks.

### Proposal B (not coded)

**The problem.** Speculative generality / phantom complexity (smell 3), in two places:
- `src/cache/` (`QueryCache`, `CacheConfig`, `withTtl`, `disabled`). `ReservationManager.listBookingsForRoom` calls `this.cache.get`, but nothing in `src/` or `tests/` ever calls `set`, `invalidate`, `withTtl`, or `disabled`, so the cache branch is dead.
- `src/notifications/notifierFactory.ts` (`registerChannel`, `registeredChannels`, `createNotificationChannel`). It is a mutable module-level registry keyed by `ChannelName`, which is the one-member union `'email'`.

Both are built inside the `ReservationManager` constructor, so they are also hidden dependencies. `QueryCache` reads `Date.now()`, and a test can't swap in a different channel.

**The decomposition.**
- Delete `src/cache/`. `listBookingsForRoom` becomes `return this.storage.findByRoom(roomId);`.
- Delete `notifierFactory.ts`. `ReservationManager`'s constructor takes the channel directly: `constructor(storage: StorageProvider = new InMemoryStorageProvider(), notifier: NotificationChannel = new EmailChannel())`. `EmailChannel`'s default from-address is already `'reservations@example.edu'`, the same as `DEFAULT_NOTIFIER_CONFIG`. Keep `channel.ts`, because the `NotificationChannel` interface is the real seam: tests can pass in a recording fake.
- The rules live in one place each afterward. Which channel to use is decided by whoever constructs the manager, not by a global registry. If caching is ever actually needed, it goes behind the existing seam as a `CachingStorageProvider implements StorageProvider` that invalidates in its own `save`/`update` and takes a clock as a parameter. That way invalidation sits next to the writes and can't be forgotten in `cancelBooking`.
- `tests/fixtures.ts` needs no change, because the new constructor parameter is optional.

**One cost.** This removes exported API: `QueryCache`, `withTtl`, `disabled`, `registerChannel`, `registeredChannels`, `createNotificationChannel`, and `DEFAULT_NOTIFIER_CONFIG`. The repo contains no callers of them, but the repo can't tell us whether code outside this module imports them. If something does, the deletion breaks it at compile time. We would have to check with the module's consumers, or keep deprecated shims for a while, which is itself more of the volume we're trying to remove.

### The thing that looks smelly but is fine

**What it is.** `validateReservationRequest` in `src/validation.ts`. It looks like a long method: a run of `if` checks split into commented sections (shape, times, duration, capacity, building rules).

**Why it is fine.** It has one responsibility: deciding whether a request is valid for a room. Its structure is a flat, ordered list of independent early-return guards:
- **No shared state:** it reads only its two arguments, `request` and `room`, plus module constants.
- **No side effects:** it doesn't touch storage, the clock, or the notifier.
- **Each guard stands alone:** it checks one rule and returns `{ valid: false, reason }`.

That structure is what gives callers exactly one clear reason for a rejection. `ReservationManager.createBooking` throws `validation.reason` as-is, and `tests/validation.test.ts` pins the specific reason for each case. The order is part of the contract. For example, `{ start: 1380, end: 1500 }` also breaks opening hours, but the test expects 'start and end must fall inside a single day' because that guard runs first. Splitting the function into helpers would add indirection without separating any responsibilities.

**What would flip your verdict.**
- **Rules varying by building or room type.** The "Booking rules for this building" section already hard-codes one building's opening hours and 15-minute boundaries. The function would grow `if building === ...` branches, and you'd want per-building policy objects.
- **A check needing storage**, such as a per-organizer booking limit. That would give the function a dependency and a second job.
