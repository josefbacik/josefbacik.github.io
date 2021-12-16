---
layout: post
title: "Btrfs: how reference counting works"
date: 2021-12-16 09:00:00 -400
categories: kernel btrfs
---
# tl;dr

Don't read this, it's a very dense description of how our extent referencing
works.  You've been warned.

# A brief description of Btrfs's reference counting

Everything is reference counted.  Every block that is allocated is reference
counted.  This is the basis for how snapshotting works.  You allocate a block on
disk, you add a reference to the extent tree for that block describing who owns
that block and how many references they have.  When you free a block you are
simply dropping your reference to that block, and when the reference count for
that block reaches 0 it is freed and able to be re-allocated in a subsequent
transaction.

This applies to all data and all metadata.  So when you snapshot a subvolume
you'll create a new root, copy the original root, and add a reference to all of
the children of that block for the new root.  Then you need to COW any of the
children twice to reclaim the block.  Nice, easy to understand reference
counting.

However for global roots, such as the extent tree or csum tree, they can't be
shared.  They're global, they aren't ever snapshotted.  For the extent tree this
means any time you have to allocate a block for the extent tree you have to add
another entry to the extent tree, which could generate more blocks which
generates more entries in the extent tree.

# This sounds expensive

It is.
   
For example, with a clean fs, if you touch a file, you'll modify 1 block for the
inode item, 1 block for the extent tree, 1 block for the free space tree, and
then of course the super block.  This is because a clean file system has 0 level
roots everywhere, so the extent references for the extent tree will be in the
same block as the reference for the subvolume block.

However consider a much larger file system, with 4 level high trees.  To modify
that inode item you have to allocate 4 blocks in the subvolume.  That's 8 extent
references that need to be updated, 4 to remove the references of the old blocks
and 4 to insert the references for the new blocks.  Say we get lucky and these 8
references are only spread across 4 leafs in the extent tree, that's 8 blocks
minimum in the extent tree, 13 blocks maximum.  Now each of those is their own
extent entry that needs to be modified.  As you can see with a larger file
system modifying 1 item suddenly explodes into a larger cost.

The cost of modifying 1 item went from 4 blocks to 40 for a larger fs.  That's a
comically stupid amount of write amplification.  However this cost is spread out
across all modifications.  Users rarely touch one file and then do nothing else,
they'll modify a bunch of items.  So if you touch 100 files in the same
directory you could still only need to modify 1 leaf in that subvolume, so your
100 item updates still only cost 40 blocks.

# So fucking why?

Because it's easy to wrap your head around.  Every programmer understands how
reference counting works.  It's easy (ish) to implement snapshotting with.  You
add references when add a snapshot, and when reference counts drop to 0 you can
free the block.

There's also some really nice benefits from this that you wouldn't think of
immediately.  For btrfs we have checksums, so if we fail to read a metadata
block we can easily query the extent tree and figure out who owned that block
and tell the user what has gone wrong.

It also means things like scrub are nice and simple.  We have a source of truth
for all allocated blocks on the file system, so we just have to walk this one
tree and read all the blocks to validate the file system.

Originally we didn't have a free space tree, we just used gaps in the extent
tree to build a view of the free space in the file system.

We can look up who owns all blocks for things like quotas so we can tell how
many blocks are shared and not shared between a subvolume and its snapshots.

We use this for relocation.  Mark a block group as read only, walk through the
extents allocated in that block group and copy them to other block groups.

We gain a lot of things from having every block tracked in a single location.
The cost is write amplification.

# Conclusion

The extent reference counting has served us well for more than a decade.  It's
one of the core concepts that Btrfs is built on.  The original paper that Btrfs
is built on described how to do the reference counting to get low-cost snapshots
with a COW B-Tree, and that's why we have this system.

However more than a decade of experience has given us new ideas of how to get
all of these features without the write amplification cost.  Which is why we're
putting effort into the extent-tree-v2 work.
