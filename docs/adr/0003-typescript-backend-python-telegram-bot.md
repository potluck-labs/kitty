# TypeScript backend on NestJS, Python Telegram bot on aiogram

We want both Python and TypeScript across Kitty, so one service gets each language. `kitty-backend` is TypeScript on NestJS with the Express adapter, running on Node 26 LTS, and `kitty-telegram-bot` is Python on aiogram. NestJS gives the backend its modules and DI container, so the backend's architecture works inside that structure and is not designed from a bare HTTP framework. The split costs two toolchains, two test setups, and two sets of dependency updates, and no types can be shared between the services, so they need an explicit API contract. Changing either language later means rewriting that service.

## Considered Options

- Python backend on FastAPI or Litestar, with a TypeScript bot on grammY. It keeps SQLAlchemy and Alembic for the ledger and leaves the layers for us to design, but nothing in the framework enforces those layers.
- Bun or Deno as the backend runtime. NestJS's docs and CI cover only Node. Bun's decorator metadata differs from `tsc` (`X | undefined` becomes `X`), and NestJS reads that metadata. Deno treats `experimentalDecorators` as deprecated. Avoiding Bun-only APIs keeps a later move possible.
- python-telegram-bot or pyTelegramBotAPI for the bot. The first is three Bot API versions behind aiogram (10.0 against 10.3). The second is on 10.3 but licensed GPL-2.0, while aiogram is MIT and our repos are public.
