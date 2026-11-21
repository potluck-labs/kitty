# Payments recorded for someone else go to Untracked money

When Ana records an Expense that Ben paid, Kitty can't know which of Ben's Accounts paid, and a Placeholder Member has no Accounts at all. So every User and every Placeholder Member has one Account named Untracked money. Whenever someone else records money moving in or out of a person, such as the Payer of an Expense or the receiver of a Loan or Settlement, that person's side goes to their Untracked money. The person moves it to a real Account by editing the Transaction, which is a normal revision. This reverses "the Payer's Default Account pays" from the permissions ticket. The Default Account is now only the Account Kitty picks first when a User records something themselves.

Untracked money is created at sign-up at 0 and has no opening balance. Its owner can't delete or rename it, pick it when recording, or put Income into it. A User has one across all Groups. A Placeholder Member has its own, because Kitty doesn't know that "Dad" in two Groups is one person.

## Opening balances and corrections

Sam pays 30 EUR on day 1 before he signs up, and enters his real Bank balance, 150 EUR, on day 3. If he then moves that payment from Untracked money to Bank, Bank would show 120, because the 150 already reflects it. So:

- An opening balance is a Posting against an Opening balance Ledger account. Its timestamp is when the Account was created and can't change. Its amount can.
- Every Transaction has a happened-at timestamp. It defaults to the time of recording, and the User may edit it.
- When a payment that happened before an Account's opening timestamp moves to that Account, the same revision adds a correction Posting that gives the amount back. Bank stays at 150.
- Deleting the payment, or moving its happened-at across the opening, cancels or adds the correction in the next revision. Statistics ignore corrections, because a correction is not a Share.

Every balance stays a sum of Postings on one Ledger account, as in ADR 0004.

## Considered Options

- The Payer's Default Account pays. Kitty would guess which Account moved money, and for a Placeholder Member there is nothing to guess.
- Filtering Account balances at read time, so Postings dated before the opening don't count. It needs no correction rows, but it breaks ADR 0004's single balance rule and has to guess payments on the opening day.
- Asking the User about same-day payments. Comparing happened-at with the opening timestamp decides every case. A User who leaves happened-at wrong fixes it with an edit.
- Transfers into and out of Untracked money. A Transfer dated after the opening would count a pre-sign-up payment twice.

## Consequences

- A correction is decided at write time. Changing an opening timestamp would mean recomputing every correction, so it is fixed. A User who wants a different start creates a new Account.
- Notifications for a payment recorded on someone's behalf must say it went to Untracked money and ask them to pick an Account.
