---
feature: local-overlay-store
start-date: (fill me in with today's date, YYYY-MM-DD)
author: (name of the main author)
co-authors: (find a buddy later to help out with the RFC)
shepherd-team: (names, to be nominated and accepted by RFC steering committee)
shepherd-leader: (name to be appointed by RFC steering committee)
related-issues: (will contain links to implementation PRs)
---

# Summary
[summary]: #summary

Add a new `local-overlay` store implementation to Nix.
This will be a local store that is layered upon another local filesystem store (local store or daemon).
This allows locally extending a shared store that is periodically extended with additional store objects.

# Motivation
[motivation]: #motivation

TODO: link Replit blog post.

## Technical motivation

Many organizational users of Nix have a large collection of Nix store objects they wish to share with a number of consumers, be they human users, build farm workers etc.
The existing ways of doing this are:

- Share nothing: Copy store objects to each consumer

- Share everything: Use single mutable NFS store, carefully synchronizing updates.

- Overlay everything: Mount all of `/nix` (store and DB) with OverlayFS.

Each has serious drawbacks:

- Share nothing wastes tones of space as many duplicate store objects are stored separately.

- Share everything incurs major overhead from synchronization, even if consumers are making store objects they don't intend any other consumer to use.

- Overlay everything cannot take advantage of new store objects added to the lower store, because its "fork" of the DB covers up the lower store's.

The new `local-overlay` store also uses OverlayFS, but just for the store directory.
The database is still a regular fresh empty one to start, and instead Nix explicitly knows about the lower store, so it can get any information for the DB it needs from it manually.
This avoids all 3 downsides:

- Store objects are never deduplicated.
  OverlayFS ensures that one copy in either store dir layer is enough, and we are careful never to wastefully include a store object in both layers.

- No excess synchronization.
  local changes are just that: local, not shared with any other consumer.
  The lower store is never written to (no modifications or even filesystem locks) so any slow NFS write and sync paths should not be encountered.

- No rigidity of the lower store.
  Since Nix sees both the `overlay-store`'s DB and the lower store, it is possible for it to be aware of and utilize objects that were added to the lower store after the upper store was created.

This gives us a "best of all three worlds" solution.

## Marketing motivation

It is quite common for organizations using Nix to first adopt it behind the scenes.
That is to say, Nix is used to prepare some artifacts which are then presented to a consumer that need not be aware they were made with Nix.
Later though, because of Nix's gaining popularity, there may be a desire to reveal its usage so consumers can use Nix themselves.
Rather than Nix being a controversial tool worth hiding, it can be a popular tool worth exposing.
Nix-unware usage can still work, but Nix-aware usage can do additional things.

The `local-overlay` store can serve as a crucial tool to bridge these two modes of using Nix.
The lower store can be however the artifacts were disseminated in the "hidden Nix" first phase of adoption, perhaps with a small tweak to expose the DB / daemon socket if it wasn't before.
The upper store is new, but purely local, separate for each user that wants to use Nix, and completely not impacting any user that doesn't.

By providing the `local-overlay` store, we are essentially completing a reusable step-by-step guide for Nix users to "Nixify their workplace" in a very conscientious and non-disruptive manner.

# Detailed design
[design]: #detailed-design

## Class hierarchy, configuration settings, and initialization

`LocalOverlayStore` is a subclass of `LocalStore` implementing the `local-overlay` store.
It has additional configuration items for:

 - the lower store, which must be a `LocalFSStore`

 - the scratch directory used as the upper layer of the OverlayFS

On initialization, it checks that an OverlayFS mount exists matching these parameters:

 - the lower layer must be the lower layer's "real store directory"

 - the upper layer must be the scratch directory specified for this purpose

The database for the `local-overlay` store is exactly like that for a regular local store:

 - same schema, including foreign key constraints

 - created empty on opening the store if it doesn't exist

 - opened existing one otherwise

## Operation

As discussed in the motivation, all store objects are stored exactly once: either in the lower store or the upper scratch directory.
No file system data should ever be duplicated.

Non-filesystem data, what goes in the DB (references, signatures, etc.) is duplicated.
Any store object from the lower store that the upper store needs has that information copied into the upper store's DB.
This includes information for the closure of any such store object, because the normal closure property enforced by the DB's foreign key constraints is upheld.

## Read-only `local` Store

In order to facilitate using `local-overlay` where the lower store is entirely read only (read only SQLite files too, not just store directory), it is useful to also implement a new "read-only" setting on the `local` store.

# Examples and Interactions
[examples-and-interactions]: #examples-and-interactions

Because the `local-overlay` store is a completely separate store implementation, the interactions with the rest of Nix are fairly minimal and well-defined.
In particular, users of other stores and not the `local-overlay` store

This section illustrates the detailed design. This section should clarify all
confusion the reader has from the previous sections. It is especially important
to counterbalance the desired terseness of the detailed design; if you feel
your detailed design is rudely short, consider making this section longer
instead.

# Drawbacks
[drawbacks]: #drawbacks

Why should we *not* do this?

# Alternatives
[alternatives]: #alternatives

What other designs have been considered? What is the impact of not doing this?

# Unresolved questions
[unresolved]: #unresolved-questions

What parts of the design are still TBD or unknowns?

# Future work
[future]: #future-work

What future work, if any, would be implied or impacted by this feature
without being directly part of the work?
