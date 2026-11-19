# The Telegram bot keeps privacy mode on and never needs to be a group admin

The production bot and the shared test bot keep Telegram's privacy mode on in BotFather. In a group, the bot then gets only commands meant for it, replies to its own messages, button presses, and service messages. The MVP records with commands, so it needs nothing more. It never asks a group to make it an admin and works the same either way. We chose this because changing the BotFather setting takes effect in a group only after the bot is removed and added again, and we can't do that for every Group once sign-up is open. Keeping it on also keeps both routes open for "line, then card" later: a command prefix such as `/k`, or a group that chooses to make the bot an admin, which changes nothing in other groups.

## Consequences

- A group admin can still make the bot an admin, and Telegram then sends it every message. The bot drops any message it doesn't act on in its first handler, before any logging, and never stores it or sends it to the backend. A test checks this, and the privacy notice says so.
- With privacy mode on, a plain `/expense` in a group reaches the bot only if it was the last bot to post there. Usage replies and help text in groups use the `/expense@<bot>` form, and the bot posts a welcome message when it is added. Plain commands still work when they arrive.
- The bot calls `getMe` at startup and refuses to start if `can_read_all_group_messages` is true. This catches a wrong BotFather setting at the next deploy. Groups added while the setting was off keep getting every message until the bot is added again.

## Considered Options

- Privacy mode off in BotFather. Every group would send every message, the bot would see plain lines such as `37 lidl` with no prefix, and the privacy notice would have to say Kitty reads all group chat. Turning it back on would need the bot re-added to every group added in the meantime.
- Asking every group to make the bot an admin in the MVP. It would also let the bot delete a refused Transfer typed in a group and get `chat_member` updates, but neither is worth reading every message.
