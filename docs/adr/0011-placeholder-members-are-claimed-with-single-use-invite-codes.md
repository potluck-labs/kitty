# Placeholder Members are claimed with single-use Invite codes

Groups often share costs with someone who hasn't signed up yet, such as a parent on a trip. So a Member can be a Placeholder Member: a name with no User behind it, which can be the Payer or have a Share. This reverses the earlier rule, set with the Channel contract in ADR 0005, that only Users can be Members. Any Member may add, rename, or remove one, and removing is allowed only while it has no Postings. Names are unique among a Group's current Members. A Placeholder Member has its own Book and its own Untracked money (ADR 0012), has no rights, and gets no Notifications. Loans and Settlements with it are recorded by the other Member.

A User takes over a Placeholder Member by a Claim. A Member picks the Placeholder Member in a Channel, Kitty issues a single-use Invite code tied to it, and the User who joins with that code becomes that Member. The Placeholder Member's Postings move into the User's Book as they are, and its Untracked money Postings move into the User's one Untracked money Account. Every Member gets a Notification when a Claim happens.

## Considered Options

- Picking a name on join, then another Member approves. It needs an approval state, and it shows the Group's Placeholder Member names to anyone with a normal Invite code.
- Picking a name on join with no approval. A stranger with the Group's Invite code could claim "Dad" and his balances.
- Merging a Placeholder Member into a User who is already a Member. Every debt between the two would become a debt to oneself, which needs its own cleanup rules. The MVP refuses the Claim instead, and merging is out of scope.
- An undo for a Claim. After a Claim the User may record Postings with their own Accounts, so an undo would be a small Erasure. A wrong Claim is fixed by the rules for removing a Member instead.

## Consequences

- A Claim is as safe as the Invite code. A Member who sends it to the wrong person has handed over the Placeholder Member's balances.
- Moving Postings to another Book on a Claim is a second exception to append-only Postings in ADR 0004, after Erasure in ADR 0009.
- Leaving a Group does not create a Placeholder Member. The former Member keeps their Book, because their Accounts span Groups.
