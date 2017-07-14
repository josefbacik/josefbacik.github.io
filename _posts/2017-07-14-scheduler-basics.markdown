---
layout: post
title: "Linux Kernel Scheduler Basics"
date: 2017-07-14 10:30:00 -0400
categories: kernel scheduler
---
# Intro and Definitions

I've recently been working on a problem with the Linux scheduler, specifically
the Completely Fair Scheduler part.  There are different modes for tasks to be
scheduled, but by and large everybody uses SCHED_OTHER/SCHED_NORMAL, which is
handled by CFS.  The problem I have been trying to fix is an imbalance of CPU
time between two equally weighted cgroups.  In one group we have our application
that does normal application things.  It goes to sleep, it wakes up, it wakes
other threads up, and it does work.  The other group is `stress -c`, which
simply calculates pi in a loop.

This problem was complex and difficult, made especially difficult by the fact
that I have 0 experience in the scheduler (outside of messing with wake affinity
a few years ago).  I spent two months on this problem and I learned a lot about
how the scheduler works.  Obviously the Linux Kernel changes constantly, so this
is likely to be outdated at some point, so I'll try to stick to just the basics.
I'm also still learning the code, so I may make mistakes.  With that in mind
lets start with a few basic definitions.

* `struct task_struct` - this is our task object.  Every process in the system
 has this, and it holds a lot of information.  It'll hold mm information, it's
 name, statistics, etc.
* `struct sched_entity` - this is the schedule-able
 object.  Every task_struct has a sched_entity, as well as group cfs_rq's.
* `struct rq` - the core per-cpu run queue object.  This holds all of the state
 information for the current CPU, including the individual schedulers private
 run queue structures.
* `struct cfs_rq` - the completely fair scheduler run queue
 object.  This contains an rb-tree of the tasks that want to run for this
 group/cpu.
* `struct task_group` - this is how we group together tasks in a
 cgroup.  Each task group has a per-cpu cfs_rq, which has a sched_entity.
* load avg - this isn't like the load average you get from top.  This is the time
spent on the cpu times the weight divided by the current time period.  The
load average is used to determine load balancing and such.
* scheduler tick - In order to make sure running tasks that never put themselves
to sleep are preempted at some point we have a timer that fires at a certain interval
(depending on how you've configured your kernel) in order to interrupt the
task and do house keeping.  Load average gets updated here, the load balancer
gets run, and decisions about rescheduling also occur here.
* `sched_entity->vruntime` - this is the virtual runtime of the `sched_entity`.
This is the key that is used to place the `sched_entity` in the cfs_rq tree,
and what is advanced every scheduler tick based on the weight of the
`sched_entity`.

# How this fits together

On fork() a new `struct task_struct` is created for the new process.  This
process's `sched_entity->vruntime` is set to the parent task's `vruntime`.

Once the parent task goes to sleep, or is preempted, we need to find the next
person to run.  The bulk of this logic is handled by __schedule().  We call into
`pick_next_task_fair`, which looks for the next runner and returns it to the
core scheduler code.  From there we context switch into the new process.

Anytime we are in the scheduler, wether we are adding or removing tasks from the
runqueue, selecting the next task to run, doing our scheduler tick, or waking up
a task we always update the load average for the currently running task.  We
will also update the load average for the `cfs_rq` for our current task.

# Well that doesn't seem particularly complicated?

With no cgroups and no `nice` value changes it is relatively simple.  The devil
is in the details however.  Lets go back to our original example, but this time
we put our tasks in a cgroup.  Now we have a private `cfs_rq` for just the tasks
in our group, and our `cfs_rq` has a `sched_entity` on the CPU `cfs_rq`.  Still
not too complicated.  Now the steps are more like this:

1. `update_curr()` - this updates the `sched_entity->vruntime` for the currently
running group.
2. `pick_next_entity()` - find the first entry on the `cfs_rq` rb_tree.  We will
assume that the only thing on this CPU is this group, which means that the
rb_tree will be empty.  Because it's empty our `cfs_rq->curr` (our currently
running `sched_entity`) will be the `sched_entity` we pick.
3. `cfs_rq = group_cfs_rq(se);` - now we get the child `cfs_rq` of our
`sched_entity`.  This will be the `cfs_rq` that has our actual tasks on it.
4. Loop to #1 and do the same things, only this time we will find our newly
fork()'ed process and break from this loop as that `sched_entity` will not have
a child `cfs_rq` as it is a real task.

# Well ok that was a little more complicated, but still manageable?

Lets talk about how weights are calculated now.  The task weights tend to not
change, they are based on the weight of the group you are put into, so there's a
static weight per task.  Every time a `sched_entity` is enqueued (added to the
runqueue) it's weight is added to the `cfs_rq->load.weight`.  Then we
recursively walk up the hierarchy and update the `sched_entity` for every
`cfs_rq` using the following code (trimmed of the comments)

```
static long calc_cfs_shares(struct cfs_rq *cfs_rq)
{
        long tg_weight, tg_shares, load, shares;
        struct task_group *tg = cfs_rq->tg;

        tg_shares = READ_ONCE(tg->shares);

        load = max(scale_load_down(cfs_rq->load.weight), cfs_rq->avg.load_avg);

        tg_weight = atomic_long_read(&tg->load_avg);

        /* Ensure tg_weight >= load */
        tg_weight -= cfs_rq->tg_load_avg_contrib;
        tg_weight += load;

        shares = (tg_shares * load);
        if (tg_weight)
                shares /= tg_weight;

        return clamp_t(long, shares, MIN_SHARES, tg_shares);
}
```
The value returned from this function is what we set the `cfs_rq`'s
`sched_entity->load.weight` to.  What this function is doing equates to
```
shares = (total shares for the group)(load of group on this cpu)
         -------------------------------------------------------
                  total load of the group on all cpus
```
We want the weight of this `cfs_rq`'s `sched_entity` to be this CPU's share of
the overal load of the entire group.

What this means in practice is that adding a heavily weighted process to one
group is not going to have an obvious direct effect to the load that the root
`cfs_rq` sees.  As more and more steps are added to the hierarchy, more and more
our weight gets diluted.

Logically we look at two tasks in two different groups, one has 10 times the
weight of the other, and we assume that means that heavier weighted task will
get to run 10 times as much.  But this isn't necessarily the case.  If the
groups themselves are equally weighted, the heavier task will get closer to
equal runtime, depending on the load average distribution of both groups across
other CPUs.

# I'll admit that's a little murkier...

Now we need to talk about load averages.  Weight is nicely correlated, adding
and removing weight from a `cfs_rq` has an immediate impact on the entire
hierarchy.  However weight doesn't matter directly to the system, load average
does.  Load average only changes as the task is running on the CPU (it decays at
enqueue time, but that's not important now).

What this detail means is that if a task from another group is currently running
on the CPU you can essentially wake up as many tasks as you want from a
different group on the same CPU and see no difference in load on that CPU.  You
aren't supposed to be able to do this because we have logic in place to prevent
it, but it turns out this logic was broken, which was part of the problem that I
was having.

# Conclusion

This is just a surface look at how CFS works and how some of the different
pieces fit together.  One of the purposes of this exercise is to provide a way
for me to sort out the details properly in my head, and preserve the knowledge
for future problems.  I will go into detail about the problem I was working on
in the next post.
