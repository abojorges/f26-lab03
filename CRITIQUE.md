# RoomReserve Critique

RoomReserve as the code actually builds it, where that differs from `DESIGN.md`, and what
I would change instead. Every claim below was checked by reading the source and by running
the compiled code.

---

## Milestone 1: The design as it is

**Data model.** A booking is not a type — there is no `Booking` class. It is an agreement
between two maps in `InMemoryStore` (lines 13-14). One maps a `"room|date"` string to a
list of two-element arrays holding a start and an end in minutes since midnight; the other
maps a longer `"room|date|start|end"` string to the person who booked it. `DESIGN.md`
documents only the first, so the second map — and the work of keeping the two in step — is
invisible in the design. Four things have to stay true for a booking to make sense, and
nothing in the code enforces any of them: the two maps agree, or `listBookings` prints
`(null)` where the owner should be; the key string is spelled identically at all seven
places it gets assembled; the array holds the start first and the end second, by
convention alone; and no two intervals in a list overlap. Rooms and dates are plain
strings that are never examined, so `"banana"` is an acceptable date.

**Operations.** A caller can create, cancel, reschedule, and list bookings, all through
`RequestHandler`, and everything crosses the boundary as strings. `createBooking` takes a
room, date, start, end, and user. `cancelBooking` and `rescheduleBooking` identify an
existing booking by its exact start and end. `listBookings` takes a room and date. The
first three answer with `OK:` or `ERROR:`, but `listBookings` answers with neither, so a
caller has to know which shape to expect from which method. `cancelBooking` takes no user,
so anyone can cancel anyone's booking.

**Structure.** Four classes exist and three participate. `ReservationApp` runs a demo
script against `RequestHandler`. `RequestHandler` holds the only object reference in the
system — `private final InMemoryStore store = new InMemoryStore()`, line 10 — and owns
much more than `DESIGN.md` credits it with: parsing, time conversion, the overlap rule,
the reschedule sequence, and response formatting. `InMemoryStore` owns the two maps.
`BookingPolicy` owns the three documented rules and is referenced by nothing — never
imported, constructed, or called, by the main code or by the tests. `DESIGN.md` draws
requests flowing from handler to policy to storage and says nothing skips a level; in the
code there is no policy level, so every request skips it.

**The no-double-booking invariant.** The rule is that two bookings for the same room on
the same day may not overlap. Three places look like they check it, and no two check the
same thing. `RequestHandler.createBooking` (lines 30-36) runs a real overlap test, but
only on the create path, and it is a hand-copied version of logic that already exists
elsewhere. `InMemoryStore.addSlot` (lines 23-27) compares intervals for exact equality, so
it turns away a second 09:00-10:00 but accepts 09:30-10:30 laid on top of one — that check
protects the map keys, not the rule. `BookingPolicy.validate` (lines 29-33) holds the
correct test and never runs. One reschedule shows where that leaves things:
`ReservationApp` line 21 calls `rescheduleBooking`, which parses its four times (69-88),
confirms the new end is after the new start (89-91), asks `store.bookerFor` whether the old
booking exists and who owns it (93), calls `store.removeSlot` to delete it (98), calls
`store.addSlot` to insert the new interval (99), and returns `OK: moved` at line 100. No
overlap check happens anywhere on that path, and the handler ignores what both storage
calls report back. So, asked where no-double-booking is enforced: on the create path only,
by a copy of the rule, inside the class meant to be handling strings. Running the code
confirms both consequences — rescheduling onto a time someone else holds stores two
overlapping bookings and reports success, and rescheduling onto an identical interval
deletes the original, stores nothing, and still reports success.

---

## Milestone 2: Two design problems

### Problem 1

**The problem.** Misplaced responsibility. The no-double-booking rule belongs to
`BookingPolicy`, which is named for the rules and holds a correct version of this one. The
version that actually runs sits in `RequestHandler`, whose job is meant to be reading
requests and writing replies. Because the rule lives in a method instead of in the class
that owns rules, applying it became a choice made once per method: the create path took a
copy, and the reschedule path took nothing.

**Where in the code.** `RequestHandler.createBooking` lines 30-36 hold the copy that runs.
`RequestHandler.rescheduleBooking` lines 93-99 are where the same check is missing.
`BookingPolicy.validate` lines 29-33 hold the version nobody calls. The same split has
already happened to a second rule: "end must be after start" is written three times, at
`createBooking:26`, `rescheduleBooking:89`, and `BookingPolicy.validate:20`.

**What it makes expensive.** Two things are already wrong. Rescheduling a booking onto a
time another person holds returns `OK: moved` and leaves both bookings overlapping in
storage. Rescheduling onto an identical interval deletes the original, stores nothing, and
still returns `OK: moved`, because `addSlot` refuses the duplicate and line 99 discards
its answer. Two rules `DESIGN.md` advertises have also never worked at all: an eleven-hour
booking and an 03:00 booking are both accepted. The next planned change makes this worse.
`DESIGN.md` calls per-building business hours "the change we expect next" and promises a
single place to edit when the rules change. That place is `BookingPolicy`, so the edit
gets made there, the tests written against it pass, and the service behaves exactly as
before, because nothing calls it. What breaks first is quieter than that: the first person
who reschedules into an occupied room is told `OK`, with no error, no log entry, and no
failing test.

### Problem 2

**The problem.** Representational gap. Nothing the service talks about exists as a type in
the code. A booking is two map entries that happen to agree. A room and a date are
fragments of a string key. A span of time is a two-element array where position decides
meaning. A person is a string kept off to one side.

**Where in the code.** `InMemoryStore` lines 13-14 declare the two maps, and the key string
is assembled by hand at seven separate points (lines 17, 29, 34, 42, 50, 58, 62).
`RequestHandler` lines 21-22 turn `"HH:MM"` into a number without checking the result.

**What it makes expensive.** Recurring bookings is the first item on the plan, and a
weekly seminar is a single idea with nowhere to live. There is no booking object to attach
a repeat rule to, and storage is organised one room-day at a time, so a semester becomes
fifteen unconnected entries under fifteen different keys. Cancelling the series has
nothing to aim at, moving it means fifteen separate reschedules that can each half-succeed,
and tying the occurrences together means a third map to keep in step with the other two.
Every later detail of a booking — an ID, an attendee count, a reason for cancelling —
arrives the same way. Something already goes wrong, too: `"WEH-5302"`, `"weh-5302"`, and
`"WEH-5302 "` are three different keys, so three people can each book the same room at the
same hour and each be told `OK`. The overlap check ran correctly all three times; it was
looking in three different places. Fixing Problem 1 would not catch this, because the clash
is invisible before any rule runs. What breaks first is two groups arriving at one room,
after a front end passes along whatever spelling a client sent.

---

## Milestone 3: Two alternative decompositions

### Alternative A

**The decomposition.** Four pieces in place of today's one busy handler. Small value types
— a room identifier, a time span, a booking — settle what is valid when they are created,
so a room name is made canonical and an impossible time is refused up front, and one
booking object replaces the two parallel maps. `BookingPolicy` holds every rule and answers
one question: is this booking allowed alongside the ones this room already has that day.
`BookingService` owns the four operations; each one fetches the day, asks the policy, then
writes, and reschedule becomes a single step that checks the new time against the day
without the booking being moved, then swaps them. `BookingStore` becomes an interface with
no rules inside it, so the in-memory version and a later database version are
interchangeable. `RequestHandler` keeps only reading requests and writing replies. Every
rule lives in `BookingPolicy`, and every write passes through it.

**One tradeoff.** The guarantee still rests on habit. `BookingService` has to remember to
ask the policy before each write, and the store will save whatever it is handed — which is
precisely how today's bug happened, with one method remembering and the other forgetting. A
fifth operation added next term can forget in the same way.

### Alternative B

**The decomposition.** Put the rule inside the thing it protects. A `RoomDay` owns the
bookings for one room on one day and is the only thing able to change them; its book,
cancel, and move methods each check for a clash before doing anything, and move cannot
half-fail because it is one step on one object. A `ScheduleRepository` hands out `RoomDay`
objects and is the only place room-and-date keys exist at all. `BookingRules` keeps the
rules that are preferences rather than promises — opening hours, maximum length,
per-building settings — and is applied before a request reaches a `RoomDay`. A thin service
ties the three together. The overlap rule lives with the data it constrains; the adjustable
rules live apart from it.

**One tradeoff.** "What does the service allow?" stops having a single answer. The rules
live in two homes, and each new rule needs a judgement about which home it belongs in. Two
features already on the plan do not fit either: a booking running past midnight belongs to
two days and therefore to no single `RoomDay`, and a limit on one person's weekly bookings
spans many rooms and days. Both need something above `RoomDay` that can see more than one
of them, and once that exists and begins holding rules, there is a third place to look.

### Preference

Alternative A, on one condition — that this service keeps growing in what its rules say
and where its data is kept, rather than in how many different ways a booking can be
written. That is the growth `DESIGN.md` actually predicts. Per-building hours is a setting
inside `BookingPolicy`. The database-backed store is a second implementation of
`BookingStore`. Recurring and past-midnight bookings are requests the service assembles and
hands to the policy. Two of those four break B outright, since neither a past-midnight
booking nor a per-person limit belongs to one room-day, so B needs a coordinator above its
own pieces before it can ship either one. A needs that coordination too, but already has
the coordinator.

The deciding argument is that the two costs are not equally fixable. A's weakness can be
answered from inside A: route every write through one place that calls the policy, or let
the store accept only a booking the policy has already approved. Either buys most of B's
safety without B's shape. B's weakness cannot be answered from inside B, because the
boundary is drawn at one room on one day, the planned features do not respect that line,
and moving the line means giving up the decomposition.

I would switch to B once a third separate thing starts writing bookings — a web front end,
an admin tool, and an import job, say. At that point "the service remembers" becomes "three
services remember," and that habit has already failed once here with a single author and
four methods; a forgotten check becomes likelier than an awkward coordinator, and B makes
forgetting impossible. If past-midnight and per-person rules also dropped off the plan, B's
cost would mostly disappear and I would take it.

And if RoomReserve stays a prototype nobody builds on, I would do neither: call
`BookingPolicy` from both write paths, delete the copy inside `createBooking`, and leave
the maps alone.
