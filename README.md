# Problem statement

It's still very easy on a standard Debian to start some huge
computation (Ethereum syncing, Firefox build, Chrome build, etc.) and
make your system so slow that you can't even kill the responsible
processes.  During the slowdown, you can't even SSH into your machine.

Usually after 10-20 minutes the computation either finishes (rare) or
you just reboot your machine.

No more!

# Useful commands

- `smem`
- `lsns`
- `systemctl show user-1000.slice | grep -i current`

You find in this directory the `memory_hog.c` file, which is very
useful to test cgroup memory settings that you have set up.  Careful
if you have swapspace enabled, it can quickly freeze your machine.

Stolen from here: https://github.com/bmwcarit/cgroup-tests

# Asking your systemd to protect the machine from your user

```shell
systemctl set-property user-1000.slice MemoryAccounting=true
systemctl set-property user-1000.slice MemoryMax=15G    # I have 24G on my machine, this makes ssh usable as root
systemctl set-property user-1000.slice CPUAccounting=true
systemctl set-property user-1000.slice CPUQuota=600%    # I have 8 entries in /proc/cpuinfo
systemctl set-property user-1000.slice TasksAccounting=true
systemctl set-property user-1000.slice TasksMax=2000    # Maximum of 2000 processes
```

Everything is self explanatory, except for memory, please read my notes about swap!

Commands that will be useful once we have unified cgroupv2:
```shell
systemctl set-property user-1000.slice MemorySwapMax=0
systemctl set-property user-1000.slice MemoryHigh=12G
```

# Important note about swap

The above `MemoryMax` setting only ensures that your user will not use
more than 15G RAM.  It says nothing about swap!  So basically with
this setting if Chrome is using 16G RAM, your machine will be already
freezing, while previously we needed 25G RAM usage for that.

Solutions: either turn off swap, or if you want to keep it, you need
to tell your `user-1000.slice` cgroup to use 0 of it.

For this you need swapaccounting, which is by default disabled on
Debian.

You can check if you have it with:

```shell
# if this doesn't exist, then you need to adopt these instructions for your distribution
ls -l /sys/fs/cgroup/memory/user.slice/user-1000.slice/memory.limit_in_bytes
# if this does exist, then you have swapaccounting
ls -l /sys/fs/cgroup/memory/user.slice/user-1000.slice/memory.memsw.limit_in_bytes
```

If you don't have swapaccounting, you can enable it by appending
`swapaccount=1` to your kernel boot line (and reboot).  You can always
check your boot line in `/proc/cmdline`.

After you have swapaccounting, you can do this as root:
```shell
cd /sys/fs/cgroup/memory/user.slice/user-1000.slice
cat memory.limit_in_bytes >memory.memsw.limit_in_bytes
```

After this your slice will not use swap at all.  Execute this after
your xsession login via sudo or something.

Note: This is what `MemorySwapMax=0` setting should achieve without
scripts, but unfortunately that is not implemented in systemd for the
cgroupv1 system, that we have currently.  Once we have cgroupv2, we
will be able to use systemd and not this shell script.

(Note to self: on errge's systems this script is called
`noswapforme`.)

# How to debug systemd

So you are reading the systemd manual and want to know which settings
changes what in `/sys/fs/cgroup`.

The easy way to achieve that is to `strace -e file,write -p 1` in one
terminal, then change any setting with `systemctl set-property`.

You will see what systemd is doing and then you can also conclude
which settings have any effect on your system and which don't.

# Cgroupv1 vs Cgroupv2

Since the cgroup subsystem is mainly developed and supported by
Google, the usual Googlism applies:
  - there is v1 which is already very obsolete,
  - there is v2 which is not ready yet.

The v2 will have one unified hierarchy tree of
CPU+memory+IO+everything else, so it will be a lot easier to debug and
change manually (outside of systemd), but the v2 is not compatible
with Docker yet.  So if you are using Docker, you are out of luck.

If you are not using Docker, you can already enable v2 with the boot
parameter of `systemd.unified_cgroup_hierarchy=1` and then you can
even set the swap usage via
systemd.

Docker support status: https://github.com/moby/moby/issues/25868

The cgroupv2 system also has some concept of MemoryHigh, which is a
kind of softer limit than MemoryMax.

On the other hand, I couldn't get CPU limiting work correctly with
systemd and cgroupv2.

All this happened on or around 2018-12-20, so feel free to try again
if you are in the future!

# Building Firefox in an even more restrictive cgroup

As root you can do this:

```shell
mkdir /sys/fs/cgroup/memory/user.slice/user-1000.slice/firefoxbuild
echo $[8*1024*1024*1024] > /sys/fs/cgroup/memory/user.slice/user-1000.slice/firefoxbuild/memory.limit_in_bytes
echo $[8*1024*1024*1024] > /sys/fs/cgroup/memory/user.slice/user-1000.slice/firefoxbuild/memory.memsw.limit_in_bytes
echo <pidof my build shell> > /sys/fs/cgroup/memory/user.slice/user-1000.slice/firefoxbuild/cgroup.procs
```

And then make sure you execute the `./mach build` command in the shell
that you have put into the cgroup.

This way your build will be limited to 8G of ram, so it will not
starve your desktop and you can continue watching some
SoloRenektonOnly during your build.
