---
layout: post
title: "Investigating performance issues"
date: 2020-09-25 11:46:00 -0400
categories: kernel btrfs tracing
---
# tl;dr
- Learn how to use BPF/BCC/bpftrace, they will make your life so much easier.
- I dump all of my scripts I write
  [here](https://github.com/josefbacik/debug-scripts), they are mostly very raw,
  but good starting points for "how the hell do I do X".

# The problem(s)

WhatsApp has been piloting their new offline storage stuff on btrfs, which has
resulted in a lot of interesting issues being discovered.  The most recent
issues have been around latencies creeping up as the file system fills up.
Obviously the first thing I suspect when these things come out is our ENOSPC
flushing mechanisms.  The more full you get, the more flushing happens, the
slower everything goes.

However throwing random patches at a problem hoping to solve the problem is a
path that leads to sadness.  Generally I like to know what exactly is going on
before I go about fixing a problem.  Enter tracing.

The initial investigation was around read latencies creeping up.  Of course the
ENOSPC flushing code shouldn't be affecting read latencies, but they could
indirectly affect them by increasing lock contention on the metadata b-trees.

# All the things that I tried first

I got pretty frustrated when looking into this problem.  There is a distinct
lack of tooling around investigating "X is slow" in the kernel.  You have a few
options here.

## perf record

`perf record` is a good option if you know you are CPU bound.  You get a nice
graph of where all the time was spent, and you can quickly spot the problem.
However the amount of investigations that were purely CPU bound that I've been
involved in are exactly 0, it's always some larger more complicated interaction.
Generally if I'm being asked to investigate something, we're already outside of
`perf record -ag` territory.

That being said there are times where it's still useful to give you an area to
dig into.  However one thing that constantly annoys me is that `perf report`
changes the reporting filtering seemingly every other day, so I often get
frustrated because it doesn't show me the same thing every time, making it a
generally useless tool for most of my investigations.

## func_graph

`func_graph` is a tracer built into `ftrace` in the kernel.  It does this cool
thing where it traces the exit and entry point of every function and gives you
this pretty little graph with times

```
 1)               |    cpuidle_select() {
 1)               |      menu_select() {
 1)               |        cpuidle_governor_latency_req() {
 1)   0.203 us    |          get_cpu_device();
 1)   0.222 us    |          pm_qos_read_value();
 1)   0.149 us    |          cpu_latency_qos_limit();
 1)   1.228 us    |        }
 1)               |        tick_nohz_get_sleep_length() {
 1)   0.214 us    |          can_stop_idle_tick();
 1)               |          tick_nohz_next_event() {
 1)   0.204 us    |            rcu_needs_cpu();
 1)               |            get_next_timer_interrupt() {
 1)   0.160 us    |              _raw_spin_lock();
 1)   0.317 us    |              __next_timer_interrupt();
 1)               |              hrtimer_get_next_event() {
 1)   0.152 us    |                _raw_spin_lock_irqsave();
 1)   0.157 us    |                _raw_spin_unlock_irqrestore();
 1)   0.757 us    |              }
 1)   1.866 us    |            }
 1)   0.152 us    |            timekeeping_max_deferment();
 1)   3.047 us    |          }
 1)               |          hrtimer_next_event_without() {
 1)   0.146 us    |            _raw_spin_lock_irqsave();
 1)   0.122 us    |            __hrtimer_next_event_base();
 1)   0.160 us    |            __hrtimer_next_event_base();
 1)   0.159 us    |            _raw_spin_unlock_irqrestore();
 1)   1.229 us    |          }
 1)   5.065 us    |        }
 1)   0.154 us    |        nr_iowait_cpu();
 1)   0.157 us    |        tick_nohz_tick_stopped();
 1)   7.987 us    |      }
 1)   8.281 us    |    }
```

This is very handy, and gives me the ability to easily see where we are spending
time without needing to figure out every little function to trace.

However this also has some drawbacks.  Firstly bugs happen, for example [I
couldn't filter on
PID](https://lore.kernel.org/lkml/20200725005048.1790-1-josef@toxicpanda.com/)
because there as a bug in the filtering.

Secondly the best way to interact with `ftrace` is via `trace-cmd`, which
handles all of the intricacies of setting up and tearing down and recording
things.  This is where I spent the majority of my time, because `trace-cmd` had
a bunch of bugs where it wasn't processing `function_graph` things properly.
`trace-cmd` has this handy tool called `trace-cmd hist` which generates a
histograph based on the recordings to tell you where you spent the most time.

I spent probably a solid 3 or 4 days hacking on this thing to get it to read
`function_graph` outputs properly.  See part of the output from `function_graph`
is the `depth` of the current callstack.  For some reason I was seeing the
`depth` field being _completely_ wrong when we had IRQs happen.  This broke the
histogram stuff because it depended on `depth` being accurate.

It turned out the `depth` wasn't wrong on purpose, we just switch to a different
CPU for the PID, but the tool was processing a single CPU at a time, which
causes problems.  However there were still other issues, and at this point I was
just speaking in swear words.

# Giving up and doing it manually, the first pass

I try to avoid writing my own BCC scripts whenever possible, mostly because
there's a time cost associated with doing that.  I have lots of debug scripts
that I've written to run down one-off issues, but generally it's a time sink I
want to avoid, I'd rather not re-invent something that I can get from another
tool.

However for this particular class of problems there's really nothing that
exists, adding a lot to my frustration and the amusement of my colleagues as my
fucks-per-second goes higher and higher.

The first pass that I used can be found
[here](https://github.com/josefbacik/debug-scripts/blob/master/timing.py).  As
you can see it's very raw and grew pretty organically.  Lots of copying and
pasting as I narrowed down the problem.

The general flow of these sort of things are like this

```
BPF_HASH(some_function_hash, u64, u64);
BPF_HASH(some_function_time, u64, u64);
BPF_HASH(main_function_hash, u64, u64);
BPF_PERF_OUTPUT(events);

typedef struct data_s {
	u64 main_function_duration;
	u64 some_function_duration;
} data_t;

int trace_some_function(struct pt_regs *ctx)
{
	u64 pid = bpf_get_current_pid_tgid();
	u64 ts = bpf_ktime_get_ns();

	some_function_hash.update(&pid, &ts);
	return 0;
}

int trace_some_function_ret(struct pt_regs *ctx)
{
	u64 pid = bpf_get_current_pid_tgid();
	u64 *val;

	val = some_function_hash.lookup(&pid);
	if (!val)
		return 0;
	u64 delta = bpf_ktime_get_ns() - *val;
	u64 zero = 0;
	val = some_function_time.lookup_or_init(&pid, &zero);
	lock_xadd(val, delta);
	return 0;
}

int trace_main_function(struct pt_regs *ctx)
{
	u64 pid = bpf_get_current_pid_tgid();
	u64 ts = bpf_ktime_get_ns();

	some_function_hash.update(&pid, &ts);
	return 0;
}

int trace_main_function_ret(struct pt_regs *ctx)
{
	u64 pid = bpf_get_current_pid_tgid();
	u64 *val;

	val = main_function_hash.lookup(&pid);
	if (!val)
		return 0;
	u64 delta = bpf_ktime_get_ns() - *val;

	if (delta < 1000000L)
		return 0;

	data_t d = {
		.main_function_duration = delta,
	};

	val = some_function_time.lookup(&pid);
	if (val)
		d.some_function_duration = va;
	events.output(ctx, &d, sizeof(d));
	return 0;
}
```

And for every `some_function` you need to add two more `BPF_HASH`'s and an entry
and exit function to trace, and then a slot in the output struct, and then the
corresponding BCC code to print out the event.  You can see how this becomes
more and more annoying the deeper down the stack you need to go.

Debugging "X is slower" is practically a very difficult thing to do.  "Reads are
slow" could be any number of things, so we want to be able to narrow down where
the slowdowns are coming from.  You can sort of infer this from `perf record` or
`offcputime.py` from BCC, but these just give you one half of the answer.  What
you really want is to see which function is actually taking up all the time.

Enter `timing.py`.  I started with the basic thing, `extent_readpages`, which is
the main entry function for reading for btrfs.  Then I worked my way down,
trying different functions to figure out where the time was being spent for any
calls that took longer than 1ms.

This isn't foolproof obviously, you can still get red-herrings.  We noticed a
problem with how btrfs was caching metadata.  Every time we had a 1ms or longer
latency spike, it was because we had to do IO to read the csum for the data
pages from disk.  This lead me to discover that we were evicting metadata pages
from cache really quickly and improperly.  Fixing this however didn't fix the
problem completely.

# Latency investigation part 2, turning my script into something less annoying

Shortly following the end of the read latency investigation, I got another
report of increased write latencies.  Again I wanted something similar to what I
used for the read investigation, but as you can see that script is highly
specialized for what I was debugging at the time.

Clearly this is an ongoing need of mine, so I sat down and made it more
[generic](https://github.com/josefbacik/debug-scripts/blob/master/timing-everything.py).
This is the highly polished version, the original version just had it so I could
copy and paste one function to add another function to trace.  Still copying and
pasting, but much easier.  The original version you can see
[here](https://github.com/josefbacik/debug-scripts/commit/a8c6dde91df9c4e972a63bba96f3315efe173045).
This time I was much faster about working out the issue, I had already learned
all of my lessons from the original investigation, so I could quickly iterate on
this one to narrow down the solution.

Above you can see we clearly have a lot of common code for each function we want
to add to the list to debug.  Figuring out commonalities is not really somtehing
that occurs to you when you have a first pass.  The second pass allowed me to
take all the information I had from my previous investigation and turn it into
something more generic and able to be used for any similar investigation that I
have in the future.

# Lessons learned

Somehow I have ended up being a lightning rod for "well that's fucked up, we
should figure out why" sort of issues.  Unfortunately for me that means I often
have a lot of spinning plates, and often hit things that nobody else notices.
The `trace-cmd` is a good example of the sort of things that happen to me in my
day-to-day work life.

* I need to investigate a latency problem
  * Lets use `function_graph`.
    * I can't trace on PID's for some reason, why is that.
      * It's broken, send a patch.
    * Use `trace-cmd hist`
      * I'm getting no stack traces, just individual functions.
        * It's relying on the `depth` field.
          * Read all of the code around `depth` in the kernel.
            * Make some changes to how it works for IRQs.
              * <1 day later> ok lets just get back to the task at hand.
        * Ignore the depth field for stack traces, we already know where we are
          in the stack trace, just internally track depth with enter->exit.
          * I have a lot of enter's with no exits, work around this.
            * This isn't working either, oh right we process per-CPU, not based
              on real time.

This whole process eats 3 days of my time.  Generally speaking I bail if I've
wandered off my primary task for more than a couple of hours, because otherwise
I'd end up rewriting some unrelated tool rather than fixing the problem that I
need to be working on.

In this particular case I knew that I have these sort of investigations _all_ of
the time.  And so investing a lot of time into having a tool that I can just run
to find the answer is time well spent in my mind.  However 3 days is excessive,
again I need to find the actual problem, not write perfect tracing tools.

I hesitated to go down the BPF/BCC route originally because I wanted to avoid
having a specialized script that would be completely useless to me in future
investigations.  What I want are solid tracing tools that can be easily picked
up and used to quickly narrow down where the problem is so I can start working
on a solution.

Unfortunately building and maintaining those tools takes time and effort that I
do not have.  BPF/BCC is invaluable for doing these sort of investigations, and
the more things I investigate the faster I can iterate on new scripts to help me
find the problem.  It's a higher barrier to entry than having a generic tool,
but in the end I spend a lot less time writing my own BPF script than trying to
get generic tools to work.
