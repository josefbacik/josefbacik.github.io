---
layout: post
title: "sched-time.py and generalizing workloads"
date: 2017-08-03 11:30:00 -0400
categories: kernel scheduler bcc bpf
---
# The problem.

My previous two blog posts talked about a problem I had been working on with a
scheduler imbalance.  This was noticed by one of our teams that does testing
with our various applications and environments.  They noticed if they ran one of
our giant applications for figuring out suggestions in one cgroup, and a cpu
stress load in another cgroup that the application would get a lot less cpu
time.

In order to reproduce this I had to get a box in this pool of boxes, pin the
task to this box specifically so I could run my kernel (otherwise our task
scheduler thing would migrate the task whenever I rebooted the box), and then I
had to wait 10-15 minutes after rebooting for the scheduler to start up and
realize it needed to start the tasks and for the tasks to then even out and hit
their steady state.

This means every time I wanted to test a kernel there was 20 minutes of time
just waiting for things to happen.  For a problem I knew nothing about that was
really really painful.  This ignores the pain of actually accomplishing things
like pinning the task to a specific box, and having that keep working for more
than a few hours.

We run into this problem all of the time.  Giant application that requires a
bunch of environment specific setup in order to reproduce the problem.  This is
a pain for me as a kernel developer, but hey at least I have access to the
environment right?  What about the guys upstream that then need to take my
patches and make sense of them, and verify what I'm seeing?

# Enter sched-time.py

Originally 
[sched-time.py](https://github.com/josefbacik/debug-scripts/blob/master/sched-time.py)
was just supposed to spit out general information about processes.  I was going
to do what we always do, poke a few places, write a C program and hope it
exhibited the same problem.  That is why the default action just spits out some
general information about the processes you are tracing.

Also keep in mind this was very much a write it while I was working on the
problem type of script, so as soon as it gave me the results I needed I
committed it and haven't touched it since.

# The BPF part of the script.

It's relatively easy to figure out how often a task is on the cpu and how often
it sleeps.  You simply trace `finish_task_switch` and you get the process going
to sleep and the process waking up.  From this we track the start and stop time
to keep track of how often the process was awake and how often it was asleep for
the tracing period.

The next piece of information we need is who wakes our task up.  A task can be
woken up for a variety of reasons including signals, the IO it was waiting on is
ready, it went to sleep because of some kernel locking, or futexes.  Keeping in
mind all we care about is how the application interacts with itself I was only
interested in futex related wakeups.  Futexes are how pthread mutex and
conditionals are implemented, which means if we trace where we enter the futex
code and capture a wake up operation then we can infer that these tasks have
some sort of locking/conditional dependency.

My script does this very generically.  We could put a little more effort into
the futex kprobes and figure out why we are waking somebody up, either because
we're unlocking a lock or because we're signaling a pthread_cond_t, but those
sort of details weren't really needed for this case.  Instead I used this to
build a very loose dependency tree between the tasks of the application I was
tracing.  I captured who woke which task up, and kept track of how many times
this occurred.

The reason we keep track of how many times a particular wakeup dependency
happens was to detect a producer->consumer relationship.  A common pattern with
multi-threaded applications is something like this

```
pthread_mutex_lock(&mylock);
list_add(&work->list, &global_work_list);
pthread_cond_broadcast(&global_work_cond);
pthread_mutex_unlock(&mylock);
```

The thing is all the workers are going to lock `mylock`, so they could end up
waking each other up and muddying the waters.  But if we notice one process
doing the bulk of the wake-ups then we know we have this sort of pattern.

# The BCC part of the script.

The BCC part of this script is super subtle and has no comments, so is a little
tricky to follow.  The following piece of code is building a view of what I
described above.  We're trying to figure out which processes woke up what other
processes.

```
waker_deps = b.get_table("wake_deps")
waker_sets = {}
for k,v in waker_deps.items():
    waker = k.waker_pid
    sleeper = k.sleeper_pid
    # we add our waker to our list because consumers may wake producers to
    # indicate they have completed their task
    if waker not in waker_sets:
        waker_sets[waker] = set([sleeper])
    elif sleeper not in waker_sets[waker]:
        waker_sets[waker].update([sleeper])
```

Assume we have pid1 as our producer and pid[234] as our consumers, the above would
give us something like the following

```
waker_sets['pid1'] = ['pid2', 'pid3', 'pid4']
waker_sets['pid2'] = ['pid1', 'pid3']
waker_sets['pid3'] = ['pid1', 'pid2']
waker_sets['pid4'] = ['pid1']
```

Then we want to reduce this dependency map into something less spaghetti like

```
def reduce(waker_sets):
    need_loop = True
    groups = {}
    counter = 0
    while need_loop:
        need_loop = False
        producer = None
        for pid,wakeset in waker_sets.items():
            found = False
            need_break = False
            for name,base in groups.items():
                if wakeset.issubset(base):
                    found = True
                    break
                elif wakeset.issuperset(base):
                    found = True
                    groups[pid] = wakeset.copy()
                    groups.pop(name, None)
                    need_break = True
                    break
                elif len(wakeset.intersection(base)):
                    need_break = True
                    waker_sets[pid] -= base
                    break
            if need_break:
                need_loop = True
                break
            if not found:
                groups[pid] = wakeset.copy()
                need_loop = True
    return groups

groups = {}
loops = 0
while True or loops > 10:
    loops += 1
    blah = reduce(waker_sets)
    if len(groups) != len(blah):
        groups = blah
        waker_sets = blah
    else:
        break
```

Ignoring that while loop is completely bogus, the purpose of this is to iterate
through the sets and reduce them to their most common forms.  Take our above
example again, it would reduce to

```
waker_sets[pid1] = ['pid2', 'pid3', 'pid4']
waker_sets[pid2] = ['pid1]
```

This is almost perfect, we want to drop the last guy, which is why we have this
bit

```
for k,v in groups.items():
    if len(v) == 1:
        groups.pop(k, None)
```

This kills any of the consumers dependency sets and leaves only the producer
dependency sets.

The next part is to build a dict that we will turn into the real json
configuration for rt-app.  The first part of this loop is the following

```
    total_time = 1000000
    runtime = v.run_time + v.preempt_time
    runevents = v.run_events
    sleeptime = v.sleep_time + v.io_time
    tdict = {}
    if v.pid in groups:
        tdict['loop'] = -1
        tdict['instance'] = 1
        if v.priority != 120:
            tdict['priority'] = v.priority - 120
        tdict['lock'] = 'mutex{}'.format(v.pid)
        tdict['broad'] = 'shared{}'.format(v.pid)
        tdict['unlock'] = 'mutex{}'.format(v.pid)
        tdict['sleep'] = 0
        threads_dict["tasks"][v.pid] = tdict
```

Here we  get our overall runtime and sleeptime.  Keep in mind we count `preempt`
time as time we wanted to run since we're on the run queue waiting to run.  This
isn't strictly accurate, I would need to fix the BPF code to ignore preempt time
in the rt-app case so we had real runtimes.

The next bit is checking to see if our pid is one of the "producers" in our wake
dependencies, and set up the appropriate mutexes and conditionals.  We name
everything after the producer pid to make it easy for the consumers to have the
right info.

The next part is for the consumers

```
    found = False
    for pid,pidset in groups.items():
        if v.pid in pidset:
            found = True
            name = "threads{}".format(pid)
            priority = 0
            if v.priority != 120:
                priority = v.priority - 120
                name = "threads{}priority{}".format(pid, priority)
            if name not in threads_dict["tasks"]:
                threads_dict["tasks"][name] = tdict
                tdict['instance'] = 0
                tdict['loop'] = -1
                if v.priority != 120:
                    tdict['priority'] = v.priority - 120
                tdict['lock'] = 'mutex{}'.format(pid)
                tdict['wait'] = { 'ref': 'shared{}'.format(pid),
                                  'mutex': 'mutex{}'.format(pid) }
                tdict['unlock'] = 'mutex{}'.format(pid)
                tdict['run'] = 0
            else:
                tdict = threads_dict["tasks"][name]
            tdict['run'] += (runtime / 1000) / runevents
            tdict['instance'] += 1
            break
```

Here we do the standard consumer pattern, which is something like

```
while (1) {
    struct work *work;
    pthread_mutex_lock(&mylock);
    pthread_cond_wait(&global_work_cond, &mylock);
    work = list_first_entry(&global_list, struct work, list);
    list_del(&work->list);
    pthread_mutex_unlock(&mylock);
    do_work(work);
}
```

Then we base the runtime on a 1 second loop.  The last bit is just for the
producer task.

```
    if found:
        continue
    tdict['instance'] = 1
    tdict['loop'] = -1
    tdict['run'] = (runtime * total_time) / (runtime + sleeptime)
    if sleeptime > 0:
        tdict['sleep'] =  (sleeptime * total_time) / (runtime + sleeptime)
    threads_dict["tasks"][v.pid] = tdict
```

We only have 1 instance of the producers, and as many instances of the consumers
as we traced.  The last part has a healthy comment for so I won't go into detail
about it here, but suffice it to say the ordering of the json elements in rt-app
is important, and python dict's are inherently un-ordered.  The last part fixes
this by ordering based on the key ordered we need for rt-app to work properly,
and then dump it to stdout.

# Conclusion

This script is kind of hairy.  It was meant to do the one thing I needed to, and
I didn't bother commenting it or making it look pretty.  There's even a fair
amount of dead code, specifically the trace_do_exit() function which was to
detect short-lived tasks.  It would probably still be useful to have that for
other applications, but the application I was tracing didn't have short-lived
threads so I just didn't bother completing that functionality.

Hopefully this is useful, the script should work for most other cpu intensive
apps, and could easily be cleaned up and expanded to fit other corner cases.
