# Double-entry ledger with per-User books and append-only Postings

Kitty records five kinds of Transaction, and they feed three kinds of balance. Open-source expense splitters use single-entry tables with a separate balance rule for each, so a new kind can be left out of a query with no error. So Kitty stores money as double-entry Postings, and every balance comes from one rule: sum the Postings on a Ledger account. Each User's Postings on a Transaction sum to zero, so one Expense can write eight Postings, and a debt appears twice as mirrored Ledger accounts ("Ben owes me" and "I owe Ana"). Amounts live only in Postings. The Transaction record holds non-money facts.

Postings never change. An edit adds cancelling Postings and new Postings as a numbered revision of the same Transaction, and a delete adds cancelling Postings and marks the Transaction as deleted. The revisions are the history log beneath a Transaction. This costs more rows per Transaction, posting rules for each kind, and queries to read an Expense back into a plain shape. In return, history is part of the data, and an Expense amount can't disagree with the balances.

## Considered Options

- Single-entry domain tables, as in Spliit, SplitPro, and ihatemoney. The schema reads like the glossary and has about half the rows, but each balance query must handle every kind that affects it.
- Domain tables plus a derived ledger. It keeps both readable tables and the zero-sum check, but the two can disagree, and one must win.
- One shared set of books, with four Postings for the same Expense. It has fewer rows, but other Members would have no Category Postings of their own, so their Category totals would come from debt Postings.
- Updating Postings in place with a history table. It has fewer rows, but history sits beside the data.

## Consequences

Erasure and retention purges must still remove personal data, so append-only has that one exception. Data protection rules for Users and Groups defines how, and a shared Transaction's amounts must stay correct.
