# Postgres data lives on a LUKS volume that we unlock by hand after a reboot

Telegram's bot terms (§4.4(a)) require data encrypted at rest and stored apart from its key. Postgres keeps its data directory on a LUKS-encrypted volume, and one of us types the passphrase over SSH after every reboot. The key never touches the server's disk, so a full-server snapshot by the provider, or by an attacker in the provider console, holds only unreadable data. Automatic reboots are off, and we schedule kernel updates ourselves. The passphrase lives in the shared password manager, and the unlock steps are in the `kitty-ops` runbook.

Until we unlock it, the backend and the bot don't run. Healthchecks.io alerts us when the backend stops. The bot uses long polling and Telegram keeps updates for 24 hours (ADR 0006), so an unlock within a day delays replies but loses no message.

## Consequences

- The server does not come back on its own after a reboot. This is on purpose.
- Root access on the running server still sees the data. No option here prevents that.
- Backups are encrypted separately by restic with its own password.
- Bot Redis runs with persistence off, so it has no data at rest.

## Considered Options

- A LUKS key file on the server's root disk, written by the deploy job. The server restarts unattended, but one snapshot of the server captures both the key and the data, so it fails the threat it exists for.
- Column encryption in the backend for names, Account names, and Merchants. Amounts and Categories must stay plain for SQL sums. It protects those columns in a leaked dump, but it blocks search and sort on them, needs a key rotation job, and adds code to every module. It can come later if a reason appears.
