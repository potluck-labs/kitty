# How finance apps model money and balances

Research for [#5](https://github.com/potluck-labs/kitty/issues/5), part of map [#2](https://github.com/potluck-labs/kitty/issues/2). It lists options and trade-offs. It does not pick one. Terms in capitals (Account, Expense, Share, Split, Settlement, Shared balance, Loan balance) are from `GLOSSARY.md` on branch `docs/glossary-and-identity-adr`.

Sources were read on 2026-10-28. Code links are pinned to the commit that was read.

## Summary

- Money amounts: the apps and payment APIs read store exact values. Stripe, Spliit, and SplitPro use integers in minor units (cents). PostgreSQL and Python's docs recommend exact decimal types. ihatemoney uses floats and carries a 0.01 tolerance in several places as a result.
- Bookkeeping: the open-source expense splitters (Spliit, SplitPro, ihatemoney) are single-entry. They store Expenses with per-person rows and compute balances by query. Payment companies (Stripe, Square, Modern Treasury, TigerBeetle) and personal finance tools (Firefly III, Beancount) use double-entry, where each transaction's entries sum to zero.
- Balances: every app read computes balances from the transactions. SplitPro first kept separate balance tables, saw them drift after edits and deletes, and replaced them with a SQL view.
- Debt simplification: the open-source apps use a greedy pass that matches the largest debtor with the largest creditor. It needs at most n-1 payments for n people. Finding the true minimum number of payments is NP-hard.
- Rounding leftovers: all approaches keep the Shares summing to the Expense amount. They differ in who gets the extra cent: largest remainder with a rotation (Spliit), a seeded shuffle (SplitPro), the first participant (Dinero.js), or random (Splitwise).
- Edits and deletes: the expense splitters update rows in place and keep some history (an activity log, soft delete, or full row versions). Ledger systems never change a posted entry. They post a reversing entry and a new one.

## 1. Representing an amount

### What the sources say

- Stripe's API takes every `amount` "in the currency's minor unit". 10 USD is `1000`. Zero-decimal currencies such as JPY use the whole unit, so 500 JPY is `500`. ([Stripe, Supported currencies](https://docs.stripe.com/currencies#minor-units)) EUR is a two-decimal currency on that page.
- PostgreSQL says `numeric` "is especially recommended for storing monetary amounts and other quantities where exactness is required", and that `real` and `double precision` "are inexact". `numeric` is "very slow compared to the integer types". `bigint` holds up to 9,223,372,036,854,775,807. ([PostgreSQL, Numeric types](https://www.postgresql.org/docs/current/datatype-numeric.html))
- The PostgreSQL wiki says not to use the `money` type: it "doesn't handle fractions of a cent" and "doesn't store a currency with the value". It suggests `numeric`, or rarely `integer`. ([PostgreSQL wiki, Don't Do This](https://wiki.postgresql.org/wiki/Don%27t_Do_This#Don.27t_use_money))
- Python's `decimal` docs show that `0.1 + 0.1 + 0.1 - 0.3` is exactly zero in `Decimal` and `5.5511151231257827e-017` in binary float, and say "decimal is preferred in accounting applications which have strict equality invariants". `quantize()` rounds to a fixed number of places, with a chosen rounding mode such as `ROUND_HALF_EVEN`. ([Python, decimal](https://docs.python.org/3/library/decimal.html))
- JavaScript's `number` is an IEEE 754 double. It holds integers exactly only up to `Number.MAX_SAFE_INTEGER`, 9,007,199,254,740,991. Above that, use `BigInt`. ([MDN, MAX_SAFE_INTEGER](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Number/MAX_SAFE_INTEGER))
- node-postgres returns `int8` (`bigint`) columns as strings by default, because a 64-bit value can overflow a JavaScript number. ([pg-types README](https://github.com/brianc/node-pg-types#readme))
- Splitwise's public API sends `cost`, `paid_share`, and `owed_share` as "a string representation of a decimal value, limited to 2 decimal places". ([Splitwise API](https://dev.splitwise.com/))

### What open-source splitters do

| App | Stack | Amount type | Source |
| --- | --- | --- | --- |
| Spliit | TypeScript, Prisma, Postgres | `Int` in minor units | [schema.prisma](https://github.com/spliit-app/spliit/blob/73478ed2f67a59e83866948caeece6bf2ecb985e/prisma/schema.prisma) |
| SplitPro | TypeScript, Prisma, Postgres | `BigInt` in minor units, migrated from `Int` in 2025 | [schema.prisma](https://github.com/oss-apps/split-pro/blob/3583568319807e193fbe57085ea9560a2e872b93/prisma/schema.prisma), [migration](https://github.com/oss-apps/split-pro/blob/3583568319807e193fbe57085ea9560a2e872b93/prisma/migrations/20250526193958_migrate_amounts_to_bigint/migration.sql) |
| ihatemoney | Python, Flask, SQLAlchemy | `Float` in major units | [models.py](https://github.com/spiral-project/ihatemoney/blob/e66a7672e8e5c41549bf53c4a824c72c43ab9079/ihatemoney/models.py) |

ihatemoney shows what floats cost. It keeps showing a deactivated member while `abs(balance) > 0.01`, and its settlement library raises an error only when the two sides differ by 0.01 or more ([debts solver](https://framagit.org/almet/debts/-/blob/master/debts/solver.py)). Spliit's balance code has a comment that float totals rounded per participant "does not preserve a sum, and the residue ended up in the group's total balance", which it fixed by moving to whole minor units ([balances.ts](https://github.com/spliit-app/spliit/blob/73478ed2f67a59e83866948caeece6bf2ecb985e/src/lib/balances.ts)).

### Options for Kitty

| Option | Python side | TypeScript side | Postgres | Trade-offs |
| --- | --- | --- | --- | --- |
| A. Integer cents | `int`, unbounded | `number` is safe to about 90 trillion euros in cents. `bigint` if you want no ceiling. | `bigint` (or `integer`, which caps at 21,474,836.47 euros) | Fast, exact, simple sums. Every input and display needs a conversion step. node-postgres returns `bigint` as a string unless you register a parser. Changing to a currency with 3 decimals later means a data migration. |
| B. Exact decimal | `Decimal` | A decimal library, or strings over the wire | `numeric(12,2)` or similar | Values read like euros. Python's `Decimal` is built in. TypeScript has no built-in decimal type, so a library or string handling is needed. Slower arithmetic in Postgres, not likely to matter at Kitty's size. |
| C. Float | `float` | `number` | `double precision` | Shown here only because ihatemoney uses it. All sources above advise against it for money. |

Kitty is EUR only for the MVP (map #2, Out of scope), so either A or B has one fixed scale of 2. A "Money" value type (amount plus currency) is a common wrapper for either. Martin Fowler's catalog names the two problems it guards against: mixing currencies, and amounts lost to rounding ([Fowler, Money](https://martinfowler.com/eaaCatalog/money.html)).

## 2. Single-entry or double-entry

### Double-entry, as the sources define it

- Each transaction has two or more entries, one per account touched, and the entries sum to zero. Beancount: "The sum of all the postings of a transaction must equal zero." So the sum of all account balances is also zero. ([Beancount, The double-entry counting method](https://beancount.github.io/docs/the_double_entry_counting_method/))
- Fowler: "For an accountant you cannot create money, it is only ever moved around." He suggests two-legged transactions (one from, one to) unless you need more legs, since multi-legged ones are harder to create and validate. ([Fowler, Accounting Transaction](https://martinfowler.com/eaaDev/AccountingTransaction.html))
- Modern Treasury: store immutable transactions and derive balances from them, since changing balances directly makes errors hard to detect and reconcile. If the debit-normal and credit-normal totals differ, the system has created or lost money. ([Modern Treasury, Accounting for developers part I](https://www.moderntreasury.com/journal/accounting-for-developers-part-i))
- Square built "Books", an append-only double-entry service where every journal entry balances to zero and a book's balance is the sum of its entries. ([Square, Books](https://developer.squareup.com/blog/books-an-immutable-double-entry-accounting-database-service/))
- Stripe's Ledger is an append-only double-entry log. Stripe checks health by looking for "clearing" accounts that should be zero at steady state but are not. ([Stripe, Ledger](https://stripe.dev/blog/ledger-stripe-system-for-tracking-and-validating-money-movement))
- Firefly III, a self-hosted personal finance app, stores each transaction as a journal with two rows: "-250 from your bank account" and "+250 for Amazon.com". Its docs say that today this is "useful to check if all records are straight", including for transfers between your own accounts. ([Firefly III docs, Transactions](https://github.com/firefly-iii/docs/blob/main/docs/docs/explanation/financial-concepts/transactions.md))

### How expense splitters do it

Spliit, SplitPro, and ihatemoney are single-entry. Each stores one Expense row with a payer and amount, plus one row per participant (a weight in Spliit and ihatemoney, an amount in SplitPro). Balances are sums over those rows. Spliit computes per participant `paid - paidFor` ([balances.ts](https://github.com/spliit-app/spliit/blob/73478ed2f67a59e83866948caeece6bf2ecb985e/src/lib/balances.ts)). ihatemoney computes `paid - spent + transferred - received` ([models.py](https://github.com/spiral-project/ihatemoney/blob/e66a7672e8e5c41549bf53c4a824c72c43ab9079/ihatemoney/models.py)). Spliit and ihatemoney record a Settlement as a special Expense: Spliit with an `isReimbursement` flag, ihatemoney with `BillType.REIMBURSEMENT`.

None of the three tracks money in bank or cash accounts. Kitty does (Account, Transfer, Income), which puts it closer to Firefly III on the personal side.

### Beancount's model for shared expenses

Beancount's "Sharing expenses" guide shows a group trip in double-entry ([Beancount, Sharing expenses](https://beancount.github.io/docs/sharing_expenses_with_beancount/)). Each person's contributions come from an account with their name (`Income:Martin:CreditCard`). Each shared expense is split into per-person expense accounts (`Expenses:Accommodation:Martin`). A person's position is the sum of every account with their name in it. A settlement is one transfer between the two people's accounts.

Mapped to Kitty's terms, an Expense of 10.00 EUR that Ana pays from her bank Account, split three ways, could post like this (amounts in cents):

| Ledger account | Entry |
| --- | --- |
| Ana's Account "Bank" | -1000 |
| Expense, Category Food, Ana's Share | +334 |
| Expense, Category Food, Ben's Share | +333 |
| Expense, Category Food, Cy's Share | +333 |
| Sum | 0 |

Ana's Shared balance position is -1000 + 334 = -666, so the group owes her 6.66. Ben and Cy each owe 3.33. The Account balance, the Category totals, and the Shared balances all come from the same entries. A Loan would credit the lender's Account and debit the borrower's Account, with a pair of Loan balance accounts to keep it apart from Shared balance, as the glossary requires.

### Options for Kitty

| Option | Shape | Trade-offs |
| --- | --- | --- |
| A. Single-entry domain tables | `expense` plus `expense_share` rows, `transfer`, `loan`, `settlement` tables. Balances by query. | What Spliit, SplitPro, and ihatemoney do. Fewer tables, schema reads like the glossary. The rule "money is conserved" lives in application code and tests. Each new Transaction kind needs its own balance query logic for Account balance, Shared balance, and Loan balance. |
| B. Double-entry ledger as the source of truth | A `transaction` table and an `entry` table (account, amount, sign). Expense, Loan, and so on are views or metadata on a transaction. | What payment companies and Firefly III do. One rule (entries sum to zero) checks every kind. One balance query for every account. More concepts to learn, and the domain words sit one layer above the data. |
| C. Domain tables plus a derived ledger | Domain tables are the source of truth. Each write also posts balanced entries, or a check job rebuilds entries and compares. | Domain stays readable, and the zero-sum rule is still checked. Two representations must stay in sync, so you have to decide which wins if they differ. |

## 3. Computing balances between Members

### Net position or pairwise

There are two shapes of "who owes whom":

- Net position per Member: one number per Member, `paid - owed`, summing to zero across the Group. Spliit and ihatemoney work this way. Who pays whom is then a suggestion computed from the net positions.
- Pairwise: one number per pair of Members. SplitPro's `BalanceView` groups Expense participant rows by (participant, payer, group, currency) ([migration](https://github.com/oss-apps/split-pro/blob/3583568319807e193fbe57085ea9560a2e872b93/prisma/migrations/20251108122842_add_balance_view/migration.sql)). Splitwise's API returns both `original_debts` (pairwise) and `simplified_debts` for a group ([Splitwise API](https://dev.splitwise.com/)).

The glossary defines Shared balance as "what one Member owes another", which is pairwise. A Settlement "says which one" it reduces, Shared or Loan. If Kitty also shows simplified debts, a Settlement may follow a simplified edge that has no pairwise debt under it. The design must then say how that Settlement reduces the pairwise balances.

### Stored or computed

- Computed on read, in application code: Spliit loads all group Expenses and sums them in TypeScript ([balances.ts](https://github.com/spliit-app/spliit/blob/73478ed2f67a59e83866948caeece6bf2ecb985e/src/lib/balances.ts)). ihatemoney does the same in Python.
- Computed on read, in SQL: SplitPro's `BalanceView`. The migration notes "We may want to migrate this to a materialized view in the future for performance reasons."
- Stored and updated on each write: SplitPro used to keep `Balance` and `GroupBalance` tables next to the Expense rows. Issue [#234](https://github.com/oss-apps/split-pro/issues/234) reports a member's balance "off exactly his portion of a deleted expense" while the `ExpenseParticipant` rows were correct, and asks why one Expense is tracked in four tables. PR [#479](https://github.com/oss-apps/split-pro/pull/479) replaced the tables with the view, running both side by side with an assertion until the view matched.
- Square's Books keeps each book's balance as the running sum of its entries, so reading a balance needs no grouping or aggregation. Entries are append-only, so the running sum never has to absorb an edit ([Square, Books](https://developer.squareup.com/blog/books-an-immutable-double-entry-accounting-database-service/)).

### Account balance, Shared balance, Loan balance

These three draw on different parts of each Transaction:

- Account balance: opening balance, plus Income, plus Transfers in, minus Transfers out, minus the full amount of each Expense paid from the Account, minus Loans and Settlements paid from it, plus Loans and Settlements received into it. Map #2 says the MVP credits the receiver's default Account for Loans and Settlements.
- Shared balance: per pair of Members in a Group, from each Expense's Shares owed to its Payer, reduced by shared Settlements.
- Loan balance: per pair of Members in a Group, from Loans, reduced by loan Settlements. The glossary says a loan Settlement reduces the overall Loan balance, not a specific Loan. That matches net-per-pair arithmetic and needs no per-Loan matching.

## 4. Simplifying debts

### What the sources say

- Splitwise's blog lists the rules its "simplify my debts" tool keeps: "Everyone owes the same net amount at the end", "No one owes a person that they didn't owe before", and "No one owes more money in total than they did before the simplification." A reply on the same post says group simplification keeps rules 1 and 3 but not 2, so it can create a debt between two people who had none. Turning group simplification off restores the original debts. ([Splitwise blog, Debts made simple](https://blog.splitwise.com/2012/09/14/debts-made-simple/))
- Verhoeff (2004) studies several debt-settling variants and calls one "an elegant NP-hard problem". The hard one is the minimum number of transfers. ([Verhoeff, Settling multiple debts efficiently, Informatics in Education 3(1)](https://infedu.vu.lt/journal/INFEDU/article/612))
- ihatemoney uses the `debts` library. It sorts creditors and debtors, pays the largest debt toward the largest credit, and repeats. Each step zeroes at least one person, so n people need at most n-1 payments. It rounds each payment with `ROUND_HALF_DOWN` to 0.01 and drops payments under 0.01. ([debts solver](https://framagit.org/almet/debts/-/blob/master/debts/solver.py))
- Spliit's version sorts positive balances before negative ones and breaks ties by participant id, "so that a participant executing a suggested reimbursement does not result in completely new repayment suggestions" ([balances.ts](https://github.com/spliit-app/spliit/blob/73478ed2f67a59e83866948caeece6bf2ecb985e/src/lib/balances.ts)).

### Options for Kitty

| Option | Trade-offs |
| --- | --- |
| A. No simplification. Show pairwise Shared balances as they are. | Simple, and each line has a direct cause. A group of 5 can show up to 10 pairs. |
| B. Greedy suggestion from net positions, computed on read, never stored | At most n-1 payments. Not always the minimum. Stable order matters, or suggestions reshuffle after each Settlement. |
| C. Exact minimum number of payments | NP-hard in general. Exhaustive search is feasible for small Groups only. |
| D. Pairwise-preserving simplification (Splitwise rule 2) | No one is asked to pay someone they never owed. Usually saves fewer payments than B. |

For any of B to D, a separate choice: is the simplified plan only a suggestion that Settlements may follow, or does it replace the pairwise balances shown to Members?

## 5. Rounding leftovers in a Split

A Split of 10.00 EUR among three Members gives 333 cents each and 1 cent left. The sources agree on one rule: the Shares must sum to the Expense amount. They differ on who gets the leftover cent.

| Method | Who gets the extra cent | Used by |
| --- | --- | --- |
| Largest remainder (Hamilton), ties broken by a rotation seeded from a hash of the Expense id | The participant with the biggest dropped fraction. Even splits rotate between participants across Expenses. | Spliit, [shares.ts](https://github.com/spliit-app/spliit/blob/73478ed2f67a59e83866948caeece6bf2ecb985e/src/lib/shares.ts) |
| Seeded shuffle of participants, seed from the amounts and the Expense date | A pseudo-random non-payer for even splits | SplitPro, [addStore.ts](https://github.com/oss-apps/split-pro/blob/3583568319807e193fbe57085ea9560a2e872b93/src/store/addStore.ts#L350-L372) |
| Spread from the first participant | The first ones in the list. 1003 split 50/50 gives 502 and 501. | Dinero.js `allocate`, [docs](https://www.dinerojs.com/docs/api/mutations/allocate) |
| Random | Random participants | Splitwise, per a staff reply on its [feedback forum](http://feedback.splitwise.com/forums/162446-general/suggestions/33733765--bug-splitting-expense-by-shares-calculation-isn) (2018) |
| No rounding | Nobody. Floats carry the fraction. | ihatemoney |

A second choice is where the rounding result lives:

- Store each Share in cents at write time (SplitPro stores `ExpenseParticipant.amount`). The past never changes if the algorithm changes. The database can check that Shares sum to the Expense amount.
- Store the Split rule (weights) and compute Shares on read (Spliit stores `shares` weights). The rule stays editable and nothing is duplicated. The function must stay deterministic forever, or old balances move. Spliit's PR #647 had to make the form preview use the same seed as the saved Expense.

The glossary already treats a Share as "in money", which matches storing it.

## 6. Edits and deletions

### What the sources say

- Fowler lists three ways to correct an accounting event ([Reversal](https://martinfowler.com/eaaDev/ReversalAdjustment.html), [Replacement](https://martinfowler.com/eaaDev/ReplacementAdjustment.html), [Difference](https://martinfowler.com/eaaDev/DifferenceAdjustment.html)):
  - Reversal adjustment: post an opposite entry and a new correct entry. It works when entries are immutable. The cost is three entries for every one you had before, and lists must filter out reversed pairs.
  - Replacement adjustment: delete the wrong entries and post new ones. It needs deletable entries, and it destroys the history of the old event.
  - Difference adjustment: post one entry with the difference. Fowler says to pick reversal if users want to see explicit cancelling, and difference if they prefer a summary.
- TigerBeetle: "Transfers in TigerBeetle are immutable, so once they are created they cannot be modified or deleted." Corrections are new transfers that reverse or adjust the earlier ones, linked to them by a shared field. ([TigerBeetle, Correcting transfers](https://docs.tigerbeetle.com/coding/recipes/correcting-transfers/))
- Stripe's Ledger: "Transactions previously published into Ledger cannot be deleted or modified." ([Stripe, Ledger](https://stripe.dev/blog/ledger-stripe-system-for-tracking-and-validating-money-movement))

### What expense splitters do

| App | Edit | Delete | History kept |
| --- | --- | --- | --- |
| Spliit | In-place `update` | Hard `delete` | An `Activity` row with the type (`UPDATE_EXPENSE`, `DELETE_EXPENSE`), Expense id, and title. Old amounts are not kept. The log write and the change are separate statements, not one database transaction. ([api.ts](https://github.com/spliit-app/spliit/blob/73478ed2f67a59e83866948caeece6bf2ecb985e/src/lib/api.ts)) |
| SplitPro | In-place, sets `updatedBy` | Soft delete with `deletedAt` and `deletedBy`. Balance view filters `deletedAt IS NULL`. | Who and when, not old values ([schema.prisma](https://github.com/oss-apps/split-pro/blob/3583568319807e193fbe57085ea9560a2e872b93/prisma/schema.prisma)) |
| Splitwise | In-place via `update_expense` | Soft delete with `deleted_at` and `deleted_by`, and `undelete_expense` to restore | ([Splitwise API](https://dev.splitwise.com/)) |
| ihatemoney | In-place | Delete | Full row versions through SQLAlchemy-Continuum (`__versioned__`), which writes a history row per change and a transaction log, and can revert deleted objects ([models.py](https://github.com/spiral-project/ihatemoney/blob/e66a7672e8e5c41549bf53c4a824c72c43ab9079/ihatemoney/models.py), [Continuum docs](https://sqlalchemy-continuum.readthedocs.io/en/latest/intro.html)) |

### Options for Kitty

| Option | Trade-offs |
| --- | --- |
| A. In-place update, hard delete, activity log | Simplest. Balances after an edit look as if the old value never existed. An audit question such as "what did this Expense say last week" has no answer unless the log stores old values. |
| B. In-place update, soft delete, row versions (history table or trigger) | Full history, and balance queries stay simple. Every balance query must filter deleted rows. History lives beside the data, not in it. |
| C. Immutable Transactions with reversing entries | Matches Fowler, TigerBeetle, Stripe, Square. History is the data. An "edit" in the bot becomes reverse plus re-post, which doubles rows per edit. Lists shown to Members must hide or group reversed pairs. |
| D. Editable domain record, immutable ledger beneath | Pairs with section 2 option C. Members edit an Expense. The backend posts a reversal and new entries. Two layers to keep consistent. |

## Open questions for the decision ticket

1. Integer cents or exact decimal, and which types in Python, TypeScript, Postgres, and JSON.
2. Single-entry, double-entry, or domain tables plus a derived ledger.
3. Shared balance shown pairwise, as net positions, or both.
4. Debt simplification: none, greedy suggestion, or pairwise-preserving. Suggestion only, or shown as the balance.
5. Balances computed on read (application code or SQL view) or stored and updated on write.
6. Who gets the leftover cent, and whether Shares are stored in cents or computed from a stored Split rule.
7. Edits and deletes: in place with history, or reversing entries.
8. Whether a User's Category totals use the full Expense amount or only their own Share. This affects the Group and personal statistics, and is not settled by the glossary.
