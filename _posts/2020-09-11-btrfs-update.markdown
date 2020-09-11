---
layout: post
title: "Btrfs development update"
date: 2020-09-11 11:19:00 -0400
categories: kernel btrfs
---
# tl;dr
- Fedora Workstation and all of its descendants are switching to btrfs by
  default, hooray!
- The core btrfs team is organizing more under
  [Github](https://github.com/btrfs).
- Development update.
- I'm going to attempt to be more communicative about what we as the btrfs
  community are up to development wise and priority wise.

# Fedora

Chris Murphy and Neal Gompa have done an excellent job driving btrfs adoption
with Fedora, and decided to kick it up a notch for Fedora 33 by switching to
btrfs as the default file system for the Fedora Workstation spin and all of its
descendants.  Community work is a long an arduous job, and they've both done
excellent work on that front.  From the Facebook side we've had Davide Cavalca
and Michel Salim working closely with Chris and Neal to address the variety of
policy and user space questions around using btrfs by default.  It has been a
ton of work and I've been very thankful that I wasn't doing any of it, I just
got to show up and answer questions and fix bugs.

# Becoming a better organized development community

Poor Dave Sterba, who does an awesome job maintaining the btrfs tree and
handling the patches and pull requests, has had a hell of a time trying to get
the rest of us to pay attention and review patches and test things more
thoroughly.  For myself I often go a few days in between checking the
mailing list and sometimes miss what needs to be reviewed and what is already in
the tree.  Often times I forget to follow up on my own patches and things get
lost.

We've had a lot of discussions over the last few weeks and have spent a bit of
time working out how we want our development workflow to work.  We clearly need
a better way to track things that are in flight.  We have previously toyed with
moving things to Github, and already have a btrfs project on Github.  He and I
have written a few documents and scripts to help with this, and they can be
found [here](https://github.com/btrfs/btrfs-workflow).

All of our patch submissions will be tracked as individual Github issues, so we
know what needs to be reviewed and what is pending.  We are tracking xfstest
failures in our fstests tree, and will start staging fixes there so we don't
step on each others toes when adding new tests.  Getting a group of developers
who have different companies and different bosses on the same page is tricky,
but we think this is a step in the right direction.

# What's coming this cycle?

Dave likes to keep things rolling but give us time to stabilize before the next
merge window, so he cuts off new big changes around halfway through the current
development cycle.  As such what's in his misc-next branch is all the big things
that are going into the 5.10 merge window.  Most of what is currently there is
not super interesting or flashy, but solid work nonetheless.

- Goldwyn Rodrigues' work to convert our O_DIRECT path from the old O_DIRECT
  path to the new iomap based O_DIRECT code.  This cleans up a lot of our cruft
  around O_DIRECT, and will enable some of his locking changes in this area.
- Filipe Manana's fsync performance improvements.  Filipe has spent a lot of
  time meticulously going through our tree log code to make it more robust and
  faster.
- Nikolay Borisov cleaned up our device handling code for seed devices.  This
  has been a weird wart that's led to a variety of strange issues when dealing
  with seed devices.  These cleanups enable us to make some other changes in a
  cleaner way.
- Anand Jain fixed a bunch of issues with seed devices and initializing them in
  a few of the lesser used configurations.  This rework was enabled by
  Nikolay's work.
- Qu Wenruo continues to chip away at different qgroup accounting issues, as
  well as some error handling fixes.
- My prep work for reworking the tree node locking has gone in.  The actual tree
  locking changes will go in for 5.11, as they're a bit disruptive to go in at
  this stage.
- I've fixed a ton of lockdep splats we've hit, hopefully we are now lockdep
  splat free (at least until I change all the locking again).
- My data reservation ticketing rework.  All btrfs metadata reservations use a
  ticketing infrastructure to make sure the space reservation is FIFO, but data
  was still a "whoever wins the race gets the reservation."  This generally was
  OK because data reservation to usage is generally 1:1, unlike metadata.
  However if we did need to flush at all then you could end up with early ENOSPC
  in a few corner cases.  This rework eliminates that class of issues, and makes
  all of the ENOSPC handling for both data and metadata unified.

# What's in the pipeline?

I can't really speak for anybody but myself, our day to day priorities change
with our daily job responsibilities.  But there are a few clear things that I
know people are working on.

- Omar Sandoval is working on RWF_ENCODED in order to support doing send/receive
  with encoded data.  Right now btrfs has built in compression, but if you
  send/receive that data, we have to decompress the data, send it over the wire,
  and then re-compress on the other side.  RWF_ENCODED will allow us to send
  over the raw compressed bytes, and then write then straight to disk on the
  other side.
- Omar is also working on subvolume encryption, which is why the RWF_ENCODED
  work is so important.  In order to send encrypted containers around we don't
  want to have to unencrypt and then re-encrypt the data on the other side,
  which is why he's doing RWF_ENCODED first.  The native subvolume encryption
  will work exactly the way
  [fscrypt](https://www.kernel.org/doc/html/v4.18/filesystems/fscrypt.html) does
  for ext4.  You get encrypted filenames and data.  It is not block level
  encryption, you will still need LUKs or something like that if you have those
  kind of requirements.
- Boris Burkov is working on polishing up the free space tree.  The free space
  tree really helps with a lot of latency related issues we've seen in
  production inside Facebook, so we want to get this cleaned up and working well
  upstream in order to switch to it as the default for all new btrfs file
  systems.
- Goldwyn is continuing to hammer on the O_DIRECT iomap stuff, cleaning up a
  locking issue we found with his conversion and a few related things.
- Naohiro Aota is plugging away at adding native zoned block device support to
  btrfs.  Being able to directly do this means you can eek out better
  performance from your zoned block devices, as we just write in a way that the
  devices handle naturally.
- Qu is working on sub-page size block size support.  This has been a long term
  ask with a variety of different attempts throughout the years.  Qu is taking a
  more piece by piece approach which may be more successful than previous
  attempts.

I'm sure there is other work that I've missed and I apologize to the authors,
but clearly we have a lot of good work going on.

# On communication

One thing that has become clear to me during this whole Fedora discussion is how
little people know about what is happening with btrfs.  People read posts on
reddit, phoronix, or other such sites and get a very warped view of what is
happening with the btrfs project.  It is clear that we as a development
community need to speak more openly about the work we are doing, what things we
are trying to address, the successes, and the failures.

I consistently saw a lot of misinformation about how Facebook uses btrfs in
production, claims that we barely used it or only used it for specific workloads
that didn't represent reality.  This is a failure on my part, one that I made a
goal of addressing this year.  Part of fulfilling that goal will be posting more
regularly about what we are doing as a community.

There is still a lot of work for us to do, and we as a development community
tend to keep our head down and do that work.  I will attempt to stick my head up
every once and a while and talk about what we're working on and what is coming
in each kernel version.
