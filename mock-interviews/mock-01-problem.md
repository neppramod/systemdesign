# Mock Interview 01 — Problem

> **How to use this:** Read the prompt below. Then, in your reply to me, present your design as if
> you were in the live round — talk through the framework steps in order. Don't look at Topic 1
> while answering (that's the point). Take ~30–40 min. When you're done, I'll play interviewer:
> I'll judge your answer, push on the deep dives you missed, and give you concrete feedback on
> where you'd have lost the round and how to fix it.
>
> It's fine to be rough and to think out loud in text. I'm grading your *process and tradeoff
> reasoning*, not diagram beauty.

---

## The Problem: Design a Ticket Booking System (e.g. Ticketmaster / BookMyShow)

Design the backend for a system that sells tickets to live events (concerts, sports, movies).

**Starting context (intentionally minimal — the rest is yours to scope):**
- Users browse events, view seat availability, select seat(s), and pay.
- A popular event can put **hundreds of thousands of seats on sale at a single instant** (e.g.
  Taylor Swift tickets drop at 10:00 AM) with **far more buyers than seats**.
- A seat must never be sold twice.

That's all you get. Everything else — scale numbers, what to include/exclude, consistency model,
data store choices — **you** must establish via the framework.

---

## What I'll specifically be evaluating (don't peek until after your attempt)

<details>
<summary>Click to expand AFTER you finish — these are the dimensions I'll grade.</summary>

1. **Did you drive requirements & estimation**, or wait to be told?
2. **Did you spot the *core hard problem*?** (Hint: it is not "store events." It's the
   concurrency/consistency of seat selection under a massive thundering herd.)
3. **Seat reservation correctness** — how do you prevent double-selling AND avoid locking a seat
   forever when a user abandons checkout? (reservation hold + TTL? optimistic vs pessimistic?)
4. **The thundering herd** — 1M people hitting "buy" at 10:00:00. How do you protect the system?
   (waiting room / virtual queue? rate limiting? )
5. **Read vs write split** — browsing (huge read) vs booking (contended write). Do you treat them
   differently?
6. **Payment flow** — what happens to the held seat during payment? What if payment fails or times out?
7. **Tradeoffs named explicitly** with a chosen side, not a survey.
8. **Failure modes** — what happens when the reservation store / payment provider / queue dies?

</details>

---

## Your turn
Reply with your design. Start at requirements. Go.
