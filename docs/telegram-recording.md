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
