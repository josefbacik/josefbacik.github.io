---
layout: post
title: "My patch review workflow"
date: 2021-10-19 11:30:00 -0400
categories: kernel
---

# tl;dr

Copy into your ~/.gitconfig git

```
[alias]
show-pager = -c 'core.pager=less -+F' show
review = "!f() { local c=${1:-HEAD}; git difftool -y \"$c\"~1 \"$c\"; }; f"
series-review = "!f() { local c=$1; for i in $(git log --format='%H' \"$c\"~1.. | tac); do git show-pager \"$i\"; git review \"$i\"; read -p \"Continue (Yn)? \" -n 1 -r; if [[ $REPLY =~ [Nn]$ ]]; then break; fi; done; }; f"
```

The general workflow I use is
```
$ git checkout -b patch-review <some common base>
$ b4 am -o- <msgid> | git am

# If it is a single patch, I do the following
$ git show
$ git review

# If it is a patch series, I do the following
$ git series-review <hash of the first patch in the series>
```

# Why we do patch review

We all have things we check when reviewing patches.  Some of us have spent a lot
of time fixing locking problems, so we pay close attention to any locking.  Some
of us have spent a lot of time dealing with error handling, so we closely pay
attention to error handling.  Everybody has different experiences which makes
them look at different things.  This is why patch review is so important, the
person writing the code and the person reviewing the code have different
viewpoints and different things they look for, so together we can come up with
better code.

That being said it's helpful to have a process, so you are an effective and
helpful patch reviewer.  Patch reviews will not catch all problems, but
maintainers should trust that you are doing a thorough job.  This isn't meant to
be an exhaustive list of what you should do, but these are fundamental things
that can be done to help make sure we're applying quality code.

# Things to check before you do a code review

Before you embark on the code review, there's a lot of mechanical things that
you should be checking that are just as important to a code review.

1. Does the patch apply?  Nothing is more annoying as a maintainer than to have
   patches not apply.  Usually there is a common base branch that everybody in a
   project works from, so at least validating that the patch applies is super
   important.  I will never stop plugging
   [`b4`](https://people.kernel.org/monsieuricon/introducing-b4-and-patch-attestation)
   for this, it is invaluable to me.
2. Does it build?  Again, sanity checks are important.  Often people will simply
   look at the patch in their mail client and add their Reviewed-by without
   making sure the patch even builds.  The automated patch building
   infrastructure is nice, but it never hurts to double check.
3. Does the commit message make sense?  Do you understand what the point of the
   patch is?  If it's a deadlock fix, is the deadlock sufficiently explained?
   If it's a panic or lockdep splat fix, is that message included in the commit
   message so that distro's can easily find a fix for a problem they may hit in
   older kernels?
4. Does the patch actually reflect what is in the commit message?  Sometimes
   while rebasing you can end up with things munged and you get code in a patch
   you didn't mean to and you don't notice it.  Are you only fixing the one
   thing you mentioned in the commit message?  1 change per commit, if the patch
   is fixing a deadlock, but changes formatting somewhere unrelated that should
   be in a different patch.
5. Does this patch make sense?  This is a more ambiguous requirement, but
   sometimes developers will fix a symptom of a problem rather than the root
   cause.  Is this fixing the root cause of the problem, or merely the symptom?
   Is the feature we're trying to add make sense to do in the first place?

Generally with experienced developers you can pick and choose how strongly
you're going to adhere to these specific rules.  For example if I get a single
patch from any of the core developers on btrfs and it's a few lines, I'll
sometimes skip step 1 and 2 because I know they likely did the correct thing.

# Using `git show`, `git reveiw`, and `git series-review`

These all serve a different purpose.  `git series-review` is simply doing the
`git show` and `git review` for all patches in a series, so mechanically is the
same as what I'll describe.

`git show` is important for these pre-checks I've listed above.  It lets you
look at the change as a whole.  You can read the commit message, you can check
the files modified by the patch to make sure there's not something crazy that
ended up there.  You can easily spot places where the author may have
accidentally included format changes in a functional change.  `git show` is a
great sanity check to validate the change as a whole makes sense.

`git review` is one of the macros I put above.  It uses `git difftool` on the
patch, which since I use `vim` I have defined in my `.gitconfig` as

```
[diff]
        tool = vimdiff
```

This gives me a side-by-side view of every change this patch makes in the code
context.  This is exceedingly valuable in code reviews, because it allows you to
go directly to the change, see how it looks in the code and validate any
side-effects it may have in the context of the code where the change happened.

# Effective code review strategies

As indicated above, this part is murkier and more driven by the reviewers
experience than anything else.  This list is highly personal, but is a good set
of core things for anybody to check, so is a good starting point for anybody
with less reviewing experience.

1. Error handling.  If we're using functions that return errors, do we properly
   check the return value of those functions?  Do those functions have special
   return codes that mean something special, and are they handled properly?  Do
   we free memory, drop locks, drop reference counts, and otherwise properly
   cleanup if we get an error?
2. Locking.  Do we lock and unlock properly?  Do any of the functions we call
   with the lock held do the correct thing with the lock held?  If we add a new
   function that requires a specific lock held, do we have `lockdep_assert_held`
   in that function to make sure we catch mistakes during testing?
3. Are we using the correct `GFP_` flags for our allocations?
4. If we're doing big loops, do we need a `cond_resched()` in there?
5. Is the reference counting done properly if there is any?

This list could be a mile long with all of the different situations and
different things you could check.  These are the big ticket items that are less
likely to be caught in testing.  Between these checks and the general checks you
can feel pretty comfortable putting your Reviewed-by tag on a patch.

# An argument for `git` hooks

We can make everybody's life easier by automating a lot of these checks.  For
btrfs we have [patch submission
guidelines](https://github.com/btrfs/btrfs-workflow/blob/master/patch-submission.md)
that outline the general things we expect from developers.  Some of this is how
to integrate with our github workflow, but part of it is getting our `git` hooks
setup on your repository.  These check a few things

1. `applypatch-msg` and `commit-msg`.  This runs `codespell` against the commit
   message when you commit it and when you apply a patch.  This means that the
   developer will get an error if they misspelled something on commit, and if
   they don't have this setup then the reviewer will get the error when they
   apply the patch for review.  A nice simply way to automate spelling checks on
   the commit message.
2. `pre-applypatch` and `pre-commit`.  This is what we run before we commit or
   apply the patch.  For our hook we do a few things.
    1. Build the code.  This is very useful for patch series, because maybe the
       whole series builds, but an individual patch in the middle doesn't.  The
       patch reviewer can catch these problems because the tree will be built
       after every patch is applied.
    2. Run `checkpatch.pl`.  Every kernel developer has something bad to say about
       this script, but it is handy for catching basic problems.  This can be
       especially good for new developers to avoid simple mistakes before
       submitting patches.  For btrfs we have the things we don't care about
       turned off, and we have a local copy so any upstream changes don't break
       our workflow.
    3. Run the `kernel-doc` checker.  We all forget the format for kernel-doc, so
       this makes sure we did the correct thing.  This has the same caveat as
       `checkpatch.pl` in that sometimes it complains when it shouldn't, but is a
       good sanity checker in general.

This set of automation is beneficial for the review and submitter alike.  It
keeps basic mistakes from being made, and it catches basic mistakes without the
reviewer needing to put too much thought into every change.

# Reviewing does not replace testing

As I stated above, reviewing is extremely important, but we're humans and not
computers, we cannot conceive every possible consequence that a patch will
induce.  For btrfs we have continual integration testing, every night xfstests
is run against our development branch with a variety of options and hardware.
Bugs slip through, that is why we test.  Review is not a replacement for
testing, but is a very important first step before we start integrating code.

We can do a lot of things as reviewers and engineers to make testing even more
effective.  Using annotations like `lockdep_assert_held()` is a good example of
how a reviewer can hold a patch submitter to a higher standard, and it will pay
dividends later on when automated testing trips over the assert in a path that
wasn't considered during review.

Not only that these sort of changes make it easier for problems to be caught
before the patch is even submitted by the testing done by the path author.

The more tools and automation we can build around these tasks the easier it is
on the developers and on the reviewers.  The easier it is for us to get patches
right the first time means that we can move faster and feel safer as we fix
things and build new features.
