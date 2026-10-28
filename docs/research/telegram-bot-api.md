# Telegram Bot API facts that shape the design

Research for [#3](https://github.com/potluck-labs/kitty/issues/3), part of map [#2](https://github.com/potluck-labs/kitty/issues/2). Read on 2026-10-28. The Bot API changelog showed Bot API 10.3 (August 24, 2026) as the latest version.

All facts come from Telegram's own docs:

- [Bot API reference](https://core.telegram.org/bots/api), cited as [API].
- [Bot features](https://core.telegram.org/bots/features), cited as [Features].
- [Bots FAQ](https://core.telegram.org/bots/faq), cited as [FAQ].
- [Webhooks guide](https://core.telegram.org/bots/webhooks), cited as [Webhooks].
- [Bots introduction](https://core.telegram.org/bots), cited as [Intro].
- [Mini Apps](https://core.telegram.org/bots/webapps), cited as [MiniApps].

Each section lists the facts first, then what they mean for Kitty. The implications are inputs for later decisions, not decisions.

## Receiving updates: webhook or long polling

Facts:

- There are "two mutually exclusive ways of receiving updates": `getUpdates` (long polling) and `setWebhook`. `getUpdates` does not work while a webhook is set. [API, Getting updates]
- Telegram keeps undelivered updates, but "not longer than 24 hours." [API, Getting updates]
- `update_id` values increase sequentially and let a bot "ignore repeated updates or restore the correct update sequence." After a week with no updates, the next id is random. [API, Update]
- `getUpdates` takes `offset`, `limit` (1 to 100, default 100), `timeout` (seconds, 0 means short polling), and `allowed_updates`. An update is confirmed once the bot calls `getUpdates` with a higher offset. [API, getUpdates]
- A webhook gets one HTTPS POST per update. Any non-2xx response is a failure, and Telegram "will repeat the request and give up after a reasonable amount of attempts." The docs give no retry count or timeout. [API, setWebhook]
- Webhook constraints: HTTPS only, ports 443, 80, 88, or 8443, TLS 1.2 or later, IPv4 only, and the certificate CN must match the domain. Self-signed certificates work if uploaded with the `certificate` parameter. [Webhooks], [API, setWebhook]
- `secret_token` (1 to 256 characters, `A-Z a-z 0-9 _ -`) is sent in the `X-Telegram-Bot-Api-Secret-Token` header of every webhook request. [API, setWebhook]
- Telegram sends webhooks from 149.154.160.0/20 and 91.108.4.0/22. [Webhooks]
- `max_connections` is 1 to 100, default 40. [API, setWebhook]
- `getWebhookInfo` returns `pending_update_count`, `last_error_date`, and `last_error_message`. [API, getWebhookInfo]
- A bot can answer a webhook request with a method call in the response body, but "it's not possible to know that such a request was successful or get its result." [API, Making requests when getting updates]
- A self-hosted [Local Bot API server](https://github.com/tdlib/telegram-bot-api) lifts some limits: HTTP webhooks on any port and local IP, uploads up to 2000 MB, downloads with no size limit, and up to 100000 webhook connections. [API, Using a Local Bot API Server]

For Kitty:

- Long polling needs no public HTTPS endpoint, so it works on a laptop and on the VPS behind no reverse proxy. A webhook needs a domain, a certificate, and an open port from the list above. That is DevOps work the team wants to learn, but it is a second moving part.
- The two modes are exclusive per bot token. Developers each need their own bot token from BotFather, or local polling breaks the production webhook.
- Both modes can deliver the same update twice: webhooks retry on non-2xx, and polling re-delivers anything not yet confirmed. The bot needs to deduplicate on `update_id` or make the backend call idempotent. This feeds the open item "Duplicate messages and retries between Channel and backend" on the map.
- If the bot is down for more than 24 hours, updates are lost.

## How the bot identifies a user and a chat

Facts:

- `User.id` "has at most 52 significant bits," so a 64-bit integer or a double stores it safely. [API, User]
- `User.first_name` is required. `username`, `last_name`, and `language_code` are optional. [API, User]
- `Chat.id` may have "more than 32 significant bits" and has at most 52. [API, Chat]
- `Chat.type` is one of `private`, `group`, `supergroup`, or `channel`. [API, Chat]
- A group can be migrated to a supergroup. The old group then sends `migrate_to_chat_id`, the new supergroup sends `migrate_from_chat_id`, and a call to the old id fails with `ResponseParameters.migrate_to_chat_id`. The supergroup has a new chat id. [API, Message], [API, ResponseParameters]
- In a private chat with a user, the chat id and the user id are the same number. This is observed behavior, not a documented guarantee. The docs do not document the sign or prefix of group ids either.
- `CallbackQuery.from` is the user who pressed the button, which may differ from the user the message was sent to. `chat_instance` identifies the chat the message was sent in. [API, CallbackQuery]
- `Message.from` "may be empty for messages sent to channels," and `sender_chat` is set when someone posts on behalf of a chat (for example an anonymous group admin). [API, Message]

For Kitty:

- Store Telegram ids as `BIGINT` in Postgres and as `int` in Python. In TypeScript, a `number` holds 52 bits exactly, so no `BigInt` is needed, but JSON between services must not cut them to 32 bits.
- A Telegram chat id is not stable for the life of a group, because of the supergroup migration. Kitty's Group must not be keyed on it. The Channel layer needs a link from a Telegram chat to a Kitty Group that it can update when `migrate_to_chat_id` arrives.
- `username` is optional and users can change it. It is a display hint, never an identity key. The only stable key is `User.id`.
- Anonymous admins post with `sender_chat` and no real user. The bot cannot tell which User recorded a Transaction from such a message, so it has to reject or ignore it.

## Private chats and group chats

Facts:

- "Bots can't start conversations with users." A user "must either add them to a group or send them a message first." [Intro]
- In groups, bots run in privacy mode by default, unless they were added as admins. [Features, Privacy mode]
- With privacy mode on, a bot receives in a group only: commands meant for it (`/command@this_bot`), general commands like `/start` if it was the last bot to post there, replies to its messages, all service messages, and messages from channels it belongs to. [Features], [FAQ]
- Admin bots and bots with privacy mode off receive all group messages except those from other bots. [FAQ]
- BotFather's `/setprivacy` changes the mode, and the bot "will need to be re-added to the group for this change to take effect." [Features]
- Bots cannot see messages from other bots in groups. [FAQ]
- Service messages such as `new_chat_members` and `left_chat_member` reach every bot. `my_chat_member` reports changes to the bot's own status. `chat_member` reports other members' status changes, but only if the bot is an admin and lists `chat_member` in `allowed_updates`. [API, Update], [Features]
- `getChatMember` is "only guaranteed to work for other users if the bot is an administrator in the chat." [API, getChatMember]
- Commands start with `/`, have up to 32 characters, and can target one bot with `/command@botname`. `setMyCommands` takes up to 100 commands and a scope. Scopes can target all private chats, all group chats, group admins, one chat, or one member of a chat. [Features], [API, setMyCommands], [API, BotCommandScope]

For Kitty:

- With default privacy mode, a group bot never sees a plain message like "12.50 pizza". Users must type a command (`/expense 12.50 pizza`), reply to a bot message, or tap a button. The other option is to turn privacy mode off or make the bot an admin. Then the bot reads every group message, which affects the privacy notice and the amount of traffic.
- Kitty cannot send a private message to a Member who has never started the bot. Private notifications (for example "you owe Sara 20 EUR") only work for Users who opened a private chat first. Group-only Members can only be reached in the group.
- Telegram group membership and Kitty Group membership are different things. A user who joins the Telegram group is not a Member until they use an Invite code. A user who leaves the Telegram group is still a Member. The bot sees join and leave service messages, so it could react to them, but it does not have to.
- Command scopes let the bot show different command menus in private chats and in groups.

## Deep links and the Invite code

Facts:

- `https://t.me/<bot>?start=<param>` opens a private chat and delivers `/start <param>`. `?startgroup=<param>` asks the user to pick a group and delivers `/start@<bot> <param>` there. [Features, Deep linking]
- The parameter allows only `A-Z a-z 0-9 _ -`, up to 64 characters. Telegram recommends base64url for other data. [Features]

For Kitty:

- ADR 0001 says the Invite code belongs to Kitty and a Channel may show it as a link. A deep link carries the code without extra encoding only if the code uses that alphabet and fits in 64 characters. If the backend picks an alphabet outside it, the bot must encode the code. Either choice is fine, but the code format should be decided with this limit in mind.

## Keyboards and callback queries

Facts:

- A reply keyboard (`ReplyKeyboardMarkup`) replaces the user's keyboard with buttons. Pressing one sends its text to the chat as a normal message. Options: `is_persistent`, `resize_keyboard`, `one_time_keyboard`, `input_field_placeholder` (1 to 64 characters), and `selective`. [API, ReplyKeyboardMarkup]
- `selective` shows the keyboard only to users @mentioned in the text or to the sender of the message the bot replies to. [API, ReplyKeyboardMarkup]
- `ForceReply` opens the reply interface "as if they had manually selected the bot's message and tapped Reply". It also has `selective`. [API, ForceReply]
- An inline keyboard (`InlineKeyboardMarkup`) is attached to one message. Pressing a callback button sends no chat message. It sends a `callback_query` update instead. [Features], [API, InlineKeyboardButton]
- `callback_data` is 1 to 64 bytes. [API, InlineKeyboardButton]
- Clients show a progress indicator until the bot calls `answerCallbackQuery`, and the bot should call it even if it shows nothing. The notification text is 0 to 200 characters, or an alert with `show_alert`. [API, CallbackQuery], [API, answerCallbackQuery]
- `CallbackQuery.from` is the user who pressed the button. [API, CallbackQuery] The reference does not restrict who may press, so in a group any member who sees the message can.

For Kitty:

- 64 bytes cannot hold a draft Expense. Buttons should carry a short key (an action name and a draft id), and the full state lives elsewhere. Where that draft state lives, bot or backend, is a design decision. If it lives in the backend, the backend learns about a Telegram conversation concept.
- In a group, the bot must check that the person pressing a button is the person whose flow it is, or another Member could pick someone else's Category.
- Under privacy mode, a multi-step flow in a group can use `ForceReply` with `selective`, because replies to the bot's messages reach it.
- Reply keyboards post the pressed label into the group, which adds noise. Inline keyboards do not.

## Editing and deleting messages

Facts:

- `editMessageText`, `editMessageReplyMarkup`, `editMessageCaption`, and `editMessageMedia` edit messages. `editMessageText` works only on messages with no reply markup or with an inline keyboard. The 48-hour edit limit applies only to business messages not sent by the bot. The docs state no time limit for a bot editing its own messages. [API, Updating messages]
- `deleteMessage` works only on messages "sent less than 48 hours ago." A bot can delete its own messages in any chat and incoming messages in private chats. In groups, it can delete other people's messages only if it is an admin. [API, deleteMessage]
- When a user edits a message the bot already saw, the bot gets an `edited_message` update. [API, Update]

For Kitty:

- A flow can edit one bot message in place (for example, a draft that updates as the user picks Payer, Category, and Split) instead of posting many messages. This also helps with the group rate limit below.
- If a user records an Expense by message and then edits that message, the bot receives the edit. Kitty must decide whether an edit changes the Transaction, or is ignored.
- A non-admin bot cannot clean up users' command messages in the group.

## Sending images and documents (charts)

Facts:

- Files can be sent by `file_id`, by HTTP URL, or by multipart upload. [API, Sending files]
- Upload limits: 10 MB for photos, 50 MB for other files. By URL: 5 MB for photos, 20 MB for other files. Resending by `file_id` has no size limit. [API, Sending files]
- A `file_id` "can't be transferred from one bot to another." [API, Sending files]
- `sendPhoto`: width plus height at most 10000, aspect ratio at most 20. [API, sendPhoto] Telegram clients show photos recompressed. This is observed behavior, not stated in the reference.
- `sendDocument` sends any file type up to 50 MB. [API, sendDocument]
- Message text is 1 to 4096 characters, and captions 0 to 1024, both counted after entity parsing. [API, sendMessage], [API, sendPhoto]
- Formatting uses `parse_mode` `MarkdownV2` or `HTML`. `Markdown` is a legacy mode. [API, Formatting options]
- Bots can download files up to 20 MB with `getFile`. [API, getFile]
- `sendChatAction` shows a status such as "uploading photo" for 5 seconds or less. [API, sendChatAction]

For Kitty:

- A monthly chart as a PNG fits easily within 10 MB. If the chart must stay sharp, it can go as a document. Whether clients recompress documents is not stated in the reference, but in practice they do not.
- A long statement may pass 4096 characters, so the bot must split it or send it as a file.
- Uploading the image by multipart means the chart service does not need a public URL. Sending by URL would need one.
- Rendering the chart is channel-neutral if the backend returns image bytes or chart data, and the bot only uploads. If the backend calls Telegram to send the photo, Telegram leaks into the backend.

## Rate limits

Facts:

- In one chat, "avoid sending more than one message per second." [FAQ]
- In one group, a bot cannot send "more than 20 messages per minute." [FAQ]
- Across all chats, bots cannot broadcast "more than about 30 messages per second" unless they pay for broadcasts (up to 1000 per second, 0.1 Stars per message above the free 30). [FAQ]
- Going over the limit returns HTTP 429. `ResponseParameters.retry_after` gives the seconds to wait. [FAQ], [API, ResponseParameters]

For Kitty:

- 20 messages per minute per group is the limit that matters. A busy group where every Expense step posts a new message can hit it. Editing one message per flow, and sending one summary per action, keeps the count low.
- Fan-out jobs, such as a monthly summary sent to every User in private chat, need a send queue that respects about 30 messages per second overall and honors `retry_after`.

## Verifying a Telegram identity outside the bot

Facts:

- Mini Apps pass `initData` to the bot's server. The server checks its `hash` with HMAC-SHA-256, using a key derived from the bot token and the constant `WebAppData`, and checks `auth_date` for freshness. A `signature` field lets third parties check the data with Telegram's Ed25519 public key, without the bot token. [MiniApps, Validating data]
- The Telegram Login Widget authorizes users on a website. It is paired with a domain through BotFather's `/setdomain`. [Features, Login Widget]

For Kitty:

- A web dashboard is out of scope for the MVP. If one comes later, these are the documented ways to prove that a web visitor owns a Telegram account and link it to an existing User. That fits ADR 0001: the Telegram account is a linked Channel identity.

## Where Telegram concepts could leak into the backend

ADR 0001 says the backend knows Users, Groups, Accounts, and Invite codes, and Telegram is one Channel. These facts push Telegram-specific concepts toward the backend, and each needs a deliberate boundary:

1. Telegram user id. The link from a Telegram `User.id` to a Kitty User has to live somewhere. If it lives in the backend's identity-linking table, the backend stores a Channel name and an opaque external id, and nothing more Telegram-specific.
2. Telegram chat id and supergroup migration. A Telegram group's chat id can change. If the backend stores it as a Group attribute, migration becomes a backend concern. Keeping the chat-to-Group link in the Channel layer, or storing it as a generic "Channel conversation" with a way to update it, keeps the Group id stable.
3. Duplicate deliveries. `update_id` is a Telegram concept. If deduplication happens only in the bot, the backend never sees it. If the backend must be idempotent anyway, the bot can pass a Channel-neutral idempotency key instead of an `update_id`.
4. Draft state behind buttons. The 64-byte `callback_data` limit forces state to live outside the button. Storing drafts in the backend is easy, but it adds a conversation concept to the backend.
5. Deep link parameters. The `start` parameter alphabet (`A-Z a-z 0-9 _ -`, 64 characters) is a Telegram limit. Fitting the Invite code to it is harmless for other Channels, but it is a Telegram rule shaping a backend value.
6. Reachability. Kitty cannot message a User in private until that User starts the bot. Whether a User can be notified is a per-Channel fact. If the backend sends notifications, it must know about it. If the bot decides delivery, the backend only emits events.
7. Display names. Telegram gives `first_name` and an optional, changeable `username`. If the backend copies them into the User, it should treat them as a Channel-supplied display name that can change.
8. Rate limits and file uploads. Sending messages and photos within Telegram's limits is Channel work. The backend should return data or image bytes, and the bot should send them.

## Gaps in the docs

- The docs do not say how many times or for how long Telegram retries a failed webhook.
- The docs do not document the sign or prefix of group and supergroup chat ids, or that a private chat id equals the user id. Code should not depend on either.
- The docs do not give a time limit for a bot editing its own non-business messages. Telegram may still enforce one that is not documented.
