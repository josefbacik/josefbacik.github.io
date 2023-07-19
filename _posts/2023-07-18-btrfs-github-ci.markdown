---
layout: post
title: "Setting up GitHub Actions to run fstests for Btrfs"
date: 2023-07-18 14:05:00 -0400
categories: kernel
---
# tl;dr
- We (the Btrfs team) have a new testing setup that uses GitHub actions, you can
  find the yml [`here.`](https://github.com/btrfs/linux/blob/ci/.github/workflows/ci.yml)
- Running fstests requires a two step process, one to build the kernel and
  deploy it onto the runners, then the step to run the tests on the runners.
- The GitHub self hosted runner masks SIGPIPE, which is pretty necessary for any
  shell based testing, so you need a wrapper like the one provided below to keep
  things from breaking.

# The old setup

We have a bunch of VM's that run a variety of different configurations through
fstests every night.  It pulls our development branch, builds it, and kicks off
the run.  This is running on an Intel NUC on my desk, not exactly fast, it takes
around 18 hours to run all the tests and get results.  Of course all of the
developers run fstests as part of their normal workflow, but this keeps us from
missing anything as we put all the patchsets together.

These VM's were setup with some random version of Fedora Rawhide around Fedora
30, so I haven't been able to update them and they sometimes break.
Additionally I'm the only one who can touch the machines, and the results are
saved in a git tree, parsed and uploaded to my VPS machine so we can keep track
of how everything is running.

For now this will continue to run, as we run the entire suite and it lets us
keep track of what tests are potentially flakey.

# The new setup

There's a lot of flaws with the old setup, but one of the more annoying things
is that only I can run it, and it's automated via cron jobs.  This means I can
run some local tests for developers, but theres a small window I can run them in
without taking the whole thing offline.

Now I have two machines, one that is x86 that runs the majority of the tests,
and a Mac Studio running Asahi Linux for the subpage blocksize tests.  When
everything is working normally I can get a full run done in around 4.5 hours,
which is a pretty drastic improvement over the old hardware.

Additionally we can now trigger the tests through pushes and pull requests.
Developers can now get their code tested before they send it for review on a
variety of configurations.  Almost nobody has an ARM64 machine that is fast
enough to run tests, so this gives us much better code coverage for work before
it's merged.

# How is the system designed

**Self hosted runners**.  Since we're formatting disks and have special storage
requirements it goes without saying we need self hosted runners.  These fall
into 2 categories.

  - **Build and deploy**.  These are the 'vmhost' tagged machines.  These are
    the boxes the VM's are running in, the x86 machine and the Mac Studio.
    They're going to build the kernels and deploy them onto the VM's.  The
    `build`/`build-arm` and the `deploy`/`deploy-arm` stages run in parallel.
  - **fstests vms**.  These are the VMs that are going to actually run fstests.
    Each of them has all of the fstests configs for their respective vmhost
    type.  This is the `config_name` in the matrix strategy.  Once the VM's are
    booted the runner will start and they'll pick up the next job.

# Setting up the VMs

Virtualization doesn't work well out of the box in Linux, there's a host of
things that go wrong and weird little annoyances that making using it as the
basis for anything a huge pain in the ass.  I did this with Fedora 38 and
libvirt, what I thought would be the most mature software stack and able to
handle my very basic use case.  This wasn't necessarily the case, there were a
whole host of annoying problems.

- **Guests created with the cloud image wouldn't boot on reboot.**  There was
  some bug that kept them from booting normally once you installed a new kernel
  from a fresh boot.  I had to use `--uefi` with virt-installer in order to get
  this to work.
- **Fedora 38 defaults to secure boot for uefi, this is broken.**  I'm trying to
  install custom built kernels, and for some reason doing the make install
  doesn't do the appropriate signing or anything, so you have to disable secure
  boot to get these things to work.  This required using the options

  ```
  --xml ./os/firmware/feature/@enabled=no
  --xml ./os/firmware/feature/@name=secure-boot
  ```
  with `virt-installer` do make sure secure boot was disabled.
- **User session will not work, don't bother.** We try to be nice and secure and
  not run things as root, unfortunately this appears to be completely untested
  and you run into all sorts of issues.  Additionally `virtiofs` doesn't
  currently work in user session, and this is important for getting the kernels
  installed.
- **You must (well mostly must) use `virtiofs`.** The directory that GitHub
  actions checks out the kernel source is based on the hash, so it's helpful to
  just live attach that directory to the VM after you've built the kernel so you
  can do the `make modules_install && make install && reboot` step, and then
  detach it after the fact.  You cannot do this with 9pfs, however you could
  still make it work with 9pfs if you did something like the process below.
- **Live attaching `virtiofs` to VMs sometimes doesn't work.**  This I'm not
  necessarily going to lay at libvirts feet, because it works fine with x86 but
  not with ARM64.  I ended up having to attach the build directory to the VM with
  ```
  sudo virsh attach-device $VM_NAME --config <hand crafted xml file>
  ```
  and then shut the VM down and bring it back up, mount the directory in the vm,
  build and install the kernel and reboot.  You can detach it live, but you
  cannot attach it live.
- **Building and installing custom kernels in Fedora sucks.** Generally you want
  to just run `make modules_install && make install && reboot` and boot into the
  new kernel.  Unfortunately Fedora doesn't do this by default, you have to make
  sure to do
  ```
  echo 'UPDATEDEFAULT=yes"' > /etc/sysconfig/kernel
  ```
  to make sure the kernel you tell it to install is actually set to the default
  one on bootup.

That was all the problems Josef, what about actually getting the things to
work?? Well that's pretty straightforward

- **Make some qcow2 images or LV's for disks.**  We're going to need disks to
  use for fstests.  For Btrfs we need 10 to test all the various RAID levels and
  utilize things like the `tree log`.  This is fairly straightforward, get the
  disks how you like, I used an extra disk with LV's on the x86 machine and then
  just qcow2 images for the ARM64 machine, and attach them to the VM.
- **Checkout and build fstests.** Again, simple enough.  Additionally you will
  want to create a local.config with all of the test sections you'll be using on
  the VM's in this VM host.  This way they can each run any config.  I would
  recommend setting
  ```
  RECREATE_TEST_DEV=true
  ```
  for each config section so that `TEST_DEV` matches the configuration you need.
- **Create a no password SSH key to install into the VMs.**  This will allow the
  VM host to run commands and such via scripts.

Most of these tips and tricks are encapsulated in my scripts that you can find
[here](https://github.com/josefbacik/virt-scripts/tree/master/fedora).

# Running fstests from GitHub Actions

The virtualization thing wasn't the onlyh part that tripped me up.  fstests is a
series of shell scripts, and it relies on the following behavior
```
some command | another command > something else
```
If you are like me and haven't really ever thought of the mechanism of how this
works, let me introduce you to `SIGPIPE`.  When we're redirecting output into
another command and that command finishes, it exits, which closes the pipe.  The
command providing the input then gets `SIGPIPE` and exits, and everybody is
happy.

If you are a network based application you don't want to get killed with
`SIGPIPE`, you'd rather get `-EPIPE` when you read from the socket, so you
disable `SIGPIPE`.  Disabling `SIGPIPE` is one of those fun things that gets
inherited by `fork()`, so fstests will get called with `SIGPIPE` disabled, and
all sorts of things will begin to break.

There's no easy way around this, the GitHub runner gets launched (on Linux
anyway) via node.js, which of course disables `SIGPIPE` by default.  However you
can't really turn it back on without attaching a signal handler to the signal,
which is overkill for what we want.

What would be ideal is if the runner itself were to `fork()` and then unmask
`SIGPIPE` before `execve()`'ing the script.  However the runner is written in
C#, and C# doesn't know what `SIGPIPE` is, so it makes this impossible.

Enter `unfuck-signal.c`.  You must wrap fstests around this, as it does the
necessary thing to unmask `SIGPIPE` and allow everything to work properly.  As
indicated above, it's just a wrapper that calls `fork()`, unmasks `SIGPIPE` and
then execs whatever was passed into it.  It's pasted below, I just build it and
copy it into the fstests directory on all of the VMs

```
#include <signal.h>
#include <unistd.h>
#include <stdio.h>
#include <sys/wait.h>

int main(int argc, char **argv)
{
        pid_t child;

        if (argc == 1) {
                fprintf(stderr, "Must specify a command to run");
                return 1;
        }

        child = fork();
        if (child < 0) {
                fprintf(stderr, "Couldn't fork");
                return 1;
        }

        if (child) {
                int wstatus;

                wait(&wstatus);
                if (WIFEXITED(wstatus))
                        return WEXITSTATUS(wstatus);
                if (WIFSIGNALED(wstatus))
                        raise(WTERMSIG(wstatus));
                return 1;
        }

        signal(SIGPIPE, SIG_DFL);
        printf("argv[1] is %s\n", argv[1]);
        return execvp(argv[1], argv+1);
}
```
# ci.yml explanation

I found the GitHub actions YML part pretty straightforward, but I'll break down
the different parts to explain how it all works

```
on:
  push:
    branches: [ "master", "ci" ]
  pull_request:
    branches: [ "master", "ci" ]
```

Easy enough, tells us when we trigger the GitHub action.  Now keep in mind for
our tree our `master` branch is just a mirror of Linus's tree, so the `ci.yml`
file actually only exists in the `ci` branch, so you have to do the PR against
that branch for it to work.

```
jobs:
  build:
    runs-on: [self-hosted, linux, x64, vmhost]
    steps:
    - uses: actions/checkout@v3
    - name: copy system config
      run: copy-config.sh ${{ github.workspace }}
    - name: make olddefconfig
      run: make olddefconfig
    - name: make
      run: make -j32
```

This is `build`, `build-arm` is the same thing but specifies the ARM machine,
this is how you tell it which self hosted runner to use.  The `uses` part is the
built in `git checkout` action, `copy-config.sh` is local to the vm host and
copies the kernel's `.config` into place so it can be built.  This is obviously
different between the x86 and ARM machine.  The rest is just the typical "build
the kernel" steps.

```
  deploy:
    runs-on: [self-hosted, linux, x64, vmhost]
    needs: build
    strategy:
      matrix:
        vm: [xfstests5, xfstests6, xfstests7, xfstests8, xfstests9, xfstests10]
    steps:
    - name: update btrfs-progs
      run: update-btrfs-progs.sh ${{ matrix.vm }}
    - name: update xfstests
      run: update-xfstests.sh ${{ matrix.vm }}
    - name: update kernel
      run: update-kernel.sh ${{ matrix.vm }} ${{ github.workspace }}
```

This runs the deploy on each of the VM's controlled by the x86 vmhost, you can
see in the actual file there's another version of this for the ARM system that
has different VM names.  Each of these scripts are in my virt-scripts, though
the update-kernel.sh is different on ARM to do the shutdown dance as described
above.  We update btrfs-progs in each VM from git, do the same with xfstests,
and then install the kernel onto the VM's.  `update-kernel.sh` waits for the VM
to boot before completing, this is important because you want the VM able to
take the next set of jobs.

```
  test:
    runs-on: [self-hosted, linux, x64, vm]
    needs: deploy
    defaults:
      run:
        working-directory: /root/fstests
    strategy:
      fail-fast: false
      matrix:
        config_name: [btrfs_normal, btrfs_compress, btrfs_holes_spacecache, btrfs_holes_spacecache_compress, btrfs_block_group_tree]
    steps:
    - name: run xfstests
      run: ./unfuck-signal ./check -E EXCLUDE -R xunit -s ${{ matrix.config_name }} -g auto
    - name: generate report
      uses: mikepenz/action-junit-report@v3
      if: success() || failure()
      with:
        report_paths: /root/fstests/results/${{ matrix.config_name }}/result.xml
```
This is the bread and butter.  We have fstests checked out in /root/fstests,
and the local.config already exists with section names that match `config_name`.
We run fstests with the wrapper, we have an EXCLUDE file in our fstests git tree
to exclude any flakey tests so we can get nice clean runs.  We're using the
option that generates an xUnit xml results file.  I'm using the
`action-junit-report` thing because it was the only thing I could find that
would properly parse our silly xUnit format.

All in all relatively straightforward, honestly this was the easiest part of
this whole endeavour.

# Conclusion

There was a lot more effort in setting all this up than I would have liked, but
in the end we have a nice system in place for running fstests on demand.  The
next steps are to integrate this with our performance testing to have that
launched on demand as well instead of the nightly runs.  Eventually we would
like to move the hardware into something like AWS or GCE and out of my basement,
but given the two step nature of the run that may be a while before I feel like
tackling that.
