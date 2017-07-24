---
layout: post
title: "CPU Scheduler imbalance with cgroups"
date: 2017-07-24 12:00:00 -0400
categories: kernel scheduler cgroup
---
# The scenario

Job isolation is so hot right now.  Everybody has loads of giant servers and
loads of jobs, but many of those jobs don't need an entire server all to
themselves.  It would be way more efficient with resource usage to be able to
stack a bunch of applications on the same server and let them do their thing.
Ideally you don't want any application to affect the other applications on the
system, so you want to employ resource limits on the applications to ensure
everybody behaves.

Enter cgroups, which allows you to put resource limits on various parts of an
applications behavior.  You can limit memory usage, cpu usage, io bandwidth,
etc.  In order to test that this works properly one of our teams was testing the
cpu part of this controller.  They made two jobs, one that was a `stress -n <#
cpus>` command, and another that was a real live application we use.  The amount
of cpu time is tracked per cgroup in `/sys/fs/cgroup/path/to/group/cpu.stat` in
cgroupsv2, and they noticed that the `stress` group got around 50% more cpu time
than the real application, despite being given equal weight.

# First things first

The first thing I needed to do was to classify our real application, because
starting all of the infrastructure for this test took a lot of time and was kind
of fragile.  Also I needed to be able to show upstream the problem, and I
couldn't very well take our giant application and hand it to them to and tell
them how to build and set it all up.

I created
[this](https://github.com/josefbacik/debug-scripts/blob/master/sched-time.py)
which I used to generate an `rt-app` configuration that was representative of
our workload.  Then a simple reproducer script
[here](https://github.com/josefbacik/debug-scripts/blob/master/unbalanced-reproducer/unbalanced.sh)
in order to quickly iterate on my problem.

# The problem generally

If you haven't read my [previous]({% post_url 2017-07-14-scheduler-basics %})
post on scheduler basics you are going to want to do that now, we're going to go
into the weeds a bit here.

The problem is generally because of how the processes behave.  Whenever a
process is runnable it sits on it's respective runqueue.  Because `stress` is
just a cpu consumer it never leaves the runqueue, which means that a lot of the
work the scheduler does just doesn't happen for any of the `stress` processes.
The real application however does real application things, it goes to sleep,
wakes up, wakes up other threads, waits on IO, etc.  All of these actions
interact with the scheduler in various ways, which means there were lots of
little ways our application would lose CPU time to the `stress` application.

The problem turned out to be a combination of 3 things.  First was the lazy
weight and load propagation up the `cfs_rq` hierarchy.  Second was how we
calculated the `cfs_rq` shares on the cpu at enqueue time.  And
finally our good friend wake affinity reared it's ugly head again and made a
mess of everything.

# Load propagation up the hierarchy

There's two load measurements we make on a `cfs_rq` and `sched_entity`, they are
the `load_avg` and `runnable_load_avg`.  `load_avg` is the load of the task that
gets updated when we are running and sleeping.  When we sleep `load_avg` goes
down, when we run `load_avg` goes up.  `runnable_load_avg` is slightly different
in that it only gets updated when we're running.  It can go down if we are
asleep for a long period of time, but it only reflects the load of the process
when it's running.  Both of these get propagated up the hierarchy, but only the
`runnable_load_avg` matters when it comes to load balancing the cpus (ish, it's
more complicated than this but for all intents and purposes it's the important
one).

Prior to Peter Zijlstra's
[patches](http://lkml.kernel.org/r/20170512164416.108843033@infradead.org) to
fix this problem we never propagated the `runnable_load_avg` of newly attached
`cfs_rq`'s up the hierarchy, only the `load_avg`.  This meant that the load
balancer was usually dealing with stale information, and would make improper
load balancing decisions.  This affected our workload because our tasks would go
on and off the runqueue while the `stress` group would not, which exacerbated
the issue.

Peter's patches fixed this problem by making the `runnable_load_avg` be
completely re-calculated every time we have a weight change in the hierarchy.
Before we would only change the `load_avg`, and we also only updated the
`load_avg` by a delta of the load change based on the new weights.  Now with his
patches we keep a `runnable_load_sum`, which is the time spent runnable, and
then calculate the `runnable_load_avg` by using our new weight in the
calculation with `runnable_load_sum`.  Changes are now immediate and more
accurate.

# A slight aside.

Peter's patches introduced a sharp regression to my test case.  I spent a good
amount of time trying to track this down, and eventually realized that his
calculation of `runnable_weight` suffered from the same problem as how we
calculated our `cfs_rq` shares as described below.  The fix for this specific
problem is in [this
patch](http://lkml.kernel.org/r/1500038464-8742-3-git-send-email-josef@toxicpanda.com).

# Mis-calculation of `cfs_rq` shares at enqueue time

The code in my previous post is the updated version of `calc_cfs_shares`, but
previously our current load was calculated like this

```
load = cfs_rq->avg.load_avg
```

which means that if we suddenly woke up 4 processes on this `cfs_rq`, the load
would be the historical load, rather than a reflection of the current load.
This code got changed to the following

```
load = max(scale_load_down(cfs_rq->load.weight), cfs_rq->avg.load_avg);
```

`load_avg` operates in the range of 0 - load.weight.  This change meant that at
_enqueue_ time we would get credit for our theoretical maximum load, as
`cfs_rq->load.weight` is updated with the `sched_entity->load.weight` at enqueue
time.

The result was that we were now getting an immediate reflection of probable load
by newly woken tasks, which made load balancing more accurate.

# Wake affinity effective load miscalculation

Wake affinity is the schedulers way of trying to prefer cache locality over
wakeup latency.  The idea is that all things being equal, we'd rather pay the
cost of migrating a task to the cpu where we think it may have cache locality
than deal with migrating cache between cpus.  This logic isn't super
complicated, we basically have a counter to see if we have a multi-waker to
wakee relationship, and if we do make a sort of half-assed load balance decision
and migrate the task to our current cpu.

The half-assed load balance decision is part of the problem.  Since we make load
balancing decisions at the cpu level, and our `sched_entity` is buried under a
`cfs_rq` hierarchy, we have to calculate the load difference each cpu would see
by waking this task up on either cpu.  How this happens is we do a recursive
walk up the `cfs_rq` hierarchy doing a modified version of `calc_cfs_shares` all
the way up until we have the delta of the load that would be visible to the cpu.

You're going to need to read that paragraph a few more times.

How this is different from the normal state of things is that we add our
`sched_entity->avg.load_avg` to the `cfs_rq->avg.load_avg`, calculate what our
theoretical shares would be for the `cfs_rq`'s `sched_entity`, subtract the
actual value of the `cfs_rq`'s `sched_entity` and do this recursively.  The
problem is in reality, at enqueue time, we _only_ add our
`sched_entity->load.weight` to our `cfs_rq->load.weight`, then we calculate our
shares from there, and set our `sched_entity->load.weight` to the new weight.
Now this isn't a huge deal, it's supposed to be an approximation and it's
definitely approximate, but it isn't a reflection of what happens in reality.
And as we see above, the `load_avg` can bias us against the group that goes to
sleep occasionally.  With load balancing decisions we want to be using the
theoretical maximum load a new task would place on the hierarchy.

Next we have a significantly more subtle problem.  We are calculating our new
`sched_entity->load.weight` using current `load_avg` numbers.  Our
`sched_entity->load.weight` that we subtract from our newly calculated weight
was computed using historic `load_avg` numbers, which could have been higher at
the time.  This means that when we're trying to calculate how much load we would
add to the cpu by moving our task to that cpu, the calculation would actually
make it look like we were removing load from the cpu.  This meant we were
constantly migrating tasks when in reality we should not have been.  Since we
can easily calculate the "old" weight by using the current numbers without the
tasks added weight we do this so our delta is consistent with our current
`load_avg` numbers.

This was a doozy.  Even once I understood everything that was going on it still
took me a few days (probably more like a week) to get the math and logic right
here and realize what was going on.

# Wake affinity ping-ponging

The last aspect of the problem was wake affinity ping-ponging our tasks.  As I
stated above, if everything is equal between two cpu's, we will prefer to wake
up the task on the waker's cpu.  This blanket assumption that these things
_maybe_ share cache meant that we were ping-ponging tasks around constantly,
which was still causing problems.  We would wake affine a process, then the load
balancer would come behind us and put the process back on another cpu.

This problem was much easier to understand and more straightforward to fix.  I
simply kept track of the last time the load balancer moves us off of a cpu, and
if it has been in the last second don't do a affinity wakeup.  This got us the
rest of the way there, and now we have an even 50-50 split between the two
groups.

# Conclusion

This problem sucked, I spent a solid 2 months on it.  Many thanks to Peter
Zijlstra and Tejun Heo for answering my questions and being my sounding board.
I had very little scheduler experience before tackling this problem, and it
ended up being much more involved than I expected.  The whole series has been
sent up to fix the problem, I imagine there will be a lot of discussion and the
patches that eventually go upstream won't look anything like these ones, but for
now you can look at them
[here](http://lkml.kernel.org/r/1500038464-8742-2-git-send-email-josef@toxicpanda.com).
