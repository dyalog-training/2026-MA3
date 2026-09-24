# Workshop: parallel is not the same as concurrent

Two small examples that demonstrate principles related to the performance of code run explicitly in parallel using APL threads and Isolates.

Requirements: Dyalog APL 18.2 or later and a machine with several cores.

## Set up

```apl
)copy isolate
]Link.Create # /path/to/isolates/workshop
```

If you prefer not to use Link, fix the four functions directly:

```apl
2 ⎕FIX 'file:///path/to/isolates/workshop/collatz.aplf'
2 ⎕FIX 'file:///path/to/isolates/workshop/primes.aplf'
2 ⎕FIX 'file:///path/to/isolates/workshop/splits.aplf'
2 ⎕FIX 'file:///path/to/isolates/workshop/time.aplf'
```

Then paste the lines of `demo.txt` into the session one at a time. The first
run of anything in a fresh workspace is slower, so run each timed line twice.
(`gritt -stdin < demo.txt` runs the whole script against a session in RIDE
serve mode, see `../RESULTS.md`.)

## The pieces

| Function | What it does |
|---|---|
| `collatz m n` | longest Collatz chain among starting numbers `m..n-1`, returned as `steps start`. One vectorised step per iteration for all chains still running; finished chains drop out. Pure integer arithmetic, tiny result. |
| `primes m n` | the primes in `[m,n)` by a segmented sieve (from the README's `psieve`). Memory-heavy: it scatters into a bit array. |
| `k splits lo hi` | `k` equal `(start end)` pairs covering `[lo,hi)`, so any range can be handed out in pieces. |
| `(f time) x` | runs `f x` and returns `(elapsed ms)(CPU ms of this process)` followed by the result. |

`IÏ` is the isolate workspace's model of the parallel-each operator `∥¨`: one
temporary isolate per item, and an array of futures back immediately. Anything
that needs the values, such as `↑`, `+/` or display, waits for them.

## Example 1: longest Collatz chain below 2 million

```apl
chunks←10 splits 2 2E6
best←{(⊃⍒⍵[;⎕IO])⌷⍵}
t r←({↑collatz¨⍵} time) chunks ⋄ t,best r              ⍝ one after another
t r←({↑⎕TSYNC collatz&¨⍵} time) chunks ⋄ t,best r      ⍝ ten APL threads
t r←({↑collatz IÏ ⍵} time) chunks ⋄ t,best r           ⍝ ten isolates
```

Output on an Apple M2 Pro (6 performance + 4 efficiency cores), Dyalog 21.0.
Columns: elapsed ms, CPU ms of this process, steps, start.

```
2812 2806 556 1723519      one after another
2736 2722 556 1723519      ten threads
 398   19 556 1723519      ten isolates
```

Points to make:

- **Threads are concurrent, not parallel.** Ten `&` threads finish in the same
  time as the plain loop and use the same CPU, because all Dyalog threads share
  one interpreter on one core. Threads are for overlapping waits, not for
  splitting computation.
- **Isolates are parallel.** Ten isolates finish 7× sooner, and this process
  spent 19 ms: the work happened in ten other processes at the same time.
- **Why 7× and not 10×.** The wall time is the slowest chunk on the slowest
  core. Four of the ten cores are efficiency cores, and the chunks differ by
  about 30% in cost (per-chunk times run from 232 to 307 ms). More chunks than
  processes (20 or 40) does not help here: extra isolates in a process are
  threads again, and each one adds coordination.
- The answer is the same every way: 1,723,519 needs 556 steps.

## Example 2: how many primes below 10⁹

```apl
chunks←10 splits 2 1E9
({+/{≢primes ⍵}¨⍵} time) chunks                        ⍝ sequential
iss←isolate.New¨10⍴⊂⎕NS'primes'                        ⍝ ten isolates holding a copy of primes
({+/iss.{≢primes ⍵}⍵} time) chunks                     ⍝ parallel
({≢⊃,/primes IÏ ⍵} time) chunks                        ⍝ ship the primes back instead
```

```
2457 2450  50847534      sequential
 988    8  50847534      ten isolates, counts come back
3423  833  50847534      ten isolates, all 50,847,534 primes come back
```

Points to make:

- **Not every computation scales.** The sieve is memory-bound: ten processes
  crossing out multiples in ten big arrays share one memory bus, so ten cores
  give 2.5×, not 7×.
- **What comes back matters.** Returning the primes instead of the counts moves
  50 million integers through sockets. The parent alone spends 0.8 s receiving
  them and the whole thing ends up slower than the sequential loop. Return a
  reduction, or a compact encoding, when you can.
- **Two ways to get code into an isolate.** `IÏ` copies only its operand
  function, so it works for a self-contained function like `primes` or
  `collatz`. A function that calls others needs isolates created from a
  namespace holding that code, as `iss` does here; those isolates persist and
  can be called repeatedly.

## Things to try

- `isolate.State ''` after a run: one process per core, and how the isolates
  were spread over them.
- Fewer processes: `isolate.Config 'processors' 4 ⋄ isolate.Reset 0`, then run
  Example 1 again with `4 splits`.
- Timings move with whatever else the machine is doing. Check the load average
  if a run looks odd, and repeat it.
