# Seats

You are running a theatre box office. Clerks sell seats with `Book`, an auditor watches the "seats-left" board, and some bookings also charge a shared card gateway. As written the code races, and once a second lock is involved it can freeze. Your job is to fix each fault with the right tool, which is not always a bigger lock.

The exercise has two tiers. Scenarios A, B and C use a single lock and cover `:Hold` as the default. Scenarios D, E and F add a second lock, where the order you take locks in starts to matter. Do the single-lock tier first.

## Setup and Use
Take a copy of the code so that you can keep the original for reference. Use Link to import your copy into your active workspace.

```apl
      ]Link.Create Seats /path/to/Seats-copy
      Seats.ScenarioA
      Seats.ScenarioB
      Seats.RunAll        ⍝ all six scenarios at once
```

## The domain

- `SEATS` is a boolean vector, one element per seat: `0` free, `1` sold.
- `SOLD` is a running count of sold seats. It repeats a fact that `SEATS` already holds.
- `'PAYMENT'` is a second lock, standing in for a shared card gateway, that the two-lock flows also take.

One rule must hold at every instant: no seat is sold twice, and `SOLD` equals `+/SEATS`. The count stays in step with the seats actually taken, and never exceeds capacity.

## Hazards

Inspect the code and find where each of these arises.

- Double-book: two clerks claim one seat.
- Torn board: an auditor reads part-way through an update.
- Throughput: the cost of a coarse lock.
- Deadlock: two locks taken in opposite orders.
- Livelock: a non-blocking retry that never makes progress.
- Starvation: a lock held across slow work.

## Tasks: one lock

You edit `Book`, `Board`, and later `Remaining` and `NewSeats`. The `Scenario*` functions are harness, so leave them alone. Search for `⍝ TODO`.

### Lock the seat
`Book` reads a seat, decides, then claims it on a later line. Because the check and the claim sit on separate lines, two clerks can both pass the check before either one writes. You cannot squeeze read, decide and write onto a single line, so this one needs a lock. Wrap the whole check-and-claim in `:Hold`, and take the same hold in `Board`. If only `Book` locks, the auditor still reads through a booking in progress and the board tears. Both sides have to take the lock.

Once both hold, `ScenarioA` and `ScenarioB` print:

```
Scenario A: PASS
Scenario B: PASS
```

Now rerun `ScenarioC`. Its time roughly doubles once `Board` holds: the auditors keep the lock through their whole render, so they serialise. The next step removes that cost.

### Give the count one source of truth
Locking the reader was the wrong fix for the tear. The board tears only because `SOLD` is a second copy of the count, updated on its own line just after the seat. So remove the copy. Delete `SOLD` and work the count out from `SEATS` directly. Do not reach instead for a one-line "atomic" update of the two globals; that is the fragile line-atomicity trap. With a single source of truth the board cannot tear, `Board` no longer needs a hold, and the auditors run lock-free. `ScenarioB` still passes with no reader lock, and `ScenarioC` drops back down.

Leave the `:Hold` in `Book`. At every tier, the double-book still depends on that gap between reading and writing.

## Tasks: two locks

Do the one-lock tasks first. Until `Book` holds `'SEATS'`, nothing here contends for it, and the scenarios pass for the wrong reason. You edit `ReserveSP` and `ReservePS` (deadlock), `ReserveRetrySP` and `ReserveRetryPS` (livelock), and `SlowReport` (starvation). Again, the `Scenario*` functions are harness.

### Deadlock
A reservation has to hold both `'SEATS'` and `'PAYMENT'`. `ReserveSP` takes them in one order, `ReservePS` in the other. Under load each takes its first lock and then waits on the second, which the other flow is holding. Neither moves. Break the cycle: take both locks together in one hold, or use a single acquisition order everywhere. `ScenarioD` should print `Scenario D: PASS`.

> A stuck cycle shows up as `DEADLOCK` (1008). Treat 1008 as something to prevent, not to handle: `:Trap 1008` catches it, but the caught thread stays suspended with its token still held. The scenario guards and times out the flows only so it can report cleanly instead of hanging. Your own flows carry no such guard.

### Livelock
`ReserveRetrySP` and `ReserveRetryPS` avoid the deadlock with non-blocking holds: take one lock, try the other without waiting, and on failure release everything and retry. They never block, so they never deadlock. But run them in opposite orders and they collide on every round and retry forever, in step. Nothing errors and no booking completes. That is harder to spot than a deadlock, because nothing signals that the work has stalled. Break the symmetry so the flows stop retrying in lockstep. `ScenarioE` should print `Scenario E: PASS`.

### Starvation
`SlowReport` holds `'SEATS'` for the length of its slow render, so bookings queue up behind it. Hold the lock for as little as possible: take a snapshot under the hold, release it, then render from the snapshot. `ScenarioF` should print `Scenario F: PASS`.

## Running it

| Scenario | Racy | Fixed |
|----------|------|-------|
| A Double-book (at rest)    | one seat sells more than once, count drifts | a single sale, no drift |
| B Torn board (concurrent)  | torn reads above `0` | `0` |
| C Throughput (checkpoint)  | coarse `:Hold` near 2× the lock-free time | lock-free, near 1× |
| D Deadlock                 | flows freeze (`DEADLOCK` 1008) | both finish |
| E Livelock                 | close to no bookings finish, and nothing errors | all finish |
| F Starvation               | bookings wait out the render | bookings run during the render |

## A green PASS is evidence, not proof

The racy scenarios fail at random, and the torn board leaves no trace at rest: once a run ends the books balance again, so a single PASS proves nothing. Run each scenario several times. What you are after is a repeated PASS.
