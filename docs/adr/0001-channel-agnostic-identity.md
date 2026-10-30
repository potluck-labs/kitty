# Channel-agnostic identity

Kitty's MVP talks to people only through Telegram, but later Channels may include Slack, Discord, or a mobile app with email or phone sign-up. So a User is Kitty's own identity, and a Telegram account is one linked Channel identity, not the User itself. Invite codes, Groups, and Accounts are backend concepts that never depend on Telegram features such as deep links. This costs an identity-linking table and a Channel-neutral API now. In return, adding a Channel later does not mean migrating every User and reference keyed on a Telegram ID.
