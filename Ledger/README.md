# The Banker's Ledger

You are tasked with implementing a ledger to track transactions in a banking system. In the system, tellers make transactions using the `Transfer` function to pass amounts between different accounts.

The `LEDGER` matrix records transactions in four columns:

- transaction ID `TID` in column 1
- account ID `ACC` in column 2
- debit amount `DEB` in column 3
- credit amount `CRED` in column 4

In this simplified model, you can think of a _debit_ amount as money given to an account, and _credit_ as money taken from an account.

Two functions are used to check the consistency of the ledger. `Balance` computes the balance of a particular account as $\sum{DEBIT} - \sum{CREDIT}$. `Audit` checks that the total credits equals the total debits across all accounts.

Two usage scenarios are provided that show concurrency failures. Your task is to implement the proposed solutions. We recommend that you take a copy of the code to edit for each solution to compare later.

## Mutual Exclusion (MutEx) using :Hold

Assuming the system must update the ledger must write or read the ledger across multiple lines, use `:Hold` to prevent simultaneous access to the ledger.

## Atomic Updates

The interpreter only switches threads at certain points in execution. Use this fact to make the code thread-safe without use of any explicit locking mechanisms.

## Reader/Writer Token Scheme

Use `⎕TALLOC`, `⎕TGET` and `⎕TPUT` to implement a locking scheme such that multiple "readers" (calls to `Audit` or `Balance`) may simultaneously check the ledger, but a writer should prevent all other readers and writers from accessing the ledger until it has finished.

Take care to:

- prevent thread starvation
- make sure tokens are released in the event a thread terminates early, for example due to an error

## Timing

Use the `_Time` operator to measure the time taken to perform multiple reads and writes in both scenarios .
