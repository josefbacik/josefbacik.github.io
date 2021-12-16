---
layout: post
title: "Extent-tree-v2: Garbage collection and no metadata reference counting"
date: 2021-12-16 12:00:00 -400
categories: kernel btrfs extent-tree-v2
---
# tl;dr

Read part 1 [here]({% post_url 2021-11-10-btrfs-global-roots.markdown %})

If you are a masochist you can read [this]({% post_url
2021-12-16-btrfs-extent-reference-counting.markdown %}) which describes how our
reference counting works which will explain why we're making some of these
changes.

These changes are more infrastructure in nature, they're setting the stage for
the more fundamental changes that are to come, but suffice it to say they allow
us to reduce write amplification and move our more complicated operations out of
the critical path.

# Problems that we're trying to solve.

1. **Reduce metadata amplification.** We don't really need to track references
   on the global roots, as they'll only ever refer to themselves.  That means
   we're inducing unnecessary churn for blocks that will never be shared.
2. **Reduce the cost of snapshotting.** Right now if you modify a shared block
   you have to do a read to see if the block is actually shared and then modify
   the references as appropriate.  This means initial COW of a shared path in a
   subvolume is potentially very expensive.
3. **Take long running operations out of the critical path.** Freeing extents in
   Btrfs is more expensive than it is on other file systems because we have to
   delete checksums.  Unfortunately the operation of deleting an extent has to
   be done in a single transaction.  If you delete a 10gib file in a single
   transacion we will have to delete around 20mib of csums.  If these csums are
   spread across the csum tree it generates a fair bit of work.  This can hold a
   transaction open for hundreds of milliseconds, which can be problematic if
   you have a latency sensitive application is waiting for a transaction commit.

# Solution #1: No longer track reference counts for metadata

Tracking references for metadata is quite expensive.  Any modification will
result in new reference updates to the extent tree.  Any time we have to modify
the extent tree we could generate more references.  This means metadata write
amplification is relatively high for individual changes, but this cost is
generally amortized across all modifications in the transaction.

Still there's a on disk write amplification cost.  There's also the cost of
running these operations in general.  For every tree modification you
potentially have to modify a bunch of other trees, and this extra work burns
CPU, means you have to read more blocks into cache, and is generally a source of
latency.  Reducing the amount of work that needs to be done per operation will
generally reduce worst case latencies and give us a nice performance boost.

However this property of Btrfs underpins basically every feature that makes
Btrfs useful.  Right now I'm throwing it out and disabling all of those fancy
features.  But don't worry, I'm replacing them with different concepts.  However
those are coming in future patches, so I'll describe them as I add the
functionality back.

# Solution #2: Garbage collection tree

I hate garbage collection.  I've always viewed garbage collection as a source of
latency, because I think in terms of worst case.  Worst case I have to stop and
wait on garbage collection at some point, and it's going to be a huge latency
spike and I'm going to be mad.  To me it was always better to spread this cost
out to the operations generating the work, make everybody equally slow instead
of surprise screwing some operation later down the line that has to pay for all
of the generated work.

However there are a few operations that we do that induce latency specifically
because we need them to be done when they happen.  The extent deletion example
above is the prime example of this.  We have a set of operations that we can't
put off until later, and if you generate enough of these operations you can have
a sad time.

Additionally I'm going to be adding a bunch of new concepts that are going to
have varying degrees of complication, and I don't want to be forced to do those
operations in the transaction where they were created.

So reluctantly I've added a garbage collection tree.  This will exist
exclusively for the cleanup operations.  Things like free'ing data extents,
updating qgroup counters, or updating shared metadata block counts.  This means
that anything that needs to wait for space to be free'd up will sometimes have
to wait on the garbage collection tree, however hopefully this will be limited
to very full file systems and not be an issue for normal operation.

For now I've only added inode eviction to the garbage collection tree.
Traditionally if you `rm <file>` it would clean up all of the items before it
returned to user space.  With the garbage collection tree we will simply add the
file to the garbage collection tree to be deleted in the background.

This will artificially make our unlink performance look amazing.  Don't be
fooled, you're going to pay the price somewhere, but if you have an unlink()
heavy microbenchmark we're going to make everybody else look stupid.  This is a
lie.

# Conclusion

These aren't super interesting changes in themselves, because for the most part
these patches are disabling all the cool things we do in Btrfs.  This sets us up
to add in the new cool concepts that will replace reference counting.  The
garbage collection tree will have new operations added to it as I add back the
functionality, and we'll talk about those in future posts.
