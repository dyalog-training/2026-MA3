# Word Search Service
You are creating a web service that allows users to add words to a central index and search the index to find similar words as defined by their Levenshtein edit distance.

## Setup and Use
Use Link or `⎕FIX` to import the application into your active workspace.

```apl
]LINK.Import WordSearch /path/to/WordSearch
```

```
2⎕FIX'WordSearch/AddWord.aplf'
2⎕FIX'WordSearch/Dist.aplf'
2⎕FIX'WordSearch/InsertNewWord.aplf'
2⎕FIX'WordSearch/ScenarioA.aplf'
2⎕FIX'WordSearch/Search.aplf'
```

## The Word List and Distance Matrix
Unique words are added to the word list, which is stored as a nested vector of character vectors.

The edit distance is computed using the `Dist` function, which gives the minimum number of edits (character additions, deletions or replacements) required to go from one word all other words in the word list.

## Add Words
The `AddWord` function calls `InsertNewWord` if it does not exist in the `WORDS` list. `InsertNewWord` appends the new word to the `WORDS` list, computes the edit distances to all existing words and appends these distances to the `DIST` matrix.

## Search Words
The `Search` function looks up a word in the `WORDS` list and returns the different word with the closest edit distance to the target word.

## Tasks
As written, the system crashes while attempting `ScenarioA`. You can use `)SIC` (clear state indicator) to clear the many suspended functions.

### Lock the Resources
Make `ScenarioA` run successfully by adding `:Hold` statements around all lines that access shared state `WORDS` and `DIST`. After doing so, `ScenarioA` should print to the session:

```
All words successfully added: PASS
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

### Reader / Writer Locking Pattern
The snapshot approach might be fragile in a larger application with multiple related shared resources, or where having full copies of shared resources is not feasible due to memory constraints. While we advise avoiding this scenario if at all possible, it is interesting to try to implement a reader/writer locking implementation using `⎕TALLOC`, `⎕TGET` and `⎕TPUT`.

Use the token pool so that:
- while the shared resources are free, any number of `Search` requests can be served at the same time
- updates via `AddWord` block `Search` threads from reading the shared resources
- only one `AddWord` thread can update the shared resources at once

### Green Futures
Another possibility is for `InsertNewWord` to put newly computed values into the token pool using `⎕TPUT`, and create permanently looping `Update` thread. The `Update` function can be called in a new thread. The challenge here is to account for errors during updates, and to make sure the `Update` loop restarts if its thread should disappear for any reason.

```
∇ Update token
  :While 1
     ⍝ You write this
  :EndWhile
∇
```
