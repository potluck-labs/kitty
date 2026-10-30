# Kitty

Kitty records money that people spend, move, and earn, alone or in small groups, and shows who paid, who owes whom, and where the money went by category.

## People and groups

**User**:
A person in Kitty, independent of any chat app. A User owns Accounts and can be a Member of many Groups.
_Avoid_: Customer, Telegram user, profile

**Channel**:
An app a User talks to Kitty through, such as Telegram. A User links one or more Channel identities.
_Avoid_: Platform, client, frontend, messenger

**Group**:
A set of one or more people who record shared Transactions together. A Group with one person is a personal ledger.
_Avoid_: Team, household, room

**Member**:
A User's place in one Group.
_Avoid_: Participant, groupmate

**Invite code**:
A code Kitty issues so a User can join a Group. A Channel may show it as a link, but the code belongs to Kitty.
_Avoid_: Invite link, Telegram link

## Money holders

**Account**:
A place that holds one User's money, such as cash or a bank account, with an opening balance. It belongs to the User, not to a Group, so one Account can pay for Expenses in many Groups.
_Avoid_: Wallet, source, bank

## Transactions

**Transaction**:
Any recorded movement of money. Every Transaction is exactly one kind: Expense, Transfer, Income, Loan, or Settlement.
_Avoid_: Row, entry, cost, record

**Expense**:
Money spent, with an amount, a Payer, the Account it left, a Category, an optional Merchant, and a Split.
_Avoid_: Cost, purchase, spending

**Transfer**:
Money moved between two Accounts owned by the same User.
_Avoid_: Move, internal transaction

**Income**:
Money that enters an Account from outside the app, such as a salary.
_Avoid_: Deposit, earning

**Loan**:
Money one Member gives another in the same Group that is expected back. It is not spending, so it never counts toward Category totals.
_Avoid_: Lending, advance, credit

**Settlement**:
Money one Member pays another inside a Group to reduce either their Shared balance or their Loan balance, and it says which one. A loan Settlement reduces the overall Loan balance, not a specific Loan.
_Avoid_: Payback, reimbursement, repayment

## Balances

**Shared balance**:
What one Member owes another in a Group because of Expense Shares.
_Avoid_: Debt, tab

**Loan balance**:
What one Member owes another in a Group because of Loans. It is kept apart from the Shared balance.
_Avoid_: Loan debt, credit

## Expense details

**Payer**:
The Member whose Account paid for an Expense.
_Avoid_: Spender, buyer

**Split**:
How one Expense's amount is divided among Members.
_Avoid_: Portion, ratio, division

**Share**:
One Member's part of a Split, in money.
_Avoid_: Portion, cut

**Category**:
The label that says what an Expense was for, such as groceries or utilities.
_Avoid_: Type, tag

**Merchant**:
The store or business an Expense was paid to.
_Avoid_: Shop, vendor, store name
