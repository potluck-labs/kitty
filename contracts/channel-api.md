# Channel API catalogue

Every endpoint and Notification type between a Channel and the backend. Decided in [Endpoints and Notification types in the Channel API](https://github.com/potluck-labs/kitty/issues/23). The mechanism and conventions are ADR 0005, from [The contract between Channels and the backend](https://github.com/potluck-labs/kitty/issues/11).

This file is the source for `contracts/channel-api.yaml`. Writing the OpenAPI file from it is the first task of milestone 1, before any code. If the two disagree, fix this file first.

These tickets add their endpoints here later: [Leaving a Group, removing a Member, and the Invite code lifecycle](https://github.com/potluck-labs/kitty/issues/42), [Dates and time zones for Transactions and statistics](https://github.com/potluck-labs/kitty/issues/43), and [Closing an Account that has Transactions](https://github.com/potluck-labs/kitty/issues/44).

## Conventions

From ADR 0005, repeated here so this file reads on its own.

### Headers

| Header | On | Value |
|---|---|---|
| `Authorization` | Every request except `GET /health` | `Bearer <Channel API key>`. The key tells the backend which Channel calls. |
| `Kitty-Actor` | Every request except `GET /health` and `GET /v1/privacy-notice` | The acting Channel identity, such as the Telegram user id. |
| `Kitty-Actor-Username` | Every request that has `Kitty-Actor` | The actor's Telegram @username without the `@`. Left out if they have none. The backend stores the latest one on the Channel identity. |
| `Idempotency-Key` | Every `POST`, `PUT`, `PATCH`, and `DELETE` | `tg:<chat_id>:<message_id>` for a command, or a UUID for a button press or a draft. |
| `If-Match` | `PUT` and `DELETE` on a Transaction only | The revision number, from the `ETag`. |

### Values

- Ids are UUIDv7 strings. A Channel id, such as a Telegram chat id, is never a Kitty id.
- Amounts are integer cents in EUR. €36.90 is `3690`.
- Timestamps are RFC 3339 in UTC. Dates are `YYYY-MM-DD`.
- JSON fields use `snake_case`.
- A Member, Category, Account, or Group in a response is an object with `id` and `name`, so one call is enough to write one message.
- The bot ignores fields it doesn't know, and handles an unknown enum value, error code, or Notification type with a generic fallback and a log warning.

### Lists

Every list in the MVP is small and bounded, so it returns all items in one response. Cursor pagination is added only to a list that can grow without limit, such as a later list of Transactions. Notifications use `limit` and a lease.

### Names

- Groups, Members, and Accounts: at least one letter or digit, at most 64 characters.
- Categories: at least one letter or digit, at most 32 characters.
- Names are unique under the matching rule below, not as typed.

### Errors

Every error is RFC 9457 Problem Details with a stable `code`.

| Status | `code` | Meaning |
|---|---|---|
| 400 | `invalid_request` | The body or parameters don't match the schema. A bug in the Channel. |
| 401 | `invalid_api_key` | The API key is missing or wrong. |
| 403 | `actor_not_registered` | No User has this Channel identity. The bot shows the privacy notice. |
| 403 | `notice_not_accepted` | The User hasn't accepted the current notice. Carries `current_version`. Every endpoint returns it except reading and accepting the notice, the export, and Erasure. A User who refuses new terms can still take their data and leave. |
| 403 | `not_permitted` | The User can see it but may not do this. |
| 404 | `not_found` | It doesn't exist, or the User can't see it. |
| 409 | `request_in_progress` | A request with this `Idempotency-Key` is still running. |
| 409 | one code per rule | A domain rule refuses the request. Each endpoint lists its codes. |
| 412 | `revision_conflict` | The `If-Match` revision is not the current one. Carries `current_revision`. |
| 422 | `idempotency_key_reused` | The same `Idempotency-Key` came with a different body. |
| 422 | `validation_failed` | Carries `errors: [{field, code}]`. Each endpoint lists its field codes. |
| 428 | `if_match_required` | A Transaction `PUT` or `DELETE` has no `If-Match`. |

The idempotency check runs before the revision check. "Errors" under each endpoint below lists only the codes added to this table.

## Name matching

The bot matches typed names against `GET /v1/me/directory`. The backend uses the same rule to keep names unique. Both services must give the same result for every case below, and each service's CI runs this table.

To turn a name into its match key:

1. Apply Unicode NFKD.
2. Remove every character in category `M` (marks).
3. Lower-case with the Unicode default mapping: `str.lower()` in Python, `toLowerCase()` in TypeScript.
4. Keep only characters in categories `L` (letters) and `N` (numbers).

Two names match when their keys are equal. An empty key means the name has no letter or digit.

| Input | Key | Note |
|---|---|---|
| `Eating out` | `eatingout` | |
| `eating-out` | `eatingout` | |
| `Eating_Out` | `eatingout` | |
| `Café` | `cafe` | Accents are ignored. |
| `CAFÉ` | `cafe` | |
| `Ben N.` | `benn` | |
| `Ｌｉｄｌ` | `lidl` | Full-width letters fold to plain ones. |
| `Москва` | `москва` | Any script works. |
| `Straße` | `straße` | `ß` is kept. `Strasse` doesn't match it. |
| `ΟΔΟΣ` | `οδος` | Both languages turn the final `Σ` into `ς`. |
| `ガス` | `カス` | NFKD splits the voiced mark off, and step 2 removes it. |
| `١٢٣` | `١٢٣` | Arabic-Indic digits stay as they are. `123` doesn't match them. |
| `Flat 3B` | `flat3b` | |
| `🏠🏠` | (empty) | No letter or digit. |
| `!!!` | (empty) | |

A Telegram @username matches without the `@` and ignoring case. A Member with a username can be named by the username or by their Member name.

## Health

### `GET /health`

No API key, no `/v1`. Returns `200` with an empty body when the backend can reach its database, and `503` otherwise. Healthchecks.io and the deploy step use it.

## Sign-up and the privacy notice

### `GET /v1/privacy-notice`

Needs the API key but no `Kitty-Actor`. Returns `{version, markdown}`, the current notice. The text has one source in `kitty` and ships with the backend.

### `POST /v1/users`

Signs up the `Kitty-Actor`. Request: `{notice_version}`. Creates the User, the Channel identity with its username, the Personal Group with its 11 default Categories, and Untracked money. It creates no other Account.

Response `201`: `{user_id, personal_group: {id, name}}`.

Errors: `409 already_registered`. `422 validation_failed` with `notice_version_outdated` when the version is not the current one.

### `POST /v1/me/notice-acceptances`

Accepts a new notice version. Request: `{version}`. Response `204`.

Errors: `422 validation_failed` with `notice_version_outdated`.

## The User's own data

### `GET /v1/me/directory`

Everything the sender can name, for the bot's resolver. One call per command.

```json
{
  "user_id": "…",
  "groups": [
    {
      "id": "…",
      "name": "Flat 3B",
      "is_personal": false,
      "has_chat": true,
      "members": [
        { "id": "…", "name": "Ana", "username": "ana", "is_me": true, "is_placeholder": false },
        { "id": "…", "name": "Dad", "username": null, "is_me": false, "is_placeholder": true }
      ],
      "categories": [ { "id": "…", "name": "Groceries" } ]
    }
  ],
  "accounts": [
    { "id": "…", "name": "Bank", "is_default": true, "is_untracked": false },
    { "id": "…", "name": "Untracked money", "is_default": false, "is_untracked": true }
  ]
}
```

- `members` lists current Members only. Former Members can't be named.
- `username` is set only for a User Member who has one.
- `accounts` has no balances. Balances come from `GET /v1/me/accounts`.

### `GET /v1/me/export`

Returns the export as one JSON document, which the bot sends as a file in the private chat. It holds `format_version`, the User, their Channel identities with usernames, their Accounts with balances, every Transaction they recorded or are a Party to with all revisions, their Group memberships with the other Members' names, their computed balances, and the accepted notice version.

### `POST /v1/me/erasure`

Runs Erasure at once. Request: `{keep_name}`, `false` by default. Response `204`. Open balances never block it. The bot does the export offer, the list of open balances from `GET /v1/me/balances`, and the confirm step first.

Writes `member_erased` to each Group the User leaves behind.

## Channel chats

`{chat_id}` is the Channel's own chat id. The API key tells the backend which Channel, so a chat id only has to be unique inside one Channel.

### `GET /v1/chats/{chat_id}`

Returns the linked Group: `{chat_id, group: {id, name}}`.

Errors: `404 chat_not_linked`.

### `POST /v1/chats`

Links a chat to a Group with the one-time code from `/group link`. Request: `{link_code, chat_id, title}`. The sender must be a current Member of the code's Group. Response `201`: `{chat_id, group: {id, name}}`.

Writes `chat_linked`.

Errors: `404 link_code_unknown`. `409 link_code_used`, `link_code_expired`, `chat_already_linked`, `group_already_linked`. `403 not_permitted`.

### `PUT /v1/chats/{chat_id}`

Moves a link to a new chat id after a supergroup migration. Request: `{new_chat_id}`. Response `200` with the link. The Group's id doesn't change.

Errors: `404 chat_not_linked`.

### `DELETE /v1/chats/{chat_id}`

Removes the link. Any current Member may do this. Response `204`. The Group and its Transactions stay.

Writes `chat_unlinked` to the chat before the link is removed.

Errors: `404 chat_not_linked`. `403 not_permitted`.

## Groups and Members

### `POST /v1/groups`

Makes a Group with the sender as its only Member and the 11 default Categories. Two forms:

- `{name}`, from `/group new`.
- `{chat: {chat_id, title}}`, from `/start@kitty_bot` in a chat with no link. The backend names the Group after the title, or "Group chat" if the title is "Personal" or has no letter or digit. It links the chat in the same step.

Response `201`: the Group as in `GET /v1/groups/{id}`, plus `name_from_fallback: true|false`.

Errors: `422 validation_failed` with `name_reserved` or `name_invalid`. `409 chat_already_linked`.

### `GET /v1/groups/{group_id}`

Returns `{id, name, is_personal, members: [{id, name, is_placeholder}], chat: {chat_id, title} | null}`. Only current Members may read it.

### `PATCH /v1/groups/{group_id}`

Renames the Group. Request: `{name}`. Any current Member. Response `200` with the Group.

Writes `group_renamed`.

Errors: `409 personal_group_restricted`. `422 validation_failed` with `name_reserved` or `name_invalid`.

### `GET /v1/groups/{group_id}/invite-code`

Returns `{code}`, the Group's one reusable Invite code. A Channel shows it as `t.me/kitty_bot?start=<code>`. Any current Member.

Errors: `409 personal_group_restricted`.

### `POST /v1/groups/{group_id}/chat-link-codes`

Returns `{code, expires_at}` for `t.me/kitty_bot?startgroup=<code>`. It works once, for 1 hour, and a newer one cancels it. Any current Member.

Errors: `409 personal_group_restricted`, `group_already_linked`.

Every code Kitty issues is at most 32 characters from `A-Z a-z 0-9 _ -`, so it fits Telegram's start parameter.

### `POST /v1/memberships`

Joins a Group with an Invite code, either the Group's code or a single-use Claim code. Request: `{invite_code, first_name, last_initial}`. `last_initial` is one letter or `null`. The backend builds the Member name from them ("Ben", then "Ben N.", then "Ben 2") and keeps neither.

With a Claim code, the sender takes over that Placeholder Member and its name.

Response `201`: `{group: {id, name}, member: {id, name}, claimed: true|false, warnings: [...]}`. A warning is `same_group_name` when the sender now has two Groups whose names match.

Writes `member_joined`, or `placeholder_claimed` for a Claim.

Errors: `404 invite_code_unknown`. `409 invite_code_used`, `already_member`.

### `POST /v1/groups/{group_id}/members`

Adds a Placeholder Member. Request: `{name}`. Any current Member. Response `201`: `{id, name, is_placeholder: true}`.

Errors: `409 name_taken`, `personal_group_restricted`. `422 validation_failed` with `name_reserved` (`me` or a name starting with `@`) or `name_invalid`.

### `PATCH /v1/groups/{group_id}/members/{member_id}`

Renames a Member. Request: `{name}`. A User renames only their own Member. Any current Member renames a Placeholder Member. Response `200`.

Errors: `403 not_permitted`. `409 name_taken`. `422 validation_failed` with `name_reserved` or `name_invalid`.

### `DELETE /v1/groups/{group_id}/members/{member_id}`

Removes a Placeholder Member. Any current Member. Response `204`. Removing a User Member is for [Leaving a Group, removing a Member, and the Invite code lifecycle](https://github.com/potluck-labs/kitty/issues/42).

Errors: `409 member_has_postings`, `not_a_placeholder`.

### `POST /v1/groups/{group_id}/members/{member_id}/claim-codes`

Issues a single-use Invite code tied to a Placeholder Member. Request: none. Any current Member. Response `201`: `{code}`. The bot always sends it to the sender's private chat.

Errors: `409 not_a_placeholder`.

## Categories

Any current Member of the Group. No Notification.

### `GET /v1/groups/{group_id}/categories`

Returns `[{id, name}]` in the Group's order.

### `POST /v1/groups/{group_id}/categories`

Request: `{name}`. Response `201`: `{id, name}`.

Errors: `409 name_taken`. `422 validation_failed` with `name_invalid`.

### `PATCH /v1/groups/{group_id}/categories/{category_id}`

Request: `{name}`. Response `200`. Every Expense in it shows the new name.

Errors: `409 name_taken`. `422 validation_failed` with `name_invalid`.

### `DELETE /v1/groups/{group_id}/categories/{category_id}`

Response `204`.

Errors: `409 category_in_use`, `last_category`.

## Accounts

Only the owner sees or changes an Account.

### `GET /v1/me/accounts`

Returns `[{id, name, balance, opening_balance, is_default, is_untracked}]`. Untracked money is in the list with its balance.

### `POST /v1/me/accounts`

Request: `{name, opening_balance}`. `opening_balance` defaults to 0 and may be below zero. Response `201` with the Account. The User's first Account becomes their Default Account, so `is_default` is `true`.

Errors: `409 name_taken`. `422 validation_failed` with `name_reserved` ("untracked") or `name_invalid`.

### `PATCH /v1/me/accounts/{account_id}`

Request: `{name?, opening_balance?}`. Changing the opening balance changes its amount only, never its timestamp. Response `200`.

Errors: `409 name_taken`, `untracked_money_restricted`. `422 validation_failed` with `name_reserved` or `name_invalid`.

### `DELETE /v1/me/accounts/{account_id}`

Response `204`. Allowed only while the only Posting is the opening balance.

Errors: `409 account_has_postings`, `default_account`, `untracked_money_restricted`.

### `PUT /v1/me/default-account`

Request: `{account_id}`. Response `200` with the Account.

Errors: `409 untracked_money_restricted`. `422 validation_failed` with `account_not_yours`.

## Transactions

One collection for all five kinds. Renames have no revisions, but every Transaction change does.

### Request bodies

Every kind has `amount`, `note` (optional, up to 200 characters), and `happened_at` (optional, defaults to now).

| `kind` | Fields |
|---|---|
| `expense` | `group_id`, `category_id`, `payer_member_id`, `account_id`, `split`, `merchant`, `origin_chat_id` |
| `loan` | `group_id`, `lender_member_id`, `borrower_member_id`, `account_id`, `origin_chat_id` |
| `settlement` | `group_id`, `payer_member_id`, `payee_member_id`, `balance` (`shared` or `loan`), `account_id`, `origin_chat_id` |
| `transfer` | `from_account_id`, `to_account_id` |
| `income` | `account_id` |

- `merchant` is optional free text, stored as typed.
- `origin_chat_id` is optional: the chat the command came from. The backend uses it only to decide `transaction_recorded`.
- The bot always sends `balance` for a Settlement. It picks it from the balances, as `docs/telegram-recording.md` says.

`split` is one of:

```json
{ "kind": "equal",   "member_ids": ["<Ana>", "<Ben>", "<Cara>"] }
{ "kind": "exact",   "shares": [ { "member_id": "<Ana>", "amount": 2000 }, { "member_id": "<Ben>", "amount": 1690 } ] }
{ "kind": "percent", "shares": [ { "member_id": "<Ana>", "percent": 60 }, { "member_id": "<Ben>", "percent": 40 } ] }
```

The backend computes the Shares in cents and gives any leftover cent to a random Member (ADR 0004).

### The `account_id` rule

`account_id` is always the sender's own side. Only the owner picks their Account.

- **Record**: when `account_id` is missing, the sender's side uses their Default Account, or Untracked money if they have none. Income with no Account is an error. Another Member's side always goes to that Member's Untracked money.
- **Edit**: when `account_id` is missing and the person on that side is unchanged, their Account stays. When the person changed, the new side follows the record rule.
- `account_id` must be the sender's own Account and not Untracked money.

### Response

`POST`, `GET`, and `PUT` return the same shape, and the `ETag` is the revision.

```json
{
  "id": "…",
  "kind": "expense",
  "revision": 1,
  "group": { "id": "…", "name": "Flat 3B" },
  "recorded_by": { "id": "…", "name": "Ana" },
  "recorded_at": "2026-11-21T09:14:03Z",
  "happened_at": "2026-11-21T09:14:03Z",
  "amount": 3690,
  "category": { "id": "…", "name": "Groceries" },
  "payer": { "id": "…", "name": "Ana" },
  "merchant": "lidl",
  "note": null,
  "split": { "kind": "equal", "member_ids": ["…", "…", "…"] },
  "shares": [
    { "member": { "id": "…", "name": "Ana" }, "amount": 1230 },
    { "member": { "id": "…", "name": "Ben" }, "amount": 1230 },
    { "member": { "id": "…", "name": "Cara" }, "amount": 1230 }
  ],
  "my_account": { "id": "…", "name": "Cash", "is_untracked": false },
  "can_change": true,
  "deleted_at": null
}
```

- `split` is the form that was sent, so the Edit button can write the command again. `shares` holds the computed cents.
- `my_account` is the sender's own side, or `null` if they have none. Other Members' Accounts never appear.
- A Loan has `lender` and `borrower`. A Settlement has `payer`, `payee`, and `balance`. A Transfer has `from_account` and `to_account`. An Income has `account`. Transfers and Income have no `group`.
- `can_change` tells the bot whether the sender may edit or delete it.

### `POST /v1/transactions`

Records a Transaction. Response `201`.

Writes `transaction_recorded` and `untracked_money_added` as the Notification rules say.

Errors: `403 not_permitted`: a Loan or Settlement where the sender is neither Member, or a Transaction in a Group they can see but no longer belong to. `422 validation_failed` with these field codes:

| Field code | When |
|---|---|
| `amount_not_positive` | The amount is zero or below. |
| `member_not_in_group` | A named Member is not a current Member of the Group. |
| `member_listed_twice` | A Split names a Member twice. |
| `split_does_not_add_up` | Exact Shares don't sum to the amount, or percentages don't sum to 100. |
| `category_not_in_group` | The Category belongs to another Group. |
| `account_not_yours` | `account_id` is not the sender's Account. |
| `account_is_untracked` | `account_id` is Untracked money. |
| `same_account` | A Transfer's two Accounts are the same. |
| `same_member` | A Loan or Settlement names one Member on both sides. |
| `no_account` | Income, and the sender has no Account yet. |
| `note_too_long` | The Note is over 200 characters. |

### `GET /v1/transactions/{id}`

Returns the current revision, including one that was deleted, with `deleted_at` set.

### `PUT /v1/transactions/{id}`

Replaces the whole Transaction as a new revision. Needs `If-Match`. The body is the same as for `POST` and must keep the same `kind`. Response `200`.

Writes `transaction_edited`, and `untracked_money_added` for a new person whose side went to Untracked money.

Errors: `403 not_permitted`. `409 transaction_deleted`. `412 revision_conflict`. `428 if_match_required`. `422 validation_failed` with the codes above, plus `kind_changed`.

### `DELETE /v1/transactions/{id}`

Deletes it. Needs `If-Match`. Response `204`.

Writes `transaction_deleted`.

Errors: `403 not_permitted`. `409 transaction_deleted`. `412 revision_conflict`. `428 if_match_required`.

There is no list of Transactions in the MVP. The export holds all of them.

## Statistics

Read-only, one endpoint per statistic and view. A Group endpoint is for current Members only. Anyone else gets `404 not_found`, under the later rule in [Who may record, edit, or delete a Transaction](https://github.com/potluck-labs/kitty/issues/22).

`from` is inclusive and `to` is exclusive, both as dates. Where a date starts is decided in [Dates and time zones for Transactions and statistics](https://github.com/potluck-labs/kitty/issues/43). `to` before or equal to `from` gets `422 validation_failed` with `period_invalid`.

| Endpoint | Returns |
|---|---|
| `GET /v1/me/accounts` | Account balances, as in Accounts. |
| `GET /v1/groups/{group_id}/balances` | `{lines: [{kind, debtor, creditor, amount}]}` for every pair in the Group. |
| `GET /v1/me/balances` | `{lines: [{kind, debtor, creditor, amount, group}]}`, the sender's lines with each Member. |
| `GET /v1/groups/{group_id}/category-totals?from=&to=` | `{from, to, total, rows: [{category, amount}]}` |
| `GET /v1/me/category-totals?from=&to=&group_id=` | `{from, to, total, rows: [{name, amount, category_ids}]}` |
| `GET /v1/groups/{group_id}/member-spending?from=&to=` | `{from, to, rows: [{member, paid, share}]}` |

- A balance line's `kind` is `shared` or `loan`. `amount` is always above zero, and `debtor` and `creditor` give the direction. Pairs that net to zero are left out, and an empty list means everyone is settled.
- Category rows have a nonzero amount and are sorted largest first. The backend returns every row, and the bot trims them.
- In the personal Category totals, Categories whose names match are one row, named after the Category with the largest amount. With `group_id`, every row is one Category.

## Notifications

### `GET /v1/notifications?limit=50`

Leases up to `limit` Notifications (1 to 50) for about 60 seconds and returns them. A Notification whose lease runs out comes back in a later pull.

```json
{
  "id": "…",
  "type": "transaction_edited",
  "recipient": { "kind": "user", "user_id": "…", "external_id": "<Telegram user id>" },
  "created_at": "2026-11-21T09:20:11Z",
  "data": { }
}
```

A Group recipient is `{kind: "group", group_id, external_chat_id}`.

### `POST /v1/notifications/acks`

Request: `{acks: [{id, result}]}`, where `result` is `delivered` or `unreachable`. Response `204`. The backend ignores an id that is already acked. An unreachable User gets no fallback.

### Who gets them

- A Placeholder Member or a former Member never gets a Notification.
- The actor never gets a Notification about their own action.
- **Group** below means: one Notification to the Group's linked chat. If the Group has no chat, one Notification to each User Member in their private chat instead.
- On one Transaction change, a User gets at most one Notification. `untracked_money_added` replaces `transaction_recorded` or `transaction_edited` for that User.
- `transaction_recorded` is written only when `origin_chat_id` is missing or is not the Group's linked chat. People in the linked chat already see the receipt there.

### Types

`data` holds a copy of everything the bot needs to write the message, so the bot needs no second call. A Transaction in `data` has the shape of `GET /v1/transactions/{id}` as the recipient would see it, so `my_account` and `can_change` are theirs.

| `type` | When | To | `data` |
|---|---|---|---|
| `transaction_recorded` | A new Transaction, by the rule above | Each Party except the recorder | `{transaction, my_share}` |
| `untracked_money_added` | A revision puts money in a User's Untracked money | That User | `{transaction, amount, direction: paid\|received, my_share}` |
| `transaction_edited` | An edit | The recorder of revision 1 and the current Parties | `{previous, current, edited_by}` |
| `transaction_deleted` | A delete | The same people | `{transaction, deleted_by}` |
| `member_joined` | A User joins | Group | `{group, member}` |
| `placeholder_claimed` | A Claim | Group | `{group, member}` |
| `group_renamed` | A Group rename | Group | `{group, old_name, new_name, renamed_by}` |
| `chat_linked` | A link | Group | `{group, chat_title, linked_by}` |
| `chat_unlinked` | An unlink | Group, sent to the chat before the link goes | `{group, chat_title, unlinked_by}` |
| `member_erased` | Erasure | Group | `{group, member}`, with the new name such as "Former member 1" |
| `inactivity_warning` | 18 months with no activity | That User | `{erasure_on}`, a date |

`my_share` is the recipient's Share in cents, or `null` if they have none. Category changes, Account changes, and Placeholder Member changes write no Notification.

## Worked examples

Ana, Ben, and Cara are Users in Flat 3B, which is linked to its Telegram chat. Ana's Accounts are Bank, her Default Account, and Cash.

**Recording in the Group's chat**

1. Ana sends `/expense 36.90 groceries at:lidl` in the Flat 3B chat.
2. The bot calls `GET /v1/chats/<chat>` and gets Flat 3B, then calls `GET /v1/me/directory` and resolves "groceries".
3. The bot sends `POST /v1/transactions` with `kind: expense`, `payer_member_id: <Ana>`, no `account_id`, an equal Split of all three, and `origin_chat_id: <Flat 3B chat>`.
4. The backend pays from Bank and returns revision 1 with `my_account: Bank`.
5. `origin_chat_id` is the linked chat, so the backend writes no Notification. Ben and Cara see the receipt in the chat.

**Recording someone else's payment in a private chat**

1. In her private chat, Ana sends `/expense 30 eating-out by:@ben split:@ana,@ben`.
2. The bot finds Flat 3B as the one Group with Ben in it, and sends `payer_member_id: <Ben>` with an `origin_chat_id` that is Ana's private chat.
3. Ben's side goes to his Untracked money. Ana has no side as Payer, so `my_account` is `null`.
4. The backend writes `untracked_money_added` to Ben with `amount: 3000`, `direction: paid`, and `my_share: 1500`. Ben gets one message, not two.
5. If Cara had a Share too, she would get `transaction_recorded`, because the Expense was not recorded in the Group's chat.

**Ben picks his Account**

1. Ben taps Pick an Account and chooses Cash. The bot sends `PUT /v1/transactions/<id>` with `If-Match: 1`, the same fields, and `account_id: <Ben's Cash>`.
2. Revision 2 pays from Ben's Cash. Ana recorded revision 1, so she gets `transaction_edited`.

**A stale edit**

1. Ana tapped Edit before Ben's change, and now replies with a corrected amount. The bot sends `PUT` with `If-Match: 1`.
2. The backend returns `412 revision_conflict` with `current_revision: 2`. The bot tells Ana and shows the current Transaction.

**Someone with no rights taps Edit**

1. Cara is in Flat 3B, but she has no Posting on the eating-out Expense and didn't record it. She taps Edit.
2. The bot calls `GET /v1/transactions/<id>`, sees `can_change: false`, and shows a popup only Cara sees.

**Joining from a chat**

1. In a new chat titled "🏠🏠", Ana sends `/start@kitty_bot`. The bot calls `GET /v1/chats/<chat>` and gets `404 chat_not_linked`.
2. The bot sends `POST /v1/groups` with `{chat: {chat_id, title: "🏠🏠"}}`. The Group is named "Group chat", with `name_from_fallback: true`, so the bot asks for a rename.
3. The bot calls `GET /v1/groups/<id>/invite-code` and posts the Join button.
4. Ben Novak taps it. The bot sends `POST /v1/memberships` with `first_name: "Ben"` and `last_initial: "N"`. The Group already has a Ben, so he becomes "Ben N.".
5. The backend writes `member_joined` to the Group, which goes to the chat.

**Delivering Notifications**

1. The bot calls `GET /v1/notifications?limit=50` and gets three.
2. It sends two. The third is to a User who never opened a private chat with the bot, so Telegram refuses it.
3. The bot sends `POST /v1/notifications/acks` with two `delivered` and one `unreachable`.
