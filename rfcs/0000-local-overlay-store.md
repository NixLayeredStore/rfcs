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
The `local-overlay` store is new, but purely local, separate for each user that wants to use Nix, and completely not impacting any user that doesn't.

By providing the `local-overlay` store, we are essentially completing a reusable step-by-step guide for Nix users to "Nixify their workplace" in a very conscientious and non-disruptive manner.

# Detailed design
[design]: #detailed-design

## Basic principle

`local-overlay` is a store representing the extension of a lower store with a collection of additional store objects.
(We don't refer to the upper layer as an "upper store" because it is not self-contained
--- it doesn't abide by the closure property because objects in the upper layer can refer to objects that are only in the lower layer.)

## Class hierarchy, configuration settings, and initialization

**TODO put in layering diagram, e.g. from blog**

`LocalOverlayStore` is a subclass of `LocalStore` implementing the `local-overlay` store.
It has additional configuration items for:

 - The lower store, which must be a `LocalFSStore`

   This is specified with an escaped URL just like the `remote-store` setting of the two SSH stores types.

 - The scratch directory used as the upper layer of the OverlayFS

On initialization, it checks that an OverlayFS mount exists matching these parameters:

 - The lower layer must be the lower layer's "real store directory"

 - The upper layer must be the scratch directory specified for this purpose

The database for the `local-overlay` store is exactly like that for a regular local store:

 - Same schema, including foreign key constraints

 - Created empty on opening the store if it doesn't exist

 - Opened existing one otherwise

## Data structure

This is a diagram for the possible states of a single store object.
Except for the closure property mandating that present store objects must have all their references also be present, store objects are independent.
We can therefore to a large extent get away with considering only a single store object.

**TODO put [state machine and partial order diagram](./0000-local-overlay-store/store-object-state-machine.drawio)**

Key:

 - Colors:

   - Red (warm colors): Store object considered not in store

   - Green/Blue (cool colors): Store object considered in store

     - Green: Store object resident of lower store

     - Blue: Store object resident of upper layer / `local-overlay` store only

     - Both, both stores

 - Arrows:

   - Up arrows (any style) "more present" partial order

   - Solid arrows: regular transitions (come from regular store operations)

   - Wide dashed arrow: part of partial order but should not happen

   - Thin dashed arrow: special operation specific to `local-overlay` store

## Operation

As discussed in the motivation, all store objects are stored exactly once: either in the lower store or the upper scratch directory.
No file system data should ever be duplicated.

Non-filesystem data, what goes in the DB (references, signatures, etc.) is duplicated.
Any store object from the lower store that the `local-overlay` needs has that information copied into the `local-overlay` store's DB.
This includes information for the closure of any such store object, because the normal closure property enforced by the DB's foreign key constraints is upheld.

## Read-only `local` Store

In order to facilitate using `local-overlay` where the lower store is entirely read only (read only SQLite files too, not just store directory), it is useful to also implement a new "read-only" setting on the `local` store.
The main thing this does is use SQLite's [immutable mode](https://www.sqlite.org/c3ref/open.html).

This is a separate feature;
it is perfectly possible to implement the `local-overlay` without this or vice-versa.
But for maximum usability, we want to do both.

# Examples and Interactions
[examples-and-interactions]: #examples-and-interactions

Because the `local-overlay` store is a completely separate store implementation, the interactions with the rest of Nix are fairly minimal and well-defined.
In particular, users of other stores and not the `local-overlay` store will not be impacted at all.

# Drawbacks
[drawbacks]: #drawbacks

## Read-only local store is delicate

SQLite's immutuable mode doesn't mean "we promise not to change the database".
It relies on the database not just being *logically* immutable (the meaning doesn't change), but *physically* immutable (no bytes of the on-disk files change).
This is because it is forgoing synchronization altogether and thus relying that nothing can be rearranged e.g. in the middle of a query (invalidating in-progress traversals of data structures, etc.).
This means there is no hope of, say "append only" mode where "publishers" only add new store objects to a local store, while read-only mode "subscribers" are able to deal with other store objects just fine.

This is an inconvenience for rolling out new versions of the lower store, but not a show stopper.
One solution is "multi version concurrency control" where consumers get an immutable snapshot of the lower store for the duration of each login.
New snapshots can only be gotten when consumers log in again, and old snapshots can only be retired once every consumer viewing them logs out.

## `local-overlay` lacks normal form for the database

A slight drawback with the architecture is a lack of a normal form.
A store object in the lower store may or may not have a DB entry in the `overlay-local` store.

> In the store object state diagriam diagram, this is represented by the fact that there are two green nodes, instead of just one like the one blue node.

This introduces some flexibility in the system: the seem "logical" layered store can be represented in multiple different "physical" configurations.
This isn't a problem per-se, but does mean there is a bit more complexity to consider during testing and system administration.

## Deleting isn't intuitive

For "deleting" lower store objects in the `local-overlay` store,
we don't actually remove them but just remove the upper DB entry.
This is somewhat surprising, but reflects the fact that the lower store is logically immutable (even when it isn't a `local` store opened in read-only mode).
By deleting the upper DB entry, we are not removing the object from the `local-overlay` store, but we are still resetting it to the initial state.

# Alternatives
[alternatives]: #alternatives

## Bind mounts instead of overlayfs

Instead of mounting the entire lower store dir underneath ours via OverlayFS, we could bind mount individual store objects as we need them.
The bind-mounting store would not longer be the union of the lower store with the additional store objects, instead the lower store acts more as a special optimized substituter.

This gives a normal form in that objects are bind-mounted if and only if they have a DB entry in the bind-mounting store;
There is no "have store object, don't yet have DB entry" middle state to worry about.

The downside of this is that Nix needs elevate permissions in order to create those bind mounts, and the impact of having arbitrarily many bind mounts is unknown.

## FUSE

We could have a single FUSE mount that could manually implement the "bind on demand" semantics described above without cluttering the mount namespace with an entry per each shared store object.
FUSE however is quite primitive, in that every read need to be shuffled via the FUSE server.
There is nothing like a "control plane vs data plane" separation where Nix could tell the OS "this directory is that other directory", and the OS can do the rest without involving the FUSE server.
That means the performance of FUSE are potentially worse than these in-kernel mounting solutions.

Also, this would require a good deal more code than the other solutions.
Perhaps this is a good thing; the status quo of Nix having to keep the OS and DB in sync is not as elegant as Nix squarely being the "source of truth", providing both filesystem and non-filesystem data about each store object directly.
But it represents as significantly greater departure from the status quo.

# Unresolved questions
[unresolved]: #unresolved-questions

How to delete paths in the upper store without needing to remount the OverlayFS.
(Expect to have answer by the time the RFC is up.)

# Future work
[future]: #future-work

We are considering a hybrid between the `local` store and `file://` store.
This would *not* use NARs, but would use "NAR info" files instead of a SQLite database.
This would side-step the currency issues of SQLite's read-only mode, and make "append only" usage of the store not require much synchronization.
(This is because the internal structure of the filesystem, unlike the internal structure of SQLite, is not visible to clients.)

It is true that this is much slower used directly --- that is why Nix switched to using SQLite in the first place --- but in combination with a `local-overlay` store this doesn't matter.
Since non-filesystem data is copied into the `local-overlay` store's DB, it will effectively act as a cache, speeding up future entires.
Each NAR info file only needs to be read once.
