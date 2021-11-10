---
layout: post
title: "Extent-tree-v2: Global Roots and Block Group Root"
date: 2021-11-10 12:00:00 -400
categories: kernel btrfs extent-tree-v2
---
# tl;dr

I'm working on a large set of on-disk format changes to address some of the more
painful parts of Btrfs's design.  There's a lot of discrete changes here, but
they'll all go under the single umbrella of "extent-tree-v2."  We've spent a few
months going back and forth on different approaches, and have finally settled on
a final set of changes.  The global roots and block group root patches have been
completed and submitted, but there's a lot more change coming.

# Definitions

- **Block Groups.**  Btrfs is unique in how it describes it's space, allocating
  "chunks" of space and assigning it to hold either metadata or data.  These
  chunks are called block groups internally, and these items are stored in our
  extent tree.
- **Extent tree.**  This is the most important tree in a Btrfs file system.  We
  reference count everything in the file system, every piece of data and
  metadata.
- **Csum tree.**  The tree that holds all of the checksums for the data on the
  file system.
- **Free space tree.**  This tree tells us where we can allocate space in each
  block group.
- **Global roots.**  The term I'm using to refer to the extent, csum, and free
  space trees, because they belong to the whole file system.

# Problems that we're trying to solve (with global and block group roots)

There are a few edge cases that are becoming less and less uncommon as Btrfs
adoption grows.

1. **The block group items are spread throughout the extent tree.**  The block
   group items are indexed on their starting `bytenr`, which means each
   individual block group item may have hundreds or thousands of extent items
   between them.  This usually isn't a problem, but since we need to load these
   items at mount time it can mean very long mount times for very large file
   systems because of the spread of the items.

2. **Lock contention on the global roots.**  For heavy applications that modify
   the file system in a multi-threaded way we'll often advise making subvolumes
   for the directories that are being modified.  This solves the lock contention
   on the file system tree problem.  Inside Facebook every workload runs inside
   of a container, and every container is their own snapshot, and so you can
   have multiple containers doing work on the same file system without
   interfering with each other.  However these applications all have to go to
   the same extent tree, csum tree, and free space tree as they do their work,
   and sometimes the lock contention on these global roots can negatively impact
   the workloads.

There's been work occasionally to address both of these problems.  We try to
make sure we don't have many threads modifying the extent roots at the same
time, instead choosing to serialize that work.  That sort of approach has
drastically reduced the pain, but you can still generate workloads that expose
this shortcoming.

# Solution #1: A block group tree

In addition to needing to be read at mount time, any modified block groups need
to be updated at transaction time.  This means we are searching down the extent
tree for every block group that was updated, which translates to more things
being read in, and more latency as we have to re-search for every block group.

Putting these items in their own tree gives us much better locality.  A 16KiB
leaf will contain ~680 block group items, or ~680gib of described space.  In the
worst case scenario with our original design you would have to read 680
different leaves to get the block group items at mount time, instead of 1 with
extent tree v2.

Of course Btrfs doesn't allow leaves to get completely full, so this is just
theoretical.  In real life 680 block groups would likely be in two different
leaves, so there would be 3 blocks total, the root and the two leaves.

This of course translates to better COW behavior when modifying those block
groups.  Take the example of modifying 2 adjacent block groups.  In the old
scheme I would likely end up modifying 2 leafs to update both block groups.
Assuming the extent tree has a height of 3, that's 5 blocks that are COW'ed to
update two adjacent block groups.  In the block group tree case they'll be in
the same leaf, and thus will get 2 blocks COW'ed instead.

This will drastically reduce the mount times for very large file systems, and
generally reduce write amplification.

# Solution #2: Multiple global roots

We have done a relatively good job of making our tree locking as discrete as
possible, so we're only ever locking exactly what we need and dropping locks at
every conceivable point.  However there is only so much you can do, and so a
generally sound solution for these types of problems is to spread the pain
around.

Instead of having single global roots for everybody to hammer on, I'm
introducing multiple global roots.  At mkfs time we will create `NR_CPUS` copies
of each global root.  Each block group will be assigned a `global_root_id`,
which corresponds to it's set of global roots.  For example, if you have 2
cpu's, you'll end up with 2 extent trees, 2 csum trees, and 2 free space trees.
Each tree will have their corresponding global root id.

If you are adding checksums for a specific bytenr, we will look up which block
group that bytenr is contained within, and then look up the csum root for that
block group.

If you have two threads writing to the disk at the same time in different block
groups, they will not induce lock contention on the csum roots if their block
groups are associated with a different global root.

Of course you can still have overlaps.  In the above example with 2 CPUs you
could possibly be writing to the same block group or writing to two block groups
that have the same global root id.

However with more and more CPUs the more spread out the locking pain will be.
If you know you have a heavily multi-threaded workload you can easily specify
more global roots at mkfs time.  In the future we may provide a way to add more
on the fly.

# Conclusion

As stated in the tl;dr, this is just the first part.  I started here because
it's the easiest part of these changes to wrap your head around.  There's a fair
bit of preparatory work that needed to be done, at the time of this writing
there are 4 patch series out with around 80 patches total just to implement
these two parts.

As I submit new portions of this work I'll add new posts to describe the
motivations and the problems it's trying to fix.  This is going to be a long
process, but hopefully in 6-12 months we'll have something users can start to
migrate to in order to take advantage of these improvements.
