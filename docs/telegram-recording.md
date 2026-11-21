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

- A Category, Account, or Group is named by its name, matched on its letters and digits only, in any script, ignoring case and accents. So `eatingout`, `eating-out`, and `Eating_Out` all match "Eating out". A name with spaces is typed without them or with `-`, because a space ends an argument.
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

- **Account**: the sender's own side, paying or receiving, uses their Default Account unless `from:` names another. Income goes into the sender's Default Account unless the command names another. A User with no Account yet has no Default Account. Their own side then uses their Untracked money, and Income gets a problem that asks them to make an Account first. Another Member's side, such as a Payer named with `by:` or the receiver of a Loan or Settlement, goes to that Member's Untracked money. That Member moves it to one of their Accounts by an edit (ADR 0012).
- **Group in a group chat**: always the Group linked to the chat. An `in:` naming another Group is a problem that points to the private chat. A chat with no linked Group gets a reply asking someone to link it.
- **Group in a private chat**: `in:` wins. Without it, the Group is the one of the sender's Groups that holds every Member the command names (`split:`, `by:`, the borrower, or the payee). If no other Member is named, it is the sender's Personal Group. Zero or two or more matching Groups is a problem that lists the candidates.
- **Settlement balance**: without `shared` or `loan`, the bot reads the balances between the two Members. If only one of them is non-zero, it uses that one. If both are zero, it uses shared. If both are non-zero, it is a problem. The receipt always says which balance it used.

### Chat rules

- `/transfer` and `/income` in a group chat record nothing. The bot replies with an "Open private chat" button.
- A receipt in a group chat leaves out the Account, because other Members never see a Member's Accounts. When the sender is the Payer, they get a private line, "Paid from your Cash (default)". When another Member's money moved, that Member gets a Notification that it went to their Untracked money, and it lets them pick an Account. A receipt in the private chat names the Account.
- Command menus differ by chat type. In private chats: `/expense /transfer /income /loan /settle /balance /stats /group /member /account /help /privacy /export /erase`. In group chats: `/expense /loan /settle /balance /stats /group /member /help`.

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
| `/expense 36.90 groceries by:@ben` | Ana records that Ben paid. The money comes from Ben's Untracked money. Ben gets a Notification and moves it to one of his Accounts by an edit. Only Ben picks his Account. |
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
| `/loan 50 @ben` | In the Flat 3B group chat: Ana lent Ben €50 from Cash. The money goes to Ben's Untracked money until he picks an Account. |
| `/loan 50 @ben from:bank` | The same Loan, from Bank. |
| `/loan 50 me by:@ana` | Sent by Ben: Ana lent Ben €50. The money comes from Ana's Untracked money and goes to Ben's Default Account. |
| `/loan 50 @cara` | In Ana's private chat: the Loan goes to Flat 3B, the only Group Ana shares with Cara. |
| `/loan 50 @ben in:lisbon` | In Ana's private chat: the Loan goes to Lisbon trip. Without `in:`, it is a problem, because Ana shares two Groups with Ben. |

**Settlement**

| Command | What it records |
|---|---|
| `/settle 23.40 @ana shared` | Sent by Ben: Ben paid Ana €23.40 from his Default Account, against the Shared balance. The money goes to Ana's Untracked money. |
| `/settle 23.40 @ana` | The same, if only one balance between them is non-zero. If Ben owes Ana on both, the bot sends a problem asking for `shared` or `loan`. |
| `/settle 50 @ana loan from:bank` | Sent by Ben: Ben paid Ana €50 from Bank, against the Loan balance. The money goes to Ana's Untracked money. |
| `/settle 23.40 me shared by:@ben` | Sent by Ana: Ana records that Ben paid her. The money comes from Ben's Untracked money and goes to Ana's Default Account. |
| `/settle 15 @cara shared in:flat3b` | In Ana's private chat: Ana paid Cara €15 in Flat 3B. The money goes to Cara's Untracked money. |

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

## Managing Groups, Members, and Accounts

Decided in [How Members manage Groups and Accounts](https://github.com/potluck-labs/kitty/issues/37). These commands follow the grammar above: fixed arguments, then `key:value` options, and names matched as in "Names".

### Commands

Three commands each take an action word. A command with no action word shows a list.

| Command | What it does | Where |
|---|---|---|
| `/group` | Shows the Group, its Members, and its chat. | Both |
| `/group new <name>` | Makes a Group with the sender as its only Member. | Private |
| `/group rename <name>` | Renames the Group. | Both |
| `/group invite` | Shows the Group's Invite code as a join link or a Join button. | Both |
| `/group link` | Sends an "Add to a group chat" button. | Private |
| `/group unlink` | Removes the link between the Group and its chat. | Both |
| `/member add <name>` | Adds a Placeholder Member. | Both |
| `/member rename <name> <new name>` | Renames a Placeholder Member. `/member rename me <new name>` renames the sender. | Both |
| `/member remove <name>` | Removes a Placeholder Member that has no Postings. | Both |
| `/member claim <name>` | Issues a single-use Invite code for a Claim of that Placeholder Member. | Both |
| `/account` | Lists the sender's Accounts with their balances. | Private |
| `/account new <name> [opening balance]` | Makes an Account. The opening balance defaults to 0. | Private |
| `/account rename <name> <new name>` | Renames an Account. | Private |
| `/account opening <name> <amount>` | Changes the amount of an Account's opening balance. | Private |
| `/account default <name>` | Makes an Account the Default Account. | Private |
| `/account delete <name>` | Deletes an Account. | Private |

- A new name runs to the end of the line or the next `key:`, so `/group new Lisbon trip` works. An existing name is one word, typed as in "Names".
- An opening balance may be zero or below zero, as for a credit card. This is the only amount that may be.
- `/account` in a group chat records nothing. The bot replies with an "Open private chat" button.
- **Which Group**: in a group chat, the command always acts on the linked Group, and `in:` gets the same problem as for recording. In the private chat, `in:` names the Group. Without `in:`, the Group is the sender's only Group besides their Personal Group. If they have none or several, it is a problem that lists them.

### Signing up and the first Account

- At sign-up, Kitty makes the User, their Personal Group, and their Untracked money. It makes no other Account, and it asks no setup questions.
- Until a User makes an Account, they have no Default Account. Their own side of a Transaction uses Untracked money, and the receipt says so. Income gets a problem that asks them to make an Account first.
- A User's first Account becomes their Default Account. Moving earlier payments into it is a normal edit, and ADR 0012's correction keeps its balance right.

### Accounts

- Account names are unique among one User's Accounts. "untracked" is reserved for Untracked money.
- Changing an opening balance changes its amount only. Its timestamp stays the time the Account was made (ADR 0012).
- Any Account except Untracked money can be the Default Account.
- An Account can be deleted only while its only Posting is the opening balance. The Default Account and Untracked money can never be deleted.

### Groups from a group chat

- In a chat with no linked Group, `/start@kitty_bot` makes a Group named after the chat title. Kitty links the chat to it, and the sender becomes its first Member. The bot posts a Join button.
- If the title is "Personal", or has no letters or digits (such as "🏠🏠" or "!!!"), the Group is named "Group chat". The reply asks for a `/group rename`.
- The sender must be a User. Otherwise Kitty makes nothing, and the bot replies with a "Sign up first" button. Messages from anonymous admins are ignored.
- In a chat that is already linked, `/start@kitty_bot` posts the Join button again.

### Joining with an Invite code

- Each Group has one Invite code, and many people can use it. A Channel shows it as a link: the Join button and the join link are the deep link `t.me/kitty_bot?start=<code>`.
- The link opens the private chat. A User joins at once. A new person first sees the privacy notice and taps "I agree". The Group gets a Notification.
- An Invite code for a Claim works once and is tied to one Placeholder Member (ADR 0011). `/member claim` always sends it to the sender's private chat. In a group chat, the bot posts "Sent to you privately".
- Expiry, cancelling a code, and leaving a Group are not decided yet.

### Linking a Group to a group chat

- A chat is linked to at most one Group, and a Group to at most one chat in each Channel. A link to a chat or Group that already has one gets a problem that says to send `/group unlink` first.
- Any Member may link or unlink. Every Member gets a Notification.
- `/group link` sends an "Add to a group chat" button, the deep link `t.me/kitty_bot?startgroup=<code>`. The Member picks a chat, Telegram adds the bot if needed, and the bot gets `/start@kitty_bot <code>` there and links the chat. The code works once, stops working after 1 hour, and a new `/group link` cancels it.
- Removing the bot from the chat keeps the link. If the bot is added again, the chat works with its Group at once.
- A Group's own id never changes. The link stores the Telegram chat id as a value that can change. When a group becomes a supergroup, the bot sends the backend the new chat id. It does this when it sees `migrate_to_chat_id` or `migrate_from_chat_id`, or when a send fails with `migrate_to_chat_id`. After a failed send, it also sends again.

### Group names

- Group names need not be unique. A User who joins a second Group with the same name as one of theirs gets a warning that `in:` matches both, and a pointer to `/group rename`.
- Any Member may rename a Group. Every Member gets a Notification.
- A Group name doesn't follow the chat title after the Group is made.
- "Personal" is reserved for Personal Groups. A Personal Group can't be renamed, has no Invite codes or Placeholder Members, and can't be linked to a chat.

### Member names

- A name belongs to the Member, so one User can be "Ben" in one Group and "Dad" in another.
- A User who joins is named after their Telegram first name. Later changes in Telegram don't change it.
- Names are unique among a Group's current Members. If the first name is taken, Kitty adds the first letter of the last name, as in "Ben N.". If that is taken too, or there is no last name, Kitty adds a number, as in "Ben 2". The bot tells the new Member how to rename.
- `me` and names starting with `@` are reserved.
- Only a User renames their own Member. Any Member renames a Placeholder Member (ADR 0011).
- After a Claim, the Member keeps the Placeholder Member's name until its User renames it.
- A former Member's name is free for others to use.

### Examples

Ana, Ben, and Cara are Users. Dad is a Placeholder Member.

**Signing up**

| Step | What happens |
|---|---|
| Ana starts the bot and taps "I agree" | Kitty makes her User, her Personal Group, and her Untracked money. She has no Default Account yet. |
| `/expense 4.50 coffee` | Recorded in Personal, from Untracked money. The receipt says: "From Untracked money. Add your Accounts with `/account new`." |
| `/income 2400` | A problem: "Make an Account first." |
| `/account new bank 150` | Bank opens at €150. It is her first Account, so it becomes her Default Account. |
| Ana edits the coffee to `from:bank` | Bank still shows €150. The coffee happened before Bank opened, so Kitty adds a correction (ADR 0012). |
| `/expense 3 coffee` | From Bank, her Default Account. |

**Accounts, in Ana's private chat**

| Command | What happens |
|---|---|
| `/account` | Lists her Accounts with their balances, and marks Bank as the Default Account. |
| `/account new cash` | Cash opens at 0. |
| `/account new visa -320` | Visa opens at −€320, a credit card debt. |
| `/account opening cash 40` | Cash's opening balance becomes €40. The time it opened stays the same. |
| `/account rename visa credit-card` | Visa becomes "Credit card". |
| `/account default cash` | Cash becomes the Default Account. |
| `/account default untracked` | A problem: Untracked money can't be the Default Account. |
| `/account new Bank` | A problem: Ana already has an Account named Bank. |
| `/account new untracked` | A problem: the name is reserved. |
| `/account delete credit-card` | Deleted, because it has only its opening balance. |
| `/account delete bank` | A problem: Bank has Transactions. |
| `/account delete cash` | A problem: Cash is the Default Account. |
| `/account` in a group chat | Records nothing. The bot replies with an "Open private chat" button. |

**A Group from a group chat**

| Step | What happens |
|---|---|
| Ana adds the bot to the chat "Flat 3B" and sends `/start@kitty_bot` | Kitty makes Flat 3B and links the chat. Ana is its first Member. The bot posts "Flat 3B is ready" with a Join button. |
| Ben taps Join | His private chat opens, and he joins at once. The group chat gets "Ben joined Flat 3B". |
| Cara taps Join | She is new to Kitty. She sees the privacy notice, taps "I agree", and joins. |
| Ben taps Join again | "You are already in Flat 3B." |
| Someone sends `/start@kitty_bot` again | The bot posts the Join button again, with the same Invite code. |
| Dan, who is not a User, sends `/start@kitty_bot` in a new chat | Kitty makes nothing. The bot replies with a "Sign up first" button. |
| An anonymous admin sends `/start@kitty_bot` | Ignored. |
| Ana sends `/start@kitty_bot` in a chat titled "🏠🏠" | Kitty makes a Group named "Group chat". The reply asks for a `/group rename`. |

**A Group from the private chat**

| Step | What happens |
|---|---|
| `/group new Lisbon trip` | Kitty makes Lisbon trip with Ana as its only Member. The bot replies with a join link to share and an "Add to a group chat" button. |
| Ben taps the join link Ana sent him | He joins Lisbon trip. |
| `/group invite` | If Lisbon trip is Ana's only Group besides Personal, the bot shows its join link. If she is also in Flat 3B, it is a problem that lists both, and she sends `/group invite in:lisbon`. |

**Linking a Group to a chat**

| Step | What happens |
|---|---|
| `/group link in:lisbon` | The bot replies with an "Add to a group chat" button. The button works once, for 1 hour. |
| Ana taps it and picks the chat "Portugal 🇵🇹" | The chat now records for Lisbon trip. Every Member gets a Notification. |
| Ben taps a forwarded copy of the same button | A problem: the button has already been used. |
| Ana links Lisbon trip to a second chat | A problem: Lisbon trip already has a chat. Send `/group unlink` first. |
| Ana links Flat 3B to "Portugal 🇵🇹" | A problem: this chat already records for Lisbon trip. |
| Cara removes the bot from "Flat 3B", then adds it again | The chat still records for Flat 3B. Kitty makes no new Group. |
| `/group unlink` in the "Flat 3B" chat | The link is removed. Flat 3B and its Transactions stay. |

**When Telegram changes the chat id**

| Step | What happens |
|---|---|
| The chat "Flat 3B" becomes a supergroup | The bot sends the new chat id to the backend, which updates the link. Flat 3B's own id doesn't change. |
| Someone taps Edit on an old receipt | It works as before. |
| A send to the old chat id fails | The bot updates the link and sends again. |

**Group names**

| Step | What happens |
|---|---|
| `/group rename Flat 3C` in the Flat 3B chat | Renamed. Every Member gets a Notification. |
| Someone changes the chat title to "Flat 3C 🎄" | The Group name stays "Flat 3C". |
| Ana, already in "Flat", joins Ben's new Group "Flat" | The join works. The bot warns: "You now have two Groups named Flat, so `in:flat` matches both. Rename one with `/group rename`." |
| `/expense 20 groceries in:flat` in Ana's private chat | A problem: two Groups match. The reply lists both. |
| `/group new Personal` | A problem: "Personal" is reserved. |
| `/group rename Home in:personal` | A problem: a Personal Group can't be renamed. |
| `/group invite in:personal` | A problem: no one can join a Personal Group. |
| `/group link in:personal` | A problem: a Personal Group can't have a chat. |

**Member names**

| Step | What happens |
|---|---|
| Ben Kowalski joins Flat 3B | His Member name is "Ben". |
| Ben Novak joins | His Member name is "Ben N.". The bot tells him: "Change it with `/member rename me <name>`." |
| A third Ben, with no last name, joins | His Member name is "Ben 2". |
| Ben Novak sends `/member rename me Benny` | He is "Benny" in Flat 3B. His names in other Groups don't change. |
| Cara sends `/member rename me Ben` | A problem: Flat 3B already has a Ben. |
| Cara sends `/member rename ben Bob` | A problem: only Ben may rename Ben. |
| Ben changes his Telegram name to "Benji" | His Member name stays "Ben". |
| `/member rename me @cara` or `/member rename me me` | A problem: these names are reserved. |

**Placeholder Members, in the Lisbon trip chat**

| Command | What happens |
|---|---|
| `/member add Dad` | Dad is added as a Placeholder Member. |
| `/expense 60 eating-out by:dad` | Dad paid, from his own Untracked money. |
| `/member rename dad Father` | Renamed. Any Member may do this. |
| `/member remove father` | A problem: Father has Postings. |
| `/member claim father` | The bot sends Ana a single-use Invite code in her private chat and posts "Sent to you privately". Dad joins with it and becomes that Member. Every Member gets a Notification, and the Member stays "Father" until Dad renames it. |
| `/member add Dad in:personal` | A problem: a Personal Group has no Placeholder Members. |

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
