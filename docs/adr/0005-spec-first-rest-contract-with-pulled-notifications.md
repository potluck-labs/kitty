# Channels use a spec-first REST contract and pull Notifications from an outbox

The backend and each Channel live in separate repos and languages (ADRs 0002 and 0003), so they share no types and need an explicit contract. A Channel calls the backend over REST with JSON, described by an OpenAPI file. The file is written first, lives in `kitty/contracts/channel-api.yaml`, and is copied word for word into each service. The backend tests its responses against its copy, and the bot generates its models from its copy. Changes are additive inside `/v1`, breaking changes go through expand and contract, and the backend always deploys first.

The backend never talks to Telegram. It writes each Notification to an outbox table in the same database transaction as the change that caused it. Each Channel pulls Notifications with a lease, delivers them at its own pace within its own rate limits, and acks each one, so delivery is at least once. A Channel authenticates with its own API key and names the acting Channel identity on each request, so the backend trusts the Channel's claim about who is acting. Conversation state stays in the Channel, and all durable data stays in the backend.

This costs a polling delay of a second or two on Notifications, an occasional duplicate Notification after a crash, and two PRs for every contract change. In return, the bot needs no open port and no database, Telegram's limits stay inside the bot, and a later Channel uses the same API.

## Considered Options

- gRPC with a `.proto` file. It gives typed stubs in both languages, but NestJS runs it apart from its HTTP stack, and a browser or mobile Channel would need a gateway.
- A message broker such as RabbitMQ, NATS, or Redis Streams. The outbox is still needed to avoid writing to two places, so a broker adds a hop and a service to run for a few messages a minute.
- The backend pushing Notifications to an HTTP endpoint on the bot. The bot would need an open port, and retries for Telegram's rate limits would be split across both services.
- Code-first OpenAPI generated from NestJS decorators. A contract change would start in backend code and reach `kitty` last, which reverses the order in ADR 0002.
- Short-lived User tokens or mTLS between services. The bot can act as any User either way, so neither adds safety while the bot is the only caller.
