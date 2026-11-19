# Recording Transactions in Telegram

The MVP records Transactions with commands. A later release switches to "line, then card". This file lists the rules the MVP bot follows so that the switch changes only the bot.

Decided in [How recording an Expense feels in Telegram](https://github.com/potluck-labs/kitty/issues/12). The prototype with all four variants is `prototypes/telegram-recording/index.html` on the branch `prototype/telegram-recording`. Open it in a browser.

## MVP: commands

- One command per kind: `/expense`, `/transfer`, `/income`, `/loan`, and `/settle`.
- A command has fixed arguments and `key:value` options, for example `/expense 36.90 groceries at:lidl from:bank`.
- The bot saves at once and replies with a receipt.
- A command it can't read gets a usage reply, and the User sends the command again.
- The bot keeps no draft. The `Idempotency-Key` is `tg:<chat_id>:<message_id>`.

The exact grammar and its defaults are decided in [Command grammar and defaults for recording in the MVP](https://github.com/potluck-labs/kitty/issues/26).

## MVP command grammar

### Commands

Fixed arguments come first, in order. Options follow in any order. An option's value runs until the next `key:` or the end of the line, so `at:Pizza Hut from:bank` works. `note:` comes last, because it takes the rest of the line. Option keys ignore case. In groups, both `/expense` and `/expense@kitty_bot` work.

| Command | Fixed arguments | Options |
|---|---|---|
| `/expense` | `<amount> <category>` | `at:`, `by:`, `from:`, `split:`, `in:`, `note:` |
| `/transfer` | `<amount> <from-account> <to-account>` | `note:` |
| `/income` | `<amount> [account]` | `note:` |
| `/loan` | `<amount> <borrower>` | `by:`, `from:`, `in:`, `note:` |
| `/settle` | `<amount> <payee> [shared\|loan]` | `by:`, `from:`, `in:`, `note:` |

- `at:` is the Merchant, free text stored as typed.
- `by:` is the Member whose money moved: the Payer of an Expense, the lender of a Loan, or the payer of a Settlement. It defaults to the sender. With `by:`, a Member can record that someone else paid, for example `/loan 50 me by:@ana` for "Ana lent me 50".
- `from:` names one of the sender's own Accounts, and it is allowed only when `by:` is the sender. Only the Payer picks their own Account.
- `in:` names a Group.
- `note:` is a Note of up to about 200 characters.

### Amounts

- The decimal mark is `.` or `,`, followed by one or two digits: `36.90`, `36,90`, `36`.
- An optional `€` may come before or after the amount.
- There are no thousands separators. `1,000` and `1.000` get a usage reply that says to write `1000`.
- The amount must be above zero.

### Names

- A Category, Account, or Group is named by its name, matched ignoring case, accents, spaces, and punctuation. So `eatingout`, `eating-out`, and `Eating_Out` all match "Eating out". A name with spaces is typed without them or with `-`, because a space ends an argument.
- An exact match wins. Otherwise a prefix that matches exactly one item wins, so `eat` works.
- A Member is named by their Telegram @username, by a tap-to-mention for people with no username, or by their name, matched the same way. `me` is the sender. The Member must be in the Group the command goes to.
- Zero matches, or two or more, is a problem. The usage reply lists the candidates.
- A Category must already exist. An unknown name gets a problem with the closest candidates, and the bot never creates it.

### Splits

- With no `split:`, every Member of the Group shares equally.
- `split:@ana,@ben` is equal between the Members listed.
- `split:@ana=20,@ben=16.90` gives exact amounts, which must add up to the amount.
- `split:@ana=60%,@ben=40%` gives whole-number percentages, which must add up to 100.
- Mixing amounts and percentages, or listing one Member twice, is a problem. The Payer is never added on their own.

### Defaults

- **Account**: the Payer's Default Account pays, and Income goes into the sender's Default Account, unless the command names another.
- **Group in a group chat**: always the Group linked to the chat. An `in:` naming another Group is a problem that points to the private chat. A chat with no linked Group gets a reply asking someone to link it.
- **Group in a private chat**: `in:` wins. Without it, the Group is the one of the sender's Groups that holds every Member the command names (`split:`, `by:`, the borrower, or the payee). If no other Member is named, it is the sender's Personal Group. Zero or two or more matching Groups is a problem that lists the candidates.
- **Settlement balance**: without `shared` or `loan`, the bot reads the balances between the two Members. If only one of them is non-zero, it uses that one. If both are zero, it uses shared. If both are non-zero, it is a problem. The receipt always says which balance it used.

### Chat rules

- `/transfer` and `/income` in a group chat record nothing. The bot replies with an "Open private chat" button.
- A receipt in a group chat leaves out the Account, because other Members never see a Member's Accounts. The Payer gets a private line, "Paid from your Cash (default)". A receipt in the private chat names the Account.
- Command menus differ by chat type. In private chats: `/expense /transfer /income /loan /settle /balance /stats /help /privacy /export /erase`. In group chats: `/expense /loan /settle /balance /stats /help`.

### Edit and Delete

Every receipt has Edit and Delete buttons, and they stay for good. The backend checks the presser's rights and the current revision on every press. A press by someone who may not change the Transaction gets a popup only they see.

- **Edit**: the bot replies with the Transaction written out as a command the User can tap to copy, and asks for a reply with the corrected command. The reply replaces the whole Transaction as a new revision, sent with `If-Match`. It must be the same kind of Transaction. The reply is a new message, so it has its own `Idempotency-Key`. The bot edits the original receipt to show the new revision.
- **Delete**: the receipt changes in place to "Delete this? Yes / No". Yes changes it to "Deleted: …" with no buttons.

### Examples

Ana is in three Groups: Personal, Flat 3B (with Ben and Cara), and Lisbon trip (with Ben). Her Accounts are Cash, her Default Account, and Bank. A Telegram username is written `@ana`.

**Expense, in the Flat 3B group chat**

| Command | What it records |
|---|---|
| `/expense@kitty_bot 36.90 groceries at:lidl` | Ana paid €36.90 from Cash. The Category is Groceries and the Merchant is Lidl. Ana, Ben, and Cara each have a €12.30 Share. |
| `/expense 36,90 groceries at:lidl` | The same Expense. A decimal comma works, and so does a plain `/expense` when the bot reaches it. |
| `/expense 36.90 groceries at:lidl from:bank` | The same Expense, paid from Bank. The group receipt leaves out the Account, and Ana gets "Paid from your Bank" privately. |
| `/expense 36.90 eat at:Pizza Hut` | "eat" is a prefix that matches only Eating out. The Merchant "Pizza Hut" has a space, which works because a value runs until the next `key:`. |
| `/expense 36.90 groceries by:@ben` | Ana records that Ben paid. The Account is Ben's Default Account, and only Ben may change it. |
| `/expense 8 groceries split:@ben,@cara` | Ana paid €8. Ben and Cara each have a €4 Share, and Ana has none. |
| `/expense 36.90 groceries split:@ana=20,@ben=16.90` | Exact amounts: Ana has €20 and Ben has €16.90. |
| `/expense 50 utilities split:@ana=60%,@ben=40%` | Percentages: Ana has €30 and Ben has €20. |
| `/expense 12 household note:light bulbs for the hall` | An Expense with a Note. |

**Expense, in Ana's private chat**

| Command | Group |
|---|---|
| `/expense 4.50 coffee` | Personal, because no other Member is named. |
| `/expense 36.90 groceries in:flat3b` | Flat 3B, named with `in:`. |
| `/expense 20 eating-out split:@ana,@cara` | Flat 3B, the only Group Ana shares with Cara. |
| `/expense 20 eating-out split:@ana,@ben` | A problem: Ana shares both Flat 3B and Lisbon trip with Ben. The usage reply lists both Groups. |
| `/expense 20 eating-out split:@ana,@ben in:lisbon` | Lisbon trip. |

**Transfer and Income, in Ana's private chat only**

| Command | What it records |
|---|---|
| `/transfer 200 bank cash` | €200 from Bank to Cash. |
| `/transfer 200 bank cash note:for the market` | The same Transfer, with a Note. |
| `/income 2400 bank note:salary november` | €2,400 into Bank, with the Note "salary november". |
| `/income 50` | €50 into Cash, the Default Account. |

In a group chat, `/transfer` and `/income` record nothing. The bot replies with an "Open private chat" button.

**Loan**

| Command | What it records |
|---|---|
| `/loan 50 @ben` | In the Flat 3B group chat: Ana lent Ben €50 from Cash. The money goes to Ben's Default Account. |
| `/loan 50 @ben from:bank` | The same Loan, from Bank. |
| `/loan 50 me by:@ana` | Sent by Ben: Ana lent Ben €50 from Ana's Default Account. |
| `/loan 50 @cara` | In Ana's private chat: the Loan goes to Flat 3B, the only Group Ana shares with Cara. |
| `/loan 50 @ben in:lisbon` | In Ana's private chat: the Loan goes to Lisbon trip. Without `in:`, it is a problem, because Ana shares two Groups with Ben. |

**Settlement**

| Command | What it records |
|---|---|
| `/settle 23.40 @ana shared` | Sent by Ben: Ben paid Ana €23.40 from his Default Account, against the Shared balance. |
| `/settle 23.40 @ana` | The same, if only one balance between them is non-zero. If Ben owes Ana on both, the bot sends a problem asking for `shared` or `loan`. |
| `/settle 50 @ana loan from:bank` | Sent by Ben: Ben paid Ana €50 from Bank, against the Loan balance. |
| `/settle 23.40 me shared by:@ben` | Sent by Ana: Ana records that Ben paid her. |
| `/settle 15 @cara shared in:flat3b` | In Ana's private chat: Ana paid Cara €15 in Flat 3B. |

**Usage replies**

| Command | Reply |
|---|---|
| `/expense groceries 8` | The fixed arguments are in the wrong order. The usage reply shows `/expense@kitty_bot <amount> <category> [at:] [by:] [from:] [split:] [in:] [note:]` and an example. |
| `/expense 1,000 rent` | The amount is ambiguous. The reply says to write `1000`. |
| `/expense 36.90 lunch` | No Category is named "lunch". The reply lists the closest Categories. |
| `/expense 36.90 groceries by:@ben from:bank` | `from:` is allowed only when the sender is the Payer. |
| `/expense 36.90 groceries split:@ana=20,@ben=50%` | Amounts and percentages are mixed. |
| `/expense 36.90 groceries in:lisbon` | Sent in the Flat 3B group chat: this chat records for Flat 3B, so the reply points to the private chat. |

**Edit, in the Flat 3B group chat**

1. Ana sends `/expense 36.90 groceries at:lidl` and taps Edit on the receipt.
2. The bot replies with `` `/expense 36.90 groceries at:lidl` `` for her to copy.
3. Ana replies to that message with `/expense 39.60 groceries at:lidl`.
4. The bot saves revision 2 and edits the receipt to show €39.60.

## Later: line, then card

- The User types what they know, in any order, for example `37 lidl`.
- The bot replies with one draft card, which it edits in place. The card has buttons for each field that is missing or ambiguous.
- Nothing is saved until the User taps Save.
- The draft lives in aiogram's FSM storage in Redis, and its `Idempotency-Key` is a UUID created when the draft starts, as ADR 0005 already allows.

## Rules for the MVP bot

These rules keep the switch to "line, then card" a change inside `kitty-telegram-bot`.

1. The backend never parses text. The bot sends only complete domain requests, as ADR 0005 requires, so both variants send the same requests.
2. Recording runs in three steps, each in its own module.
   - The parser turns text into a draft. The draft holds the fields it found and a list of problems.
   - The resolver looks up names and fills in defaults.
   - The request builder turns a complete draft into the backend request.

   Commands and "line, then card" share all three steps. Only the reply to a draft with problems differs: commands send a usage reply, and the card shows buttons.
3. A problem is data, not text. Examples are a missing amount, an unknown Category "lunch", or "ben" matching Members in two Groups. Each problem carries its kind, the input that caused it, and the candidates the bot found.
4. The resolver returns every candidate for a name, not the first match. Commands treat zero or two or more candidates as a problem. The card shows them as buttons.
5. The draft is one typed model, such as a Pydantic model, that can be serialized. Commands build it and throw it away. The card stores it in Redis under the same shape.
6. One formatter per kind writes the receipt. The card uses the same formatter with a "not saved yet" line and buttons.
7. Every valid command stays valid after the switch. The new parser accepts each MVP command and turns it into the same draft, so people who learned the commands don't relearn anything.
8. Command words are plain words, not symbols. Words the new parser reads, such as `from`, `in`, `at`, and `for`, are not reused with another meaning.

## What the switch adds

- In the bot: the card renderer, button handlers, drafts in Redis with a time limit, and a check on every button press that the presser owns the draft.
- In the bot: a deep link that carries a draft id, so a Transfer or Income typed in a group can open as a card in the private chat. The id must fit the 64-character start parameter.
- In groups: plain lines reach the bot only with a command prefix such as `/k`, or when the bot is a group admin. [Privacy mode or group admin for the bot](https://github.com/potluck-labs/kitty/issues/25) decides which.
- In the backend, optional and additive within `/v1`: a dry-run call that returns the exact Shares. A preview can't show them otherwise, because the backend picks at random who gets a leftover cent.

## Not part of the switch

The prototype also showed editing a recorded line by editing your own Telegram message. This needs the backend to store a link from a Telegram message to a Transaction. Neither the MVP nor "line, then card" includes it.
