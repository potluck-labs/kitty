# Erasure hands shared Postings to a placeholder Member

An Expense that Ana paid and Ben shared is personal data of both, and Ben's balances and the Group's statistics depend on it. So Erasure removes what belongs to the User alone, and hands what other Members depend on to a placeholder Member in each Group that still has another User. The rule: nothing shared that affects another Member's balance or Share is deleted. It stays as it is or moves to the placeholder.

- Removed: the User, their Channel identities, Accounts with names and opening balances, Transfers, Income, pending Notifications, the accepted notice record, and every Group with no other User left, including personal ledgers.
- Moved to the placeholder: the Member row, the User's own Postings on shared Transactions, and every "recorded by" and revision-author reference. Payer Postings move to the placeholder's Untracked money (ADR 0012).
- Kept unchanged: other Members' Postings, and the amounts, dates, Categories, and Merchants of shared Transactions.

The placeholder is an ordinary placeholder Member with no special kind, so edits, Group statistics, and claiming work the same as for any placeholder. It is named "Former member N" unless the User chooses at Erasure to keep their name. The remaining Members get a Notification that suggests a rename, with no name filled in, and may rename it. Kitty keeps no link to the erased person. A returning person signs up as a new User and claims placeholders when Group Members invite them.

Erasure runs right away after a confirm step that offers an export and shows open balances with an offer to record a Settlement. Open balances never block it. Inactivity Erasure after 24 months uses the same code path.

## Consequences

- Moving and removing Postings is the one exception to append-only Postings in ADR 0004.
- A restore from backup must re-run Erasures from the last 30 days. A deletion log keeps the internal ids of erased Users for that long, with no names.
- Keeping shared rows rests on the remaining Members' legitimate interest in a correct ledger. The privacy notice must say from the start that shared Transactions stay under a placeholder. Whether this is a lawful answer to an erasure request is a point for legal review.

## Considered Options

- Deleting every Transaction that names the User. It is the cleanest erasure, but it changes other Members' balances and leaves gaps in their history.
- Deleting only the User's own Postings and keeping the other Members' mirrored lines. Per-User books keep the others' balances correct, but the Group's Category totals and "paid" column drop by the User's part, and an edit has no Payer side to cancel.
- Refusing Erasure while balances are open. The GDPR has no "settle first" exception.
- A grace period before Erasure runs. It adds an account state and a scheduled job, and the export already covers a change of mind.
- Claim codes in the export so a returning person can get their placeholders back. It keeps a link to the erased person for a small gain.
