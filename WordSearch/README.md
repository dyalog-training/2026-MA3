# Word Search Service
You are creating a web service that allows users to add words to a central index and search the index to find similar words as defined by their Levenshtein edit distance.

## Setup and Use
Take a copy of the code so that you can keep the original for reference. Use Link to import the application into your active workspace.

```apl
]LINK.Create WordSearch /path/to/WordSearch-copy
```

## The Word List and Distance Matrix

- `WORDS` is a nested vector of the words the service knows.
- `DIST` is a square numeric matrix; `DIST[i;j]` is the edit distance between `WORDS[i]` and `WORDS[j]`.

When implemented correctly:

- every word appears in `WORDS` exactly once
- `DIST` is a square matrix with shape `2⍴≢WORDS`
- a search always reads a matched `WORDS`/`DIST` pair
- no word is lost or duplicated
- no reader ever sees a half-built index.

The edit distance is computed using the `Dist` function, which gives the minimum number of edits (character additions, deletions or replacements) required to go from one word all other words in the word list.

## Hazards

Inspect the code and determine the sources of these potential errors.

- Corrupt / duplicate write: two potential errors
- Torn read

## Tasks
As written, the system crashes while attempting `ScenarioA`. You can use `)SIC` (clear state indicator) to clear the many suspended threads.

### Lock the Resources
Make `ScenarioA` run successfully by adding `:Hold` statements around all lines that access shared state `WORDS` and `DIST`.

To begin with, only lock the writer function `AddWords`. Then, run `ScenarioA` again and observe how the readers fail. The failure path is non-deterministic so you might need to try multiple times to see the failure.

Then add a lock to the `Search` function. After doing so, `ScenarioA` should print to the session:

```
Scenario A: PASS
```

### Search with a Snapshot
Having the shared resources locked under `:Hold` creates two new problems.

Firstly, by serialising access you lose the benefits of concurrency. You can use the `⎕AI` to check the total running time of the `:While` loop in `ScenarioA`.

```
time ← ⎕AI[3]
expr1 ⋄ expr2 ⋄ expr3
⎕←'Time taken: ',⍕⎕AI[3]-time
```

Secondly, allowing many requests that block others risks thread starvation.

`WORDS` and `DIST` must always be in agreement, so keeping them separate risks them being out of step at certain times. Instead of two variables, merge them into a single shared `INDEX←WORDS DIST`. Think about how you can remove the lock in the reader `Search` function. Does the writer `AddWords` function still need a lock as well?

Once you have implemented this, check that the test passes. You should notice the overall run time decrease as well.

### Bonus: Reader / Writer Locking Pattern
The snapshot approach might be fragile in a larger application with multiple related shared resources, or where having full copies of shared resources is not feasible due to memory constraints. While we advise avoiding this scenario if at all possible, it is interesting to try to implement a reader/writer locking implementation using `⎕TALLOC`, `⎕TGET` and `⎕TPUT`.

Build a reader/writer lock from the token pool with a **negative "gate" token** and **positive "write lock" token**, so that:
- while the index is free, any number of `Search` requests may read at once
- an `AddWords` update waits for current readers to finish and blocks new `Search` threads
- only one `AddWords` updates at a time

### Bonus: Green Futures
Another possibility is for `AddWords` to put newly computed values into the token pool using `⎕TPUT`, and create permanently looping `Update` thread. The `Update` function can be called in a new thread. The challenge here is to account for errors during updates, and to make sure the `Update` loop restarts if its thread should disappear for any reason.

```
∇ Update token
  :While 1
     ⍝ You write this
  :EndWhile
∇
```
